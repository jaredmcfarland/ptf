---
description: Retry a specific failed or blocked task with fresh context
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Retry a specific failed task by resetting its state and re-executing.

**Usage:**
- `/ptf:retry task-id` - Retry specific task

**Behavior:**
1. Validates task exists and is in failed/blocked state
2. Resets task state to "ready"
3. Clears blocked status from dependent tasks (if this was only blocker)
4. Optionally spawns orchestrator to execute the task immediately

**After this command:** Task is ready for execution. Run `/ptf:execute` to execute.
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
</execution_context>

<context>
@.orchestrator/state/execution.yaml
@.orchestrator/decomposition/graph.yaml
</context>

<process>

## Phase 1: Parse Arguments

Extract task ID from command arguments.

```bash
TASK_ID="$1"

if [ -z "$TASK_ID" ]; then
  echo "ERROR: Task ID required"
  echo "Usage: /ptf:retry task-id"
  exit 1
fi
```

## Phase 2: Validate Task

Check task exists and is in retryable state.

```bash
TASK_FILE=".orchestrator/state/tasks/${TASK_ID}.yaml"

if [ ! -f "$TASK_FILE" ]; then
  echo "ERROR: Task ${TASK_ID} not found"
  echo "Available tasks:"
  ls .orchestrator/state/tasks/*.yaml 2>/dev/null | xargs -I{} basename {} .yaml
  exit 1
fi

STATUS=$(grep "^status:" "$TASK_FILE" | awk '{print $2}')

if [ "$STATUS" != "failed" ] && [ "$STATUS" != "blocked" ]; then
  echo "ERROR: Task ${TASK_ID} has status '${STATUS}'"
  echo "Only failed or blocked tasks can be retried."
  echo ""
  echo "Task status:"
  cat "$TASK_FILE"
  exit 1
fi
```

## Phase 3: Reset Task State

Invoke state manager to reset task for retry.

```
Task(prompt="
Reset task ${TASK_ID} for retry.

1. Read current task state from .orchestrator/state/tasks/${TASK_ID}.yaml

2. Update task state:
   - status: ready
   - Clear error field
   - Clear blocked_by field (if present)
   - Preserve attempts array (for history)
   - Add retry entry: {attempt: N+1, status: pending, reset_at: timestamp, reset_reason: manual_retry}

3. Log task_retry event:
   echo '{\"ts\":\"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'\",\"event\":\"task_retry\",\"task\":\"${TASK_ID}\",\"reason\":\"manual_retry\"}' >> .orchestrator/history/events.jsonl

4. Return confirmation with task details.
", subagent_type="ptf-state-manager")
```

## Phase 4: Clear Cascade Blocks

If this task was blocking others, check if they can now proceed.

```
Task(prompt="
Check cascade effects for task ${TASK_ID}.

1. Find tasks with blocked_by containing ${TASK_ID}:
   grep -l \"blocked_by:\" .orchestrator/state/tasks/*.yaml | while read f; do
     if grep -q \"${TASK_ID}\" \"\$f\"; then
       echo \"Found: \$f\"
     fi
   done

2. For each blocked task found:
   - Read its blocked_by array
   - Remove ${TASK_ID} from blocked_by
   - If blocked_by is now empty:
     - Set status: ready
     - Log task_unblocked event
   - If blocked_by still has entries:
     - Keep status: blocked
     - Log partial_unblock event

3. Update execution.yaml counters:
   - Decrement tasks_failed
   - Increment tasks_pending

4. Return list of affected tasks.
", subagent_type="ptf-state-manager")
```

## Phase 5: Report Result

Show retry status and next steps.

```markdown
## Task Reset for Retry

**Task:** {task_id}
**Previous Status:** {failed | blocked}
**New Status:** ready
**Attempts:** {N} previous, resetting for attempt {N+1}

### Cascade Effects

{If no blocked tasks affected:}
No dependent tasks were blocked by this task.

{If blocked tasks cleared:}
The following tasks are now unblocked:
- {task-id}: was blocked by {task_id}, now ready

{If blocked tasks still blocked:}
The following tasks remain blocked by other failures:
- {task-id}: still blocked by {other_task}

### Next Steps

Task is ready for execution.

**Options:**
1. `/ptf:execute` - Execute current wave (includes this task if in current wave)
2. `/ptf:execute-all` - Execute all remaining waves
3. `/ptf:status` - Check current execution state
```

</process>

<error_handling>

## Task Not Found

```markdown
## Error: Task Not Found

Task ID `{task_id}` does not exist in the current execution.

**Available tasks:**
{list of task files}

**Tip:** Use `/ptf:status` to see all tasks and their current status.
```

## Task Not Retryable

```markdown
## Error: Task Not Retryable

Task `{task_id}` has status `{status}` and cannot be retried.

**Retryable statuses:** failed, blocked
**Current status:** {status}

{If status == completed:}
Task already completed successfully. No retry needed.

{If status == running:}
Task is currently running. Wait for completion.

{If status == ready:}
Task is already ready to execute. Run `/ptf:execute`.
```

## State Manager Error

If state manager invocation fails:
- Report the error
- Do NOT modify any state manually
- Suggest running `/ptf:status` to check state consistency

</error_handling>

<success_criteria>
- [ ] Task ID parsed from arguments
- [ ] Task existence validated
- [ ] Task status checked (failed or blocked)
- [ ] Task state reset to ready
- [ ] Attempt history preserved
- [ ] task_retry event logged
- [ ] Blocked dependents checked and updated if applicable
- [ ] Execution counters updated
- [ ] Clear summary with next steps
</success_criteria>
