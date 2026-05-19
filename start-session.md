---
name: start-session
description: Open every work session with a deterministic context pull. Verifies the knowledge graph is reachable, surfaces today's calendar, scans recent inbound (email + DMs), lists open tasks, and pre-resolves entity context for anyone on today's schedule. Reports a strategic snapshot and waits for direction. Run at the start of any working session before doing real work.
location: project
allowed-tools: Bash, Read
metadata:
  author: template
  version: "1.0"
---

# Start Session

Every work session follows the same shape: **get context → do work → close session**. This skill owns the first leg. By the time it finishes, the agent knows what's on the schedule, who matters today, what's still open from prior sessions, and what's brand new since last close.

The principle, stolen from any decent operator: you don't start work by reacting to whatever surfaces first. You orient, then act.

## When to Use

- First prompt of every working session
- Returning after a long gap (>4 hours) and you want fresh state
- After any unattended period (overnight, weekend, travel)
- Any time the user says "let's start", "ok let's go", "what's on the docket", "where are we"

Skip when:
- The user has already given you a concrete task and explicitly says "skip the intro"
- You're inside a sub-agent invocation (the parent owns context)

## Configuration

This skill expects three environment values. Defaults sit alongside but the user should override for their own setup:

| Env | Purpose | Example |
|---|---|---|
| `KB_HOST` | Knowledge-base API endpoint | `http://localhost:<your-port>` |
| `KB_USER_ID` | Your tenant / user id in the knowledge base | `your-name-personal` |
| `MAIL_QUERY_USER` | Gmail SA user id (if mail polling is wired) | `me` |

If any of the above is missing, the skill should report the gap and continue with what it has rather than blocking.

---

## Process

### Step 1: Health check

Verify the knowledge graph is reachable. If it isn't, every downstream step degrades to "best effort" and the user must be told.

```bash
curl -s "${KB_HOST}/health"
```

Expect `{"ok": true}` or equivalent. On failure: flag the outage in the report and continue with file-system fallbacks only.

### Step 2: Strategic snapshot

Pull the most-recently-touched entities to surface anything that needs attention before the user even asks.

```bash
kb entities list \
  --type contact \
  --user-id "${KB_USER_ID}" \
  --limit 20
```

Look for:
- Entities whose `last_observation_at` is more than 14 days old but whose status is `active` / `prospect` — they may be going dormant
- Entities created in the last 24 hours — new arrivals from any inbound pipeline
- Anything flagged in the entity snapshot (`needs_followup`, blockers, owed responses)

### Step 3: Today's calendar

Pull events for the local day. Calendar is the single best predictor of who the user will need context on.

```bash
START="$(date -u +%Y-%m-%dT00:00:00Z)"
END="$(date -u -v+1d +%Y-%m-%dT00:00:00Z)"
cal events list \
  --start "$START" \
  --end "$END"
```

For each event, extract:
- Time (start)
- Title
- Attendees (everyone who isn't the user)

Flag protected blocks (lunch, deep-work, recurring focus time) — never schedule across them.

### Step 4: Recent inbound scan

What changed since the last close? Two sources matter:

1. **Email** since the last close — anything starred, anything from a known active prospect, anything with `IMPORTANT` label
2. **DMs / messaging surfaces** — last 24h of unread / unanswered threads

```bash
# Email since last close
mail messages list \
  --query "in:inbox is:unread newer_than:1d" \
  --max 20

# DMs (surface-specific — adapt to your bridge)
ls -t "${DM_DATA_DIR}"/*.jsonl 2>/dev/null | head -5
```

Don't enumerate every message in the report. Cluster by sender, surface the 2–3 that look most load-bearing.

### Step 5: Open tasks

Pull tasks where `status=active` and `due_by <= today + 1 day`. These are what's at risk of slipping if not surfaced.

```bash
kb entities list \
  --type task \
  --user-id "${KB_USER_ID}" \
  --status active \
  --limit 30
```

Group by:
- **Due today** — must close or reschedule
- **Overdue** — surface separately; aging tasks need either action or a deliberate cancel
- **Due this week** — informational

### Step 6: Pre-resolve context for today's attendees

For every non-user attendee on today's calendar, run `get-context` (the sister skill) at low depth. The goal: when the user says "what was the last thing with X?", you already know.

```bash
for slug in "${todays_attendee_slugs[@]}"; do
  kb entities search "$slug" --user-id "${KB_USER_ID}" --limit 1
  kb observations list --entity-id "$entity_id" --limit 5
done
```

Don't dump all of it into the report. Hold it in working memory; it surfaces when relevant.

### Step 7: Today's qualifier (priming the close)

Draft a short candidate qualifier (≤ 80 chars) for the eventual session_close. This is a soft commitment to today's shape: it forces the agent to think about the day's theme up-front.

Examples:
- `"Tuesday — client session + infra gates"`
- `"Wednesday — heads-down on the integration test"`

Hold it; offer it back at close time.

### Step 8: Report

Output a structured, scannable snapshot. **Keep it under ~30 lines.** The user reads this every time; respect that.

```markdown
## Today's schedule
| Time | Block |
|---|---|
| 09:30 | <event title> |
| 10:15 | <event title> + <attendee> |
| 12:00 | Lunch (protected) |

## Open from last close
- <bullet — what carries over>
- <bullet — what's at risk>

## New since last close
- <bullet — inbound that matters>
- <bullet — DM / signal>

## Stale relationships
- <slug>: <days since last touch>, status <status>

## Suggested qualifier
"<draft qualifier ≤ 80 chars>"

## Ask
What do you want to work on?
```

### Step 9: Wait

Do not start work. Wait for the user to direct.

The reflex to "be helpful" by picking a task off the open-tasks list is wrong here. The user's priorities at session start are theirs to set, not the agent's to assume.

---

## Failure modes & guardrails

- **Knowledge graph unreachable.** Report the outage, continue with calendar + mail + file-system only. Flag that observations will be lossy until reconnect.
- **Calendar empty.** Still run the rest. A calm day is real signal.
- **Long gap since last close (>72h).** Note this explicitly — "last close was X days ago, expect drift in entity context." Encourage a fuller catch-up before any high-stakes work.
- **Conflict between calendar attendees and entity records.** Don't auto-create entities at start-session. Flag the unknowns; let `prospect-intake` or `ingest` own creation.

## Source of truth hierarchy

| Data | Primary | Fallback |
|---|---|---|
| Entity state | Knowledge graph | File mirror (read-only) |
| Calendar | Calendar API | None — flag absence |
| Email | Mail API | None — flag absence |
| Tasks | Knowledge graph `task` entities | Plain-text TODO list |

## Pairs with

- `get-context` — called by Step 6 for each attendee
- `close-session` — consumes the qualifier drafted in Step 7
- `ingest` — any inbound surfaced in Step 4 that hasn't been logged yet
