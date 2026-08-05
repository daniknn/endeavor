# Endeavor Stack Reference

Full rationale for every tool choice in the standard.

## Data layer

### Snowflake (primary data warehouse)
**Why:** Endeavor already runs on Snowflake. It provides elastic compute separation,
native vector search (avoiding a separate vector DB), row-access policies for multi-market
isolation, and Snowflake Cortex for in-warehouse LLM/embedding inference.

**Naming conventions:**
- Databases: `ENDEAVOR_<ENV>` (e.g., `ENDEAVOR_PROD`, `ENDEAVOR_STAGING`)
- Schemas map to data zones: `RAW`, `STAGED`, `CURATED`, `AI_READY`
- Tables: `SCREAMING_SNAKE_CASE`
- Views: prefix `V_` (e.g., `V_MENTOR_SESSIONS`)
- Stored procs / UDFs: `SCREAMING_SNAKE_CASE` with doc comment

**Cost guardrails:**
- Auto-suspend warehouses after 5 minutes of inactivity.
- Use separate warehouses for ingestion, dbt transforms, and AI inference.
- Cluster keys on large tables: `(market_id, created_at::DATE)`.
- Query result cache: leverage it by avoiding `NOW()`/`CURRENT_TIMESTAMP()` in WHERE
  clauses on non-time-sensitive queries.

### dbt (transformations)
**Why:** version-controlled SQL, lineage documentation, built-in testing, schema.yml for
PII tagging, native Snowflake adapter.

**Layer conventions:**
| Layer | dbt folder | Snowflake schema | Purpose |
|---|---|---|---|
| Sources | `models/staging/sources/` | `RAW` | Raw landing, no transforms |
| Staging | `models/staging/` | `STAGED` | Type casting, renaming, PII flag |
| Intermediate | `models/intermediate/` | `CURATED` | Business logic joins |
| Marts | `models/marts/` | `CURATED` | Final wide tables / aggregates |
| AI Ready | `models/ai_ready/` | `AI_READY` | Consent-filtered, chunk-ready |

**Required tests per model:** `not_null` + `unique` on primary key; `accepted_values`
on enum columns; `relationships` on foreign keys.

### Snowflake Cortex (AI inference)
**Why:** runs inside Snowflake — data never leaves the warehouse for embedding or LLM
inference. Simplifies governance (no external API calls for sensitive content).

**Functions used:**
- `SNOWFLAKE.CORTEX.EMBED_TEXT_768()` — embedding (768-dim)
- `SNOWFLAKE.CORTEX.COMPLETE()` — LLM completion
- `VECTOR_COSINE_SIMILARITY()` — retrieval scoring

**When to use external APIs (OpenAI / Anthropic):**
- When Cortex's available models are insufficient for a specific task.
- Requires: explicit approval, data anonymized before sending, logged in audit.

## Orchestration

### Prefect (durable pipelines)
Use for: multi-step ingestion flows, embedding pipelines, scheduled batch jobs with
retry logic and observability.

### Snowflake Tasks + DAGs (lightweight)
Use for: simple scheduled SQL, maintenance jobs, dbt run triggers that don't need
external retry logic.

### dbt Cloud (dbt orchestration)
Use for: scheduled dbt runs, CI for dbt models, lineage UI.

## Backend

### Python 3.12 + FastAPI
Standard for API services. Use `snowflake-snowpark-python` for Snowflake connections.
Session management: one Snowpark session per request context (not shared globally).

### Node/TypeScript
Acceptable for lightweight API services or edge functions where Python is overkill.

## Frontend

### Next.js (App Router)
Default for internal tools and portals. Host on Vercel.

### Vite + React
For simpler single-page tools that don't need SSR.

## Auth

### Clerk
Default for new web applications.

### Endeavor SSO (OIDC/SAML)
Where Endeavor's corporate identity provider is available, use it. Clerk supports
SAML Enterprise SSO — configure it rather than managing users separately.

## Infrastructure

### Fly.io
Python/Node API services. Dockerfile-based. Auto-scale to zero for non-critical services.

### Vercel
Next.js frontends. Preview deployments on every PR.

### GitHub Actions
CI/CD. Required gates: lint, typecheck, tests, dbt compile (if applicable).

## Observability

### LangSmith
Trace all LLM calls (prompts, completions, latency, cost). Required for AI features.
Set `LANGCHAIN_TRACING_V2=true` and `LANGCHAIN_PROJECT=endeavor-<service>`.

### OpenTelemetry
Instrument all FastAPI services. Export to your observability backend (Datadog, Grafana
Cloud, or similar).

### Snowflake Query History
Audit all queries via `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`. Alert on:
- Queries scanning > 1 TB.
- Service account queries outside expected patterns.
