# Amebo + Abra: Architecture Plan

**Date:** 2026-04-04
**Status:** Draft — proposals for discussion, not final decisions. Open for input and alternative suggestions.

---

## What This Document Is

A concrete engineering plan for integrating Abra's binding/knowledge model into Amebo. Based on analysis of both codebases as they exist today, not what we hope they'll become.

## The Honest Starting Point

**Amebo** (github.com/Cooperation-org/amebo) is a working product:
- ~5K lines of Python backend (FastAPI), ~3K lines Next.js frontend
- Slack ingestion (real-time events + backfill scheduler), document extraction (PDF/DOCX/TXT/MD)
- RAG pipeline: ChromaDB semantic search -> quality filter -> Claude answer generation
- Multi-tenant: organizations -> workspaces -> documents -> users
- PostgreSQL for metadata (15 tables, views, triggers), ChromaDB for content + vectors
- Production deployed with systemd, JWT auth, encrypted Slack tokens

**Abra** (github.com/Cooperation-org/abra) is a working knowledge store:
- 4 PostgreSQL+pgvector tables (catcode_registry, content, bindings, hot_tags)
- ~800 lines Python CLI (query.py, write_binding.py)
- 12 CLI commands (who, about, when, search, related, refs, names, hot, read, store, bind, hot set/unset)
- Embeddings via all-MiniLM-L6-v2 (384 dimensions, local, free)
- Odoo CRM connector for PII-safe contact management
- 11K+ bindings, 4K+ names already indexed

**Abra-ai** (github.com/Cooperation-org/abra-ai) is a design document:
- 41-line bash script wrapping Claude CLI with a custom system prompt
- 3 contextual prompts (capture-protocol, trust-evaluation, working-discipline)
- Design docs and governance principles
- Zero application code

The "merge" is not merging two codebases of equal size. It is:
1. Adding Abra's 4 tables to Amebo's database
2. Replacing ChromaDB with pgvector
3. Adding a binding service to Amebo's backend
4. Enriching the QA pipeline with binding context
5. Adopting facilitation prompt ideas for answer generation

---

## Architecture Decisions

### Decision 1: Evolve Amebo in Place (Don't Fork)

**Choice:** Build directly on Amebo's existing repo. No fork.

**Reasoning:**
- Amebo has no external users depending on its current API contract. It's early enough to reshape.
- Forking creates two diverging repos that a small team can't maintain.
- "Preserve Kene's history" works just as well with linear evolution — new commits on top.
- A fork implies the original goes stale. There's no reason to preserve a "vanilla Amebo" separately.

**What this means:** All changes happen as commits to `github.com/Cooperation-org/amebo`. Abra's table schema gets added. ChromaDB gets replaced. New services get added. The frontend evolves incrementally.

### Decision 2: Single Database, Shared Schema

**Choice:** Add Abra's tables to Amebo's PostgreSQL database with the `abra_` prefix.

**Reasoning:**
- Amebo already has PostgreSQL with 15+ tables and a connection pool.
- Abra's tables are simple (4 tables, no complex FKs between them).
- Single database means one backup, one connection pool, one migration path.
- Vectors and metadata live together — enables JOIN queries across messages and bindings.
- Cross-database queries (via FDW or application joins) add complexity for no benefit at this scale.

**New tables in Amebo's database:**
```sql
-- Enable pgvector extension (one-time)
CREATE EXTENSION IF NOT EXISTS vector;

-- Abra's tables, adapted for multi-tenancy
abra_catcode_registry (catcode, parent_catcode, org_id, label, created_at)
abra_content          (id, org_id, workspace_id, source_file, content, embedding vector(384), note_date, catcode, created_at)
abra_bindings         (id, org_id, workspace_id, scope, name, relationship, target_type, target_ref, qualifier, permanence, source_date, catcode, created_at)
abra_hot_tags         (scope, name, org_id, priority, added_at, expires_at)
```

**Key change from Abra's current schema:** Added `org_id` and `workspace_id` columns. Abra currently uses flat scopes ("golda", "linkedtrust"). In Amebo, bindings are scoped to organizations, optionally to workspaces. This preserves Amebo's 4-layer tenant isolation while supporting Abra's scope model (scope becomes a label within the org).

### Decision 3: Replace ChromaDB with pgvector

**Choice:** pgvector replaces ChromaDB as the vector store.

**Reasoning:**

| Factor | ChromaDB (current) | pgvector (proposed) |
|--------|-------------------|-------------------|
| Deployment | Separate process, file-based persistence | Inside PostgreSQL, already running |
| Joins | Impossible — separate system | Native SQL JOINs with metadata |
| Multi-tenancy | Collection naming convention (`workspace_X_messages`) | WHERE clauses on org_id/workspace_id |
| Backup | Separate backup strategy | One database backup covers everything |
| Binding integration | Two-system glue code needed | Same table space, single query |
| Embeddings | Built-in (handles embedding for you) | Manual — must call embedding model before insert |
| Abra compatibility | Incompatible — Abra uses pgvector | Native — same extension, same query patterns |

**What we lose:** ChromaDB's built-in embedding. We now manage embeddings ourselves.

**What we gain:** One database for everything. The killer feature is not "pgvector is better than ChromaDB" — it's that bindings and message vectors share the same query infrastructure. A single SQL query can search messages AND check binding relationships.

**Embedding model choice:** `all-MiniLM-L6-v2` (384 dimensions). Same model Abra already uses. Runs locally, no API cost, fast enough for batch processing. If quality becomes an issue later, swap to a larger model — the vector column dimension is the only thing that changes.

**New table replacing ChromaDB:**
```sql
message_vectors (
    id SERIAL PRIMARY KEY,
    org_id INT NOT NULL REFERENCES organizations(org_id),
    workspace_id VARCHAR(20) NOT NULL REFERENCES workspaces(workspace_id),
    message_id INT NOT NULL REFERENCES message_metadata(message_id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    embedding vector(384) NOT NULL,
    channel_id VARCHAR(20),
    channel_name VARCHAR(255),
    user_id VARCHAR(20),
    user_name VARCHAR(255),
    slack_ts VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_message_vectors_workspace ON message_vectors(workspace_id);
CREATE INDEX idx_message_vectors_org ON message_vectors(org_id);
CREATE INDEX idx_message_vectors_embedding ON message_vectors USING hnsw (embedding vector_cosine_ops);
CREATE INDEX idx_message_vectors_message ON message_vectors(message_id);
```

The HNSW index enables fast approximate nearest-neighbor search. ChromaDB's query pattern `collection.query(query_texts=[...])` becomes:
```sql
SELECT mv.*, 1 - (mv.embedding <=> %s::vector) AS similarity
FROM message_vectors mv
WHERE mv.workspace_id = %s
ORDER BY mv.embedding <=> %s::vector
LIMIT 10;
```

### Decision 4: Amebo Is the Product, Not a Service for Abra

**Choice:** Single backend (Amebo's FastAPI), with an optional MCP interface later. Not Amebo-as-MCP-server-for-Abra.

**Reasoning:**
- The data flows both directions — Abra needs conversation history, the QA service needs binding context. A one-way service boundary doesn't fit.
- Two services means two deployments, two monitoring setups, network hops for every query. Overhead for a small team.
- The facilitation logic (system prompts, binding-enriched answers) belongs in the QA service, not in a separate process calling back.
- The `coop` launcher (Abra-ai's bash script) can use Amebo's REST API or a future MCP interface for read access. That's a thin layer on top, not the primary architecture.

**What this means concretely:**
```
Amebo Backend (FastAPI) — single process
├── /api/qa/            existing Q&A (now binding-enriched)
├── /api/documents/     existing document management
├── /api/workspaces/    existing workspace management
├── /api/bindings/      NEW — binding CRUD (query, write, hot tags)
├── /api/knowledge/     NEW — unified search across messages + bindings + documents
│
├── services/
│   ├── qa_service.py           existing (modified to query bindings alongside vectors)
│   ├── query_service.py        existing (pgvector replaces ChromaDB calls)
│   ├── binding_service.py      NEW — wraps Abra's write_binding.py + query.py logic
│   ├── knowledge_service.py    NEW — unified search across all content types
│   ├── backfill_service.py     existing (modified to generate embeddings on insert)
│   └── document_service.py     existing (modified to store vectors in pgvector)
│
└── db/
    ├── connection.py           existing (same pool, same database)
    ├── pgvector_client.py      NEW — replaces chromadb_client.py
    └── repositories/
        ├── binding_repo.py     NEW — data access for abra_bindings, abra_content, etc.
        └── (existing repos)
```

### Decision 5: Slack Becomes One Integration, Not The Integration

**Choice:** Abstract ingestion behind a common interface. Slack-specific code stays but is no longer the only path.

**Reasoning:**
- Golda's point: "Slack is just ONE of the integrations, not only/primary necessarily."
- Abra's data comes from meeting notes, LinkedIn, Google contacts, manual input — not Slack.
- Future sources: Discord, email, meeting transcripts, manual binding input via web UI.
- The current code already has clean separation between "fetch from Slack" and "store in vector DB." The abstraction is small.

**Concretely:** The backfill service and slack listener continue to work as-is. The document service continues to work as-is. Both now write to pgvector instead of ChromaDB. Future sources (Discord, email, manual) follow the same pattern: fetch content -> generate embedding -> store in `message_vectors` or `abra_content` + create bindings.

No need to build a formal `ContentSource` interface right now. That's premature abstraction. When the second non-Slack source arrives, the pattern will be clear.

### Decision 6: Binding-Enriched QA (The New Capability)

**Choice:** When the QA service builds context for Claude, it also queries Abra bindings to enrich the prompt.

**What changes in `qa_service.py`:**

Today's flow:
```
Question -> ChromaDB semantic search -> quality filter -> build context string -> Claude generates answer
```

New flow:
```
Question -> pgvector semantic search -> quality filter -> build context string
                                                              |
                                        binding enrichment: query abra_bindings
                                        for names mentioned in results
                                              |
                                        combined context -> Claude generates answer
```

Example: User asks "What's the status of the LTQ1 project?"

Today: Amebo finds Slack messages mentioning "LTQ1" and summarizes them.

With bindings: Amebo finds Slack messages AND checks `abra_bindings` for:
- `ltq1 IS "LinkedTrust Quarterly 1"` (identity)
- `ltq1 RELATED peter "potential partner"` (relationship)
- `ltq1 ABOUT content/42 "project proposal"` (linked content)
- `ltq1` has a hot tag (high priority right now)

The Claude prompt gets richer context: not just "here are messages mentioning LTQ1" but "LTQ1 is LinkedTrust Quarterly 1, Peter is a potential partner, here's the project proposal, and this is currently a priority."

**Implementation:** Add a `_enrich_with_bindings()` method to QAService that:
1. Extracts names/terms from search results
2. Queries `abra_bindings` for matching names (scope = org scope)
3. Fetches linked `abra_content` blobs for ABOUT bindings
4. Appends structured relationship context to the prompt

This is an addition to `qa_service.py`, not a rewrite. The existing retrieval, filtering, and answer generation stay the same.

### Decision 7: System Prompt Becomes Configurable

**Choice:** The QA service's system prompt for Claude is stored as configuration, not hardcoded.

**Reasoning:**
- Today's prompt in `qa_service.py` is hardcoded: "You are a helpful teammate answering questions about your Slack workspace."
- Abra Coop's system prompt is different: "You are Abra Coop, a cooperation facilitator..."
- Different communities may want different dispositions (direct Q&A vs. facilitation vs. analysis)

**Implementation:** A `prompts/` directory in the backend, or a `system_prompts` table. The QA service loads the appropriate prompt based on workspace configuration. Default is the current Amebo prompt. Organizations can override with a facilitation-oriented prompt.

This is Phase 5 work. Not urgent. The current hardcoded prompt is fine for now.

---

## Phased Implementation Plan

### Phase 1: ChromaDB -> pgvector Migration

**Goal:** Replace ChromaDB with pgvector. Amebo works identically but vectors live in PostgreSQL.

**Files that change:**
| File | Change |
|------|--------|
| `backend/src/db/chromadb_client.py` | Replaced by `pgvector_client.py` |
| `backend/src/db/pgvector_client.py` | NEW — same interface (search_messages, add_message, add_messages_batch, etc.) |
| `backend/src/services/query_service.py` | Import `PgvectorClient` instead of `ChromaDBClient` |
| `backend/src/services/backfill_service.py` | Use pgvector_client for storing messages |
| `backend/src/services/document_service.py` | Use pgvector_client for storing document chunks |
| `backend/src/db/schema.sql` | Add `message_vectors` table, pgvector extension |
| `backend/requirements.txt` | Add `psycopg2-binary[vector]`, `sentence-transformers`; remove `chromadb` |

**Migration script:** Reads all documents from ChromaDB collections, generates embeddings with all-MiniLM-L6-v2, inserts into `message_vectors` table. Run once, verify counts match, then remove ChromaDB dependency.

**The interface stays the same:**
```python
# Old (ChromaDB)
self.chromadb.search_messages(workspace_id, query_text, n_results)
self.chromadb.add_messages_batch(workspace_id, messages)

# New (pgvector) — same method signatures
self.pgvector.search_messages(workspace_id, query_text, n_results)
self.pgvector.add_messages_batch(workspace_id, messages)
```

Callers (qa_service, query_service, backfill_service) change one import line. The rest of the code is unaffected.

**Estimated scope:** ~300 lines new (pgvector_client.py), ~20 lines changed across existing services, ~100 lines migration script. Remove ~425 lines (chromadb_client.py).

### Phase 2: Add Abra Binding Tables

**Goal:** Abra's 4 tables exist in Amebo's database with multi-tenant columns. A binding service provides CRUD.

**Files that change/add:**
| File | Change |
|------|--------|
| `backend/src/db/schema.sql` | Add `abra_bindings`, `abra_content`, `abra_catcode_registry`, `abra_hot_tags` |
| `backend/src/db/repositories/binding_repo.py` | NEW — data access for binding tables |
| `backend/src/services/binding_service.py` | NEW — business logic (adapted from Abra's write_binding.py + query.py) |
| `backend/src/api/routes/bindings.py` | NEW — REST endpoints for binding CRUD |
| `backend/src/api/main.py` | Register binding routes |

**What gets ported from Abra:**
- `write_binding.py`'s AbraWriter class -> `binding_service.py` (PII checking, binding CRUD, catcode management, hot tags)
- `query.py`'s read commands -> `binding_repo.py` (who, about, search, related, names, hot)
- Multi-tenancy added: all queries filter by `org_id`

**What doesn't get ported:**
- The CLI interface (Abra's `query.py` argparse) — the CLI keeps working by pointing at the same database
- The Odoo connector — stays in Abra's repo, works independently

### Phase 3: Source-Agnostic Ingestion

**Goal:** Clean separation between "where data comes from" and "how it's stored and searched."

**Concretely:** Refactor the backfill and document services so their storage logic is reusable. When a Discord connector or email ingester arrives, they call the same storage functions.

This is a light refactor, not an abstraction framework. Move the "generate embedding + insert into pgvector" logic into a shared function that any source can call.

### Phase 4: Binding-Enriched Retrieval

**Goal:** The QA service queries bindings alongside vector search when building context.

**Files that change:**
| File | Change |
|------|--------|
| `backend/src/services/qa_service.py` | Add `_enrich_with_bindings()` method, modify `_build_context()` |
| `backend/src/services/knowledge_service.py` | NEW — unified search across messages + bindings + documents |

**This is the integration payoff.** A question like "Who knows about the accessibility project?" now returns:
- Slack messages discussing accessibility (vector search)
- Binding: `accessibility-project RELATED sarah "team lead"` (structured knowledge)
- Binding: `accessibility-project ABOUT content/57 "project charter"` (linked document)

### Phase 5: Configurable System Prompts

**Goal:** Organizations can configure the AI's disposition — standard Q&A, facilitation mode, analysis mode.

**Files that change:**
| File | Change |
|------|--------|
| `backend/src/services/qa_service.py` | Load system prompt from config instead of hardcoded string |
| `backend/src/db/schema.sql` | Add `workspace_prompts` table or use `bot_config` |
| `backend/prompts/` | NEW directory — default prompts, facilitation prompt, etc. |

---

## What Stays the Same

These parts of Amebo are not affected by the integration:
- **Frontend** (Next.js) — existing pages work as-is. New pages for bindings added later.
- **Auth system** — JWT, bcrypt, user/org management unchanged.
- **Slack connector** — slack_listener.py, slack_commands.py, slack_client.py unchanged (only their storage target changes from ChromaDB to pgvector).
- **Multi-tenant model** — org -> workspace -> user hierarchy preserved. Bindings slot into it.
- **API routes** — existing routes untouched. New routes added alongside.
- **Deployment** — same systemd services. ChromaDB process removed (simplification).

## What the Abra CLI Does

The Abra CLI (`abra who`, `abra about`, etc.) continues to work. It's just SQL queries — point it at Amebo's PostgreSQL and it reads the `abra_*` tables. No code change needed in Abra's query.py beyond the connection string (or add the abra_ table prefix).

The `coop` launcher (abra-ai's bash script) continues to wrap Claude CLI. If it needs workspace knowledge, it can call Amebo's REST API. A future MCP server interface would make this more natural for AI agent use, but that's optional and later.

## Open Questions for Discussion

1. **Embedding model quality:** all-MiniLM-L6-v2 is fast and free but not the best quality. Is it good enough for the Q&A use case? Amebo's ChromaDB uses its default model (also sentence-transformers based). We should test retrieval quality after migration.

2. **PII boundary:** Abra enforces no-PII in the binding store (PII goes to Odoo CRM). Amebo stores Slack messages which may contain PII (names, emails in message text). Do we apply the same PII rule to message_vectors, or is that a different concern?

3. **Catcode adoption:** Abra's catcode system is a hierarchical spatial coordinate system for organizing knowledge. Should Amebo adopt catcodes for organizing workspaces/channels/documents, or is the existing org/workspace hierarchy sufficient?

4. **Hot tags UX:** Hot tags are powerful for maintaining focus, but they're CLI-only today. How should they surface in Amebo's web UI? A "pinned topics" widget? Priority badges on search results?

5. **Scope mapping:** Abra uses flat scopes ("golda", "linkedtrust"). Amebo uses org_id + workspace_id. The proposed mapping: org = top-level scope, workspace = sub-scope. But some bindings are org-wide (team relationships) while others are workspace-specific (Slack conversation context). Need to define the scoping rules.

6. **Migration of existing Abra data:** The current Abra database has 11K+ bindings and 4K+ names. These need to be migrated into Amebo's database with org_id/workspace_id assigned. What org/workspace do they belong to?

---

## Summary

The integration is adding Abra's structured knowledge model to Amebo's existing ingestion and search infrastructure. Not a rewrite, not a fork, not a separate service. Five phases, each independently valuable:

1. **pgvector** — removes ChromaDB, unifies storage
2. **Binding tables** — adds structured knowledge alongside unstructured messages
3. **Source abstraction** — Slack becomes one source among many
4. **Enriched retrieval** — QA answers get better with binding context
5. **Configurable prompts** — different communities, different dispositions

Phase 1 is the foundation. Everything else builds on it.
