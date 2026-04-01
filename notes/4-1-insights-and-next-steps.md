# Insights and Next Steps from Claude Code Deep Dive

## Date: 2026-04-01

## Engineering Patterns Worth Adopting

1. **Tool interface as a protocol.** Every tool: schema (zod), async generator call(), prompt(), permissions check, validation. Clean — adding a tool doesn't touch core code. Adopt this for abra query, trust check, delegation tools.

2. **Async generators for streaming.** Tools yield progress + final result. Long-running operations show intermediate state. Good for any tool that takes time (web research, trust evaluation across multiple claims).

3. **Read-only concurrent, write serial.** Simple rule, avoids most races. Applies to community context too — multiple people reading bindings simultaneously is fine, writes need sequencing.

4. **Stale-write protection.** Track when files were last read, reject writes if modified since. Prevents overwriting someone else's changes. Essential for shared community documents.

5. **Contextual prompt assembly.** Core identity stays small. Situational prompts load when needed. We built this with the `prompts/` directory and `--with` flag. Could evolve into something more dynamic.

6. **Prompt caching with boundary markers.** Static content (identity, disposition) cached globally. Dynamic content (session-specific, community-specific) per-session. Worth designing for from the start — it's a significant cost factor.

7. **VCR test fixtures.** Record API responses, replay in tests. Dehydrate paths/timestamps for portability. Good pattern for testing any AI-powered tool without hitting the API every time.

8. **Sub-agents as isolated conversations.** Own message history, own tools, own log. Results serialized back as text. Fork pattern shares prompt cache across parallel tasks.

9. **MCP as extension mechanism.** External services connect via standard protocol. Tool schemas loaded dynamically, discovered via search. This is how abra, trust tools, CRM connectors should plug in.

10. **Everything event-logged.** Every API call, tool use, error. Analytics built into the product loop, not bolted on. For a cooperation tool: every decision captured, every trust query logged, every intention check recorded. The audit trail IS the trust.

## Security: What Claude Code Does and What We Should Do Differently

### What Claude Code restricts (and why)
- **Banned network commands** (curl, wget, nc, telnet, browsers) — forces all external access through controlled WebFetch/WebSearch tools. Prevents prompt injection via fetched URLs, but also prevents tool-chaining to external APIs outside the observable tool layer.
- **No interactive commands** — `< /dev/null` on every shell command, `GIT_EDITOR=true`, no `-i` flags. Prevents TTY sessions that bypass the tool layer.
- **cd restricted to children of original cwd** — can't navigate to /etc, ~, etc.
- **alias banned** — prevents redefining commands to bypass other restrictions.
- **Command injection detection** — tree-sitter AST analysis of bash commands. Backticks, $(), process substitution flagged.
- **Malware refusal in system prompt** — repeated twice. Hidden LS safety check ("do these files seem malicious?").
- **File write requires prior read** — prevents blind overwrites.

### The deeper pattern
All of this forces everything through the tool layer where it's observable, loggable, permission-checkable. The shell is deliberately hobbled to be a tool, not an escape hatch. The API knows it's Claude Code calling (x-app header, beta flag) so server-side behavior can differ.

### What Abra Coop should do differently
- **We DO want web access** — research is core. But through a defined tool, not raw curl. The tool boundary is where observability lives.
- **We DON'T need cd restrictions** — community data could be anywhere. But we might want awareness of what space we're in (which community's context).
- **We DON'T need malware paranoia** — we're facilitating conversations and decisions, not writing executable code. But we DO need PII awareness (abra already has "no PII in the binding store" rule).
- **We DO want tool-boundary enforcement** — when delegating to Claude Code, it should go through a defined interface, not raw `claude -p` via bash. Otherwise no tracking of what got delegated, what it cost, what permissions it used.
- **We DO want the read-before-write pattern** — stale-write protection prevents real data loss regardless of domain.
- **We probably DON'T want banned commands** — but we may want a lighter version: awareness of what's being run and why, not blanket bans. Trust the user more, but log everything.
- **Trust evaluation IS our security model** — Claude Code's security is permissions-based (can this tool run?). Ours should be trust-based (should we believe this claim? who attested to it?). LinkedClaims is the security layer, not command bans.

## Community Tool Patterns (from examining Claude Code's newer features)

### What's directly useful
- **Team memory** — shared context synced across an org. Map to community shared context.
- **Skills as markdown files** — drop a file, get a capability. Low-code extensibility for communities.
- **Hooks on lifecycle events** — trigger custom behavior on tool use, session start/end, etc. Could trigger intention checks, trust evaluation.
- **Task management with ownership and blocking** — works for any kind of collaborative work.
- **Deferred tool loading** — keyword search over available tools. Essential when you have many integrations (Slack, CRM, task managers, trust services).

### What needs rethinking
- **Verification agent** — Claude Code's verifies "does code work?" We need "does this align with intentions?" and "is this claim trustworthy?"
- **Plan mode** — Claude Code's is "explore codebase, design implementation." Ours is "explore context, understand landscape, propose approach, get alignment."
- **The output style system** — Claude Code has output style configs. We need something similar but for facilitation styles (direct, socratic, summarizing, bridging perspectives).

## Governance Principles (to integrate — see separate handoff doc)

Key concepts from Golda's work that should inform the tool's disposition:
- **Transparency not permission** — visibility into decisions can be as powerful as voting, without the delay
- **Earned governance** — governance allocated through peer-reviewed contributions, not purchased or granted
- **Ranked choice for delegation** — not simple majority, avoids spoiler effects
- **Work-weighted voting** — when votes happen, weighted by verified contribution
- **Opportunity to object** — safety valve, not veto gate
- **Anti-capture design** — governance structures that resist hostile takeover and mission drift

The tool should NOT default to "let's take a vote" as the resolution mechanism. Transparency and objection-based processes first, formal voting when actually needed, and then ranked choice + work-weighted.

Full details in `notes/4-1-governance-principles-handoff.md`.

## Next Steps

### Immediate (next session)
1. Deep dive on governance principles — get all of Golda's writings on earned governance, transparency-not-permission, ranked choice delegation into context
2. Update system prompt and add a governance prompt in `prompts/`
3. Talk to Kene about Amebo merge (see `notes/4-1-amebo-merge-possibly.md`)

### Near term
4. Define the tool interface protocol (inspired by Claude Code's but for cooperation tools)
5. Build abra query tool (MCP server or direct integration)
6. Build trust evaluation tool (LinkedClaims SDK integration)
7. Build Claude Code delegation tool (defined interface, not raw bash)
8. Test with real community conversations

### Research needed
9. What's the equivalent of AST/LSP for community artifacts? Structured analysis for non-code knowledge.
10. What existing work exists on computational deliberation, structured argumentation, collective intelligence tools?
11. What facilitation patterns can be encoded as prompts/skills?
12. How do non-technical users interact? Web UI (Amebo's Next.js), Slack (/ask pattern), something else?

## Files Created Today

- `~/abra-ai/` — the tool repo (github.com/Cooperation-org/abra-ai)
- `~/4-1-claude-code-for-humans.md` — full analysis of Claude Code's engineer bias with line pointers
- `~/abra-repo/` — cloned abra spec repo
- `~/coop-projects/` — cloned projects repo (has earnedgov materials)
- `~/amebo/` — cloned Amebo repo
- `~/claude-code-original-src/` — newer Claude Code source (was already present)
- `~/claude-code-sourcemap/` — older 0.2.8 Claude Code source (was already present)
