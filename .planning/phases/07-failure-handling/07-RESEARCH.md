# Phase 7: Failure Handling - Research

**Researched:** 2026-01-18
**Domain:** Failure recovery strategies, retry policies, cascade handling, human escalation
**Confidence:** HIGH

## Summary

Phase 7 implements graceful failure handling with retry, skip, escalate, and replan strategies. Research confirms the PTF founding document (Section 8) provides comprehensive specifications for failure modes, recovery strategies, cascade behavior, and the failure record format. The existing implementation (Phase 5 execution engine, Phase 6 verification) already has hooks and placeholders for failure handling that need to be fully implemented.

Key findings:
- **Four failure strategies** defined: retry (with exponential backoff), skip (with cascade options), escalate (pause for human), replan (re-decompose)
- **Cascade handling** is critical: failed tasks can block dependent tasks, controlled by per-task `propagate_failure` policy
- **Failure records** in `.orchestrator/failures/` directory provide debugging context
- **Human escalation** presents clear options (retry, skip, abort, replan) with task context
- **Existing infrastructure** (state manager, hooks, executor) needs extension, not replacement
- **Commands required**: `/ptf:retry`, `/ptf:abort` (new), `/ptf:resume` (extend for failure recovery)

The failure handling subsystem closes the loop on robust execution. Without it, any task failure terminates the entire plan with no recovery path.

**Primary recommendation:** Implement failure handling as five components:
1. Failure strategy handlers in orchestrator (retry, skip, escalate, replan)
2. `/ptf:retry [task]` command for manual task retry
3. `/ptf:abort` command for clean execution stop
4. Failure record creation in state manager
5. Cascade policy enforcement in orchestrator

## Standard Stack

### Core Components
| Component | Source | Purpose | Why Standard |
|-----------|--------|---------|--------------|
| Exponential backoff | Algorithm pattern | Retry delay calculation | Standard distributed systems pattern |
| YAML failure records | PTF convention | Debugging context | Human-readable, grep-able |
| JSONL failure events | Phase 4 | Audit trail | Append-only, structured |
| Shell exit codes | Unix convention | Strategy decision trigger | 0 = success, non-0 = failure |

### From Prior Phases
| Component | Phase | Purpose | Integration |
|-----------|-------|---------|-------------|
| task-state.schema.yaml | Phase 4 | Status field includes failed, blocked, skipped | Direct use |
| execution-state.schema.yaml | Phase 4 | status: failed, blockers array | Direct use |
| on-failure hook | Phase 5 (HOOK-03) | Trigger failure handling | Invoke on task failure |
| ptf-state-manager | Phase 4 | State persistence | task_failed, mark_blocked operations |
| ptf-orchestrator | Phase 5 | Wave execution | handle_failure integration |
| task.schema.yaml | Phase 1 | on_failure field with strategy, max_attempts | Read for policy |

### From Founding Document (Section 8)
| Component | Section | Purpose | Implementation |
|-----------|---------|---------|----------------|
| RetryStrategy | 8.2 | Retry with backoff | max_attempts, backoff type, backoff_base_seconds |
| SkipStrategy | 8.2 | Continue without task | propagate_failure flag |
| EscalateStrategy | 8.2 | Human decision point | message, pause_execution, options array |
| ReplanStrategy | 8.2 | Re-decompose portion | scope, preserve_completed |
| Failure modes | 8.1 | Classification | verification_failure, execution_failure, partial_output, dependency_failure, timeout |
| Cascade behavior | 8.4 | Dependent task handling | Default cascade, optional dependency.required: false |

## Architecture Patterns

### Recommended Project Structure

```
.claude/
├── commands/ptf/
│   ├── retry.md                   # /ptf:retry [task] command
│   ├── abort.md                   # /ptf:abort command
│   └── resume.md                  # (extend for failure recovery)
│
├── agents/
│   ├── ptf-orchestrator.md        # (extend with failure handling)
│   └── ptf-state-manager.md       # (extend with failure operations)
│
└── hooks/ptf/
    └── on-failure.md              # (already exists, complete implementation)

.orchestrator/
└── failures/
    └── {task-id}-attempt-{N}.yaml # Per-attempt failure records
```

### Pattern 1: Failure Strategy Decision Tree

**What:** Determine recovery action based on failure mode and task policy.

**When to use:** After task fails (executor returns BLOCKED or verification fails).

**Implementation (from founding document Section 8.5):**

```
handle_failure(task, error):

  1. LOG failure event with full context

  2. CREATE failure record in failures/

  3. DETERMINE strategy from task.on_failure
     - Get failure_mode from error category
     - Look up strategy for this mode
     - Fall back to final_fallback if no specific handler

  4. IF strategy == "retry" AND attempts < max_attempts:
     → Calculate backoff delay
     → Mark task as "ready"
     → Increment attempt counter
     → Return "retry" (will retry on next dispatch)

  5. IF strategy == "skip":
     → Mark task as "skipped"
     → IF propagate_failure:
         → Mark all dependents as "blocked:dependency_failure"
     → Return "continue" (proceed with remaining tasks)

  6. IF strategy == "escalate":
     → Pause execution
     → Present failure to human with options
     → Wait for human decision
     → Execute chosen option
     → Return based on chosen option

  7. IF strategy == "replan":
     → Trigger re-decomposition
     → Preserve completed work
     → Return "replan_triggered"
```

### Pattern 2: Exponential Backoff

**What:** Increase delay between retry attempts to avoid hammering.

**When to use:** Retry strategy with backoff: exponential.

**Implementation:**

```
calculate_backoff(attempt, config):
  backoff_type = config.backoff or "exponential"
  base_seconds = config.backoff_base_seconds or 2

  if backoff_type == "none":
    return 0
  elif backoff_type == "linear":
    return base_seconds * attempt
  elif backoff_type == "exponential":
    return base_seconds * (2 ** (attempt - 1))
    # attempt 1: 2s, attempt 2: 4s, attempt 3: 8s, etc.
```

**Cap recommendation:** Max backoff of 60 seconds to avoid excessive waits.

### Pattern 3: Cascade Policy Enforcement

**What:** Block dependent tasks when prerequisite fails.

**When to use:** After task definitively fails (retries exhausted or unrecoverable).

**Implementation (from founding document Section 8.4):**

```
handle_cascade(failed_task):
  # Find all tasks that depend on this one
  dependents = find_dependents(failed_task.id)

  for dependent in dependents:
    # Check if this dependency is required
    dep_edge = get_dependency(from=failed_task.id, to=dependent.id)

    if dep_edge.required == false:
      # Optional dependency - task can attempt anyway
      log_event("dependency_optional_skipped", dependent.id, failed_task.id)
      continue

    # Default: cascade the failure
    if failed_task.on_failure.propagate_failure != false:
      mark_task_blocked(dependent.id, "dependency_failure", failed_task.id)
      log_event("task_blocked", dependent.id, failed_task.id, "cascade_failure")
```

**Cascade vs. no cascade:**
- Default (propagate_failure: true): Dependents blocked
- Override (propagate_failure: false): Dependents attempt with missing input (will likely fail on missing_input, but that's their problem)
- Optional dependency (required: false): Dependent can proceed without this input

### Pattern 4: Human Escalation Interface

**What:** Present failure context and options to human for decision.

**When to use:** Escalate strategy triggered or retry exhausted with escalate fallback.

**Implementation:**

```markdown
## Execution Paused: Task Failed

**Task:** {task.id}
**Name:** {task.name}
**Wave:** {task.wave}

### Failure Details

**Error Category:** {error_category}
**Error Message:** {error}
**Attempts Made:** {attempts}/{max_attempts}

### Failure Record

Full context saved to: `.orchestrator/failures/{task_id}-attempt-{N}.yaml`

### Affected Tasks

These tasks are now blocked:
{for each dependent in cascade_blocked:}
- {dependent.id}: depends on {task.id}

### Options

1. **`/ptf:retry {task.id}`** - Reset attempt counter, try again
   Use when: Task might succeed with another attempt

2. **`/ptf:skip {task.id}`** - Mark as skipped, continue execution
   Use when: Task is not critical, can proceed without it
   Warning: {N} dependent tasks will be blocked

3. **`/ptf:abort`** - Stop execution, preserve all state
   Use when: Need to investigate or fix something first

4. **`/ptf:edit {task.id}`** - Modify task definition, then retry
   Use when: Task requirements need adjustment

5. **`/ptf:replan`** - Re-decompose from current state
   Use when: Task breakdown was wrong, need different approach
```

### Pattern 5: Failure Record Format

**What:** Detailed failure context for debugging.

**When to use:** Every task failure, stored in `.orchestrator/failures/`.

**Implementation (from founding document Section 8.1):**

```yaml
# .orchestrator/failures/{task-id}-attempt-{N}.yaml
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
  actual: "function not found in file"

context:
  inputs_loaded:
    - /src/types/user.ts
    - /src/config/auth.ts
  files_written:
    - /src/services/authService.ts (partial)

recovery_action: retry
next_attempt: 3

agent_logs: |
  [10:40:05] Reading input files...
  [10:40:10] Generating auth service...
  [10:41:00] Writing to /src/services/authService.ts
  [10:41:30] Running verification...
  [10:42:30] FAIL: contains check - authenticate function not found
```

### Anti-Patterns to Avoid

- **Unbounded retries:** Always have max_attempts. Default: 3 for quick failures, 10 for Ralph loops.
- **Silent failure cascade:** Always log and inform user which tasks are blocked.
- **Retrying unrecoverable errors:** Don't retry missing_input (wait for fix) or unclear_requirements (needs human).
- **Losing failure context:** Always create failure record before retry or escalation.
- **Ignoring backoff:** Quick retries hammer the system; always use delay between retries.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Backoff timing | Custom timer | setTimeout or shell sleep | Standard mechanism |
| Failure persistence | In-memory tracking | YAML failure records | Survives session interruption |
| Cascade detection | Manual graph walking | Dependency graph from Phase 3 | Already computed |
| State updates | Direct file writes | State manager operations | Atomic checkpoints |
| Retry counting | Ad-hoc tracking | attempts array in task-state | Already exists |

**Key insight:** Phase 4 (state) and Phase 5 (execution) already provide the infrastructure. Failure handling is policy on top of existing primitives.

## Common Pitfalls

### Pitfall 1: Infinite Retry Loops

**What goes wrong:** Task retries forever, never completing or escalating.

**Why it happens:** No max_attempts configured, or ignoring max_iterations from Ralph loop.

**How to avoid:**
- ALWAYS configure max_attempts (default: 3)
- Respect max_iterations for Ralph-style tasks
- After max, MUST escalate or skip - never continue retrying

**Warning signs:** Same task appearing in logs repeatedly with incrementing attempt numbers.

### Pitfall 2: Cascade Blindness

**What goes wrong:** Dependent tasks try to execute with missing inputs, fail with confusing errors.

**Why it happens:** Not blocking dependents when prerequisite fails.

**How to avoid:**
- When task fails, immediately identify and mark dependents
- Log cascade effects clearly
- Show user which tasks are affected
- Respect propagate_failure policy

**Warning signs:** Multiple tasks failing with "missing_input" for same file.

### Pitfall 3: Lost Failure Context

**What goes wrong:** After retry or resume, no information about what went wrong.

**Why it happens:** Not creating failure records, only logging events.

**How to avoid:**
- Create failure record BEFORE any retry attempt
- Include: inputs loaded, files written, error details, agent output
- Keep records even after eventual success (audit trail)

**Warning signs:** Debugging failures with only "task_failed" event, no details.

### Pitfall 4: Retry Without Backoff

**What goes wrong:** Rapid retries overwhelm system, no time for transient issues to resolve.

**Why it happens:** Immediately retrying without delay.

**How to avoid:**
- Always apply backoff (minimum: linear with 1s base)
- For external dependencies: exponential backoff
- Cap backoff at reasonable maximum (60s)

**Warning signs:** Logs show rapid fire of retry attempts.

### Pitfall 5: Escalation Without Options

**What goes wrong:** Execution pauses, user doesn't know how to proceed.

**Why it happens:** Escalating without presenting clear options.

**How to avoid:**
- Always present: retry, skip, abort, edit, replan
- Explain consequences of each option
- Show which tasks are affected by each choice
- Provide failure record location for debugging

**Warning signs:** User asks "what now?" after escalation.

### Pitfall 6: Cascade Overkill

**What goes wrong:** One failed task blocks entire plan unnecessarily.

**Why it happens:** propagate_failure: true for all tasks, no optional dependencies.

**How to avoid:**
- Mark non-critical tasks with propagate_failure: false
- Use required: false for optional dependencies
- Consider task criticality during decomposition
- Allow partial wave completion when safe

**Warning signs:** Many blocked tasks when only one failed, user complaints about rigidity.

## Code Examples

### Exponential Backoff Calculation (Bash)

```bash
# Calculate backoff delay in seconds
calculate_backoff() {
  local attempt="$1"
  local backoff_type="${2:-exponential}"
  local base_seconds="${3:-2}"
  local max_seconds="${4:-60}"

  case "$backoff_type" in
    none)
      echo 0
      ;;
    linear)
      delay=$((base_seconds * attempt))
      ;;
    exponential)
      # 2^(attempt-1) * base
      delay=$((base_seconds * (1 << (attempt - 1))))
      ;;
  esac

  # Cap at maximum
  if [ "$delay" -gt "$max_seconds" ]; then
    delay="$max_seconds"
  fi

  echo "$delay"
}

# Usage:
# attempt 1: calculate_backoff 1 exponential 2 → 2
# attempt 2: calculate_backoff 2 exponential 2 → 4
# attempt 3: calculate_backoff 3 exponential 2 → 8
# attempt 5: calculate_backoff 5 exponential 2 → 32
# attempt 7: calculate_backoff 7 exponential 2 → 60 (capped)
```

### /ptf:retry Command Structure

```markdown
---
name: ptf:retry
description: Retry a failed task with fresh context
allowed-tools:
  - Read
  - Write
  - Bash
  - Task
---

<objective>
Retry a specific failed task by resetting its state and re-executing.

**Usage:**
- `/ptf:retry task-id` - Retry specific task

**Behavior:**
1. Validates task exists and is in failed/blocked state
2. Resets task state to "ready"
3. Clears blocked status from dependents (if any)
4. Spawns orchestrator to execute the task
</objective>

<process>

## Phase 1: Validate Task

Check task exists and is retryable:

```bash
TASK_FILE=".orchestrator/state/tasks/${TASK_ID}.yaml"
if [ ! -f "$TASK_FILE" ]; then
  echo "ERROR: Task ${TASK_ID} not found"
  exit 1
fi

STATUS=$(grep "^status:" "$TASK_FILE" | awk '{print $2}')
if [ "$STATUS" != "failed" ] && [ "$STATUS" != "blocked" ]; then
  echo "ERROR: Task ${TASK_ID} has status ${STATUS}, cannot retry"
  exit 1
fi
```

## Phase 2: Reset Task State

Invoke state manager to reset task:

```
Task(prompt="
Reset task ${TASK_ID} for retry.

1. Update task state:
   - status: ready
   - Clear error field
   - Preserve attempts array (for history)

2. If task was blocking others:
   - Find tasks with blocked_by containing ${TASK_ID}
   - Re-evaluate their status (may become ready if this was only blocker)

3. Log task_retry event:
   {\"ts\": timestamp, \"event\": \"task_retry\", \"task\": \"${TASK_ID}\", \"reason\": \"manual_retry\"}

4. Update execution.yaml:
   - Decrement tasks_failed
   - Increment tasks_pending
   - Clear blockers entry for this task if present
", subagent_type="ptf-state-manager")
```

## Phase 3: Execute Retry

Spawn orchestrator to execute the single task:

```
Task(prompt="
Execute single task retry: ${TASK_ID}

This task was previously failed and has been reset.

1. Load task definition from .orchestrator/decomposition/tasks/
2. Dispatch task executor with fresh context
3. Process result (success or failure)
4. Update state appropriately

Note: This is a single-task execution, not a full wave.
", subagent_type="ptf-orchestrator")
```

## Phase 4: Report Result

Show retry outcome:

```markdown
## Retry Result

**Task:** {task_id}
**Previous Status:** failed
**Retry Outcome:** {completed | failed}

{If completed:}
Task completed successfully on retry.
Dependent tasks may now be able to proceed.
Next: Run `/ptf:execute` to continue wave execution.

{If failed again:}
Task failed again.
Error: {error}

Options:
1. `/ptf:retry {task_id}` - Try again
2. `/ptf:skip {task_id}` - Skip and continue
3. `/ptf:abort` - Stop execution
```

</process>
```

### /ptf:abort Command Structure

```markdown
---
name: ptf:abort
description: Stop execution and preserve state for later resume
allowed-tools:
  - Read
  - Write
  - Bash
  - Task
---

<objective>
Cleanly stop PTF execution while preserving all state.

**Usage:**
- `/ptf:abort` - Stop execution now

**Behavior:**
1. Marks any running tasks as interrupted
2. Updates execution status to "paused"
3. Creates final checkpoint
4. Logs session_aborted event
</objective>

<process>

## Phase 1: Check Execution State

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
```

## Phase 2: Handle Running Tasks

Find and mark any currently running tasks:

```
Task(prompt="
Abort execution in progress.

1. Find all tasks with status: running
2. For each running task:
   - Mark status: ready (will retry on resume)
   - Add to current attempt: status: interrupted
   - Log task_interrupted event

3. Update execution.yaml:
   - status: paused
   - Add blocker: \"Execution aborted by user\"
   - Update progress counters

4. Log session_aborted event

5. Create checkpoint (follow checkpoint protocol)
", subagent_type="ptf-state-manager")
```

## Phase 3: Report Abort

```markdown
## Execution Aborted

**Status:** Paused
**Wave:** {current_wave} of {total}

### Summary

- Tasks completed: {count}
- Tasks pending: {count}
- Tasks interrupted: {count}

### State Preserved

All progress has been saved. You can:

- `/ptf:resume` - Continue from current point
- `/ptf:status` - Review current state
- `/ptf:retry {task}` - Retry specific task before resuming

### Session

Session {session_id} aborted at {timestamp}.
Previous work preserved in checkpoint.
```

</process>
```

### State Manager: mark_blocked Operation

```yaml
# New operation for ptf-state-manager.md

<operation name="mark_blocked">
**Mark Task as Blocked**

Called when a task cannot proceed due to dependency failure.

**Input:** task_id, reason, blocked_by (task_id of failed dependency)

**Steps:**

1. Update task state file:
   ```yaml
   task_id: {id}
   status: blocked
   blocked_by:
     - {failed_task_id}
   error: "Blocked by failed dependency: {failed_task_id}"
   ```

2. Update execution.yaml:
   - Increment tasks_blocked
   - Decrement tasks_pending

3. Append event:
   ```bash
   echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","event":"task_blocked","task":"'${TASK_ID}'","blocked_by":"'${FAILED_TASK}'","reason":"cascade_failure"}' >> .orchestrator/history/events.jsonl
   ```

**Output:** Task marked as blocked
</operation>
```

### State Manager: create_failure_record Operation

```yaml
# New operation for ptf-state-manager.md

<operation name="create_failure_record">
**Create Failure Record**

Creates detailed failure record for debugging.

**Input:** task_id, attempt, error_details, context

**Steps:**

1. Create failures directory if needed:
   ```bash
   mkdir -p .orchestrator/failures
   ```

2. Write failure record:
   ```bash
   cat > .orchestrator/failures/${TASK_ID}-attempt-${ATTEMPT}.yaml << EOF
   task_id: ${TASK_ID}
   task_name: ${TASK_NAME}
   wave: ${WAVE}
   attempt: ${ATTEMPT}

   started: ${STARTED}
   failed: $(date -u +%Y-%m-%dT%H:%M:%SZ)
   duration_s: ${DURATION}

   failure_mode: ${FAILURE_MODE}

   error:
     type: ${ERROR_TYPE}
     category: ${ERROR_CATEGORY}
     message: ${ERROR_MESSAGE}

   context:
     inputs_loaded: ${INPUTS}
     files_written: ${OUTPUTS}

   recovery_action: ${NEXT_ACTION}
   EOF
   ```

3. Append failure_record_created event:
   ```bash
   echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","event":"failure_record_created","task":"'${TASK_ID}'","attempt":'${ATTEMPT}',"path":"'${RECORD_PATH}'"}' >> .orchestrator/history/events.jsonl
   ```

**Output:** Failure record file path
</operation>
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Fail-fast execution | Configurable failure strategies | Standard practice | More resilient execution |
| Fixed retry count | Exponential backoff | Standard practice | Better resource management |
| Silent failure cascade | Explicit cascade with policy | PTF design | User understands impact |
| Lost failure context | Persistent failure records | PTF design | Debuggable failures |
| Binary success/fail | Partial success with skip | PTF design | More flexible execution |

**Key insight from founding document:**
> "Failure is information. The framework's job is to make failures actionable: classify them, record context, present options, and enable recovery."

## Integration Points

### Inputs from Prior Phases

| Artifact | Location | Used For |
|----------|----------|----------|
| Task definitions | .orchestrator/decomposition/tasks/*.yaml | on_failure policy |
| Task state | .orchestrator/state/tasks/*.yaml | Current status, attempts |
| Dependency graph | .orchestrator/decomposition/graph.yaml | Cascade target identification |
| HOOK-03 | .claude/hooks/ptf/on-failure.md | Failure handling trigger |
| Verification results | task-state.yaml verification field | Determine verification_failure |

### Outputs Produced

| Artifact | Location | Consumed By |
|----------|----------|-------------|
| Failure records | .orchestrator/failures/*.yaml | Debugging, audit |
| Updated task states | .orchestrator/state/tasks/*.yaml | Resume, status display |
| Blocked task states | .orchestrator/state/tasks/*.yaml | Wave planning |
| Failure events | events.jsonl | Audit trail |

### Hook Integration (HOOK-03)

| Event | Hook Action | State Manager Call |
|-------|-------------|-------------------|
| Task fails verification | log_failure | task_failed |
| Retry exhausted | check_cascade_policy | mark_blocked (for dependents) |
| Cascade triggered | update_failure_record | create_failure_record |
| Human escalation | Present options | (orchestrator handles) |

## Open Questions

1. **Backoff maximum cap**
   - What we know: Exponential can grow very large
   - What's unclear: What's a reasonable maximum wait?
   - Recommendation: 60 seconds max, configurable in config.yaml

2. **Partial output cleanup**
   - What we know: Failed tasks may produce partial outputs
   - What's unclear: Should partial outputs be deleted before retry?
   - Recommendation: Keep partials (may inform next attempt), but mark as unverified

3. **Replan scope**
   - What we know: Replan re-decomposes portion of plan
   - What's unclear: How to determine scope of replan?
   - Recommendation: Start with failed task and its dependents; user can expand

4. **Concurrent failures in wave**
   - What we know: Multiple tasks in wave can fail simultaneously
   - What's unclear: Handle one at a time or batch?
   - Recommendation: Collect all failures, present together, user chooses per-task

5. **Retry delay implementation**
   - What we know: Need to delay between retries
   - What's unclear: Does Claude Code Task tool support timed delays?
   - Recommendation: Use shell sleep in orchestrator; if not feasible, immediate retry with backoff count in state

## Sources

### Primary (HIGH confidence)
- PTF Founding Document (PARALLEL-TASK-FRAMEWORK.md) - Section 8: Failure Handling and Recovery
- task.schema.yaml - FailurePolicy definition (strategy, max_attempts)
- task-state.schema.yaml - Status enum includes failed, blocked, skipped
- execution-state.schema.yaml - Status includes failed, blockers array
- on-failure.md hook - Trigger conditions and actions
- ptf-orchestrator.md - handle_wave_results integration point
- ptf-state-manager.md - task_failed operation exists

### Secondary (MEDIUM confidence)
- Phase 5 Research (05-RESEARCH.md) - Pitfall 5 covers wave blocking without escalation
- Phase 6 Research (06-RESEARCH.md) - Pattern 5 covers verification-driven retry decisions
- resume.md command - Existing failure recovery options (retry/skip/abort)

### Tertiary (LOW confidence - derived)
- Exponential backoff algorithm - Standard distributed systems pattern
- Circuit breaker patterns - Similar failure handling in microservices

## Metadata

**Confidence breakdown:**
- Failure strategies: HIGH - Founding document Section 8.2 specifies exactly
- Cascade handling: HIGH - Founding document Section 8.4 specifies
- Failure records: HIGH - Founding document shows exact format
- State integration: HIGH - Existing schemas and operations support
- Backoff algorithm: HIGH - Standard pattern, well-documented
- Replan capability: MEDIUM - Founding document mentions but less detailed
- Command interfaces: MEDIUM - Derived from existing patterns

**Research date:** 2026-01-18
**Valid until:** 2026-02-17 (30 days - failure handling patterns are stable once defined)
