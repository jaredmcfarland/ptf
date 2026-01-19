---
name: post-task-complete
description: Hook fired after each task completes (success or failure)
trigger: task_completion
hook_id: HOOK-01
---

<when>
This hook fires immediately after a task executor returns, regardless of whether
the task succeeded or failed.

**Trigger conditions:**
- Task executor outputs `VERIFICATION PASSED`
- Task executor outputs `BLOCKED: [reason]`
- Task executor times out or errors

**Not triggered by:**
- Task still running (intermediate state)
- Task not yet started
</when>

<actions>
Default actions performed when this hook fires:

1. **log_event**
   - Append task completion event to events.jsonl
   - Include: task_id, status, duration, attempt number

2. **update_state**
   - Update task state file (.orchestrator/state/tasks/{task-id}.yaml)
   - Set status to completed, failed, or blocked
   - Record completion timestamp

3. **register_artifacts**
   - For successful tasks: register produced artifacts in manifest.yaml
   - Compute checksums for each artifact
   - Record artifact producer task
</actions>

<context_available>
Data available to this hook:

| Variable | Type | Description |
|----------|------|-------------|
| `task_id` | string | Task identifier |
| `task_name` | string | Human-readable task name |
| `wave` | number | Wave this task belongs to |
| `status` | enum | completed, failed, blocked |
| `result` | object | Full executor return |
| `result.outputs` | array | Paths to produced files |
| `result.error` | string | Error message (if failed/blocked) |
| `result.attempt` | number | Attempt number (for Ralph mode) |
| `result.duration_s` | number | Execution duration in seconds |
| `result.verification` | object | Verification step results |
</context_available>

<event_format>
Event logged to events.jsonl:

```json
{
  "ts": "2026-01-18T10:32:00Z",
  "event": "task_completed",
  "task": "auth-schema",
  "wave": 1,
  "status": "completed",
  "duration_s": 45,
  "attempt": 1,
  "outputs": ["src/schema.ts"]
}
```

For failures:

```json
{
  "ts": "2026-01-18T10:32:00Z",
  "event": "task_failed",
  "task": "auth-service",
  "wave": 2,
  "status": "failed",
  "duration_s": 30,
  "attempt": 3,
  "error": "verification_failed: contains check failed"
}
```
</event_format>

<customization>
To customize this hook's behavior, edit this file.

**Add custom actions:**

```yaml
# Example: Send notification on completion
custom_actions:
  - name: notify_slack
    when: status == "completed"
    action: |
      curl -X POST $SLACK_WEBHOOK -d '{"text": "Task {task_id} completed"}'
```

**Modify default behavior:**

Edit the <actions> section to change what happens on task completion.

**Disable actions:**

Remove or comment out actions you don't want:

```yaml
actions:
  # - log_event        # Disabled
  - update_state
  - register_artifacts
```

**Conditional logic:**

Add when clauses to make actions conditional:

```yaml
actions:
  - name: register_artifacts
    when: status == "completed"
```
</customization>

<integration>
This hook is invoked by the orchestrator after each task executor returns.

**Invocation point:**
```
After dispatch_task() returns
→ Parse executor result
→ Fire post-task-complete hook
→ Continue with next task or wave handling
```

**State manager interaction:**

The orchestrator calls state manager operations based on hook actions:
- `task_completed(task_id, outputs)` for successful tasks
- `task_failed(task_id, error, attempt)` for failed tasks

Hook actions are processed via state manager, not directly.
</integration>
