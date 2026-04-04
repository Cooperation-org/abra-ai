# Abra Implementation Plan — Context Handoff

## Date: 2026-04-04

## Decision: Fork Amebo, Build Abra

Fork Amebo (github.com/Cooperation-org/amebo) and rename to Abra. Preserves Kene's commit history. He can pull changes back into original Amebo. Merge this repo (abra-ai) into the fork — system prompt, design docs, prompts, governance notes all come along.

The name-mapping piece (abra bindings) will be renamed to **lingo**.

## Where the Work Happens

On the server where Amebo currently runs. The server also has a Claude system prompt with LinkedClaims knowledge, which will be used for trust evaluation.

## What Amebo Gives Us

- **Slack integration** — Socket Mode listener, multi-workspace, `/ask` command, message backfilling. Working code.
- **FastAPI backend** — Python, async, uvicorn.
- **PostgreSQL schema** — messages, channels, users, conversations, auth, audit logs.
- **ChromaDB vector store** — semantic search over ingested content. Collection-per-workspace isolation.
- **Document ingestion** — PDF/DOCX/TXT/MD extraction and chunking.
- **Next.js frontend** — dashboard, auth, workspace management.
- **Conversation manager** — multi-turn thread tracking in postgres.
- **APScheduler** — background task scheduling.

Key files:
- `backend/src/main.py` — unified entry point, starts FastAPI + Slack listener + scheduler
- `backend/src/services/slack_listener.py` — Socket Mode event listener
- `backend/src/services/qa_service.py` — RAG Q&A using Anthropic Claude
- `backend/src/services/conversation_manager.py` — multi-turn conversation history
- `backend/src/services/query_service.py` — hybrid PostgreSQL + ChromaDB queries
- `backend/src/db/chromadb_client.py` — vector store client
- `backend/src/db/schema.sql` — full postgres schema
- `backend/src/db/schema_auth.sql` — organizations, users, API keys, audit logs

## What Needs to Change

### Instance Config Decoupling

Amebo bakes instance config into the codebase. Database URL, ChromaDB path (`./chromadb_data`), Slack tokens, API keys — all in `.env` and hardcoded defaults. Schema SQL with INSERT defaults lives in the repo. `bot_config` table has hardcoded defaults.

For Abra where each instance serves a different community:
- Instance config directory (outside repo) that the app loads from
- Schema migrations as versioned files, not a single SQL dump
- All credentials and paths — instance config, not app config
- Runtime config defaults — instance decision, not code decision

### Multi-Tenancy Model Change

Amebo's model: one SaaS platform, many organizations, each org has workspaces.
Abra's model: one instance per community. The `organizations` → `org_workspaces` → `workspaces` hierarchy is SaaS scaffolding that doesn't apply.

### Vector Store

Don't hardcode to pgvector OR ChromaDB. The vector store is a plugin — some instances will have pgvector available, others might use ChromaDB or something else. Define the interface, let the instance config choose the backend. We have pgvector available on the server.

### Open Source Consideration

Some parts may be open sourced later, but not necessarily all. Keep this in mind for module boundaries — clean separation between what's public tooling and what's proprietary to specific deployments.

## Architecture: Plugin Protocol

**This is the first and most important abstraction to design.**

Every external integration (Slack, email, Discord, news scanner, CRM, trust endpoint) is a plugin that conforms to a common protocol. Amebo currently has Slack hardwired — `slack_listener.py` imports `ChromaDBClient` and `BackfillService` directly. That can't continue.

### What Every Plugin Must Answer

- What channels does it watch?
- How does it identify the sender?
- What actions can it take? (read, reply, forward, delete, etc.)
- What trust level is needed for each action?

### Email Plugin (First New Plugin After Slack)

Each instance has an email address (Gmail). The tool checks its inbox on the poll loop, discards spam, acts on real messages.

- Gmail API (OAuth, not IMAP) — poll with history ID for incremental fetch
- Allowlist of accepted sender addresses (instance config)
- Spam: Gmail's spam folder handles most of it; second pass via quick Claude call on subject/sender/preview for what gets through
- Actions: read, reply, forward, archive, flag for human attention

### People Model

A person exists across channels. The same human may be a Slack user, an email sender, a web UI user. The people model maps these identities together. Each person has:
- Known identities (Slack ID, email address, etc.)
- Trust level (from the trust endpoint)
- Access permissions (what resources they can see, what actions they can take)
- Conversation history (across channels)

### Trust Endpoint

An API endpoint. Takes a URI (entity identifier), returns a dict of trust score aspects. This is LinkedClaims-based — contextual, directional, evidenced trust. The rules config for the instance gates:
- Access to the Abra bot itself
- Access to specific named resources

Every resource in the system has a name so it can be gated.

Admins (who may not be technical) can set rules in natural language: "only let linda send emails." The tool translates that to the rules config. (Not first priority, but the architecture should support it.)

### Request Queue

Messages from all channels feed into a prioritized request queue. The tool:
- Identifies the sender (people model)
- Checks trust/permissions (trust endpoint + rules config)
- Prioritizes (some requests are urgent, some can wait)
- Processes (may spawn sub-agents for concurrent work, same as Claude Code)

## Persistence Model

**This is NOT Claude Code.** The tool runs as a persistent daemon process. It sleeps between events.

Poll loop with exponential backoff:
- Activity detected → check again immediately
- No activity for 30 seconds → start backing off
- Max backoff: check every 5 minutes
- Webhooks for Slack and email can wake it up immediately (email webhook has a gate)

### Context Management

When conversation context hits limits: make notes to a file, truncate earlier context. No lossy summarization/compaction. Interact with conversation context checkpoints directly.

## Sequence

1. **Fork Amebo, merge abra-ai into it.** Rename to Abra. Kene's commits preserved.
2. **Design and implement the plugin protocol and people/identity model.** This comes before cleaning because the protocol shape determines what stays, what gets ripped out, and what gets reshaped in the existing code. Specifically:
   - Plugin interface (channel in, channel out, sender identity, action permissions)
   - People model (cross-channel identity)
   - Request queue (prioritized, trust-gated)
   - Trust check as API call to external endpoint
3. **Clean Amebo to fit the abstractions.** Rip out SaaS multi-tenancy, make vector store pluggable, extract Slack integration into a plugin, separate instance config from code.
4. **Apply Claude Code patterns:**
   - Tool schema and validation on every plugin
   - Contextual prompt assembly (system prompt cached, situational context per-query)
   - Read-only concurrent, write serial
   - Stale-write protection
   - Event logging on all interactions (the audit trail IS the trust)
   - Sub-agent spawning for concurrent work
5. **Build email plugin** as second plugin. Validates the protocol generalizes beyond Slack.
6. **Test and iterate.**

## Claude Code Patterns to Apply (Priority Order)

1. **Tool interface as protocol** — schema (validation), async call, permissions check. Every plugin, every tool. This is the plugin protocol itself.
2. **Contextual prompt assembly** — core identity small and cached. Situational prompts (who's talking, what community, what's the current topic) loaded per interaction. Already started with `prompts/` directory.
3. **Event logging everywhere** — every interaction, every trust check, every tool delegation logged. Amebo has `audit_logs` but only for auth. For a cooperation tool, the audit trail is core.
4. **Sub-agents as isolated conversations** — for concurrent request processing. Own message history, own tools, results serialized back.
5. **Read-only concurrent, write serial** — simple rule for the request queue and shared state.
6. **Stale-write protection** — track when data was last read, reject writes if modified since.
7. **Deferred tool loading** — keyword search over available plugins/tools. Essential as the plugin count grows.
8. **Prompt caching with boundary markers** — static content (identity, disposition) cached globally, dynamic content (who's talking, community context) per-session. Cost factor.

## What's On the Server

- Amebo running (current deployment)
- PostgreSQL with pgvector available
- Claude system prompt with LinkedClaims knowledge (for trust evaluation)

## Related Files in This Repo

- `system-prompt.md` — Abra Coop identity and disposition
- `DESIGN.md` — architecture overview, context types, tool delegation
- `ORIGIN.md` — why this exists, what Claude Code analysis revealed
- `prompts/` — contextual prompts (capture-protocol, trust-evaluation, working-discipline)
- `notes/4-1-insights-and-next-steps.md` — Claude Code deep dive findings
- `notes/4-1-governance-principles-handoff.md` — earned governance, transparency, ranked choice
- `notes/4-1-amebo-merge-possibly.md` — original merge analysis
- `learnings.md` — development learnings
- `coop` — launcher script (wraps Claude CLI with custom system prompt)

## Key Decisions Made in This Session

- The tool is a persistent daemon, not an interactive CLI session. Sleep/poll with exponential backoff.
- Webhooks for immediate wake on Slack/email (email webhook has a gate).
- Vector store is pluggable, not hardcoded to pgvector or ChromaDB.
- Plugin protocol is the first abstraction to design — before cleaning existing code.
- Email is the first new plugin (Gmail, allowlisted senders, spam filtering).
- Every resource has a name so it can be gated by trust rules.
- Trust is an API endpoint call, not inline logic.
- People model maps identities across channels.
- Context management: notes + truncation, not compaction.
- Parts may be open sourced later but not all — keep module boundaries clean.
- Name-mapping piece renamed to **lingo**.
