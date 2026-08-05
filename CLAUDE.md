# Endeavor — Data & Engineering Standard (canonical)

> This is the **normative, machine-readable** standard for every Endeavor data/engineering repo.
> Humans: the visual guide is [`README.md`](./README.md). Agents: follow this file.
> When a downstream project imports this, these rules apply **in addition to** that project's
> own domain rules. Conflicts: project domain rules win for domain logic; this standard wins
> for cross-cutting concerns (security, data governance, multi-market, AI, git flow).

**Inspired by** [DashOne Engineering Standard](https://github.com/JuliusCesarius/dashone-guidelines)
— §1 Engineering and §4 PM/Process are directly adopted; §3 Architecture is adapted for
Snowflake + multi-market; §6 Data Governance and §7 AI/Knowledge Base are Endeavor-original.

Conventions:
- **MUST** = required, non-negotiable. **SHOULD** = strong default; deviating needs a written note.
- `[stack-agnostic]` = applies regardless of language/framework.

---

## 1. Engineering

- **MUST** run the test suite after every change; **never commit if tests fail**.
- **MUST NOT** silence warnings (deprecation/security/lint). No `# noqa`, `eslint-disable`,
  `ts-ignore`, `filterwarnings=ignore` without explicit human approval.
  - Safe, non-breaking fix → apply immediately.
  - Potentially breaking fix → explain the risk and ask first.
- **MUST** audit dependencies after install (`npm audit`, `uv`/`pip` advisories). Fix CVEs
  before moving on.
- **MUST** check how a symbol is already imported before adding `from X import Y` /
  `import {Y}` — package name ≠ import name (`python-ulid`→`ulid`, `PyJWT`→`jwt`).
- **SHOULD** write code that reads like the surrounding code: match naming, comment density,
  and idioms. Prefer editing existing patterns over introducing new ones.
- **MUST** keep user-facing strings in **English** (global surface) or the relevant market
  language when the surface is market-specific; code, identifiers, and internal docs in
  **US English**.
- **Learning loop:** when a runtime error could have been prevented by a rule, add that rule
  to the project's guardrails (and propose it here) so it never recurs.
- **SHOULD** enforce a CI gate: lint + typecheck + tests on every PR to `staging`/`main`;
  failing CI blocks merge.

### Language-specific guardrails
- **Python + pydantic-ai / pydantic:** with `from __future__ import annotations`, any type
  used in `Annotated[...]` inside a tool **MUST** be imported at **module level**
  (`get_type_hints()` reads the module globals).
- **Python + Snowflake Snowpark:** session objects are not thread-safe — **MUST** create one
  session per thread/worker; never share across async tasks.
- **dbt:** every model **MUST** have a `description:` in schema.yml. Raw/source models
  **MUST NOT** be referenced directly in downstream marts — always stage first.
- **TypeScript:** `strict: true`. No `any` without a `// reason:` note.

---

## 2. Design

Endeavor's brand identity governs all customer-facing and internal surfaces.
Consult the Endeavor Brand Portal for official tokens (colors, fonts, logo usage).

### Universal rules (apply regardless of brand)

#### Accessibility (a11y) — MUST
- Meet **WCAG AA** contrast (4.5:1 normal text, 3:1 large/UI elements).
- Be keyboard-navigable with a **visible focus state** on every interactive element.
- Provide `alt` on images, `aria-label` on icon-only controls, `<label>` on inputs.
- Respect `prefers-reduced-motion` — disable non-essential animations.

#### Mechanics
- **Touch targets ≥ 44×44px** on all interactive elements.
- **MUST** reuse base components from the project's design system over bespoke ones.
- Shadows: soft, low opacity. No heavy drop shadows.

---

## 3. Architecture — cross-cutting contracts `[stack-agnostic]`

These hold regardless of language/framework. They are what makes Endeavor's systems
**feel the same** and interoperate across markets and products.

### Multi-market tenancy
- Every data record **MUST** carry a `market_id` (ISO country code or Endeavor market slug,
  e.g., `mx`, `ar`, `us`) **and** a `network_id` when the surface spans sub-networks.
- Data access **MUST** be scoped to the caller's market by default. Cross-market aggregations
  are only allowed for Endeavor Global roles.
- Snowflake: implement market scoping via **row-access policies** on `market_id`.
  No table without a policy. Migrations **SHOULD** be checked by CI for missing policies.

### Observability & audit
- **`trace_id`:** mint a ULID `trc_…` at ingress; propagate end-to-end across services,
  pipelines, and AI calls. It is the join key across logs, traces, and the audit log.
- **`actor`:** stamp who/what initiated at ingress
  `{type: user|schedule|agent|staff, id: "...", market_id: "..."}` and preserve it
  across the whole chain (API → pipeline → Snowflake → AI response).
- **Audit log:** append-only event log in Snowflake, queryable by `actor` / `trace_id` /
  `event_type` / date. **Never delete** audit rows. Partition by month for cost control.

### Reliability
- **Idempotency:** dedupe durable/background work by `job_id` (ULID), recorded in the same
  transaction as the side effect. Snowflake Tasks that call external services **MUST** be
  idempotent.
- **Versioned envelopes:** events/messages carry `schema_version`. Bump on breaking changes.
- **Async-first:** a single synchronous request **MUST NOT** block on background work.
  Reply immediately, run async, deliver the result on the same `trace_id`.

### Secrets & config
- Centralized in the team's approved secrets manager (1Password Secrets Automation, AWS
  Secrets Manager, or equivalent). **Never hardcode credentials or keys.**
- Adding a secret: secrets manager config + `.env.example` (placeholder only).
  Code reads env and **fails fast** with a clear message if missing.
- Snowflake credentials **MUST** use key-pair authentication or OAuth; no password auth
  in production services.

### Migrations & schema
- SQL migrations applied **automatically** by a runner with a tracking table.
  **Never** ask a human to run them manually.
- Snowflake schema changes: use **Schemachange** or **dbt** migrations. Document every
  DDL change with `COMMENT ON TABLE/COLUMN`.

### Orchestration tiers
| Workload | Tool |
|---|---|
| Durable multi-step pipelines (ingestion, embeddings, batch AI) | **Prefect** or **Snowflake Tasks + DAGs** |
| dbt transformations | **dbt Cloud** scheduled jobs or Prefect operator |
| Light API-triggered jobs | Snowflake Tasks or a `jobs` table + worker |
| Ad-hoc SQL | Snowflake Worksheets (non-production only) |

---

## 4. PM / Process

- **Specs-first:** **MUST NOT** start a feature or use case without its spec in `specs/`.
- **Per-use-case closing ritual** (after tests pass, before moving on):
  1. End-user test plan: What to test / Steps / Expected result / Quick CLI or SQL check.
  2. Run the verifiable checks and show output.
  3. Self-score 1–10 the likelihood it works end-to-end. If **< 8**, fix before presenting.
     If **≥ 8** with environment-dependent uncertainty, say so and include the number.
  4. Ask: *"Are you going to test this, or should I continue with UC{N+1}? (confidence: N/10)"*
     and wait for the answer.
- **Git flow:** `feat/<slug>` | `fix/<slug>` from `main` → PR to **`staging`** → human OK →
  PR to **`main`**. **Never** push directly to `main`/`staging`; **never** skip staging.
  After pushing, open a PR (ready for review, not draft).
- **Versioning:** SemVer in `pyproject.toml` / `package.json`; bump in the PR that introduces
  the change. **Conventional Commits** (`feat:`, `fix:`, `chore:`, `docs:` …). PRs link the
  work (`Closes UC-N` / issue).
- **Release notes:** brief, user-facing summary when promoting `staging` → `main`.
- **Honest reporting:** if tests fail, say so with output; if a step was skipped, say so;
  when verified, state it plainly.
- **Maintenance:** keep this standard and project `CLAUDE.md` current when persistent rules change.

---

## 5. Stack

See [`docs/stack.md`](./docs/stack.md) for the full matrix and rationale.

| Layer | Default | Alternative | Avoid |
|---|---|---|---|
| Data warehouse | **Snowflake** | — | Moving data out of Snowflake for transforms that can run in-warehouse |
| Transformations | **dbt Core/Cloud** | Snowpark Python | Ad-hoc SQL with no version control |
| AI / embeddings | **Snowflake Cortex** (in-warehouse) | OpenAI API / Anthropic API (external) | Embedding outside Snowflake then re-importing |
| Vector search | **Snowflake vector similarity** (`VECTOR_COSINE_SIMILARITY`) | pgvector (separate Postgres) | Custom ANN implementations |
| Orchestration | **Prefect** (durable) | Snowflake Tasks + DAGs (light) | Ad-hoc cron scripts |
| Backend API | **Python 3.12 + FastAPI** | Node/TS | Flask/Django for new services |
| Frontend | **Next.js (App Router)** | Vite + React (light UIs) | CRA |
| Auth | **Clerk** or Endeavor SSO (OIDC) | — | Rolling your own auth |
| Secrets | Team secrets manager (1Password / AWS) | — | `.env` committed to git |
| Observability | **LangSmith** (LLM traces) + **OpenTelemetry** | — | `print()`-only logging |
| CI/CD | **GitHub Actions** | — | Manual deploys |
| Deploy | **Fly.io** (services), **Vercel** (Next.js) | AWS ECS | Snowflake VMs as app servers |

---

## 6. Data Governance

Endeavor's core data assets include mentorship recordings, entrepreneur profiles, and mentor
intellectual property. These require explicit governance rules beyond standard engineering.

### Consent — MUST
- **Recording consent:** both the mentor and entrepreneur **MUST** have signed a consent form
  before a session can be recorded or ingested into the knowledge base.
  - Store consent record: `consent_id`, `actor_id`, `market_id`, `session_id`, `signed_at`,
    `scope` (`recording|transcript|ai_indexing`), `version`.
  - Consent is **revocable**. Revocation **MUST** trigger deletion of that actor's
    content from the knowledge base within 30 days.
- **Scope**: consent for recording ≠ consent for AI indexing. Track separately.

### PII handling
- **PII actors:** entrepreneurs (`entrepreneur_id`), mentors (`mentor_id`), Endeavor staff.
- **MUST NOT** expose raw PII (full name, email, phone) in AI responses or search results
  unless the querying user has explicit access to that market's data.
- Aggregate/research use cases: **MUST** anonymize or pseudonymize before processing.
- **MUST** document which fields are PII in `schema.yml` (`meta: pii: true`).

### Data zones (Snowflake)
```
RAW      →  STAGED      →  CURATED     →  AI_READY
(landing)   (cleaned,       (business       (chunked,
            typed, PII      logic,           embedded,
            flagged)        validated)       consent-checked)
```
- Data flows **one way**: never write back to RAW from downstream zones.
- AI_READY zone only contains sessions where **all consent scopes are active**.

### Retention
- Recordings: follow Endeavor legal/compliance team's policy per market.
- Transcripts: same retention as recordings.
- Embeddings: deleted when source transcript is deleted or consent is revoked.
- Audit log: **permanent** (never deleted, see §3).

### Access control
- Market staff: read/write own `market_id` only.
- Endeavor Global (analytics): read-only cross-market aggregates, no raw PII.
- AI pipeline service account: read AI_READY zone only, no RAW access.
- Document each role's permissions in `docs/access-matrix.md`.

---

## 7. AI & Knowledge Base

The mentorship knowledge base is Endeavor's core data product. These contracts govern
how it is built, queried, and maintained.

### Ingestion pipeline contract
Every session entering the knowledge base **MUST** pass through this pipeline in order:

```
1. Consent check      → abort if any party's ai_indexing consent is missing
2. Transcription      → speaker-diarized (mentor vs. entrepreneur), stored in STAGED
3. Topic tagging      → auto-tag with domain (finance, ops, growth, …) + market_id
4. Chunking           → max 512 tokens, overlap 64, preserve speaker turns
5. Embedding          → Snowflake Cortex (preferred) or specified external model
6. Index              → Snowflake vector column in AI_READY.mentorship_chunks
7. Audit event        → log ingestion with trace_id, session_id, model version, chunk_count
```

Every chunk record **MUST** carry:

| Field | Description |
|---|---|
| `chunk_id` | ULID |
| `session_id` | Source session |
| `mentor_id` | Mentor in that session |
| `market_id` | Market where session occurred |
| `topic_tags` | Array of domain tags |
| `speaker` | `mentor` or `entrepreneur` |
| `embed_model` | Model name + version used |
| `embed_model_version` | Semantic version |
| `chunk_text` | The raw text of the chunk |
| `embedding` | VECTOR column |
| `ingested_at` | Timestamp |

### Retrieval contract
AI-powered responses **MUST**:
- Include `source_citations`: `[{session_id, mentor_id, date, topic_tags, similarity_score}]`
  for every retrieved chunk used.
- Apply a **minimum similarity threshold** (default `0.75`). Below threshold: respond with
  "No closely matching mentorship sessions found; here is general guidance:" — never silently
  fabricate a source.
- Filter retrieved chunks to the **requesting user's market** unless they have Global role.
- Log every query: `trace_id`, `query_text` (hashed), `retrieved_chunk_ids`, `model_used`,
  `latency_ms`, `actor`.

### Quality & feedback loop
- FDEs **SHOULD** review a sample of AI responses weekly and mark them
  `helpful | partially_helpful | not_helpful` in the feedback table.
- Feedback is the signal for chunk pruning and re-ranking weight tuning.
- If a response is marked `not_helpful` 3+ times, flag the source chunks for human review.

### Model versioning
- **MUST** record `embed_model_version` on every chunk at ingestion time.
- When upgrading the embedding model: re-embed **all** chunks in a transaction, do not
  mix model versions in the same search index.
- Keep the previous model's embeddings in a separate column until the new index is validated.

### Guardrails
- AI responses **MUST NOT** attribute advice to a specific mentor by name in the public
  response unless the mentor has granted attribution consent (separate consent scope).
- **MUST NOT** generate or infer PII about entrepreneurs from session context.
- Implement output filtering for: phone numbers, email addresses, financial figures linked
  to named individuals.

---

## 8. Adoption

How a project repo consumes this standard.

```bash
# Submodule approach (recommended)
git submodule add https://github.com/<endeavor-org>/endeavor-guidelines .endeavor
echo "@.endeavor/CLAUDE.md" >> CLAUDE.md
git submodule update --remote .endeavor
```

Template approach: copy `templates/CLAUDE.project.md`, fill in project specifics.

**Precedence:** a project's `CLAUDE.md` may **add** stricter rules and domain rules; it
**MUST NOT** silently relax a `MUST` here. To deviate, document the exception explicitly
with a rationale and an owner.
