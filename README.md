# Endeavor — Data & Engineering Standard

Engineering, data governance, and AI guidelines for Endeavor's technical teams.

## What this is

This repository is the **canonical source of truth** for how Endeavor builds, governs,
and evolves its data products — starting with the mentorship knowledge base.

It is modeled after the [DashOne Engineering Standard](https://github.com/JuliusCesarius/dashone-guidelines),
adapted for Endeavor's specific context:

| Concern | Endeavor reality |
|---|---|
| Primary data store | Snowflake |
| Tenancy model | Multi-market (`market_id` = country/region) |
| Core AI use case | RAG over mentorship sessions |
| Key actors | Entrepreneurs, mentors, FDEs, Endeavor staff |
| Critical constraint | Consent for recording AND AI indexing (tracked separately) |

## Quick start for a new project

```bash
# 1. Add this repo as a submodule
git submodule add https://github.com/daniknn/endeavor .endeavor

# 2. Import into your project's CLAUDE.md (for Claude Code / AI agents)
echo "@.endeavor/CLAUDE.md" >> CLAUDE.md

# 3. Scaffold from the project template
cp .endeavor/templates/CLAUDE.project.md CLAUDE.project.md
# Fill in the <Project> sections

# 4. Keep it current
git submodule update --remote .endeavor
```

## Structure

```
endeavor/
├── CLAUDE.md                    # Normative rules (machine + human readable)
├── README.md                    # This file
├── docs/
│   ├── stack.md                 # Full stack matrix and rationale
│   ├── data-governance.md       # PII, consent, data zones detail
│   ├── ai-rag.md                # RAG pipeline and retrieval contracts
│   └── access-matrix.md        # Role → data access mapping (to be created per deployment)
└── templates/
    ├── CLAUDE.project.md        # Per-project CLAUDE.md template
    └── compliance-checklist.md  # Pre-PR checklist
```

## Key sections in CLAUDE.md

| Section | Summary |
|---|---|
| §1 Engineering | Testing, warnings, dependency audits, language guardrails |
| §2 Design | Endeavor brand + universal a11y rules |
| §3 Architecture | Multi-market tenancy, Snowflake RLS, tracing, audit, orchestration |
| §4 PM / Process | Specs-first, git flow, UC closing ritual |
| §5 Stack | Approved tools (Snowflake, dbt, Cortex, Prefect, FastAPI) |
| §6 Data Governance | Consent, PII, data zones (RAW→STAGED→CURATED→AI_READY), retention |
| §7 AI & Knowledge Base | Ingestion pipeline, retrieval contract, source attribution, guardrails |
| §8 Adoption | How to consume this standard in a project |

## The mentorship RAG use case

The flagship use case driving these standards:

```
Mentorship session
        │
        ▼
[Consent check] ──✗──▶ stop; do not index
        │ ✓
        ▼
[Transcription + diarization]   mentor vs. entrepreneur turns
        │
        ▼
[Topic tagging]   domain: finance | growth | ops | legal | hr | …
        │
        ▼
[Chunking + embedding]   Snowflake Cortex (in-warehouse)
        │
        ▼
[AI_READY.mentorship_chunks]   with market_id, mentor_id, session_id, topic_tags
        │
        ▼  (query time)
[Similarity search]   cosine similarity ≥ 0.75, filtered to user's market
        │
        ▼
[Response + citations]   answer + [{session_id, mentor_id, date, topic, score}]
```

A response without citations is a policy violation.
A response below the similarity threshold must declare it rather than fabricate.

## Maintenance

- Open a PR against `main` when a rule needs updating.
- Tag the FDE team for review on governance changes (§6, §7).
- Run `templates/compliance-checklist.md` before merging any PR that touches `CLAUDE.md`.
