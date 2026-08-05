# Pre-PR Compliance Checklist

Run this before opening any PR that touches data models, AI pipeline, or governance code.
Check each item; document any "N/A" with a brief reason.

## Engineering (§1)
- [ ] Tests pass locally (`pytest` / `npm test` / `dbt test`)
- [ ] No silenced warnings (no new `# noqa`, `eslint-disable`, `ts-ignore`)
- [ ] `npm audit` / `pip audit` clean — no unresolved CVEs introduced
- [ ] No hardcoded credentials, keys, or secrets

## Architecture (§3)
- [ ] New Snowflake tables have `market_id` column
- [ ] New Snowflake tables have a row-access policy on `market_id`
- [ ] New API endpoints propagate `trace_id` and stamp `actor`
- [ ] Audit events emitted for new state-changing operations
- [ ] Background work is async; no request blocks on it

## Data Governance (§6)
- [ ] New PII fields tagged with `meta: pii: true` in `schema.yml`
- [ ] Any new data landing in AI_READY zone passes consent gate
- [ ] No raw PII in AI_READY zone columns
- [ ] Retention policy documented for any new data type

## AI / Knowledge Base (§7) — skip if no AI changes
- [ ] Every chunk record includes: `session_id`, `mentor_id`, `market_id`, `embed_model`, `embed_model_version`
- [ ] Retrieval responses include `source_citations`
- [ ] Similarity threshold ≥ 0.75 enforced (or deviation documented)
- [ ] Fallback response used when no chunks pass threshold
- [ ] `mentor_name` in citations only if attribution consent is active
- [ ] AI calls logged to LangSmith

## Process (§4)
- [ ] Spec exists in `specs/` for this feature
- [ ] Branch named `feat/<slug>` or `fix/<slug>`
- [ ] PR targets `staging`, not `main`
- [ ] Conventional commit messages (`feat:`, `fix:`, `chore:`, …)
- [ ] Version bumped in `pyproject.toml` / `package.json` if applicable

---

**Reviewer:** confirm each checked item before approving.
