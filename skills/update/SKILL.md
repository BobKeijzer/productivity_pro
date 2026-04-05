---
name: update
description: Sync tasks and refresh memory from your current activity. Use when pulling new assignments from your project tracker into TASKS.md, triaging stale or overdue tasks, filling memory gaps for unknown people or projects, or running a comprehensive scan to catch todos buried in chat and email. This skill reads across all sessions in the project, so it picks up context from every chat, not just the current one.
argument-hint: "[--comprehensive]"
---

# Update Command

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../../CONNECTORS.md).

Keep your task list and memory current. Two modes:

- **Default:** Sync tasks from external tools, triage stale items, check memory for gaps
- **`--comprehensive`:** Deep scan chat, email, calendar, docs — flag missed todos and suggest new memories

Both modes read across all sessions in the project (not just the current chat) and save a changelog of what changed to `memory/updates/`.

## Usage

```bash
/productivity:update
/productivity:update --comprehensive
```

## Cross-Session Reading

Users often work in multiple chats within the same project — one per client, use case, or topic. The update skill needs to see what happened across all of them, not just the current chat. Without this, information from other chats would be invisible and the update would be incomplete.

### How it works

1. **Check the last update timestamp.** Read `memory/.last_update` to find when the update skill last ran. If the file doesn't exist, this is the first run — treat all recent session content as new.

2. **List and filter sessions** using `list_sessions`. This returns every local Cowork session on the machine — including sessions from other projects. Filter to only sessions that belong to the current project:

   - Determine the current workspace folder name by running `ls ./mnt/` (e.g. `bob-workspace`)
   - For each session returned by `list_sessions`, extract the session name from its `cwd` field (e.g. `cwd: /sessions/friendly-vibrant-noether` → session name is `friendly-vibrant-noether`)
   - Check which workspace folder is mounted for that session: `ls /sessions/[session-name]/mnt/ 2>/dev/null`
   - **Only include sessions whose mounted workspace folder name matches the current project's workspace folder.** Sessions with a different or missing workspace folder belong to a different project — skip them.

3. **Read new content from each session** using `read_transcript`. For each session:
   - Read the most recent messages (use `limit` parameter to get the last 20-50 messages)
   - Filter to only messages that occurred **after** the `.last_update` timestamp
   - If a session has no new messages since the last update, skip it entirely

4. **After completing the update**, write the current timestamp to `memory/.last_update`.

This approach ensures that even if you run the update from a chat you barely used today, it still picks up everything from your other active chats. And if you skip a day, the next update catches everything since it last ran.

## Default Mode

### 1. Load Current State

Read `TASKS.md` and `memory/` directory. If they don't exist, suggest `/productivity:start` first.

Read `memory/.last_update` for the timestamp of the previous run. Read new content from all sessions (see Cross-Session Reading above).

### 2. Sync Tasks from External Sources

Check for available task sources:
- **Project tracker** (e.g. Asana, Linear, Jira) (if MCP available)
- **GitHub Issues** (if in a repo): `gh issue list --assignee=@me`

If no sources are available, skip to Step 3.

**Fetch tasks assigned to the user** (open/in-progress). Compare against TASKS.md:

| External task | TASKS.md match? | Action |
|---------------|-----------------|--------|
| Found, not in TASKS.md | No match | Offer to add |
| Found, already in TASKS.md | Match by title (fuzzy) | Skip |
| In TASKS.md, not in external | No match | Flag as potentially stale |
| Completed externally | In Active section | Offer to mark done |

Present diff and let user decide what to add/complete.

### 3. Triage Stale Items

Review Active tasks in TASKS.md and flag:
- Tasks with due dates in the past
- Tasks in Active for 30+ days
- Tasks with no context (no person, no project)

Present each for triage: Mark done? Reschedule? Move to Someday?

### 4. Decode Tasks for Memory Gaps

For each task, attempt to decode all entities (people, projects, acronyms, tools, links):

```
Task: "Send PSR to Todd re: Phoenix blockers"

Decode:
- PSR → Pipeline Status Report (in glossary)
- Todd → Todd Martinez (in people/)
- Phoenix → ? Not in memory
```

Track what's fully decoded vs. what has gaps.

### 5. Fill Gaps

Present unknown terms grouped:
```
I found terms in your tasks I don't have context for:

1. "Phoenix" (from: "Send PSR to Todd re: Phoenix blockers")
   → What's Phoenix?

2. "Maya" (from: "sync with Maya on API design")
   → Who is Maya?
```

Add answers to the appropriate memory files (people/, projects/, glossary.md).

### 6. Capture Enrichment

Tasks and session transcripts often contain richer context than memory. Extract and update:
- **Links** from tasks → add to project/people files
- **Status changes** ("launch done") → update project status, demote from CLAUDE.md
- **Relationships** ("Todd's sign-off on Maya's proposal") → cross-reference people
- **Deadlines** → add to project files
- **New information from other sessions** → decisions, new contacts, project updates discovered in cross-session reading

### 7. Save Update Changelog

Create `memory/updates/` if it doesn't exist. Each update run writes to a file named with today's date: `memory/updates/DD-MM-YYYY.md`. If a file for today already exists (from an earlier update run the same day), append to it. Never write to a file from a previous date — each new day gets its own file. Over time, `memory/updates/` grows into a dated history of what changed and when.

This changelog captures what actually changed during this update run. The reason this matters: without it, memory/ and CLAUDE.md get updated in place and you lose track of what changed when. The changelog gives you a timeline — you can look back and see "on April 3rd we learned X, updated Y, and added Z."

**Changelog format:**

```markdown
# Update DD-MM-YYYY

## Sessions scanned
- [Session name] — [X new messages since last update]
- [Session name] — [X new messages since last update]

## Memory changes
- Added: [new person/project/term added to memory]
- Updated: [existing memory file that was enriched]
- Promoted to CLAUDE.md: [item moved to hot cache]
- Demoted from CLAUDE.md: [item removed from hot cache]

## Task changes
- Added: [new tasks added to TASKS.md]
- Completed: [tasks marked done]
- Triaged: [stale tasks rescheduled or moved]

## Key insights
- [Notable decisions, patterns, or context discovered across sessions]
```

Skip empty sections. Keep it concise — this is a reference log, not a narrative.

If there's already a changelog for today (from an earlier update run), append to it rather than overwriting.

### 8. Update Timestamp

Write the current timestamp to `memory/.last_update`:

```
YYYY-MM-DDTHH:MM:SS
```

### 9. Report

```
Update complete:
- Sessions scanned: X (Y with new content)
- Tasks: +3 from project tracker, 1 completed, 2 triaged
- Memory: 2 gaps filled, 1 project enriched
- Changelog: memory/updates/DD-MM-YYYY.md
- All tasks decoded
```

## Comprehensive Mode (`--comprehensive`)

Everything in Default Mode, plus a deep scan of recent activity.

### Extra Step: Scan Activity Sources

Gather data from available sources:

**Always check first:**
- **Previous update changelogs:** Read recent entries from `memory/updates/` (last 5-7 days). These show what changed recently and help spot patterns or gaps.

**Then check MCP sources (if connected):**
- **Chat:** Search recent messages, read active channels
- **Email:** Search sent messages
- **Documents:** List recently touched docs
- **Calendar:** List recent + upcoming events

### Extra Step: Flag Missed Todos

Compare activity against TASKS.md. Surface action items that aren't tracked:

```
## Possible Missing Tasks

From your activity, these look like todos you haven't captured:

1. From chat (Jan 18):
   "I'll send the updated mockups by Friday"
   → Add to TASKS.md?

2. From meeting "Phoenix Standup" (Jan 17):
   You have a recurring meeting but no Phoenix tasks active
   → Anything needed here?

3. From email (Jan 16):
   "I'll review the API spec this week"
   → Add to TASKS.md?
```

Let user pick which to add.

### Extra Step: Suggest New Memories

Surface new entities not in memory:

```
## New People (not in memory)
| Name | Frequency | Context |
|------|-----------|---------|
| Maya Rodriguez | 12 mentions | design, UI reviews |
| Alex K | 8 mentions | DMs about API |

## New Projects/Topics
| Name | Frequency | Context |
|------|-----------|---------|
| Starlight | 15 mentions | planning docs, product |

## Suggested Cleanup
- **Horizon project** — No mentions in 30 days. Mark completed?
```

Present grouped by confidence. High-confidence items offered to add directly; low-confidence items asked about.

## Notes

- Never auto-add tasks or memories without user confirmation
- External source links are preserved when available
- Fuzzy matching on task titles handles minor wording differences
- Safe to run frequently — only updates when there's new info
- `--comprehensive` always runs interactively
- Cross-session reading respects the `.last_update` timestamp — it only reads what's new
- If `list_sessions` or `read_transcript` tools are unavailable, fall back gracefully to current-session-only mode