# Golda's Raw Direction — 2026-04-04

Exact words from the conversation, in order.

Note: the tool may be called **abra-ai** for now since the server already has something called 'abra' and we may not have time to rename it to lingo everywhere.

---

> ok i've been thinking about this, i think the coop tool needs to talk to people where they are - so it should have a sense that communications come in over channels like email, slack, other messaging apps; it may need to have a tool that scans news, social media, as well as doing research on the web, because we may need to respond to events of the day.  definitely CRM with tight acl/privacy per user, and the tool may be talking to more than one user, so it needs to have a sense who its talking to and what they can access, so a trust endpoint also.  it could be that people need certain trust levels to access some things.  it should have something it refers to for that.  its not real security since the tool does have more access, in some cases we need real security like a key or deletion but don't worry about that for now.
>
> The main thing is since this is a community tool it needs to like be persistent in the conversation with more than one person involved, and track who is talking.  Of ocurse when it comes to context limit it needs to do something, probably just make notes and chop off earlier context, not try to compact.  so it should interact with the conversation context checkpoints.
>
> its very important too it shuld have voice tools/interface
>
> many of thes things are tools not the core app, we should keep the thools separate from this repo, they will be cli tools, but the use of them may be different than claude code

---

> since this is not claude code exactly, its using insights from claude code but its our tool why can't it be persistent with sleeps?

---

> right so this critter could check interactions, if there are any then check right away for more; when there haven't been any for like 30 secs do a exponential backoff kind of thing til its only checking every 5 min.  since the check doesn't hit the api it should be no problem to wake up and check every 5 min, also it can have a webhook for incoming slack or email
>
> the webhook for email can have a gate
>
> the trust is a API endpoint to check for entities, it checks the uri of an entity and returns a dict of trust score aspects
>
> then we have a rules config for the instance that gates access to the abra bot itself and to specific resources
>
> every resource in it should have a name so it can be gated by name
>
> its also important for the admins who may not be technical, an admin can say "only let linda send emails' or whatever, but don't worry about that yet
>
> we're going to fork amebo and call it abra, so it has kene's commits but we change a lot
>
> he can also pull our stuff back into the original amebo
>
> the name-mapping piece we'll rename to 'lingo'

---

> now, leaning mostly on external APIs and cli tools, and starting with amebo (see ../amebo) which already has slack integration, but with learnings from claude an an instance confiurable knowledge base that can grow from research using the pgvector store

---

> it can have limited addresses it accespt from
>
> right so thats the first abstraction to design

---

[On the plan sequence:]

> ok we're probably going to do this work on a server where amebo is currently running
>
> we don't need to hardcode it to pgvector, other instances may want chromadb, we just do have pgvector available but everyone might not
>
> later we might open source part of this but not all necessarily

---

> also save all my exact words from this convo.  note we may call the tool abra-ai for now since the server already has one called 'abra' and i dn't know if we'll have time to rename it to lingo everywhere
