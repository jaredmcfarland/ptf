---
name: on-session-end
description: Hook fired when execution ends (complete, abort, or interrupt)
trigger: session_end
hook_id: HOOK-04
# NOTE: This file is a design specification, not executable code.
# Actual behavior is implemented by ptf:orchestrator and ptf:state-manager.
---

<when>
This hook fires when an execution session ends, regardless of the reason.

**Trigger conditions:**
- All waves complete successfully (plan complete)
- Execution aborted via /ptf:abort
- Execution paused due to failures
- Session interrupted (Ctrl-C, timeout, error)

**End reasons:**
- `complete` - Plan finished successfully
- `paused` - Stopped due to task failures
- `blocked` - Cannot proceed, intervention needed
- `abort` - User explicitly aborted
- `interrupt` - Session was interrupted unexpectedly
</when>

<actions>
Default actions performed when this hook fires:

1. **final_checkpoint**
   - Write final state to all state files
   - Ensure execution.yaml reflects final status
   - Guarantee state consistency for resume

2. **cleanup_temp_files**
   - Remove temporary files created during execution
   - Clean up partial outputs from failed tasks (configurable)
   - Preserve important artifacts

3. **summary_report**
   - Generate session summary
   - Include: duration, waves completed, tasks completed/failed
   - Write to events.jsonl as session_ended event
</actions>

<context_available>
Data available to this hook:

| Variable | Type | Description |
|----------|------|-------------|
| `end_reason` | string | complete, paused, blocked, abort, interrupt |
| `session_id` | string | Current session identifier |
| `session_started` | timestamp | When session began |
| `session_duration_s` | number | Total session duration |
| `waves_completed` | number | Waves finished this session |
| `waves_total` | number | Total waves in plan |
| `tasks_completed` | number | Tasks completed this session |
| `tasks_failed` | number | Tasks failed this session |
| `tasks_remaining` | number | Tasks not yet executed |
| `artifacts_produced` | number | Artifacts registered |
| `final_status` | string | Final execution status |
</context_available>

<event_format>
Event logged to events.jsonl:

```json
{
  "ts": "2026-01-18T11:30:00Z",
  "event": "session_ended",
  "session": "session-003",
  "end_reason": "complete",
  "duration_s": 3600,
  "waves_completed": 5,
  "tasks_completed": 12,
  "tasks_failed": 0,
  "artifacts_produced": 24,
  "final_status": "completed"
}
```

For interrupted session:

```json
{
  "ts": "2026-01-18T11:30:00Z",
  "event": "session_ended",
  "session": "session-003",
  "end_reason": "interrupt",
  "duration_s": 1847,
  "waves_completed": 2,
  "tasks_completed": 6,
  "tasks_failed": 0,
  "artifacts_produced": 12,
  "final_status": "running",
  "checkpoint_wave": 2,
  "resume_point": "wave-3"
}
```
</event_format>

<cleanup_policy>
## Temporary File Cleanup

**Always removed:**
- `.orchestrator/temp/*` - Temporary work files
- `.orchestrator/cache/*` - Cached computations

**Removed on complete:**
- Task attempt logs (success path doesn't need history)
- Intermediate outputs superseded by final outputs

**Preserved on failure/interrupt:**
- Partial outputs from failed tasks (for debugging)
- Task attempt logs (for investigation)
- State files (for resume)

**Never removed:**
- State files (.orchestrator/state/*)
- Event log (events.jsonl)
- Registered artifacts
- Configuration files
</cleanup_policy>

<customization>
To customize this hook's behavior, edit this file.

**Add completion notification:**

```yaml
custom_actions:
  - name: notify_complete
    when: end_reason == "complete"
    action: |
      echo "PTF execution complete!" | mail -s "PTF Done" user@example.com
```

**Generate detailed report:**

```yaml
custom_actions:
  - name: generate_report
    action: |
      cat << EOF > .orchestrator/session-report.md
      # Execution Report

      Session: {session_id}
      Duration: {session_duration_s}s
      Status: {final_status}

      ## Summary
      - Waves: {waves_completed}/{waves_total}
      - Tasks: {tasks_completed} completed, {tasks_failed} failed
      - Artifacts: {artifacts_produced}
      EOF
```

**Custom cleanup rules:**

```yaml
cleanup:
  on_complete:
    remove:
      - ".orchestrator/temp/*"
      - ".orchestrator/cache/*"
    preserve:
      - ".orchestrator/state/*"

  on_failure:
    preserve_all: true   # Keep everything for debugging
```

**Skip cleanup (debugging):**

```yaml
actions:
  - final_checkpoint
  # - cleanup_temp_files   # Disabled for debugging
  - summary_report
```
</customization>

<integration>
This hook is invoked at the very end of execution.

**Invocation point:**
```
execute_plan() / execute_wave()
→ Complete or error
→ Fire on-session-end hook
→ Return final result to command
```

**State manager interaction:**

Final checkpoint ensures all state is persisted:
- `checkpoint_wave(current_wave, results)` if mid-wave
- Update execution.yaml with final_status
- Flush events.jsonl

Hook runs AFTER state manager operations complete.
</integration>

<resume_preparation>
When end_reason is not "complete", this hook prepares for resume:

1. **Mark interrupted tasks:**
   - Tasks with status "running" are noted
   - Resume will re-execute these tasks

2. **Record resume point:**
   - Current wave and task positions saved
   - execution.yaml updated with resume hint

3. **Validate checkpoint:**
   - Verify state files are consistent
   - Report any inconsistencies in summary

**Resume command:**
After interruption, user runs `/ptf:resume` which:
1. Reads execution.yaml for resume point
2. Validates completed artifacts
3. Continues from interrupted position
</resume_preparation>

<summary_output>
When execution ends, display summary based on reason:

**On complete:**
```
Execution complete!
- Duration: 45m 30s
- Waves: 5/5 (100%)
- Tasks: 12 completed, 0 failed
- Artifacts: 24 registered

Run /ptf:verify for final verification.
```

**On pause/abort:**
```
Execution paused.
- Progress: Wave 3 of 5
- Tasks: 8 completed, 1 failed
- State saved at wave 3 checkpoint

Run /ptf:resume to continue or /ptf:status for details.
```

**On interrupt:**
```
Session interrupted.
- Last checkpoint: Wave 2
- State preserved for resume

Run /ptf:resume when ready to continue.
```
</summary_output>
