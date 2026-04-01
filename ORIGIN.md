# How Abra Coop Came To Be

Tool-level context. This explains why the design is what it is.

## The Discovery (April 1, 2026)

We examined Claude Code's source code — two versions, the early 0.2.8 and a much newer ~1900-file version. Claude Code has excellent infrastructure: layered context, team memory, hooks, skills, multi-agent orchestration.

But its entire identity assumes the user is a software engineer. The first sentence of the system prompt: "You are an interactive agent that helps users with software engineering tasks." Every behavioral instruction, every example, every tool — code, code, code. The engineering worldview isn't the skin, it's the skeleton. (Full analysis with line pointers: ~/4-1-claude-code-for-humans.md)

This can't be fixed by swapping a sentence. A cooperation tool needs a different soul.

## What Communities Need That Code Tools Don't Provide

Communities produce intentions, decisions, relationships, narratives, resources — not source code. Most of it is natural language. AST doesn't apply. The equivalent of "code intelligence" for communities is an open research question this tool helps answer.

The core problem: communities write vision documents that nobody reads. Values that gather dust. Abra Coop exists to make intentions alive — present in every interaction, tested against, evolved through use.

## Why Abra

Abra is a naming/binding system — local pet names with typed relationships, scoped to people and groups. This is exactly how communities actually talk: "the peter contract", "our accessibility goal." Abra makes that machine-readable without making it less human. It's the knowledge substrate.

## Why Trust Is Core

LinkedClaims provides trust that's contextual, directional, evidenced, composable, and decentralized. Without trust measurements, a cooperation tool can't evaluate claims, weigh perspectives, or respect the trust relationships a community has built. Trust isn't a permission layer — it's how communities actually make decisions.

## Why This Tool Builds Itself

The first user of Abra Coop is its own development process. The tool helps design itself — capturing decisions, evolving its own prompts, testing whether its facilitation actually works. This is both practical (we need it working) and philosophical (a cooperation tool should be cooperatively built).

## Learnings From Claude Code Architecture

Useful patterns to adapt (not copy):
- **Layered context**: community-level > team-level > personal instructions
- **Skills system**: shareable automation units with frontmatter metadata
- **Hooks**: lifecycle events that can trigger custom behavior
- **Memory taxonomy**: user, feedback, project, reference — maps to community context types
- **Tool delegation**: coop delegates to specialized tools rather than doing everything
- **Deferred loading**: don't load everything into context — discover tools on demand

Patterns to reject:
- Engineer identity baked into every instruction
- Code-centric behavioral rules
- Terse terminal output style as default
- Verification = "does it compile/pass tests"
