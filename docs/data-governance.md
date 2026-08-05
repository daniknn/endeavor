# Data Governance — Detail

This document expands on §6 of `CLAUDE.md`. Refer to `CLAUDE.md` for the normative rules;
this doc provides implementation guidance.

## Actors and their data

| Actor | Identifier | Sensitive fields |
|---|---|---|
| Mentor | `mentor_id` (ULID) | name, email, company, phone, LinkedIn |
| Entrepreneur | `entrepreneur_id` (ULID) | name, email, company, revenue, headcount |
| Endeavor Staff | `staff_id` (ULID) | name, email, market |
| AI pipeline | service account | no PII stored |

PII fields **MUST** be tagged in dbt `schema.yml`:
```yaml
columns:
  - name: mentor_email
    description: "Mentor's contact email"
    meta:
      pii: true
      pii_category: contact
```

## Consent model

```
CONSENT
├── session_id       FK → SESSIONS
├── actor_id         mentor_id or entrepreneur_id
├── actor_type       mentor | entrepreneur
├── market_id
├── scope            recording | transcript | ai_indexing   ← track separately!
├── status           active | revoked
├── signed_at        timestamptz
├── revoked_at       timestamptz (null if active)
├── form_version     version of the consent form presented
└── consent_id       PK (ULID)
```

### Consent check query (Snowflake)
```sql
-- Returns true only if ALL required scopes are active for ALL parties in a session
SELECT
    s.session_id,
    BOOLOR_AGG(c.status = 'active' AND c.scope = 'ai_indexing') AS all_consented
FROM STAGED.SESSIONS s
JOIN STAGED.CONSENT c ON c.session_id = s.session_id
WHERE s.session_id = :session_id
GROUP BY s.session_id
HAVING COUNT(DISTINCT c.actor_id) >= 2  -- both mentor and entrepreneur
   AND all_consented = TRUE;
```

## Data zones

### RAW
- Landing zone. Exact copy of source data (no transforms, no deletions).
- Access: ingestion service account only (write), data engineers (read).
- PII present and unmasked.

### STAGED
- Typed, renamed, PII flagged (but not masked).
- All columns have `description:` in schema.yml.
- Tests: `not_null`, `unique` on PKs.
- Access: data engineers + dbt service account.

### CURATED
- Business logic applied. Joins, aggregations, derived metrics.
- No direct PII in marts (use `actor_id` as foreign key, not embedded name/email).
- Access: analytics roles, FDE dashboards.

### AI_READY
- Consent-filtered. Only sessions where **all** parties have active `ai_indexing` consent.
- Contains embedded chunks (`VECTOR` column).
- **No raw PII in any column.**
- Access: AI pipeline service account (read only), data engineers (read for debugging).

## Revocation handling

When a consent is revoked (`status = 'revoked'`):

1. Mark `CONSENT.status = 'revoked'`, set `revoked_at`.
2. Trigger `revocation_job` (Prefect or Snowflake Task):
   - Delete all `AI_READY.MENTORSHIP_CHUNKS` where `session_id` matches any session
     involving the revoked `actor_id` AND that actor's scope is revoked.
   - Log deletion in audit: `{event_type: consent_revocation, actor_id, session_ids_affected, deleted_at}`.
3. Complete within **30 days** of revocation (target: 24 hours).
4. Send confirmation to the actor's registered email.

## Access matrix

| Role | RAW | STAGED | CURATED | AI_READY |
|---|---|---|---|---|
| Data engineer | R | R/W | R | R (debug) |
| dbt service account | — | R/W | R/W | R/W |
| FDE (own market) | — | — | R (own market) | — |
| Endeavor Global Analytics | — | — | R (aggregate, no PII) | — |
| AI pipeline service account | — | — | — | R |
| Ingestion service account | W | — | — | — |

Implemented via Snowflake roles + row-access policies on `market_id`.

Full role definitions: see `docs/access-matrix.md` (to be created per deployment).

## Data retention schedule

| Data type | Retention | Legal basis |
|---|---|---|
| Recordings (raw audio/video) | Per market legal policy | Consent + legitimate interest |
| Transcripts (STAGED/CURATED) | Same as recording | Derived from recording |
| Embeddings (AI_READY) | Until source transcript deleted or consent revoked | Derived data |
| Consent records | Permanent (legal evidence) | Legal obligation |
| Audit log | Permanent | Compliance |
| Query logs (LangSmith) | 90 days | Operational |

Retention enforcement: Snowflake data retention + Lifecycle tasks scheduled to run monthly.
