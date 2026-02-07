---
name: ptf:status
description: Show current PTF execution state - progress, waves, recent activity, blockers
allowed-tools:
  - Read
  - Bash
  - Glob
---

<objective>
Display current execution state from .orchestrator/state/.

Shows:
- Overall status and progress percentage
- Wave status breakdown
- Recent events from history
- Any blockers preventing progress

If no execution state, shows decomposition/plan status instead.

**After this command:** `/ptf:execute` to continue execution, `/ptf:resume` if interrupted.
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
</execution_context>

<context>
@.orchestrator/state/execution.yaml (if exists)
@.orchestrator/config.yaml
@.orchestrator/plan.md
</context>

<process>

## Phase 1: Load State

Determine what state exists and where we are in the PTF workflow.

```bash
if [ -f .orchestrator/state/execution.yaml ]; then
  echo "EXECUTION_STATE: exists"
elif [ -f .orchestrator/plan.md ]; then
  echo "PLAN_STATE: exists but execution not started"
  echo "Next step: /ptf:execute"
elif [ -d .orchestrator/decomposition ] && [ -f .orchestrator/decomposition/tasks/*.yaml ] 2>/dev/null; then
  echo "DECOMPOSITION_STATE: exists, run /ptf:plan next"
elif [ -f .orchestrator/goal.md ]; then
  echo "INIT_STATE: goal exists, run /ptf:decompose next"
else
  echo "NO_STATE: No PTF state found. Run /ptf:init first"
fi
```

If no execution state exists, display appropriate status based on what does exist:

**No state at all:**
```
# PTF Status: Not Initialized

No PTF project found in this directory.

**Next:** Run `/ptf:init <goal>` to start a new project.
```

**Goal exists (after /ptf:init):**
```
# PTF Status: Initialized

Project initialized but not yet decomposed.

**Goal:** {first line of goal.md}
**Domain:** {domain from config.yaml}

**Next:** Run `/ptf:decompose` to break goal into tasks.
```

**Decomposition exists (after /ptf:decompose):**
```
# PTF Status: Decomposed

Goal decomposed into tasks but plan not yet generated.

**Tasks:** {count of task files}

**Next:** Run `/ptf:plan` to generate execution plan.
```

**Plan exists (after /ptf:plan):**
```
# PTF Status: Planned

Execution plan ready but not started.

**Tasks:** {N} in {M} waves

**Next:** Run `/ptf:execute` to begin execution.
```

If execution.yaml exists, continue to Phase 2.

## Phase 2: Gather Status

Read all state sources and compile status data.

**1. Read execution.yaml:**
```bash
cat .orchestrator/state/execution.yaml
```

Extract:
- status: overall execution status
- current_wave: current wave number
- waves_total: total waves
- progress: task counts by status
- session: current session info
- blockers: any blockers
- execution_mode: classic or teams (default: classic)
- teams: team state if teams mode (team_name, worker_count, tasks_in_flight)

**2. Count task files by status:**
```bash
# Count tasks in each state
for STATUS in completed running pending failed blocked ready; do
  COUNT=$(grep -l "^status: ${STATUS}" .orchestrator/state/tasks/*.yaml 2>/dev/null | wc -l)
  echo "${STATUS}: ${COUNT}"
done
```

**3. Read recent events:**
```bash
# Last 5 events from event log
tail -5 .orchestrator/history/events.jsonl
```

Parse each event line for display.

**4. Read wave summaries:**
```bash
# Get wave status from execution.yaml wave_summary section
grep -A50 "^wave_summary:" .orchestrator/state/execution.yaml | grep "^  [0-9]:"
```

**5. Calculate progress:**
```bash
COMPLETED=$(grep "^  tasks_completed:" .orchestrator/state/execution.yaml | awk '{print $2}')
TOTAL=$(grep "^  tasks_total:" .orchestrator/state/execution.yaml | awk '{print $2}')
PERCENT=$((COMPLETED * 100 / TOTAL))
```

## Phase 3: Display Status

Format and display the status output.

**Detect output format:**
```bash
if [ -t 1 ]; then
  # TTY - human-readable output
  OUTPUT_FORMAT="human"
else
  # Pipe/redirect - machine-readable output
  OUTPUT_FORMAT="json"
fi
```

**Human-readable output (TTY):**

```markdown
# PTF Status: {STATUS}

## Progress

{progress_bar} {completed}/{total} tasks ({percent}%)

| Metric | Count |
|--------|-------|
| Completed | {tasks_completed} |
| Running | {tasks_running} |
| Ready | {tasks_ready} |
| Pending | {tasks_pending} |
| Failed | {tasks_failed} |
| Blocked | {tasks_blocked} |

## Current Wave

Wave {current_wave} of {waves_total}

## Wave Summary

| Wave | Status | Tasks |
|------|--------|-------|
| 1 | completed | auth-schema, user-model |
| 2 | running | auth-service (1/2 complete) |
| 3 | pending | auth-routes, auth-tests |
| ... | ... | ... |

## Recent Activity

| Time | Event | Details |
|------|-------|---------|
| 10:35:22 | task_completed | auth-schema (179s) |
| 10:32:01 | task_started | auth-schema |
| 10:32:00 | wave_started | wave 1 |
| ... | ... | ... |

## Session

**ID:** {session.id}
**Started:** {session.started}
**Last Update:** {session.last_update}

## Blockers

{If blockers array is not empty:}
- {blocker 1}
- {blocker 2}

{If blockers array is empty:}
No blockers. Ready to continue.

{If execution_mode == "teams":}

## Teams Execution

**Mode:** Dynamic scheduling (Agent Teams)
**Team:** {teams.team_name}
**Workers:** {teams.worker_count}

### Tasks In Flight

| Worker | Task | Started |
|--------|------|---------|
| {worker} | {task_id} | {started} |
| ... | ... | ... |

### Task Progress (Flat View)

| Status | Tasks |
|--------|-------|
| Completed | {list of completed task IDs} |
| In Progress | {list of running task IDs with worker} |
| Ready | {list of unblocked pending task IDs} |
| Blocked | {list of blocked task IDs with reason} |

{End teams section}

---

**Next:** `/ptf:execute` to continue, `/ptf:resume` if previously interrupted
```

**Progress bar rendering:**
```bash
render_progress_bar() {
  local completed=$1
  local total=$2
  local width=20
  local filled=$((completed * width / total))
  local empty=$((width - filled))

  # Build bar using Unicode block characters
  local bar="["
  for ((i=0; i<filled; i++)); do bar+="="; done
  for ((i=0; i<empty; i++)); do bar+=" "; done
  bar+="]"

  echo "$bar"
}
```

**Machine-readable output (non-TTY):**

When output is piped or redirected, emit structured JSON:

```json
{
  "status": "running",
  "current_wave": 2,
  "waves_total": 5,
  "progress": {
    "tasks_total": 12,
    "tasks_completed": 3,
    "tasks_running": 2,
    "tasks_pending": 7,
    "tasks_failed": 0,
    "tasks_blocked": 0,
    "percent": 25
  },
  "wave_summary": {
    "1": "completed",
    "2": "running",
    "3": "pending",
    "4": "pending",
    "5": "pending"
  },
  "session": {
    "id": "session-003",
    "started": "2026-01-18T10:32:00Z",
    "last_update": "2026-01-18T10:35:22Z"
  },
  "blockers": [],
  "recent_events": [
    {"ts": "2026-01-18T10:35:22Z", "event": "task_completed", "task": "auth-schema"},
    {"ts": "2026-01-18T10:32:01Z", "event": "task_started", "task": "auth-schema"}
  ]
}
```

</process>

<output_formats>

## Human-Readable (TTY)

When output goes to terminal:
- Progress bar with visual indicators
- Formatted markdown tables
- Clear section headers
- Status-appropriate next action hint

## Machine-Readable (Pipe/Redirect)

When output is piped or redirected:
- Single JSON object on stdout
- All status information structured
- Easy to parse with jq or other tools

**Detection logic:**
```bash
if [ -t 1 ]; then
  # Terminal - use human format
  format_human_output "$STATUS_DATA"
else
  # Pipe/redirect - use JSON format
  format_json_output "$STATUS_DATA"
fi
```

**Usage examples:**
```bash
# Human-readable
/ptf:status

# Machine-readable for scripting
/ptf:status | jq '.progress.percent'

# Check if running
/ptf:status | jq -e '.status == "running"'
```

</output_formats>

<success_criteria>
- [ ] Status displayed from execution.yaml if exists
- [ ] Progress counts accurate (match task file states)
- [ ] Wave summary shows all waves with status
- [ ] Recent events shown (last 5 from events.jsonl)
- [ ] Blockers displayed if any
- [ ] Appropriate fallback for missing state
- [ ] Human-readable format for TTY
- [ ] JSON format for non-TTY
- [ ] Next action hint appropriate to current state
</success_criteria>
