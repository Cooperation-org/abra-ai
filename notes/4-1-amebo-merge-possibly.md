# Amebo + Abra Coop: Possible Merge

## Date: 2026-04-01

## Context

Kene built Amebo (github.com/Cooperation-org/amebo) — an AI-powered workspace intelligence tool that ingests Slack conversations and documents into vector search (ChromaDB) with a FastAPI backend and Next.js frontend. It's a working product with real code, his commit history, and his name on it.

Today we built abra-ai — system prompts, contextual prompts, design docs, governance principles for a cooperation facilitator tool. No application code yet, just the soul and the architecture thinking.

These projects overlap significantly. The question is how they fit together.

## What Each Side Brings

### From Amebo (Kene's — built first, has working code)
- Slack SDK integration + message backfilling (working, tested)
- Document extraction pipeline (PDF/DOCX/TXT/MD → chunks)
- FastAPI backend structure (Python)
- Next.js frontend with auth, multi-tenant workspace isolation
- `/ask` Slack command pattern — non-technical users ask questions in Slack
- PostgreSQL multi-tenant schema
- ChromaDB vector store for semantic search
- Full git history and his ownership

### From abra-ai (today's design work)
- System prompt + contextual prompts architecture (`prompts/` directory)
- The `coop` launcher pattern (wrapping Claude CLI with custom identity)
- Design docs: DESIGN.md, ORIGIN.md, governance principles handoff
- Separation of tool / instance / community data (from abra's three-level pattern)
- Trust evaluation via LinkedClaims as a core concept
- Facilitation disposition (not just Q&A retrieval)

### What neither has yet
- Abra binding integration (pet names, typed relationships)
- LinkedClaims/trust tooling
- Governance-aware facilitation connected to actual data
- The community intention tracking we designed

## If We Merge

### Preserve
- **Kene's commit history** — fork Amebo, build on top. His name is first in the git log.
- **Ingestion pipeline** — Slack connector, document parsing, backfill scheduler. That's real working code.
- **Python backend** — abra's pgvector impl is Python, Odoo connector is Python, Amebo backend is Python. Consistency.
- **Next.js frontend scaffold** — non-technical community members need a browser UI, not a terminal. This is a starting point.

### Change
- **ChromaDB → pgvector** — unify with abra's existing vector store. One store, not two.
- **Simple Q&A retrieval → binding-aware retrieval** — not just "find the chunk" but "find the chunk and connect it to known names and relationships."
- **Generic AI responses → Abra Coop system prompt** — trust evaluation, facilitation disposition, community intentions.
- **Backend becomes abra-binding-aware** — queries go through the binding layer, not just raw vector search.
- **Frontend will evolve significantly** — the current Amebo dashboard is a Slack Q&A interface. The cooperation tool needs intention tracking, relationship views, decision logs, trust indicators.

### Add
- System prompt + contextual prompts layer
- Abra binding integration (query/write pet names, relationships)
- LinkedClaims trust tooling
- `coop` launcher for CLI usage alongside the web UI
- `prompts/` directory for composable context loading

## Branding

Amebo is a better name than abra-ai for the overall product. Kene named it, his project was first. "Abra Coop" could be the name for the facilitation layer/mode specifically, or it could fold in entirely under Amebo.

This is Kene's call to weigh in on.

## Engineering Decisions to Make Together

1. **ChromaDB vs pgvector** — Kene may have reasons for ChromaDB. Worth discussing, not dictating.
2. **Backend architecture** — how much reshaping vs building alongside.
3. **Frontend scope** — what stays from current Amebo dashboard, what changes for cooperation use cases.
4. **Deployment** — Amebo has a deploy.sh. How does the cooperation layer affect that.

## The Principle

This is a project about earned governance and credibility through contributions. Kene built first. That matters. The merge should feel like his project gaining new capabilities, not his work getting absorbed.
