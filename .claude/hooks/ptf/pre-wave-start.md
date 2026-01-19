---
name: pre-wave-start
description: Hook fired before first task in a wave is dispatched
trigger: wave_start
hook_id: HOOK-02
# NOTE: This file is a design specification, not executable code.
# Actual behavior is implemented by ptf-orchestrator and ptf-state-manager.
---

<when>
This hook fires at the beginning of wave execution, before any tasks are dispatched.

**Trigger conditions:**
- Orchestrator begins executing a wave
- Wave dependencies validated and satisfied
- About to dispatch first task

**Not triggered by:**
- Wave already in progress (mid-wave)
- Wave skipped due to dependency failure
- Resume from interrupted state (fires on fresh start only)
</when>

<actions>
Default actions performed when this hook fires:

1. **checkpoint_state**
   - Checkpoint current execution state before starting wave
   - Ensures previous wave results are persisted
   - Atomic write of all state files

2. **validate_dependencies**
   - Verify all input dependencies for wave tasks exist
   - Check artifact checksums match expected values
   - Report any missing or corrupted dependencies

3. **log_wave_start**
   - Append wave_started event to events.jsonl
   - Include: wave number, task count, timestamp
</actions>

<context_available>
Data available to this hook:

| Variable | Type | Description |
|----------|------|-------------|
| `wave` | number | Wave number about to start |
| `tasks` | array | Task IDs in this wave |
| `task_count` | number | Number of tasks in wave |
| `depends_on_waves` | array | Prerequisite wave numbers |
| `previous_wave_status` | string | Status of prior wave |
| `expected_inputs` | array | Input files required by wave tasks |
| `execution_status` | string | Overall execution status |
</context_available>

<event_format>
Event logged to events.jsonl:

```json
{
  "ts": "2026-01-18T10:35:00Z",
  "event": "wave_started",
  "wave": 2,
  "tasks": ["auth-service", "config-loader", "user-model"],
  "task_count": 3,
  "depends_on_waves": [1]
}
```

Pre-wave validation event:

```json
{
  "ts": "2026-01-18T10:34:59Z",
  "event": "wave_validated",
  "wave": 2,
  "dependencies_checked": 5,
  "all_satisfied": true
}
```
</event_format>

<customization>
To customize this hook's behavior, edit this file.

**Add pre-execution setup:**

```yaml
custom_actions:
  - name: setup_environment
    action: |
      # Load environment variables for this wave
      source .env.test

  - name: clear_cache
    when: wave > 1
    action: |
      rm -rf .cache/
```

**Add validation rules:**

```yaml
custom_validations:
  - name: check_disk_space
    check: |
      [ $(df -k . | tail -1 | awk '{print $4}') -gt 1000000 ]
    error: "Insufficient disk space for wave execution"
```

**Skip checkpoint for speed (not recommended):**

```yaml
actions:
  # - checkpoint_state    # Skipped - DANGEROUS
  - validate_dependencies
  - log_wave_start
```

Warning: Skipping checkpoint risks losing progress on interruption.
</customization>

<integration>
This hook is invoked by the orchestrator at wave start.

**Invocation point:**
```
execute_wave(wave_number)
→ Validate wave dependencies
→ Fire pre-wave-start hook
→ Invoke state manager: start_wave
→ Dispatch tasks
```

**State manager interaction:**

After hook actions complete, orchestrator calls:
- `start_wave(wave, tasks)` to initialize wave state

Hook actions may influence whether wave proceeds:
- If validate_dependencies fails, wave is blocked
- If checkpoint fails, execution should halt
</integration>

<failure_handling>
If pre-wave-start hook fails:

1. **Dependency validation failure:**
   - Wave is marked blocked
   - Missing dependencies listed
   - Execution paused for investigation

2. **Checkpoint failure:**
   - Critical error - execution should stop
   - Cannot guarantee state consistency
   - Return EXECUTION BLOCKED

3. **Custom action failure:**
   - Logged as warning
   - By default, wave proceeds (soft failure)
   - Can configure to block on custom action failure
</failure_handling>
