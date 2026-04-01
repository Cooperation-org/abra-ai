# Governance Principles Deep Dive — Handoff Notes

## Context

This is a handoff from a long session on 2026-04-01 where we:
1. Examined Claude Code source (0.2.8 + newer ~1900 files) in depth
2. Identified its engineer-brain problem (every prompt assumes software engineering)
3. Created Abra Coop (~/abra-ai, github.com/Cooperation-org/abra-ai) — a cooperation facilitator tool
4. Got it running (launcher works, identity comes through clean)
5. Need to go deeper on the governance philosophy that should be baked into the tool's thinking

## The Three Pillars (from LinkedTrust case study)

These are battle-tested, 6+ years, not theoretical:

### 1. Transparency Not Permission
"With enough eyes, all bugs are shallow" applies to more than code. Visibility into resource allocation and decision making can be AS POWERFUL as voting, WITHOUT the delay.

Key insight: vote-gating is problematic — it slows everything down and creates bottlenecks. Transparency lets leaders drive ahead while maintaining team trust. People can object if they see something wrong, but they don't have to approve every action.

### 2. Earned Governance
Governance allocated to those doing valuable work, based on peer-reviewed tasks. No overhead of separate governance processes — it emerges from the work itself. Task → Peer Review → Completed → Earnings Recorded → Attestation (LinkedClaims) → Equity/Governance Updated.

Work-weighted voting: when votes DO happen, weighted by contribution. Not one-person-one-vote, not one-dollar-one-vote — one-contribution-one-vote.

### 3. Democratic Leadership
CEO elected by ranked choice, work-weighted vote. Profits spent by members submitting proposals to rotating steering committee. "Opportunity to object" baked into process.

## Key Concepts the Tool Should Understand

- **Ranked choice voting for delegation** — not simple majority, avoids spoiler effects and strategic voting
- **Work-weighted voting** — voting power proportional to verified contributions, not headcount or capital
- **Transparency not permission** — the default is visibility and action, not asking for approval. Objection-based, not permission-based.
- **Earned governance vs purchased governance** — anti-capture design. You can't buy your way to power.
- **Opportunity to object** — everyone has the right, but it's not a veto gate. It's a safety valve.
- **Peer review as trust-building** — review isn't gatekeeping, it's how trust gets built and evidenced (via LinkedClaims attestations)
- **Portable credentials** — contributions create verifiable claims that follow the person, not locked to one org
- **Anti-capture design** — governance structures that resist hostile takeover and mission drift ("corporate body snatching")

## What This Means for Abra Coop

The tool shouldn't hard-bake specific governance mechanisms, but its disposition should reflect these principles:
- Default to transparency over permission when suggesting how groups should work
- Understand that voting isn't always the answer — sometimes visibility is better
- When voting IS needed, understand ranked choice and work-weighted as better defaults than simple majority
- Trust is earned through contributions, not granted by position
- Governance should emerge from the work, not be a separate overhead layer

## Source Materials (read these for the deep dive)

Already cloned:
- `~/coop-projects/Active/earnedgov/2-27-program.txt` — Earned Equity Accelerator proposal (source of truth)
- `~/coop-projects/Active/earnedgov/` — slide deck PNGs showing the three pillars
- `~/coop-projects/Ideas/toolkit/initial-thoughts.md` — Cooperation Toolkit vision (core flow, pluggable components, principles)
- `~/coop-projects/Ideas/toolkit/README.md` — toolkit overview
- `~/coop-projects/plans/playbook.md` — LinkedTrust Growth Playbook (the three pillars articulated clearly)

Need to find/collect:
- Golda's writings on ranked choice voting for delegation specifically
- More on "transparency not permission" as a principle
- The "frictional currencies" essay with Leanne Usher (in progress)
- The Rebooting Web of Trust XI publication (co-authored with Georgetown & MIT)
- Anything on sortition (mentioned in the playbook)

Also read:
- `~/abra-repo/concept-notes.md` — the abra vision (hot tags, pet names, reducing cognitive load)
- `~/abra-repo/arch_notes.md` — binding format, catcode space, relationship types
- `~/abra-ai/BIRTH.md` — removed from repo but concepts now in ORIGIN.md
- `~/abra-ai/ORIGIN.md` — why abra coop exists, what we learned from Claude Code
- `~/abra-ai/DESIGN.md` — current architecture thinking
- `~/abra-ai/system-prompt.md` — the tool's current identity
- `~/4-1-claude-code-for-humans.md` — full analysis of Claude Code's engineer bias

## What the Deep Dive Should Produce

1. Updated system-prompt.md that reflects governance principles (not hard-coded mechanisms, but disposition)
2. A governance-concepts.md in the tool repo — reference doc the tool can read about how cooperation actually works
3. Thoughts on which "tools" the tool needs that are governance-related (not consensus/voting as I suggested earlier — that was too simplistic. More like: transparency dashboards, contribution tracking, delegation flows, objection handling)
4. How abra bindings map to governance concepts (earned equity as RELATED bindings? contribution history as ABOUT? trust attestations via LinkedClaims?)

## Important Correction from User

The initial tool list I proposed included things like "Consensus & Voting Tool" and "Conflict Resolution Tool" — these were too conventional. The user's approach is:
- NOT vote-gating (problematic, slow)
- NOT simple majority voting
- Transparency as the primary mechanism, not voting
- When voting happens: ranked choice, work-weighted
- Delegation via ranked choice, not hierarchy
- Objection-based, not permission-based

The tool's instincts should reflect this. Don't suggest "let's take a vote" as the default resolution mechanism.
