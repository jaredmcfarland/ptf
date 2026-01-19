---
name: ptf:resume
description: Resume PTF execution from last checkpoint after interruption
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Continue execution from last checkpoint after interruption.

This command:
- Validates artifacts exist and are unchanged before skipping completed tasks
- Handles interrupted tasks (were "running" when session ended)
- Safe: won't re-execute completed work unless artifacts are invalid
- Creates new session ID for the resumed execution

**After this command:** `/ptf:execute` continues from current wave.
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
</execution_context>

<context>
@.orchestrator/state/execution.yaml
@.orchestrator/artifacts/manifest.yaml
@.orchestrator/config.yaml
</context>

<process>

## Phase 1: Load State

Check that execution state exists and determine current status.

```bash
if [ ! -f .orchestrator/state/execution.yaml ]; then
  echo "ERROR: No execution state found."
  echo "Next: Run /ptf:execute to start execution first."
  exit 1
fi
```

Read the master state file:
```bash
cat .orchestrator/state/execution.yaml
```

Extract:
- status: current execution status
- current_wave: which wave we're on
- session: previous session info
- blockers: any existing blockers

## Phase 2: Handle Terminal States

Check if execution is in a terminal state (completed or failed).

**If status == "completed":**

```markdown
# Resume Check

Execution already complete. Nothing to resume.

**Status:** completed
**Total Tasks:** {tasks_total}
**Completed:** {tasks_completed}

---

All tasks finished successfully. No action needed.
```

Exit with success - nothing to do.

**If status == "failed":**

Report the blockers and ask user how to proceed:

```markdown
# Resume Check

Execution is in failed state with blockers.

**Status:** failed
**Blockers:**
{for each blocker in blockers array:}
- {blocker}

## Options

1. **retry** - Clear failed status, retry failed tasks from current wave
2. **skip** - Mark failed tasks as skipped, continue to next wave
3. **abort** - Exit without changing state
```

Use response to determine action:

- **retry**: Update failed tasks to status: ready, clear blockers, continue
- **skip**: Update failed tasks to status: skipped, continue to next wave
- **abort**: Exit without changes

After handling choice, continue to Phase 3.

## Phase 3: Validate Completed Artifacts

Spawn state manager to validate all artifacts from completed tasks.

```
Task(prompt="
Validate all artifacts in .orchestrator/artifacts/manifest.yaml

For each artifact:
1. Check file exists at path
2. Compute checksum: shasum -a 256 {path}
3. Compare with stored checksum (format: sha256:{hash})

Return VALIDATION RESULTS with:
- Total artifacts checked
- Valid count
- Invalid list (path, reason, producing task)
", subagent_type="ptf-state-manager")
```

**Process validation results:**

If all artifacts valid:
- Log artifact_verified events for each
- Continue to Phase 4

If any artifacts invalid (missing or checksum mismatch):

1. Identify the producing task for each invalid artifact
2. Update those task states to status: ready
3. Append artifact_validation_failed events to log:
   ```bash
   echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","event":"artifact_validation_failed","path":"'${PATH}'","task":"'${TASK_ID}'","reason":"'${REASON}'"}' >> .orchestrator/history/events.jsonl
   ```
4. Report which tasks will be re-executed

```markdown
## Artifact Validation Failed

{count} artifact(s) require regeneration:

| Artifact | Issue | Producing Task |
|----------|-------|----------------|
| {path} | Missing | {task-id} |
| {path} | Checksum mismatch | {task-id} |

These tasks will be re-executed to regenerate artifacts.
```

## Phase 4: Handle Interrupted Tasks

Tasks with status: running were interrupted mid-execution when the previous session ended.

Check for interrupted tasks:
```bash
grep -l "^status: running" .orchestrator/state/tasks/*.yaml 2>/dev/null
```

For each interrupted task:

1. Check if all expected outputs exist and are valid:
   ```bash
   # Read task file for expected outputs
   outputs=$(grep -A10 "^outputs:" .orchestrator/decomposition/tasks/${TASK_ID}.yaml)
   # Check each output path exists and compute checksum
   ```

2. **If outputs valid (task completed but state not updated):**
   - Mark task as completed
   - Register artifacts in manifest
   - Append task_completed event:
     ```bash
     echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","event":"task_completed","task":"'${TASK_ID}'","wave":'${WAVE}',"recovered":true}' >> .orchestrator/history/events.jsonl
     ```

3. **If outputs invalid or missing (task needs retry):**
   - Mark task as ready (will be picked up in current wave)
   - Append task_retry event:
     ```bash
     echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","event":"task_retry","task":"'${TASK_ID}'","wave":'${WAVE}',"reason":"interrupted"}' >> .orchestrator/history/events.jsonl
     ```

Report handling results:

```markdown
## Interrupted Task Recovery

Found {count} task(s) that were running when session ended:

| Task | Status | Action |
|------|--------|--------|
| {task-id} | Outputs valid | Marked completed |
| {task-id} | Outputs missing | Will retry |
```

## Phase 5: Prepare Continuation

Set up new session and prepare for execution to continue.

**1. Generate new session ID:**
```bash
NEW_SESSION_ID="session-$(date +%s | tail -c 6)"
```

**2. Update session info in execution.yaml:**
```yaml
session:
  id: {new-session-id}
  started: {now}
  last_update: {now}
  resumed_from: {previous-session-id}
```

**3. Determine resume point:**
```bash
# Check current wave status
CURRENT_WAVE=$(grep "^current_wave:" .orchestrator/state/execution.yaml | awk '{print $2}')
PENDING_IN_WAVE=$(grep -l "^status: \\(ready\\|pending\\)" .orchestrator/state/tasks/*.yaml 2>/dev/null | wc -l)

if [ "$PENDING_IN_WAVE" -gt 0 ]; then
  echo "Resuming wave ${CURRENT_WAVE} with ${PENDING_IN_WAVE} pending tasks"
else
  NEXT_WAVE=$((CURRENT_WAVE + 1))
  echo "Wave ${CURRENT_WAVE} complete, advancing to wave ${NEXT_WAVE}"
fi
```

**4. Append session_resumed event:**
```bash
echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","event":"session_resumed","session":"'${NEW_SESSION_ID}'","from_session":"'${OLD_SESSION}'","from_wave":'${CURRENT_WAVE}'}' >> .orchestrator/history/events.jsonl
```

**5. Update execution.yaml with new session:**
Write updated execution.yaml with new session info.

**6. Report resume status:**

```markdown
## Resume Summary

**New Session:** {new-session-id}
**Previous Session:** {previous-session-id}
**Resuming from:** Wave {N}

### Validation Results

- Artifacts checked: {count}
- Artifacts valid: {count}
- Tasks re-validated: {count}
- Tasks to retry: {count}

### Ready to Continue

State validated and session resumed.

---

**Next:** Run `/ptf:execute` to continue execution from wave {N}
```

</process>

<validation_logic>

## Checksum Validation

Compute SHA-256 checksums for artifact validation:

```bash
# Cross-platform checksum computation (macOS/Linux compatible)
compute_checksum() {
  local file_path="$1"

  if command -v shasum &> /dev/null; then
    # macOS and some Linux
    shasum -a 256 "$file_path" | cut -d' ' -f1
  elif command -v sha256sum &> /dev/null; then
    # Most Linux distros
    sha256sum "$file_path" | cut -d' ' -f1
  else
    echo "ERROR: No SHA-256 tool available"
    exit 1
  fi
}

# Validate a single artifact
validate_artifact() {
  local path="$1"
  local stored_checksum="$2"  # Format: sha256:abc123...

  # Check file exists
  if [ ! -f "$path" ]; then
    echo "MISSING:$path"
    return 1
  fi

  # Compute actual checksum
  local actual=$(compute_checksum "$path")
  local expected="${stored_checksum#sha256:}"  # Strip prefix

  if [ "$actual" != "$expected" ]; then
    echo "MISMATCH:$path:expected=${expected}:actual=${actual}"
    return 1
  fi

  echo "VALID:$path"
  return 0
}
```

## Validation Workflow

For each artifact in manifest.yaml:
1. Extract path and stored checksum
2. Call validate_artifact
3. Collect results
4. Report summary

```bash
# Example validation loop
MANIFEST=".orchestrator/artifacts/manifest.yaml"
VALID_COUNT=0
INVALID_COUNT=0
INVALID_TASKS=()

# Parse manifest and validate each artifact
while IFS= read -r line; do
  if [[ "$line" =~ ^[[:space:]]*-[[:space:]]path: ]]; then
    CURRENT_PATH="${line#*: }"
  elif [[ "$line" =~ ^[[:space:]]*checksum: ]]; then
    CURRENT_CHECKSUM="${line#*: }"
  elif [[ "$line" =~ ^[[:space:]]*produced_by: ]]; then
    CURRENT_TASK="${line#*: }"

    # Validate this artifact
    RESULT=$(validate_artifact "$CURRENT_PATH" "$CURRENT_CHECKSUM")
    if [[ "$RESULT" == VALID:* ]]; then
      ((VALID_COUNT++))
    else
      ((INVALID_COUNT++))
      INVALID_TASKS+=("$CURRENT_TASK")
    fi
  fi
done < "$MANIFEST"

echo "Valid: $VALID_COUNT, Invalid: $INVALID_COUNT"
```

</validation_logic>

<interrupted_task_handling>

## Detecting Interrupted Tasks

Tasks with status "running" in their state file were interrupted:

```bash
# Find interrupted tasks
INTERRUPTED_TASKS=$(grep -l "^status: running" .orchestrator/state/tasks/*.yaml 2>/dev/null)

for TASK_FILE in $INTERRUPTED_TASKS; do
  TASK_ID=$(basename "$TASK_FILE" .yaml)
  echo "Found interrupted task: $TASK_ID"
done
```

## Recovering Interrupted Tasks

For each interrupted task, check if its outputs were actually produced:

```bash
recover_task() {
  local task_id="$1"
  local task_file=".orchestrator/state/tasks/${task_id}.yaml"
  local definition=".orchestrator/decomposition/tasks/${task_id}.yaml"

  # Get expected outputs from task definition
  local outputs=$(grep -A20 "^outputs:" "$definition" | grep "path:" | awk '{print $2}')

  local all_valid=true
  for output_path in $outputs; do
    if [ ! -f "$output_path" ]; then
      all_valid=false
      break
    fi
    # Could also validate checksums if we have expected values
  done

  if [ "$all_valid" = true ]; then
    # Task completed but state wasn't updated
    echo "RECOVERED:$task_id"
    # Mark completed, register artifacts
  else
    # Task needs to be retried
    echo "RETRY:$task_id"
    # Mark ready for retry
  fi
}
```

## State Updates for Recovery

**Marking recovered task as completed:**
```yaml
# .orchestrator/state/tasks/{task-id}.yaml
task_id: {id}
wave: {N}
status: completed
recovered: true
recovered_at: {timestamp}
attempts:
  - attempt: 1
    started: {original_start}
    completed: {now}
    status: recovered
outputs_produced:
  - path: {output_path}
    checksum: sha256:{computed}
```

**Marking task for retry:**
```yaml
# .orchestrator/state/tasks/{task-id}.yaml
task_id: {id}
wave: {N}
status: ready
retry_reason: interrupted
attempts:
  - attempt: 1
    started: {original_start}
    status: interrupted
```

</interrupted_task_handling>

<success_criteria>
- [ ] Execution state loaded from execution.yaml
- [ ] Completed state handled (reports nothing to do)
- [ ] Failed state handled with user options (retry/skip/abort)
- [ ] All artifacts validated with checksums
- [ ] Invalid artifacts trigger task re-execution
- [ ] Interrupted tasks (status: running) detected and handled
- [ ] Interrupted task outputs checked for recovery
- [ ] New session ID generated
- [ ] session_resumed event logged
- [ ] Resume point determined (current wave or next)
- [ ] Clear summary of validation and recovery actions
- [ ] User knows to run /ptf:execute next
</success_criteria>
