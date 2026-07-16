# 📚 Knowledge Base Ingestion & Vector Processing

This document explains the technical specs for uploading documents, processing them into semantic chunks, generating vector embeddings, and running retrieval searches.

---

## 📥 Ingestion Specifications

The platform supports `PDF` and `TXT` files uploaded by administrators through the dashboard.

| Parameter | Constraint | Note |
| :--- | :--- | :--- |
| **Max File Size** | 5 MB | Checked on both client-side and backend Edge Function |
| **Supported Formats** | `.txt`, `.pdf` | Managed by specific parser routes |
| **Ingestion Engine** | Supabase Edge Function | Direct upload via Supabase Storage buckets |

---

## ✂️ Text Chunking Strategy

To maintain context coherence and stay within Gemini's context constraints, text must be chunked carefully. Ad-hoc division is not allowed. We use a **Recursive Character Text Splitter** approach:

1. **Splitting Hierarchy:** Split characters in sequence: double newlines (`\n\n`), single newlines (`\n`), spaces (` `), and empty strings (`""`).
2. **Chunk Size:** Target **1,000 characters** per chunk (roughly ~150-200 English words).
3. **Chunk Overlap:** **200 characters** overlap. This overlap ensures context is preserved across adjacent chunks.
4. **Metadata Attachment:** Each chunk is tagged with its parent `document_id` and calculated `token_count`.

---

## 🧪 Embedding Generation

We use the Google Gemini Embedding API to transform text chunks into vectors:
* **Model:** `text-embedding-004`
* **Output Dimensions:** 768 dimensions
* **Batch Processing:** To optimize network calls, chunks are sent in batches of up to 100 per API call.

---

## 🔍 Semantic Search & Vector Matching

To retrieve relevant context for a user's question, we perform a Cosine Similarity search. We execute this via a database RPC function named `match_documents` inside Supabase:

```sql
create or replace function public.match_documents (
  query_embedding vector(768),
  match_threshold float,
  match_count int,
  filter_org_id uuid
)
returns table (
  id uuid,
  content text,
  similarity float
)
language plpgsql stable
as $$
begin
  return query
  select
    dc.id,
    dc.content,
    1 - (dc.embedding <=> query_embedding) as similarity
  from public.document_chunks dc
  join public.documents d on dc.document_id = d.id
  where d.org_id = filter_org_id
    and d.processing_status = 'completed'
    and 1 - (dc.embedding <=> query_embedding) > match_threshold
  order by dc.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

### Prompt Construction (Context Synthesis)
When a user submits a query `Q` to the Telegram bot, the engine:
1. Generates query vector `V` for `Q`.
2. Calls `match_documents` with `V`, `match_threshold = 0.50`, and `match_count = 3`.
3. Concatenates the matching `content` text fields into a single `CONTEXT` block.
4. Assembles the final prompt for Gemini 1.5 Flash:

```text
[System Instructions]
You are a helpful customer support bot. Rely strictly on the Context below to answer the User Query. If the answer cannot be found in the Context, reply with: "I'm sorry, I don't have information about that."

[Context]
---------------------
${CONTEXT}
---------------------

[User Query]
${USER_QUERY}
```
