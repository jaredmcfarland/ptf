---
name: on-failure
description: Hook fired when a task fails after all retries exhausted
trigger: task_failure_exhausted
hook_id: HOOK-03
---

<when>
This hook fires when a task definitively fails - after all retry attempts are exhausted
or when a blocking condition is encountered that cannot be retried.

**Trigger conditions:**
- Task reaches max_iterations without passing verification
- Task encounters BLOCKED: missing_input (cannot proceed)
- Task encounters BLOCKED: unclear_requirements (needs human input)
- Task executor crashes or times out after retries

**Not triggered by:**
- Individual failed attempt within Ralph loop (retry in progress)
- Temporary failures that will be retried
- Successful task completion
</when>

<actions>
Default actions performed when this hook fires:

1. **log_failure**
   - Append task_failed event to events.jsonl
   - Include: task_id, final error, attempt count, failure category

2. **check_cascade_policy**
   - Identify tasks that depend on this failed task
   - For each dependent task:
     - If propagate_failure: true (default), mark as blocked
     - If propagate_failure: false, allow to attempt anyway
   - Record cascade effects

3. **update_failure_record**
   - Create/update failure record file
   - Include execution context for debugging
   - Store last error, attempts, outputs produced
</actions>

<context_available>
Data available to this hook:

| Variable | Type | Description |
|----------|------|-------------|
| `task_id` | string | Task identifier |
| `task_name` | string | Human-readable task name |
| `wave` | number | Wave this task belongs to |
| `error` | string | Final error message |
| `error_category` | string | verification_failed, missing_input, execution_error, max_iterations |
| `attempts` | number | Total attempts made |
| `max_iterations` | number | Configured max iterations |
| `outputs_partial` | array | Any outputs produced before failure |
| `cascade_targets` | array | Tasks dependent on this one |
| `execution_log` | array | Summary of each attempt |
</context_available>

<event_format>
Event logged to events.jsonl:

```json
{
  "ts": "2026-01-18T10:45:00Z",
  "event": "task_failed",
  "task": "auth-service",
  "wave": 2,
  "error_category": "verification_failed",
  "error": "contains check failed: 'authenticate' function not found",
  "attempts": 10,
  "max_iterations": 10,
  "cascade_blocked": ["auth-routes", "auth-tests"]
}
```

Cascade event:

```json
{
  "ts": "2026-01-18T10:45:01Z",
  "event": "task_blocked",
  "task": "auth-routes",
  "wave": 3,
  "blocked_by": "auth-service",
  "reason": "cascade_failure"
}
```
</event_format>

<failure_categories>
## Error Categories and Recommended Actions

| Category | Cause | Retry? | User Action |
|----------|-------|--------|-------------|
| `verification_failed` | Output doesn't match expected | May retry | Review task requirements |
| `missing_input` | Required input file missing | No | Check prior task status |
| `execution_error` | Error during execution | May retry | Review error details |
| `max_iterations` | Exhausted retry limit | No | Manual intervention |
| `unclear_requirements` | Can't parse task | No | Clarify task definition |
| `timeout` | Execution too long | May retry | Increase timeout or simplify |

</failure_categories>

<customization>
To customize this hook's behavior, edit this file.

**Add notifications:**

```yaml
custom_actions:
  - name: notify_on_failure
    action: |
      echo "Task {task_id} failed: {error}" >> failures.log
      # Or send to monitoring system
```

**Customize cascade policy:**

```yaml
cascade_policy:
  default: propagate       # Block dependents
  exceptions:
    - task_pattern: "*-optional"
      policy: skip         # Don't block dependents
```

**Auto-escalation:**

```yaml
escalation:
  after_failures: 3        # After 3 task failures in session
  action: pause_execution
  message: "Multiple failures detected. Review before continuing."
```

**Failure record location:**

```yaml
failure_record:
  path: .orchestrator/failures/{task_id}.yaml
  include:
    - error_details
    - execution_log
    - partial_outputs
    - suggested_fixes
```
</customization>

<integration>
This hook is invoked by the orchestrator after final task failure.

**Invocation point:**
```
handle_wave_results(wave, results)
→ For each failed task:
   → If attempts >= max_iterations or unrecoverable:
      → Fire on-failure hook
      → Process cascade policy
→ Determine wave outcome
```

**State manager interaction:**

Orchestrator calls state manager operations:
- `task_failed(task_id, error, attempts)`
- `mark_blocked(dependent_task_ids, reason)`

Hook actions flow through state manager for consistency.
</integration>

<recovery_options>
After hook fires, execution typically pauses with these options:

1. **`/ptf:retry {task_id}`**
   - Reset attempt counter
   - Clear blocked status
   - Try task again with fresh context

2. **`/ptf:skip {task_id}`**
   - Mark task as skipped
   - Cascade blocked status to dependents
   - Continue with remaining tasks

3. **`/ptf:edit {task_id}`**
   - Open task definition for modification
   - Adjust requirements or inputs
   - Retry after edits

4. **`/ptf:abort`**
   - Stop execution
   - Preserve state for later resume
   - Exit cleanly
</recovery_options>
