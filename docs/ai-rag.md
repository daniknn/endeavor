# AI & Knowledge Base — Implementation Guide

Expands on §7 of `CLAUDE.md`. Normative rules are in `CLAUDE.md`; this doc provides
implementation patterns and example SQL/Python.

## Ingestion pipeline

### Pipeline inputs
- `session_id`: ULID of the mentorship session
- `recording_url`: secure URL to audio/video (S3 or equivalent)
- `trace_id`: propagated from the triggering event

### Step 1 — Consent gate (Snowflake)
```sql
-- In dbt model or Snowpark Python before any processing
CREATE OR REPLACE FUNCTION AI_READY.CHECK_SESSION_CONSENT(SESSION_ID VARCHAR)
RETURNS BOOLEAN
AS $$
  SELECT BOOLOR_AGG(status = 'active')
  FROM STAGED.CONSENT
  WHERE session_id = SESSION_ID
    AND scope = 'ai_indexing'
  HAVING COUNT(DISTINCT actor_id) >= 2
$$;
```

If `CHECK_SESSION_CONSENT` returns FALSE or NULL: log audit event `consent_blocked`,
stop pipeline, do not write to STAGED or AI_READY.

### Step 2 — Transcription
Use a speaker-diarized transcription service (e.g., AssemblyAI, Whisper + pyannote).
Output format per turn:
```json
{
  "turn_id": "trn_01j...",
  "session_id": "ses_01j...",
  "speaker": "mentor",
  "start_ms": 0,
  "end_ms": 45000,
  "text": "Lo que yo haría en tu caso es..."
}
```
Store raw turns in `STAGED.SESSION_TURNS`.

### Step 3 — Topic tagging
Use Snowflake Cortex COMPLETE with a classifier prompt, or a fine-tuned classifier:
```sql
SELECT
    turn_id,
    SNOWFLAKE.CORTEX.COMPLETE(
        'mistral-7b',
        CONCAT(
          'Classify this mentorship excerpt into one or more of: ',
          'finance, growth, operations, legal, hr, product, sales, fundraising, leadership, other. ',
          'Return a JSON array. Text: ', text
        )
    )::VARIANT AS topic_tags
FROM STAGED.SESSION_TURNS
WHERE session_id = :session_id;
```

### Step 4 — Chunking
Chunk by: 512 tokens max, 64-token overlap, **never split a speaker turn mid-sentence**.

```python
def chunk_turns(turns: list[dict], max_tokens=512, overlap=64) -> list[dict]:
    """
    Group consecutive same-speaker turns into chunks ≤ max_tokens.
    Overlap: carry last `overlap` tokens of previous chunk into next.
    Each chunk inherits the speaker of the majority turn.
    """
    ...
```

Store chunks in `STAGED.SESSION_CHUNKS` before embedding.

### Step 5 — Embedding (Snowflake Cortex)
```sql
INSERT INTO AI_READY.MENTORSHIP_CHUNKS (
    chunk_id, session_id, mentor_id, market_id,
    topic_tags, speaker, embed_model, embed_model_version,
    chunk_text, embedding, ingested_at
)
SELECT
    UUID_STRING() AS chunk_id,  -- replace with ULID UDF
    c.session_id,
    s.mentor_id,
    s.market_id,
    c.topic_tags,
    c.speaker,
    'snowflake-arctic-embed-m' AS embed_model,
    '1.0.0' AS embed_model_version,
    c.chunk_text,
    SNOWFLAKE.CORTEX.EMBED_TEXT_768('snowflake-arctic-embed-m', c.chunk_text) AS embedding,
    CURRENT_TIMESTAMP() AS ingested_at
FROM STAGED.SESSION_CHUNKS c
JOIN STAGED.SESSIONS s USING (session_id)
WHERE c.session_id = :session_id;
```

### Step 7 — Audit event
```python
await audit_log.append({
    "event_type": "session_ingested",
    "trace_id": trace_id,
    "actor": pipeline_actor,
    "session_id": session_id,
    "chunk_count": chunk_count,
    "embed_model": "snowflake-arctic-embed-m",
    "embed_model_version": "1.0.0",
    "ingested_at": datetime.utcnow().isoformat(),
})
```

---

## Retrieval contract

### Query flow
```
User query (text)
    │
    ▼
Embed query with same model as index
    │
    ▼
VECTOR_COSINE_SIMILARITY search, filtered to user's market_id
    │
    ├─ score ≥ 0.75 → include in context, add to citations
    └─ score < 0.75 → exclude; if no chunks pass threshold → fallback response
    │
    ▼
LLM completion with retrieved context + citations
    │
    ▼
Return: {answer, source_citations: [{session_id, mentor_id, date, topic_tags, score}]}
```

### Retrieval SQL
```sql
WITH query_embedding AS (
    SELECT SNOWFLAKE.CORTEX.EMBED_TEXT_768(
        'snowflake-arctic-embed-m',
        :query_text
    ) AS qvec
)
SELECT
    c.chunk_id,
    c.session_id,
    c.mentor_id,
    s.session_date,
    c.topic_tags,
    c.chunk_text,
    VECTOR_COSINE_SIMILARITY(c.embedding, q.qvec) AS score
FROM AI_READY.MENTORSHIP_CHUNKS c
JOIN STAGED.SESSIONS s USING (session_id)
CROSS JOIN query_embedding q
WHERE c.market_id = :market_id          -- market scoping
  AND VECTOR_COSINE_SIMILARITY(c.embedding, q.qvec) >= 0.75
ORDER BY score DESC
LIMIT 8;
```

### Fallback response template
```
No closely matching mentorship sessions were found in our knowledge base for this query.

Here is general guidance based on common patterns in [topic]:

[LLM-generated general advice, clearly labeled as not from a specific session]

---
If you have access to relevant sessions, please share them with your FDE to add them to the
knowledge base.
```

### Citation format (returned to client)
```json
{
  "answer": "Based on mentorship experience in this network...",
  "source_citations": [
    {
      "session_id": "ses_01j...",
      "mentor_id": "mnt_01j...",
      "mentor_name": null,          // null unless attribution consent granted
      "session_date": "2024-03",    // month precision only for privacy
      "topic_tags": ["finance", "fundraising"],
      "similarity_score": 0.87
    }
  ],
  "trace_id": "trc_01j..."
}
```

`mentor_name` is populated **only** when `CONSENT.scope = 'attribution'` is active for
that mentor. Otherwise always null.

---

## Feedback loop

### Feedback table
```sql
CREATE TABLE AI_READY.RESPONSE_FEEDBACK (
    feedback_id     VARCHAR PRIMARY KEY,  -- ULID
    trace_id        VARCHAR NOT NULL,
    query_hash      VARCHAR NOT NULL,     -- SHA-256 of query_text
    rating          VARCHAR NOT NULL,     -- helpful | partially_helpful | not_helpful
    chunk_ids_used  ARRAY,               -- chunks that contributed to the answer
    rated_by        VARCHAR NOT NULL,     -- staff_id of the FDE
    rated_at        TIMESTAMPTZ NOT NULL,
    notes           VARCHAR
);
```

### Escalation rule
```sql
-- Flag chunks for review if their sessions get ≥ 3 not_helpful ratings
SELECT
    UNNESTED.value::VARCHAR AS chunk_id,
    COUNT(*) AS not_helpful_count
FROM AI_READY.RESPONSE_FEEDBACK f,
     LATERAL FLATTEN(input => f.chunk_ids_used) UNNESTED
WHERE f.rating = 'not_helpful'
GROUP BY chunk_id
HAVING not_helpful_count >= 3;
```

---

## Model upgrade procedure

When upgrading the embedding model:

1. Create new column: `embedding_v2 VECTOR(FLOAT, <new_dim>)` in `AI_READY.MENTORSHIP_CHUNKS`.
2. Re-embed all chunks into `embedding_v2` in batches (respect Cortex rate limits).
3. Validate: run a test set of queries against both columns; compare recall@8.
4. If new model wins: rename columns (`embedding` → `embedding_v1`, `embedding_v2` → `embedding`).
5. Update `embed_model` + `embed_model_version` on all rows.
6. Keep `embedding_v1` for 30 days, then drop.
7. Log the upgrade in audit: `{event_type: model_upgrade, old_model, new_model, rows_re_embedded}`.
