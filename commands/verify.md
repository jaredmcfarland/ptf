---
description: Verify task outputs against declared criteria
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Run verification checks for a specific task or set of tasks.

**Usage:**
- `/ptf:verify task-id` - Verify specific task
- `/ptf:verify --all` - Verify all completed tasks
- `/ptf:verify --wave N` - Verify all tasks in wave N

**Requires:**
- Task definition in `.orchestrator/decomposition/tasks/`
- Task outputs should exist (for non-exists checks)

**After this command:**
- If all pass: Continue execution or commit changes
- If failures: Fix issues and re-verify, or use `/ptf:retry`
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
@.claude/agents/ptf-verifier.md
@.claude/agents/ptf-state-manager.md
</execution_context>

<context>
@.orchestrator/state/execution.yaml (if exists)
@.orchestrator/decomposition/tasks/ (task definitions)
@.orchestrator/state/tasks/ (task state files)
</context>

<process>

## Phase 1: Parse Arguments

Extract verification target from command arguments.

**1.1 Determine verification mode:**

```
IF no arguments:
  Display usage:
    "Usage: /ptf:verify <task-id> | --all | --wave N"
  Exit

IF argument == "--all":
  mode = "all"
  Collect all tasks with status: completed from task state files

IF argument == "--wave" followed by number:
  mode = "wave"
  wave_number = argument value
  Collect all tasks assigned to that wave from graph.yaml

ELSE:
  mode = "single"
  task_id = argument
```

**1.2 Validate task IDs exist:**

```bash
# For single mode:
if [ ! -f ".orchestrator/decomposition/tasks/${task_id}.yaml" ]; then
  echo "ERROR: Task definition not found: ${task_id}"
  exit 1
fi

# For all/wave modes:
# Collect task IDs from state/graph files
# Validate each exists in decomposition/tasks/
```

**1.3 Build task list:**

```
tasks_to_verify = []

IF mode == "single":
  tasks_to_verify.push(task_id)

IF mode == "all":
  # Read .orchestrator/state/tasks/*.yaml
  # Filter: status == "completed"
  # Add each task_id to list

IF mode == "wave":
  # Read .orchestrator/decomposition/graph.yaml
  # Find wave section for wave_number
  # Add each task in wave to list
```

IF tasks_to_verify is empty:
  Display: "No tasks to verify in specified scope."
  Exit

## Phase 2: Load Tasks

Load task definitions and verification criteria for each task.

**2.1 For each task in tasks_to_verify:**

```bash
# Read task definition
cat .orchestrator/decomposition/tasks/${task_id}.yaml
```

Extract from each task:
- `outputs[]` - Expected output artifacts
- `verify[]` - Verification criteria

**2.2 Validate task has verification criteria:**

```
IF task.verify is empty OR undefined:
  Use default verification:
    - type: exists
      target: {each output path}
  Log: "Task {task_id} has no explicit verification, using exists checks"
```

**2.3 Collect verification payloads:**

```
verifications = []
for task in tasks_to_verify:
  verifications.push({
    task_id: task.id,
    outputs: task.outputs,
    verify: task.verify,
    adapter: config.adapter  # For adapter-specific strategies
  })
```

## Phase 3: Dispatch Verifier

Spawn ptf-verifier subagent for each task.

**3.1 For each task, spawn verifier:**

```
Task(prompt="
Verify task: {task_id}

Verification criteria:
```yaml
{yaml_dump(task.verify)}
```

Expected outputs:
```yaml
{yaml_dump(task.outputs)}
```

Adapter: {adapter_name}

Run all verification checks in fail-fast order:
1. exists checks
2. syntax checks
3. contains checks
4. runs checks
5. custom checks

Return VERIFICATION PASSED or VERIFICATION FAILED with detailed results.
", subagent_type="ptf-verifier")
```

**3.2 Collect verifier results:**

Each verifier returns structured result with:
- status: passed | failed
- results: array of check results
- failed_check: first failing check (if any)

Aggregate results:
```
verification_results = []
for each verifier_response:
  verification_results.push({
    task_id: task.id,
    status: parsed_status,
    checks_total: count(results),
    checks_passed: count(results where passed == true),
    results: parsed_results
  })
```

## Phase 4: Record Results

Update task state via state manager for each verified task.

**4.1 For each verification result, update state:**

```
Task(prompt="
Record verification for task: {task_id}

Status: {passed | failed}
Timestamp: {current_timestamp}
Results:
```yaml
{yaml_dump(verification_results)}
```

Update task state file with verification section.
Append verification_completed event to log.
Update artifact manifest verified status.
", operation="record_verification", subagent_type="ptf-state-manager")
```

**4.2 Wait for state manager confirmation:**

Each state update returns:
- Updated task state path
- Event logged confirmation
- Artifacts updated count

## Phase 5: Display Summary

Present verification results to user.

**5.1 Compute aggregates:**

```
total_tasks = len(verification_results)
passed_tasks = count(results where status == "passed")
failed_tasks = count(results where status == "failed")
total_checks = sum(checks_total for all results)
passed_checks = sum(checks_passed for all results)
```

**5.2 Display based on outcome:**

**All tasks pass:**

Display VERIFY COMPLETE with success summary.

**Some tasks fail:**

Display VERIFY FAILED with failure details and remediation steps.

</process>

<structured_returns>

## VERIFY COMPLETE

When all verification checks pass:

```markdown
## VERIFY COMPLETE

**Tasks verified:** {N}
**Status:** All passed

### Results

| Task | Checks | Status |
|------|--------|--------|
| task-1 | 3/3 | PASS |
| task-2 | 2/2 | PASS |

### Verification Details

{task-1}:
- [PASS] exists: src/auth.ts
- [PASS] syntax: TypeScript compiles
- [PASS] contains: "export function authenticate"

### State Updated

Verification results recorded in task state files.
Event logged: verification_completed

### Next Step

All verifications passed. Continue execution or commit changes.
```

---

## VERIFY FAILED

When one or more checks fail:

```markdown
## VERIFY FAILED

**Tasks verified:** {N}
**Passed:** {M}
**Failed:** {F}

### Failures

| Task | Failed Check | Details |
|------|--------------|---------|
| task-1 | contains | Pattern 'export class' not found in src/auth.ts |
| task-3 | runs | Command 'npm test' exited with code 1 |

### Failure Details

{task-1}:
- [PASS] exists: src/auth.ts
- [FAIL] contains: Pattern 'export class' not found
  Expected: "export class AuthService"
  Target: src/auth.ts

{task-3}:
- [PASS] exists: src/lib.ts
- [FAIL] runs: npm test
  Exit code: 1
  Output: "TypeError: Cannot read property..."

### Passed Tasks

- task-2: 2/2 checks

### State Updated

Verification results recorded (including failures).
Failed tasks marked with verification.status: failed

### Next Step

Fix failed tasks and re-run `/ptf:verify {task-id}` or `/ptf:retry {task-id}`.
```

---

## VERIFY SKIPPED

When no tasks match the criteria:

```markdown
## VERIFY SKIPPED

**Reason:** {reason}

### Details

{mode == "all"}:
No completed tasks found. Run `/ptf:execute` first.

{mode == "wave"}:
Wave {N} has no tasks or tasks not yet executed.

{mode == "single"}:
Task {task_id} not found or has no outputs.

### Suggestion

Run `/ptf:status` to see current execution state.
```

</structured_returns>

<error_handling>

## Common Error Conditions

### Task not found

```
ERROR: Task definition not found: {task_id}
→ Check task ID spelling. Run /ptf:status to see available tasks.
```

### No verification criteria

```
WARNING: Task {task_id} has no explicit verification
→ Using default exists checks for declared outputs.
```

### Verifier subagent failure

```
ERROR: Verifier failed to complete for task {task_id}
→ Check task definition for valid verification criteria.
→ Ensure output paths are correct.
```

### State manager unavailable

```
ERROR: Could not record verification results
→ Results computed but not persisted.
→ Re-run /ptf:verify to retry state recording.
```

### Wave not found

```
ERROR: Wave {N} not found in execution plan
→ Check wave number. Run /ptf:plan to see wave assignments.
```

</error_handling>

<state_integration>

## State Files Involved

**Read:**
- `.orchestrator/decomposition/tasks/{task-id}.yaml` - Task definitions
- `.orchestrator/decomposition/graph.yaml` - Wave assignments
- `.orchestrator/state/tasks/{task-id}.yaml` - Task state (for --all mode)
- `.orchestrator/config.yaml` - Adapter configuration

**Modified (via state manager):**
- `.orchestrator/state/tasks/{task-id}.yaml` - Updated with verification section
- `.orchestrator/artifacts/manifest.yaml` - Artifact verified status
- `.orchestrator/history/events.jsonl` - verification_completed events

## Verification Section Schema

Task state files include verification results:

```yaml
verification:
  status: passed | failed
  last_verified: 2026-01-18T10:36:00Z
  results:
    - type: exists
      target: src/auth.ts
      passed: true
      message: "File exists"
    - type: contains
      target: src/auth.ts
      expected: "export function"
      passed: true
      message: "Pattern found at line 15"
```

</state_integration>

<success_criteria>
- [ ] Arguments parsed correctly (single, --all, --wave)
- [ ] Task definitions loaded with verification criteria
- [ ] Verifier subagent dispatched for each task
- [ ] Results aggregated correctly
- [ ] State manager records all verification results
- [ ] Summary displayed with appropriate format
- [ ] Error conditions handled with helpful messages
</success_criteria>
