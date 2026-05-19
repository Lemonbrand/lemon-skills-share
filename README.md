# Three Skills for a Work Session

These three files came out of a conversation about how a work session is shaped — and how the agent should know where it is in that shape.

The principle is older than agents: any unit of work follows **get context → do work → close out**. Skip the bookends and you spend the middle re-deriving state. Do the bookends well and the middle gets shorter.

## The three files

### `start-session.md` — the opening

Runs at the start of every working session. Verifies the knowledge graph is reachable, pulls today's calendar, scans unread inbound, lists active tasks, and pre-resolves entity context for everyone on today's schedule. Reports a snapshot and **waits** — the user sets priorities, not the agent.

The key move: a clean handoff from "I just sat down" to "I know what today's shape is" before any work begins.

### `get-context.md` — the middle's first move

The most under-used skill in any agent toolbox: pulling enough context before answering. Takes a target (person, company, project, decision), resolves it, fetches the snapshot, observation history, linked tasks, recent emails / DMs / call transcripts, and walks **one** edge of the relationship graph.

Synthesizes a brief. Does not dump data. The brief answers four questions in 30 seconds: who they are, last touch, open commitments, suggested next move.

Three depth tiers (`snapshot`, `standard`, `full`) so other skills can call it cheaply during loops or deeply before a high-stakes draft.

### `close-session.md` — the closing

The hardest of the three to get right. Writes a `session_close` record to the graph, logs every outbound interaction as a first-class event, writes individual `task` entities for follow-ups, and writes a redundant project-level task ledger as a human-readable safety net.

Two non-obvious bits:
- **Budgeted bullets.** The closing entity renders into the next session's opening. Limit: 8 bullets, ≤ 240 chars each. Over-budget closes silently disappear. The skill enforces the budget before writing.
- **Outbounds are logged twice.** Once as a contact observation (rollup), once as a standalone `interaction` (event). Inbound surfaces are biased toward inbound; outbounds must be written explicitly or they're lost.

## How they connect

```
              ┌─────────────────┐
              │  start-session  │
              │  (the opening)  │
              └────────┬────────┘
                       │  loops over today's attendees,
                       │  calls get-context for each
                       ▼
              ┌─────────────────┐
              │   get-context   │◀──────┐
              │  (any topic,    │       │ called repeatedly
              │   any time)     │       │ during the work
              └────────┬────────┘       │
                       │                │
                       │   the actual work happens here
                       │                │
                       ▼                │
              ┌─────────────────┐       │
              │  close-session  │───────┘
              │  (the closing)  │  consumes entity IDs
              └─────────────────┘  returned by get-context
```

## What's intentionally not here

These are skill *shapes*, not skill *implementations*. The CLI calls (`kb`, `cal`, `mail`) are placeholders for whatever knowledge graph / calendar / mail tooling you wire in. The structure — what runs, in what order, what gets persisted — is the durable part.

A few notes on adapting them:

- **The knowledge graph.** The skills assume a graph with `contact` / `company` / `project` / `task` / `interaction` / `observation` / `decision` / `session_close` entity types. Any equivalent shape works — adjust the entity-type names to match your schema.
- **Tenant / user ID.** Every call is scoped by `${KB_USER_ID}`. Set this in your shell env. If you're multi-tenant, this is also your isolation boundary.
- **The bullet budget.** The numbers (≤ 80 char qualifier, ≤ 8 bullets, ≤ 240 chars each) are tuned for a 5-line render block. If your render target is different, retune. The discipline is the point, not the exact numbers.
- **Idempotency keys.** Every entity has a deterministic name pattern (`Observation: X YYYY-MM-DD`, `interaction-<slug>-<surface>-<message_id>`). This lets you re-run the close safely if it fails halfway.

## Why ship these as `.md` files

Two reasons.

One, the agent reads them at invocation time. Their purpose is to load deliberate process into the agent's context window, not to be executable. Procedures-as-prompts.

Two, they should be readable by a human. The skill file *is* the SOP. If you can't read the file and understand what the skill does, the skill is wrong. Length is not the failure mode — opacity is.

## Provenance

Extracted on 2026-05-19 from a working agent setup. Names, IDs, hosts, and surfaces have been scrubbed; the structure and copy are intact. Use them, adapt them, throw out what doesn't fit.
