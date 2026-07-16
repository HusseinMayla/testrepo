# 🛠️ System Architecture & Database Design

This document details the core database architecture, encryption methodology, and runtime integration workflow of `chatbotRAG`. All database operations, vector lookups, and session transactions are hosted on **Supabase**.

---

## 💾 Database Schema (PostgreSQL / Supabase)

To support document chunk storage, fast cosine similarity vector search, usage auditing, and multi-tenant bot associations, we define the following tables.

### Vector Extension
We enable the `pgvector` extension for vector operations:
```sql
create extension if not exists vector with schema public;
```

### Table 1: `organizations`
Tracks tenant metadata and subscription tiers.
```sql
create table public.organizations (
    id uuid default gen_random_uuid() primary key,
    name text not null,
    plan_tier text not null default 'starter' check (plan_tier in ('starter', 'growth', 'pro')),
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);
```

### Table 2: `telegram_bots`
Stores credentials and system parameters for the chatbot instances. Telegram API keys are stored encrypted.
```sql
create table public.telegram_bots (
    id uuid default gen_random_uuid() primary key,
    org_id uuid references public.organizations(id) on delete cascade not null,
    encrypted_bot_token text not null,
    encryption_nonce text not null,
    is_active boolean default true not null,
    webhook_secret_token text not null,
    system_prompt text default 'You are a helpful customer service assistant.' not null,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);
```

### Table 3: `documents`
Logs knowledge-base files uploaded by administrators.
```sql
create table public.documents (
    id uuid default gen_random_uuid() primary key,
    org_id uuid references public.organizations(id) on delete cascade not null,
    filename text not null,
    file_size_bytes integer not null,
    mime_type text not null,
    processing_status text default 'pending' check (processing_status in ('pending', 'processing', 'completed', 'failed')),
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);
```

### Table 4: `document_chunks`
Stores text fragments and their associated vector embeddings.
```sql
create table public.document_chunks (
    id uuid default gen_random_uuid() primary key,
    document_id uuid references public.documents(id) on delete cascade not null,
    content text not null,
    embedding vector(768) not null, -- Dimension 768 matches Gemini text-embedding-004
    token_count integer not null,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);

-- Indexing for fast Cosine Distance vector search
create index on public.document_chunks using hnsw (embedding vector_cosine_ops);
```

### Table 5: `usage_logs`
Enforces rate-limits and token quotas.
```sql
create table public.usage_logs (
    id uuid default gen_random_uuid() primary key,
    org_id uuid references public.organizations(id) on delete cascade not null,
    telegram_chat_id bigint not null,
    tokens_consumed integer not null default 0,
    created_at timestamp with time zone default timezone('utc'::text, now()) not null
);
```

---

## 🔒 Telegram Key Encryption (pg_sodium)

To ensure client-provided Telegram API keys are highly secure at rest, we employ Supabase's native `pg_sodium` extension or Web Crypto APIs inside Supabase Edge Functions:
1. **Encryption Key:** A master key stored securely in Supabase vault.
2. **Algorithm:** Authenticated Encryption with Associated Data (AEAD) using `AES-GCM-256`.
3. **Nonce:** A unique salt (`encryption_nonce`) is generated per bot entry, preventing pattern matching attacks.
4. **Decryption:** The decryption process only occurs temporarily in-memory within the Supabase Edge Function during webhook verification and setup.

---

## 🛰️ Webhook Processing (Supabase Edge Functions)

The primary gateway of the Telegram bot runs inside a Supabase Edge Function:
* **Webhook Endpoint:** `/functions/v1/telegram-webhook`
* **Security Validation:** Every request from Telegram must carry the `X-Telegram-Bot-Api-Secret-Token` header matching the `webhook_secret_token` stored in the database.
* **Payload Routing:**
  1. Parse the incoming JSON message structure.
  2. Query `telegram_bots` matching the incoming route's identifier.
  3. Validate usage quota & rate limits (see [security_and_billing.md](file:///c:/Users/Abbas/dev/testrepo/security_and_billing.md)).
  4. Perform Cosine Similarity vector search via database RPC.
  5. Run RAG logic on Gemini API and respond to the Telegram API.
