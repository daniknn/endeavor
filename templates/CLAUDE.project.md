# <Project Name> — Engineering Notes

@.endeavor/CLAUDE.md

> Fill in the sections below. Delete any section that doesn't apply.
> This file is read by Claude Code agents working in this repo.
> Sections here are ADDITIVE to the base standard; they do not relax any MUST.

---

## Project overview

<!-- One paragraph: what this service/pipeline does, who uses it, which market(s) it serves. -->

## Domain rules

<!-- Rules specific to this project that go beyond the base standard. -->
<!-- Example: "This service is the consent authority — all consent writes must go through it." -->

## Key files and entry points

<!-- List the most important files an agent should know about. -->
<!-- Example: -->
<!-- - `app/main.py` — FastAPI entrypoint -->
<!-- - `models/marts/mentorship.yml` — dbt schema for mentorship marts -->
<!-- - `flows/ingest_session.py` — Prefect flow for session ingestion -->

## Environment variables

<!-- List required env vars (no values). Keep secrets in secrets manager. -->
<!-- Example: -->
<!-- - `SNOWFLAKE_ACCOUNT` -->
<!-- - `SNOWFLAKE_USER` -->
<!-- - `SNOWFLAKE_PRIVATE_KEY_PATH` -->
<!-- - `SNOWFLAKE_DATABASE` — e.g. `ENDEAVOR_PROD` -->
<!-- - `LANGCHAIN_API_KEY` -->

## Market scope

<!-- Which markets does this service serve? -->
<!-- - [ ] Mexico (mx) -->
<!-- - [ ] Argentina (ar) -->
<!-- - [ ] Colombia (co) -->
<!-- - [ ] Global (all markets — requires Global role) -->

## AI features

<!-- If this project uses the knowledge base or AI inference: -->
<!-- - Embedding model used: -->
<!-- - Similarity threshold override (if different from 0.75): -->
<!-- - Attribution consent required: yes | no -->

## Deviations from base standard

<!-- Document any explicit deviations from CLAUDE.md MUSTs. Each must have a rationale and owner. -->
<!-- Example: -->
<!-- ### Deviation: no staging environment -->
<!-- Rationale: internal tooling used by 2 staff; risk of staging env is not justified. -->
<!-- Owner: @<github-handle> -->
<!-- Approved: <date> -->
