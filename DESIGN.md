# Claude Coop Design

## What This Is

A general-purpose cooperation tool. Claude Code helps engineers ship software. Claude Coop helps people cooperate.

Users are community members, organizers, facilitators, team leads, volunteers — anyone working with others toward shared goals. They may or may not be technical.

## Separation of Concerns

Like abra, like Claude Code — the tool is not the data.

- **The tool** knows how to facilitate, how to query bindings, how to check trust, how to capture decisions. General purpose.
- **The community's data** lives wherever the community keeps it — a repo, a shared drive, an abra store, a database. The tool connects to it.
- **Instance config** tells the tool where this community's data is, what services are available, who's who.

Claude Code works in whatever directory you run it from. Claude Coop should work the same way — you point it at a community's space and it understands what's there.

## Core Tools

Claude Coop delegates to specialized tools rather than doing everything itself:

| Tool | What It Does | Status |
|---|---|---|
| **abra** | Query/write bindings — pet names, relationships, context. The community's shared language. | Exists (pgvector impl, flat file possible) |
| **trustcheck** | LinkedClaims verification — who trusts whom, about what, with what evidence. Contextual, directional, evidenced. | SDK exists, needs integration |
| **claude-code** | Delegated engineering tasks. When actual coding is needed. | Works (call via CLI) |
| **web research** | External context — web search, API calls, document fetch. | Works (WebFetch, WebSearch) |
| **file tools** | Read/write/search documents in the community's space. | Works (inherited from Claude Code) |

Others will emerge. The tool list is open.

## Context Types Communities Produce

| Type | Examples | How Coop Works With It |
|---|---|---|
| Intentions | Mission, vision, values, goals | Reads them, reflects them back, checks alignment, helps evolve them |
| Decisions | Proposals, votes, rationale, dissent | Captures them with full context, searches precedent |
| Relationships | Who cares about what, roles, trust | Queries abra bindings + LinkedClaims attestations |
| Narrative | Stories, history, lessons learned | Distills, preserves institutional memory |
| Resources | Budget, time, capacity | Tracks constraints, checks feasibility |
| External | Legal, regulatory, ecosystem | Researches via web tools |

## What "Alive Intentions" Means

Static: a vision document in a wiki nobody reads.

Alive: the tool knows the community's stated values and reflects them in every interaction. Tensions between goals get named. Decisions capture rationale and dissent. New members get context through the community's own language (hot tags, pet names). The intentions live in the conversation loop, not in a file.

## Analysis Approaches (Research Needed)

Code has AST and LSP. What do community artifacts have?

- Structured data (budgets, catcode hierarchies, decision records) → schema validation, constraint checking
- Natural language (vision statements, meeting notes) → semantic search, embedding similarity (abra pgvector already does this)
- Relationships (binding graphs, trust networks) → graph traversal (abra + LinkedClaims)
- Temporal (decision history, goal evolution) → trend detection, drift analysis
- Multi-perspective (different stakeholders' views) → alignment scoring, tension mapping (needs design)

## The System Prompt

The system prompt (system-prompt.md) defines the tool's identity and disposition. Key differences from Claude Code:

- Identity: cooperation facilitator, not software engineer
- Trust as core: LinkedClaims concepts baked into reasoning, not an afterthought
- Facilitation over direction: helps people think, doesn't think for them
- Community language: uses pet names, avoids jargon
- Tension-aware: names conflicts rather than resolving them silently

## How a Community Sets Up

(To be designed. Rough sketch:)

1. Community has a space — a repo, a directory, a shared location
2. That space has community-level docs: intentions, decisions, bindings, whatever format they use
3. They run coop pointed at that space (like running `claude` in a project directory)
4. Coop reads whatever's there — CLAUDE.md for instructions, abra bindings for context, intention files for values
5. As they work, coop captures to durable files in the community's space
6. Next session, that context is there

## Open Questions

- What's the right community-space convention? (Like CLAUDE.md is for Claude Code)
- How does instance config work? (What services, what stores, who's who)
- How do multiple communities interact? (Scoped bindings help, but trust across communities?)
- What's the capture protocol? (What gets saved where, in what format, automatically vs manually)
- How do non-technical users interact? (Terminal is for engineers. Web UI? Chat integration? Voice?)
