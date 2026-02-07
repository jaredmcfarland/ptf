---
name: on-failure
description: Hook fired when a task fails after all retries exhausted
trigger: task_failure_exhausted
hook_id: HOOK-03
# NOTE: This file is a design specification, not executable code.
# Actual behavior is implemented by ptf:orchestrator and ptf:state-manager.
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
Actions performed when this hook fires (invoked by orchestrator):

**1. Create failure record (state manager)**

```
Task(prompt="
Create failure record for task {task_id}.

Task: {task_id}
Name: {task_name}
Wave: {wave}
Attempt: {attempt}
Started: {started}
Duration: {duration_s}

Failure mode: {failure_mode}
Error type: {error_type}
Error category: {error_category}
Error message: {error}

Context:
- Inputs loaded: {inputs_loaded}
- Files written: {outputs_partial}

Next action: {based on strategy}
", subagent_type="ptf:state-manager")
```

This creates `.orchestrator/failures/{task_id}-attempt-{N}.yaml`.

**2. Apply cascade policy (state manager)**

If task's `on_failure.propagate_failure` is true (default):

```
Task(prompt="
Apply cascade policy for failed task {task_id}.

1. Find all tasks that depend on {task_id} in graph.yaml

2. For each dependent task:
   - Check if dependency is marked required: false
   - If required (default): mark task as blocked
   - If not required: log optional dependency skipped

Invoke mark_blocked for each dependent:
- blocked_by: {task_id}
- reason: cascade_failure

Return list of blocked tasks.
", subagent_type="ptf:state-manager")
```

**3. Determine next action**

Based on task's `on_failure` policy:

| Strategy | Retries Left | Action |
|----------|--------------|--------|
| retry | Yes | Mark task ready, calculate backoff delay |
| retry | No | Apply final_fallback (escalate or skip) |
| skip | N/A | Mark task skipped, continue execution |
| escalate | N/A | Pause execution, present options to user |

**4. Log events**

Events logged by state manager operations:
- task_failed (logged by orchestrator before hook)
- failure_record_created (from create_failure_record)
- task_blocked (from mark_blocked, for each dependent)
- task_skipped (if skip strategy applied)

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

<failure_record_format>
## Failure Record Structure

Records are created at `.orchestrator/failures/{task_id}-attempt-{N}.yaml`:

```yaml
task_id: auth-service
task_name: Create Authentication Service
wave: 2
attempt: 2

started: 2025-01-17T10:40:00Z
failed: 2025-01-17T10:42:30Z
duration_s: 150

failure_mode: verification_failure

error:
  type: verification_failed
  category: contains
  target: /src/services/authService.ts
  expected: "export function authenticate"
  message: |
    Pattern not found in file.
    File exists but does not contain expected export.

context:
  inputs_loaded:
    - /src/types/user.ts
    - /src/config/auth.ts
  files_written:
    - /src/services/authService.ts (partial)

recovery_action: retry
next_attempt: 3
```

## Failure Modes

| Mode | Trigger | Typical Recovery |
|------|---------|------------------|
| verification_failure | Output doesn't match expected | Retry or manual fix |
| execution_error | Error during task execution | Review error, retry |
| missing_input | Required input file missing | Check prior task |
| timeout | Task exceeded time limit | Simplify or increase limit |
| max_iterations | Ralph loop exhausted | Manual intervention |
| unclear_requirements | Can't parse task instructions | Clarify task definition |

</failure_record_format>

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
## Orchestrator Integration

The orchestrator invokes this hook's logic through the handle_failure operation:

```
# In ptf:orchestrator.md handle_wave_results:

for result in task_results where result.status == "failed":

  # 1. Create failure record
  Task(prompt="Create failure record...", subagent_type="ptf:state-manager")

  # 2. Read task's on_failure policy
  policy = read_on_failure_policy(result.task_id)

  # 3. Apply strategy
  if policy.strategy == "retry" and result.attempt < policy.max_attempts:
    backoff = calculate_backoff(result.attempt, policy)
    # Task will retry after backoff delay
    continue_wave = true

  elif policy.strategy == "skip" or (exhausted and policy.final_fallback == "skip"):
    Task(prompt="Mark task skipped...", subagent_type="ptf:state-manager")
    if policy.propagate_failure:
      apply_cascade(result.task_id)
    continue_wave = true

  else:  # escalate
    return "pause"  # Triggers escalation output
```

## Hook File Purpose

This file documents the hook's behavior. The actual execution logic lives in:
- ptf:orchestrator.md (handle_failure operation)
- ptf:state-manager.md (mark_blocked, create_failure_record operations)

The hook file serves as:
1. Documentation of trigger conditions
2. Reference for customization options
3. Event format specification
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
