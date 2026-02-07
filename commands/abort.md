---
name: abort
description: Stop PTF execution cleanly and preserve all state
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Cleanly stop PTF execution while preserving all state for later resume.

**Usage:**
- `/ptf:abort` - Stop execution now

**Behavior:**
1. Marks any running tasks as interrupted
2. Updates execution status to "paused"
3. Creates final checkpoint
4. Logs session_aborted event
5. State is preserved for `/ptf:resume`

**After this command:** Execution is stopped. Run `/ptf:resume` to continue later.
</objective>

<context>
@.orchestrator/state/execution.yaml
</context>

<process>

## Phase 1: Check Execution State

Verify execution is in progress.

```bash
if [ ! -f ".orchestrator/state/execution.yaml" ]; then
  echo "No execution in progress. Nothing to abort."
  exit 0
fi

STATUS=$(grep "^status:" .orchestrator/state/execution.yaml | awk '{print $2}')

if [ "$STATUS" = "completed" ]; then
  echo "Execution already completed. Nothing to abort."
  exit 0
fi

if [ "$STATUS" = "paused" ]; then
  echo "Execution already paused."
  cat .orchestrator/state/execution.yaml
  exit 0
fi
```

## Phase 2: Handle Running Tasks

Find and mark any currently running tasks as interrupted.

```
Task(prompt="
Handle abort for execution in progress.

1. Find all tasks with status: running:
   grep -l \"^status: running\" .orchestrator/state/tasks/*.yaml 2>/dev/null

2. For each running task:
   - Read current task state
   - Update attempt entry: status: interrupted, interrupted_at: timestamp
   - Set task status: ready (will retry on resume)
   - Log task_interrupted event:
     echo '{\"ts\":\"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'\",\"event\":\"task_interrupted\",\"task\":\"'${TASK_ID}'\",\"reason\":\"user_abort\"}' >> .orchestrator/history/events.jsonl

3. Return count of interrupted tasks.
", subagent_type="ptf:state-manager")
```

## Phase 3: Update Execution State

Set execution status to paused.

```
Task(prompt="
Update execution state for abort.

1. Read current execution.yaml

2. Update fields:
   - status: paused
   - Add to blockers array: \"Execution aborted by user at {timestamp}\"
   - Update progress counters (tasks_running -> 0, tasks_pending += interrupted count)

3. Log session_aborted event:
   echo '{\"ts\":\"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'\",\"event\":\"session_aborted\",\"session\":\"'${SESSION_ID}'\",\"wave\":'${CURRENT_WAVE}'}' >> .orchestrator/history/events.jsonl

4. Create checkpoint (follow checkpoint protocol):
   - Write task state files
   - Write wave state file (if wave in progress)
   - Update execution.yaml LAST

5. Return checkpoint confirmation.
", subagent_type="ptf:state-manager")
```

## Phase 4: Report Abort Status

Show abort summary and next steps.

```markdown
## Execution Aborted

**Status:** Paused
**Session:** {session_id}
**Wave:** {current_wave} of {total}

### Summary

| Metric | Count |
|--------|-------|
| Tasks completed | {count} |
| Tasks pending | {count} |
| Tasks interrupted | {count} |
| Tasks failed | {count} |
| Tasks blocked | {count} |

### Interrupted Tasks

{If any tasks were interrupted:}
The following tasks were running and have been marked for retry:
- {task-id}: will retry from attempt {N}

{If no tasks were interrupted:}
No tasks were actively running.

### State Preserved

All progress has been saved to `.orchestrator/state/`.
Checkpoint complete at wave {N}.

### Next Steps

When ready to continue:

1. **`/ptf:resume`** - Continue from current point
2. **`/ptf:status`** - Review detailed execution state
3. **`/ptf:retry {task}`** - Retry specific task before resuming

### Session History

| Event | Time |
|-------|------|
| Started | {session_started} |
| Aborted | {now} |
| Duration | {duration} |
```

</process>

<error_handling>

## No Execution to Abort

```markdown
## No Execution in Progress

There is no active execution to abort.

**Current state:** No `.orchestrator/state/execution.yaml` found.

**Possible actions:**
- `/ptf:init [goal]` - Start new project
- `/ptf:execute` - Start execution (if already decomposed)
```

## Already Completed

```markdown
## Execution Already Complete

The plan has finished executing. There is nothing to abort.

**Status:** completed
**Tasks completed:** {count}

**Possible actions:**
- `/ptf:status` - View final execution summary
- `/ptf:verify` - Run final verification
```

## State Manager Error

If state manager fails during abort:
- Report the error
- Execution may be in inconsistent state
- Suggest `/ptf:status` to assess
- Manual intervention may be needed

</error_handling>

<success_criteria>
- [ ] Execution state checked (exists, not already complete)
- [ ] Running tasks identified and marked interrupted
- [ ] Execution status set to paused
- [ ] Blocker added with abort reason
- [ ] session_aborted event logged
- [ ] Checkpoint created (state preserved)
- [ ] Clear summary with task counts
- [ ] Next steps provided (resume, status, retry)
</success_criteria>
