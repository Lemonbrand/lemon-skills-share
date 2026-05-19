---
name: close-session
description: Archive the working session at the end. Writes a session_close record to the knowledge graph, logs every outbound interaction as a first-class event, writes task entities for follow-ups, and writes a project-level task ledger as a human-readable safety net. Run at the end of every working session to ensure nothing is lost.
location: project
allowed-tools: Read, Write, Edit, Bash
metadata:
  author: template
  version: "2.1"
---

# Close Session

The graph is the primary store. Local markdown is the git backup. Entity files in any "people/" mirror are pre-migration snapshots — never edit them; write graph observations instead.

A close is **not complete** until it has: a `session_close`, observations for everyone touched, individual `task` entities for follow-ups, and a project-level task ledger.

## When to Use

- At the end of any working session ("wrap up", "done for now", "close it out", "log this")
- When updating the "last session" block in your top-level agent instructions
- Proactively: if you're about to overwrite that block, run this first

---

## Configuration

| Env | Purpose |
|---|---|
| `KB_HOST` | Knowledge-base API endpoint |
| `KB_USER_ID` | Tenant / user id |
| `REPO_ROOT` | Path to your agent repo (for archive writes) |

---

## Process

### Step 1: Archive to local sessions/ (git backup)

1. Read your top-level agent instructions (the file that holds your "last session" block).
2. Extract content between the AUTOGEN markers.
3. Parse the date from the `**Date:**` line.
4. Append to `${REPO_ROOT}/sessions/YYYY-MM-DD-session.md` (with `---` separator if file exists).

This file is a git backup only. The graph is the live source.

### Step 2: Pull daily activity context

Run whatever daily-compiler you have:

```bash
python3 "${REPO_ROOT}/scripts/daily-compiler.py" --date "$(date +%Y-%m-%d)" --dry-run
```

Note commits, calls, open items. Include a brief summary at the bottom of the session archive.

### Step 3: Store the `session_close` entity ← PRIMARY

This is what nightly cron reads to regenerate your agent instructions. Write it first.

**Budget — hard limits, enforced by the renderer:**

- `len(qualifier) <= 80` (one short phrase, e.g. `"Tuesday — Client X briefing + Project Y slides"`)
- `len(bullets) <= 8` (max 8 entries in the array)
- `all(len(b) <= 240 and "\n" not in b for b in bullets)` (one tweet-length line per bullet, no embedded newlines)
- Total rendered last-session block ≤ ~2500 chars

Mirror these limits in your renderer. The renderer's validator is the load-bearing gate: if an entity busts the budget, the renderer **refuses to render it** and falls back to `sessions/*.md`. So an over-budget `session_close` silently disappears from your agent instructions. Don't ship one.

Why: each bullet + the qualifier render as text inside the AUTOGEN block of your agent instructions, which loads into every future session. One over-budget bullet (or one bloated qualifier) is the difference between a 5-line block and a 100-line block.

If detail won't fit, **it belongs in `sessions/YYYY-MM-DD-session.md`** (the archive — Step 1 already wrote it there). Reference it from the bullet: `"**Block 3 — Vercel cutover.** 17min outage, clean rollback. Details in archive."`

Before writing:

```python
assert len(qualifier) <= 80, f"qualifier is {len(qualifier)} chars > 80 — shrink the day-of-week summary"
assert len(bullets) <= 8, f"too many bullets: {len(bullets)} > 8 — compress before writing"
for i, b in enumerate(bullets):
    assert len(b) <= 240 and "\n" not in b, f"bullet {i} busts budget: move detail to sessions/ archive: {b[:80]}..."
```

If any assertion fires, **do not write** — report which field busted the budget and trim before retrying.

```bash
kb store --user-id "${KB_USER_ID}" --entities '[
  {
    "entity_type": "session_close",
    "name": "Session YYYY-MM-DD: qualifier",
    "canonical_name": "Session YYYY-MM-DD: qualifier",
    "date": "YYYY-MM-DD",
    "qualifier": "day of week — brief phrase",
    "bullets": [
      "**Bold lead.** Concrete details: file names, outcomes, counts.",
      "**Bold lead.** Concrete details."
    ],
    "previous_session_file": "sessions/YYYY-MM-DD-session.md"
  }
]'
```

Rules for bullets:
- Start each with a bold summary phrase
- Include concrete details: file paths, names, numbers
- 4–8 bullets per session, **≤ 240 chars each**
- Last bullet: open items (what's unfinished) — or a pointer to the archive section listing them
- Structured attributes are fine for non-rendered detail: `project`, `summary`, `files_touched`, `decisions`, `open_items`

### Step 4: Write entity observations to the graph ← NOT to markdown files

For anyone discussed in this session, write an observation. Never edit the mirror entity files directly.

```bash
kb store --user-id "${KB_USER_ID}" --entities '[
  {
    "entity_type": "observation",
    "name": "Observation: <Contact Name> YYYY-MM-DD",
    "canonical_name": "Observation: <Contact Name> YYYY-MM-DD",
    "target_slug": "<contact-slug>",
    "summary": "<what happened, what changed>",
    "surface": "email",
    "direction": "outbound",
    "occurred_at": "YYYY-MM-DDT00:00:00Z"
  }
]'
```

If the entity doesn't exist yet, create the contact entity first, then add the observation.

### Step 4b: Log every outbound as a standalone `interaction` ← in ADDITION to Step 4

The contact observation is rollup data. The `interaction` row is the first-class event the brief / digest / response-detection pipelines read. Surface ingests are inbound-biased — outbounds either lag or never arrive. At close, write one row per outbound the user sent this session.

**Find the real `message_id` first** (never invent it — it's what future auto-ingests dedupe against):

- **Email:** `mail messages list --query "in:sent to:<recipient> newer_than:1d" --max 3`
- **DMs:** check the appropriate conversation snapshot directory for the latest outbound and pull its `msg_id`
- **No auto-id available:** synthesize a manual key (e.g. `manual-<surface>-<slug>-<unix-ts>`) so future dedupe is at least possible

Then write:

```json
{
  "entity_type": "interaction",
  "name": "<Surface> outbound to <Target> — <Topic>",
  "canonical_name": "<Surface> outbound to <Target> — <Topic>",
  "target_slug": "<contact-slug>",
  "target_name": "<Display Name>",
  "surface": "email | linkedin | whatsapp | <other>",
  "channel": "<source channel>",
  "direction": "outbound",
  "subject": "<subject if email>",
  "thread_id": "<thread_id / conv_id>",
  "message_id": "<actual surface message_id>",
  "occurred_at": "<ISO UTC timestamp from the actual send>",
  "summary": "<what was sent, key terms, ask, what changes next>"
}
```

Idempotency key: `interaction-<slug>-<surface>-<message_id>`. Safe to reuse if you re-run the close.

This is **in addition to** the contact observation in Step 4, not instead of it.

### Step 5: Write `task` entities for follow-ups ← NOT to a freeform notes file

For follow-ups or pending actions identified in this session:

```bash
kb store --user-id "${KB_USER_ID}" --entities '[
  {
    "entity_type": "task",
    "name": "Brief description of the task",
    "canonical_name": "Brief description of the task",
    "entity_slug": "<related-person-or-project-slug>",
    "due_by": "YYYY-MM-DD",
    "priority": "high",
    "status": "active",
    "notes": "Context for why this matters."
  }
]'
```

Priority: `high` / `medium` / `low`. Status: `active` (open work) unless your schema has a more specific enum.

A freeform suggestions file remains a fallback for structural/meta ideas that don't map to a person entity. People-related tasks always go to the graph.

### Step 5b: Project task ledger ← SAFETY NET

For every active project touched in the session, write one human-readable task ledger observation. This is mandatory because task snapshots can be thin and search can be noisy.

```bash
kb store --user-id "${KB_USER_ID}" --entities '[
  {
    "entity_type": "observation",
    "name": "Project remaining task ledger YYYY-MM-DD",
    "canonical_name": "Project remaining task ledger YYYY-MM-DD",
    "target_slug": "<project-slug>",
    "surface": "session_close",
    "direction": "bidirectional",
    "occurred_at": "YYYY-MM-DDT00:00:00Z",
    "summary": "Plain-English list of what remains and the launch gate.",
    "tasks": [
      {
        "title": "Concrete task",
        "owner": "Owner name or role",
        "due_by": "YYYY-MM-DD",
        "status": "active",
        "priority": "high"
      }
    ]
  }
]'
```

The ledger intentionally duplicates the open tasks. It is the human-readable fallback the next session can trust.

### Step 5c: Verify

Before declaring "session closed":

```bash
kb entities search "Session YYYY-MM-DD" --user-id "${KB_USER_ID}"
kb entities search "<a representative task name>" --user-id "${KB_USER_ID}"
kb entities search "Project remaining task ledger YYYY-MM-DD" --user-id "${KB_USER_ID}"
```

If all three return hits, the close is complete.

### Step 6: Update the "last session" block

Your top-level agent instructions hold a block that renders into every future session. The nightly cron regenerates it from the `session_close` entity, but the next working session may start before that cron runs. So update it now.

Replace ONLY the content between the AUTOGEN markers:

```markdown
<!-- AUTOGEN:last-session START -->
<!-- regenerated daily by the renderer. Edit sessions/ files, not this block. -->

**Date:** YYYY-MM-DD (day of week, brief qualifier)
- **Bold lead.** Details.
- **Bold lead.** Details.
- Previous session: `sessions/YYYY-MM-DD-session.md`

<!-- AUTOGEN:last-session END -->
```

This is a local write for the next session. The renderer will overwrite it from the graph at its next scheduled run.

---

## Source of truth hierarchy

| Data | Primary | Backup |
|------|---------|--------|
| Session summary | Graph `session_close` entity | `sessions/*.md` |
| Entity interactions | Graph `observation` | Local entity mirror (frozen snapshot) |
| Tasks / follow-ups | Graph `task` entity | Freeform notes (structural only) |
| "Last session" block | Renderer from graph | Manual write at close |

## Safety net

If the graph is unreachable at close:
1. Write to `sessions/*.md` as normal
2. Note "graph unavailable — session not stored" in the archive
3. The renderer fallback reads `sessions/*.md` when no `session_close` is found

## Pairs with

- `start-session` — the next session's opening consumes what this close writes
- `get-context` — returned entity IDs feed Step 4 (observations) and Step 5 (tasks)
- `ingest` — owns parsing of new inbound; this skill owns the close-out side
