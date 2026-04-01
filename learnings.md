# Claude Coop Development Learnings

Tool-level learnings from building coop. What we discovered about designing a cooperation tool. Survives across sessions, accumulates over time.

Format: date, what happened, what was learned, what changed.

---

## 2026-04-01 — Initial Design Session

- Examined Claude Code source (0.2.8 + newer). Engineer identity is structural, not surface.
- System prompt is the soul — replacing it changes everything. Tested: `--system-prompt` flag works, identity comes through clean.
- Abra's three-level separation (spec / implementation / instance) applies to coop too: tool / instance / community data. Almost made the mistake of putting community data dirs in the tool repo.
- Trust (LinkedClaims) belongs at the core, not as an outer permission layer. "When evaluating claims, reach for trust measurement" — not afterthought.
- The tool should help build itself. First user = the development process.
- Need to define: community-space conventions, instance config, capture protocol, non-technical user interface.
