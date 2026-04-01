# Abra Coop

A general-purpose cooperation tool. Like Claude Code is for engineering, Abra Coop is for communities.

Read these in order: ORIGIN.md (why this exists), DESIGN.md (architecture), learnings.md (what we've discovered).

## Three Levels (same as abra's spec/impl/instance)

1. **Tool** (this repo) — system prompt, launcher, protocols, documentation, and code. Universal.
2. **Instance** — a running coop connected to a specific community's data. Config, auth, store connections. Not in this repo.
3. **Community data** — the community's own bindings, intentions, decisions, notes. Lives in the community's space. Coop connects to it, doesn't own it.

This repo is level 1 only. No community-specific data here.

## For Claude Code (when writing code in this repo)

- Read DESIGN.md for architecture before making changes.
- The system-prompt.md file is the tool's identity. Discuss before changing.
- Code should serve non-technical users. No engineering jargon in user-facing output.
- The tool delegates to specialized tools (abra, trustcheck, claude-code, web research). Keep that separation.
- ORIGIN.md has context on why the design is what it is.

## For Abra Coop (when building itself)

- Capture what you learn in learnings.md.
- Propose design changes in DESIGN.md — add sections, note when thinking evolved and why.
- You can propose changes to system-prompt.md but discuss with the human first.
- When you need code written, describe what's needed clearly so Claude Code can do it.

## Rules

- System prompt changes: discuss before editing.
- concept-notes.md in ~/abra-repo is human-edited only.
- No community-specific data in this repo.
- No instance configuration in this repo.

## Related

- **~/abra-repo** — The naming/binding spec and implementations. Read concept-notes.md and arch_notes.md.
- **~/4-1-claude-code-for-humans.md** — Analysis of Claude Code's engineer bias with source pointers.
- **~/claude-code-original-src** — Claude Code source, reference for architecture patterns.
- **LinkedClaims** (github.com/Cooperation-org/LinkedClaims) — Trust/attestation standard. Core to how coop evaluates claims.
