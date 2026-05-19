---
name: get-context
description: Pull everything-you-need-to-know about a topic — a person, a company, a project, a decision — before doing the work. Resolves the target, fetches the entity snapshot, the full observation history, linked tasks, recent communication threads (email, DMs, call transcripts), and walks one hop of the relationship graph (company → people, person → company + collaborators). Synthesizes a brief instead of dumping data. Call this before drafting anything (email, deck, decision memo) and before any conversation where the user wants to walk in primed.
location: project
allowed-tools: Bash, Read
metadata:
  author: template
  version: "1.0"
---

# Get Context

The single most under-used skill in the toolbox: pulling enough context before answering. The cost of doing it is low (a handful of API calls). The cost of skipping it is high (hallucinated history, missed commitments, generic drafts that ignore what was already said).

The rule: any time the user names a person, project, company, or decision that this skill could pre-load, run it before synthesizing.

## When to Use

- User names a person, company, or project mid-conversation: `"prep me for the X call"`, `"draft the Y email"`, `"what's the status on Z?"`
- Before drafting any outbound — email, DM, deck, proposal, follow-up
- Before any meeting prep ("what was the last thing with X?")
- Before any decision memo where prior commitments / constraints matter
- Proactively, in the loop of the `start-session` skill, for each calendar attendee
- When the user voice-transcribes a name (often slightly wrong) — fuzzy-resolve, don't guess

Skip when:
- The user has already shown they have the context loaded ("you remember the X situation")
- The target is purely internal (no entity, no thread history)
- Latency matters more than completeness (the user is waiting on a one-line answer)

## Configuration

| Env | Purpose |
|---|---|
| `KB_HOST` | Knowledge-base API endpoint |
| `KB_USER_ID` | Tenant / user id |
| `MAIL_DAYS` | How far back to scan email threads (default 14) |
| `TRANSCRIPT_DIR` | Where call transcripts land (optional — skip if unset) |

---

## Process

### Step 1: Resolve the target

The user almost never gives you a clean entity ID. They say a name, a half-name, a misheard transcript token, or a project slug. Resolve before fetching.

```bash
kb entities search "<user-provided name>" --user-id "${KB_USER_ID}"
```

Apply these rules:
- **Multiple hits.** If two entities match, ask the user which one — never silently pick the most recent. Common failure mode: two people share a first name.
- **Voice-mishears.** If the input came from a transcript (`"Pierre de Rue"`), search aliases too. Spelling drift is normal.
- **No hits.** Don't fabricate. Report "no entity found" and ask whether to create one (route to `prospect-intake` or `ingest`).
- **Status drift.** If the entity exists but is `archived` or `merged_to_entity_id` is set, follow the merge pointer and report the redirect.

### Step 2: Snapshot

Pull the structured entity record. This is rollup data — fast, but stale relative to observations.

```bash
kb entities get "<entity_id>"
```

Extract:
- Canonical name, status, last observation timestamp
- Relationship to the user (client / prospect / partner / community / dormant)
- Owner, segment, source surface
- Any flags (`needs_followup`, `at_risk`, blockers)

### Step 3: Observation history

The snapshot is a view. The full event history lives in observations. Read them in reverse-chronological order; stop when you've covered the last 30 days OR the last 20 entries, whichever is shorter.

```bash
kb observations list --entity-id "<entity_id>" --limit 20
```

Extract for each:
- What happened (interaction / decision / status change)
- Surface (email, call, DM, web, file)
- Who else was involved
- Open commitments mentioned

### Step 4: Linked tasks

What's still owed? Pull active tasks pinned to this entity.

```bash
kb entities list \
  --type task \
  --user-id "${KB_USER_ID}" \
  --entity-slug "<slug>" \
  --status active
```

Group:
- Owed by the user (must do)
- Owed by the entity (waiting on)
- Joint (need coordination)

### Step 5: Recent communication threads

For people/companies with email or DM history, pull the last `${MAIL_DAYS}` of threads. This catches the most recent unprocessed exchange — the thing that's freshest in the human's mind but might not be in observations yet.

```bash
# Email
mail messages list \
  --query "from:<email> OR to:<email> newer_than:${MAIL_DAYS}d" \
  --max 5

# DMs (surface-specific)
grep -l "<entity_slug>" "${DM_DATA_DIR}"/*.jsonl 2>/dev/null | head -3
```

Pull subjects + first 200 chars of body for each. Don't dump full threads.

### Step 6: Recent call transcripts

If the entity attended a meeting in the last 30 days, the transcript is gold. It's the highest-fidelity record of what was actually said.

```bash
ls "${TRANSCRIPT_DIR}"/*<entity_slug>*.json 2>/dev/null | sort -r | head -3
```

For the most recent transcript:
- Read the first 200 lines (opening — sets the frame)
- Read the last 200 lines (closing — usually contains commitments)
- Skip the middle unless the user is asking about a specific topic

### Step 7: One-hop graph walk

The target almost never matters in isolation. Walk one edge out.

**For a person:**
- Their company (if linked)
- Other people at the same company
- Anyone tagged as a co-participant in the last 5 interactions

**For a company:**
- All people linked to it (with role + status)
- Any active project entity owned by the company
- Recent decisions involving the company

**For a project:**
- The driver (person who owns it)
- The stakeholders (anyone tagged as participant)
- Linked tasks, even if not directly attached to the project entity

```bash
kb entities relationships --entity-id "<entity_id>" --depth 1
```

Do not go further than one hop. Two hops blows the context budget and rarely improves the answer.

### Step 8: Synthesize a brief

Don't dump data. Synthesize. The brief should be readable in 30 seconds and answer four questions:

```markdown
## <Display Name> — context brief

**Relationship.** <one line: who they are, status, what stage>
**Last touch.** <when, what surface, what was said>
**Open commitments.** <bulleted: who owes what by when>
**Graph context.** <one line: who else is in this orbit>
**Suggested next move.** <one line: what the user would probably do given everything above>
```

If any of the four is missing data, say "no signal" — don't pad.

### Step 9: Return raw entity IDs

At the bottom of the brief, list the entity IDs you touched. The user (or follow-up skills) needs them to write observations / tasks / decisions back.

```markdown
---
**Entity IDs touched:**
- person: `ent_xxx`
- company: `ent_yyy`
- recent tasks: `ent_zzz`, `ent_aaa`
- linked transcripts: `<filename>.json`
```

---

## Depth tiers (callable from other skills)

When invoked by another skill, accept a `--depth` flag:

| Depth | Steps run | Use for |
|---|---|---|
| `snapshot` | 1, 2 | Cheap lookup during `start-session` (loop over many attendees) |
| `standard` | 1–6 | Default. Before drafting anything. |
| `full` | 1–9 | Pre-call prep, high-stakes drafts, decision memos |

## Failure modes & guardrails

- **Voice-mishears.** Always alias-search before failing. Names from transcripts are routinely 80% right.
- **Two-X problem.** When multiple entities match, surface the disambiguation explicitly — never pick silently.
- **Stale snapshot.** If the snapshot says `last_observation_at` is older than the most recent observation you see, trust the observation. The snapshot is a recompute; observations are events.
- **Email volume.** If the entity has > 50 threads in the window, sample the most recent 5 + the most starred 3. Don't try to read everything.
- **Privacy.** If the brief will be shown to a third party, scrub it. Phone numbers, personal addresses, private slugs come out. Names, roles, public URLs stay.
- **Don't fabricate.** No signal is better than invented signal. If the data isn't there, say so.

## Source of truth hierarchy

| Data | Primary | Fallback |
|---|---|---|
| Entity facts | Knowledge graph | File mirror (read-only) |
| Interaction history | Observations + transcripts | Email threads |
| Open commitments | `task` entities | Inline mentions in observations |
| Relationship graph | Knowledge graph edges | Co-occurrence in observations |

## Pairs with

- `start-session` — calls this at low depth for each calendar attendee
- `close-session` — consumes the entity IDs returned in Step 9 when writing observations back
- `ingest` — owns creation; this skill is read-only
- `prospect-intake` — handles the "no entity found" branch
