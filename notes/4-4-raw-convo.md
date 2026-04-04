
❯ ok i've been thinking about this, i think the coop tool needs to talk to
people where they are - so it should have a sense that communications come in
over channels like email, slack, other messaging apps; it may need to have a
tool that scans news, social media, as well as doing research on the web,
because we may need to respond to events of the day.  definitely CRM with tight
 acl/privacy per user, and the tool may be talking to more than one user, so it
 needs to have a sense who its talking to and what they can access, so a trust
endpoint also.  it could be that people need certain trust levels to access
some things.  it should have something it refers to for that.  its not real
security since the tool does have more access, in some cases we need real
security like a key or deletion but don't worry about that for now.

The main thing is since this is a community tool it needs to like be persistent
 in the conversation with more than one person involved, and track who is
talking.  Of ocurse when it comes to context limit it needs to do something,
probably just make notes and chop off earlier context, not try to compact.  so
it should interact with the conversation context checkpoints.

its very important too it shuld have voice tools/interface

many of thes things are tools not the core app, we should keep the thools
separate from this repo, they will be cli tools, but the use of them may be
different than claude code

⏺ Good thinking. Let me make sure I'm tracking the key distinctions:

  Core app (this repo) — identity, disposition, multi-party conversation
  management, context checkpointing, trust-aware access control (soft, not
  crypto). The thing that knows who it's talking to, what they can see, and how
  to facilitate.

  Separate tool repos — the actual integrations:
  - Channel adapters (email, Slack, messaging apps)
  - News/social media scanner
  - Web research
  - CRM (with per-user ACL/privacy)
  - Trust endpoint (who has what trust level, what that unlocks)
  - Voice interface
  - Context checkpoint manager (notes + truncation, not compaction)

  The multi-party persistence piece is the big architectural difference from
  Claude Code — CC is one human, one session. This is multiple humans, ongoing,
  with the tool maintaining continuity across all of them and knowing who said
  what and who can see what.

  And the context strategy is intentionally simple: make notes at checkpoints,
  drop old context. No lossy summarization.

  The voice piece — that's for actual spoken interaction, not just TTS of text
  output? That changes the interaction model quite a bit (turn-taking,
  interruption, ambient listening vs. explicit invocation).

  How do you want to capture this? Update DESIGN.md, or a new note first while
  it's still forming?

❯ well i need critical thought i need this to work, then we can add the harder
bits, don't worry about the voice for now

critical thought about what could be achieved today

⏺ Thinking critically about what's real today:

  What already works:
  - coop launcher with custom system prompt — done, in this repo
  - MCP protocol for tool connections — Claude Code already supports it
  - Slack has existing MCP servers (multiple open source ones)
  - Linear MCP — already connected in your setup

  The simplest multi-party path today: Slack.
  Slack already solves user identity, persistence, threading, and multi-party. A
   Slack MCP server lets coop receive messages tagged with who sent them. The
  system prompt instructs it to track speakers. That's multi-party conversation
  without building any infrastructure.

  Trust/ACL today: a config file. JSON mapping users to trust levels, trust
  levels to access scopes. A tiny CLI tool that coop calls to check. No
  database, no service.

  CRM today: structured files (YAML/JSON per contact) behind a simple MCP
  server. Per-user ACL enforced by the trust tool before returning data.

  Context checkpoints today: a tool that writes conversation notes to a file
  when invoked. System prompt says "before context gets long, use the checkpoint
   tool." Not automatic, but functional.

  What's NOT achievable today:
  - True persistent daemon (Claude Code sessions end — someone has to restart
  it)
  - Push-based channel monitoring (Slack MCP is pull, not a live listener)
  - Multiple simultaneous channels in one session

  The honest gap is the persistence/daemon piece. Everything else can be
  duct-taped together with MCP servers and config files. The Slack bot approach
  gets you the furthest fastest, but it's still request-response, not a
  persistent presence.

  What's the actual use case you'd want to demo first?

✻ Churned for 35s

❯ no i dn't want you to drive i only wanted feedback.  i'm driving.  dont try
to drive.

⏺ Understood.

❯ since this is not claude code exactly, its using insights from claude code
but its our tool why can't it be persistent with sleeps?

⏺ You're right, no reason it can't. It's our tool — we can run a long-lived
  process that sleeps between events. Claude Code chose not to because it's an
  interactive CLI for a single engineer in a terminal. Different use case,
  different constraints.

❯ right so this critter could check interactions, if there are any then check
right away for more; when there haven't been any for like 30 secs do a
exponential backoff kind of thing til its only checking every 5 min.  since the
check doesn't hit the api it should be no problem to wake up and check every 5
min, also it can have a webhook for incoming slack or email

the webhook for email can have a gate

the trust is a API endpoint to check for entities, it checks the uri of an
entity and returns a dict of trust score aspects

then we have a rules config for the instance that gates access to the abra bot
itself and to specific resources

every resource in it should have a name so it can be gated by name

its also important for the admins who may not be technical, an admin can say
"only let linda send emails' or whatever, but don't worry about that yet

we're going to fork amebo and call it abra, so it has kene's commits but we
change a lot

he can also pull our stuff back into the original amebo

the name-mapping piece we'll rename to 'lingo'

⏺ Got it, all noted.

❯ now, leaning mostly on external APIs and cli tools, and starting with amebo
(see ../amebo) which already has slack integration, but with learnings from
claude an an instance confiurable knowledge base that can grow from research
using the pgvector store

take a look at amebo and see if we need to decouple the instance decisions and
storage from the code base, and what are the highest priority claude code
learnings we could apply

  Read 11 files, listed 5 directories (ctrl+o to expand)

⏺ OK here's what I see:

  Decoupling needed — instance vs codebase:

  The big one: instance config is baked into the code. Database URL, ChromaDB path (./chromadb_data), Slack tokens,
  API keys — all in .env and hardcoded defaults. The schema SQL lives in the repo. For abra where instances are
  independent deployments for different communities, you'd want:
  - Instance config directory (outside repo) that the app loads from
  - Schema migrations as versioned files, not a single SQL dump
  - ChromaDB path, DB connection, all credentials — instance config, not app config
  - bot_config table defaults shouldn't be in schema — they're instance decisions

  The workspace_id isolation pattern is good but it's the wrong boundary for abra. Amebo's multi-tenancy is "one
  SaaS, many orgs." Abra's is "one instance per community." The organizations → org_workspaces → workspaces
  hierarchy is SaaS scaffolding you probably don't need.

  Highest priority Claude Code learnings to apply:

  1. Tool interface as protocol. Amebo has services (qa_service, query_service, slack_listener) but they're tightly
  coupled classes instantiating each other directly. No schema, no permissions check, no streaming. Making these
  proper tool interfaces means they can be swapped, gated by trust, and called from outside the process.
  2. Contextual prompt assembly. qa_service.py has the Claude interaction but the prompt is built inline. The system
   prompt / identity should be separate and cacheable, with situational context loaded per-query. You already have
  this pattern in abra-ai's prompts/ directory.
  3. Read-only concurrent, write serial. The DatabaseConnection uses a connection pool but there's no read/write
  discipline. The conversation manager does direct writes with no conflict protection.
  4. Stale-write protection. conversation_manager.py does INSERT-only which is fine, but anything that updates
  shared state has no check for concurrent modification.
  5. Event logging. There's an audit_logs table but it's in the auth schema only. The QA interactions, trust checks,
   and tool delegations aren't logged. For a cooperation tool the audit trail is core, not optional.

  What's actually good in amebo and worth keeping:
  - Slack Socket Mode listener with multi-workspace support — real-time event handling is already there
  - ChromaDB collection-per-workspace isolation pattern (just needs the boundary redefined)
  - Conversation history in postgres — the bones of multi-party tracking
  - The hybrid PostgreSQL (metadata) + vector store (content) architecture
  - APScheduler for background tasks — could drive your sleep/backoff polling loop

  ChromaDB vs pgvector: amebo uses ChromaDB. You said pgvector. That's a simplification win — one database instead
  of two, and pgvector is production-grade. Removes the chromadb_data directory, the ChromaDB client, and the
  dual-store coordination.

✻ Cogitated for 58s

❯ ok good.  i also want to prioritize ability to add plugins/tools starting with email, i want each instance to hae
 an email address like on gmail and for it to check its mail in its primary box, quickly discord spam and act on
other messages

⏺ That fits cleanly with the poll loop you already described — email check is just another thing it does on wake.
  Gmail API over IMAP — OAuth once, then poll with messages.list using a history ID so you only get new stuff. Spam
  filtering is mostly already done by Gmail itself (spam folder), but you'd want a second pass for the stuff Gmail
  lets through — probably just a quick Claude call with the subject/sender/first few lines, cheaper than any custom
  classifier and better at catching context-specific junk.

  The plugin interface question is the real one here. Amebo has no plugin boundary — slack_listener.py imports
  ChromaDBClient and BackfillService directly. Adding email the same way means another tightly coupled service. If
  you define the tool protocol first (schema, permissions check, async call), email becomes the first real test of
  it rather than another hardwired service.

  Each plugin needs to answer the same questions: what channels does it watch, how does it identify the sender, what
   actions can it take (read, reply, forward, delete), and what trust level is needed for each action. That's the
  same shape whether it's email, Slack, or Discord later.

❯ it can have limited addresses it accespt from

right so thats the first abstraction to design

you may suggest a plan, like
1) fork amebo, merge this repo into it
2) clean and add abstractions
    plugins and permissions (api checking uris)
    people - conversation participants
    requests, we may need to prioritize and go in some order, we'll need to spin off subagents same as claude code
3) apply claude code learnings and patterns
4) test and iterate

⏺ That's a solid sequence. One adjustment I'd suggest:

  1. Fork amebo, merge this repo into it
  2. Define the plugin protocol and people/identity model first, before cleaning. If you clean amebo's existing code
   before knowing the target abstractions, you'll clean it twice. The protocol shape determines what stays, what
  gets ripped out, and what gets reshaped. Specifically:
    - Plugin interface (channel in, channel out, sender identity, action permissions)
    - People model (who is this person across channels — Slack user, email address, same human)
    - Request queue (prioritized, with trust-gated actions)
    - Trust check as an API call to an external endpoint, not inline logic
  3. Then clean amebo to fit those abstractions — rip out the SaaS multi-tenancy, replace ChromaDB with pgvector,
  extract the Slack integration into a plugin, separate instance config from code
  4. Apply Claude Code patterns — tool schema/validation, contextual prompt assembly, stale-write protection, event
  logging, sub-agent spawning for concurrent work
  5. Build email plugin as second plugin (Slack already exists as first), validating the protocol actually works for
   two different channel types
  6. Test and iterate

  The reason to swap 2 and 3: amebo's Slack code works. Refactoring it before you know what shape a plugin is wastes
   effort. Design the plugin interface, then make Slack conform to it, then email proves it generalizes.

❯ ok we're probably going to do this work on a server where amebo is currently running

we don't need to hardcode it to pgvector, other instances may want chromadb, we just do have pgvector available but
 everyone might not

later we might open source part of this but not all necessarily

oets write all your current context into a file and we'll commit and move the work to the server.  it also has a
claude system prompt that knows more about our linkedclaims which we'll use for trust

  Read 4 files (ctrl+o to expand)

⏺ Write(notes/04-04-implementation-plan.md)
  ⎿  Wrote 183 lines to notes/04-04-implementation-plan.md
      1 # Abra Implementation Plan — Context Handoff
      2
      3 ## Date: 2026-04-04
      4
      5 ## Decision: Fork Amebo, Build Abra
      6
      7 Fork Amebo (github.com/Cooperation-org/amebo) and rename to Abra. Preserves Kene's commit history. He
         can pull changes back into original Amebo. Merge this repo (abra-ai) into the fork — system prompt,
        design docs, prompts, governance notes all come along.
      8
      9 The name-mapping piece (abra bindings) will be renamed to **lingo**.
     … +173 lines (ctrl+o to expand)

⏺ Written to notes/04-04-implementation-plan.md. Has everything from this session — what Amebo has, what changes,
  the plugin protocol as first priority, email plugin specifics, people model, trust endpoint, persistence/polling
  model, the sequence, Claude Code patterns to apply, and all the decisions made today.

✻ Sautéed for 1m 27s

❯ also save all my exact words from this convo.  note we may call the tool abra-ai for now since the server already
 has one called 'abra' and i dn't know if we'll have time to rename it to lingo everywhere

⏺ Write(notes/04-04-golda-raw-direction.md)

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
 Create file
 notes/04-04-golda-raw-direction.md
╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌
   1 # Golda's Raw Direction — 2026-04-04
   2
   3 Exact words from the conversation, in order.
   4
   5 Note: the tool may be called **abra-ai** for now since the server already has something called 'abra' and we
      may not have time to rename it to lingo everywhere.
   6
   7 ---
   8
   9 > ok i've been thinking about this, i think the coop tool needs to talk to people where they are - so it sho
     uld have a sense that communications come in over channels like email, slack, other messaging apps; it may n
     eed to have a tool that scans news, social media, as well as doing research on the web, because we may need
     to respond to events of the day.  definitely CRM with tight acl/privacy per user, and the tool may be talkin
     g to more than one user, so it needs to have a sense who its talking to and what they can access, so a trust
      endpoint also.  it could be that people need certain trust levels to access some things.  it should have so
     mething it refers to for that.  its not real security since the tool does have more access, in some cases we
      need real security like a key or deletion but don't worry about that for now.
  10 >
  11 > The main thing is since this is a community tool it needs to like be persistent in the conversation with m
     ore than one person involved, and track who is talking.  Of ocurse when it comes to context limit it needs t
     o do something, probably just make notes and chop off earlier context, not try to compact.  so it should int
     eract with the conversation context checkpoints.
  12 >
  13 > its very important too it shuld have voice tools/interface
  14 >
  15 > many of thes things are tools not the core app, we should keep the thools separate from this repo, they wi
     ll be cli tools, but the use of them may be different than claude code
  16
  17 ---
  18
  19 > since this is not claude code exactly, its using insights from claude code but its our tool why can't it b
     e persistent with sleeps?
  20
  21 ---
  22
  23 > right so this critter could check interactions, if there are any then check right away for more; when ther
     e haven't been any for like 30 secs do a exponential backoff kind of thing til its only checking every 5 min
     .  since the check doesn't hit the api it should be no problem to wake up and check every 5 min, also it can
      have a webhook for incoming slack or email
  24 >
  25 > the webhook for email can have a gate
  26 >
  27 > the trust is a API endpoint to check for entities, it checks the uri of an entity and returns a dict of tr
     ust score aspects
  28 >
  29 > then we have a rules config for the instance that gates access to the abra bot itself and to specific reso
     urces
  30 >
  31 > every resource in it should have a name so it can be gated by name
  32 >
  33 > its also important for the admins who may not be technical, an admin can say "only let linda send emails'
     or whatever, but don't worry about that yet
  34 >
  35 > we're going to fork amebo and call it abra, so it has kene's commits but we change a lot
  36 >
  37 > he can also pull our stuff back into the original amebo
  38 >
  39 > the name-mapping piece we'll rename to 'lingo'
  40
  41 ---
  42
  43 > now, leaning mostly on external APIs and cli tools, and starting with amebo (see ../amebo) which already h
     as slack integration, but with learnings from claude an an instance confiurable knowledge base that can grow
      from research using the pgvector store
  44
  45 ---
  46
  47 > it can have limited addresses it accespt from
  48 >
  49 > right so thats the first abstraction to design
  50
  51 ---
  52
  53 [On the plan sequence:]
  54
  55 > ok we're probably going to do this work on a server where amebo is currently running
  56 >
  57 > we don't need to hardcode it to pgvector, other instances may want chromadb, we just do have pgvector avai
     lable but everyone might not
  58 >
  59 > later we might open source part of this but not all necessarily
  60
  61 ---
  62
  63 > also save all my exact words from this convo.  note we may call the tool abra-ai for now since the server
     already has one called 'abra' and i dn't know if we'll have time to rename it to lingo everywhere
