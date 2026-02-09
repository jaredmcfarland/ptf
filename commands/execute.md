---
description: Execute single wave or next pending wave
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Execute a single wave of the execution plan. Dispatches all tasks in the wave
in parallel (respecting max_parallel), waits for completion, handles failures,
and checkpoints state.

**Usage:**
- `/ptf:execute` - Execute next pending wave
- `/ptf:execute 2` - Execute wave 2 specifically

**Requires:**
- `.orchestrator/decomposition/graph.yaml` (waves computed)
- `.orchestrator/state/execution.yaml` (execution initialized)

**After this command:**
- Run again for next wave, OR
- Run `/ptf:execute-all` for automatic progression
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
@.claude/agents/ptf-orchestrator.md
@.claude/agents/ptf-executor.md
</execution_context>

<context>
@.orchestrator/state/execution.yaml (if exists)
@.orchestrator/decomposition/graph.yaml
@.orchestrator/config.yaml
</context>

<process>

## Phase 1: Load State

Check prerequisites and determine target wave.

**1.1 Verify prerequisites:**

```bash
if [ ! -f .orchestrator/decomposition/graph.yaml ]; then
  echo "ERROR: graph.yaml not found. Run /ptf:plan first."
  exit 1
fi

if [ ! -f .orchestrator/state/execution.yaml ]; then
  echo "ERROR: Execution state not initialized."
  echo "→ Run /ptf:plan to generate plan and initialize execution state."
  echo "  (If you already have graph.yaml, re-running /ptf:plan will create execution.yaml)"
  exit 1
fi
```

**1.2 Read current execution state:**

```bash
cat .orchestrator/state/execution.yaml
```

Extract:
- `status`: Overall execution status (pending, running, paused, completed, failed)
- `current_wave`: Current wave number
- `waves_total`: Total waves in plan

**1.3 Handle terminal states:**

```
IF status == "completed":
  Display: "Execution already complete. All waves finished successfully."
  Show final summary from execution.yaml
  Exit with success

IF status == "failed":
  Display: "Execution previously failed. Use /ptf:resume to recover or /ptf:retry to fix."
  Show failure details
  Exit with status
```

**1.4 Determine target wave:**

```
IF argument provided (e.g., /ptf:execute 2):
  target_wave = argument
  IF target_wave > current_wave:
    Display warning: "Requested wave {target_wave} ahead of current wave {current_wave}"
ELSE:
  target_wave = current_wave from execution.yaml
```

## Phase 2: Validate Wave Ready

Ensure target wave can be executed.

**2.1 Load wave definition from graph.yaml:**

```bash
# Parse wave from graph.yaml
# Get: wave number, task list, depends_on_waves
```

**2.2 Check wave dependencies satisfied:**

```
For each wave in depends_on_waves:
  Read wave status from execution.yaml wave_summary
  IF wave.status != "completed":
    Display: "Wave {target_wave} blocked: dependency wave {dep} not completed"
    List completed waves
    Suggest running those waves first
    Exit with blocked status
```

**2.3 Get tasks for wave:**

```bash
# List tasks assigned to this wave from graph.yaml
# Load task definitions from .orchestrator/decomposition/tasks/{task-id}.yaml
```

IF no tasks in wave:
  Display error: "Wave {target_wave} has no tasks"
  Exit

## Phase 3: Dispatch Orchestrator

Spawn the ptf-orchestrator subagent to handle wave execution.

**3.1 Prepare orchestrator context:**

Gather information for orchestrator:
- Target wave number
- Execution mode: "single-wave"
- Task list with definitions
- Configuration (max_parallel, execution mode)

**3.2 Spawn orchestrator:**

```
Task(prompt="
<mode>single-wave</mode>
<target_wave>{wave_number}</target_wave>

Execute wave {wave_number} of the plan.

Current state:
- Status: {status}
- Current wave: {current_wave}
- Waves total: {waves_total}

Tasks in this wave:
{list of task IDs}

Configuration:
- max_parallel_tasks: {from config.yaml or default 5}
- execution_mode: {ralph or single-shot}
- max_iterations: {from config.yaml or default 10}

Follow execution flow:
1. Invoke state manager to start wave
2. Dispatch tasks in parallel (respecting max_parallel)
3. Collect results
4. Invoke state manager to checkpoint wave
5. Return structured result

**CRITICAL: Event Logging Verification**
After EACH state-manager call:
1. Check return includes `events_logged` array
2. Verify event was written: `tail -1 .orchestrator/history/events.jsonl`
3. If missing, retry state-manager call once
Event log path: `.orchestrator/history/events.jsonl` (ONLY this path)

Return WAVE COMPLETE, EXECUTION PAUSED, or EXECUTION BLOCKED.
", subagent_type="ptf-orchestrator")
```

**3.3 Wait for orchestrator completion:**

Task tool waits for subagent to return.
Parse structured return from orchestrator.

## Phase 4: Handle Results

Process orchestrator return and display appropriate output.

**4.1 Parse orchestrator return:**

Look for structured markers:
- `WAVE COMPLETE` - Wave executed successfully
- `EXECUTION PAUSED` - Some tasks failed, intervention may be needed
- `EXECUTION BLOCKED` - Cannot proceed, blocking issue

**4.2 Display results based on outcome:**

**On WAVE COMPLETE:**

```markdown
## Wave {N} Complete

**Status:** {completed | partial}
**Duration:** {time}

### Tasks Executed

| Task | Status | Duration | Artifacts |
|------|--------|----------|-----------|
| {task-id} | completed | 45s | 2 files |
| ... | ... | ... | ... |

### Artifacts Produced

- {path} (sha256:{first-8}...)

### Checkpoint

State files updated. Safe to resume from wave {N+1}.

---

**Next Steps:**

{If more waves:}
Wave {N+1} ready with {M} tasks.
Run `/ptf:execute` to continue or `/ptf:execute-all` for auto-progression.

{If last wave:}
All waves complete!
Run `/ptf:status` for final summary.
Run `/ptf:verify` for final verification.
```

**On EXECUTION PAUSED:**

```markdown
## Wave {N} Partially Complete

**Status:** paused
**Reason:** {reason from orchestrator}

### Completed Tasks

| Task | Status | Artifacts |
|------|--------|-----------|
| {task-id} | completed | {files} |

### Failed Tasks

| Task | Error | Attempts |
|------|-------|----------|
| {task-id} | {error} | {N} |

### Blocked Dependents

These tasks in later waves cannot proceed:
- {task-id}: depends on {failed-task}

---

**Options:**

1. `/ptf:retry {task-id}` - Retry failed task
2. `/ptf:skip {task-id}` - Skip and continue (mark dependents blocked)
3. `/ptf:execute` - Try to continue (may re-attempt failed)
4. `/ptf:abort` - Stop execution, preserve state

Review failure details and choose an option.
```

**On EXECUTION BLOCKED:**

```markdown
## Wave {N} Blocked

**Status:** blocked
**Reason:** {specific reason}

### Blocking Issue

{Detailed description from orchestrator}

### Resolution Required

{What needs to happen to unblock}

### Current State

Progress preserved at wave {N-1} checkpoint.

---

Resolve the issue and run `/ptf:resume` to continue.
```

</process>

<state_integration>

## State Files Involved

**Read:**
- `.orchestrator/state/execution.yaml` - Overall execution state
- `.orchestrator/decomposition/graph.yaml` - Wave and task assignments
- `.orchestrator/config.yaml` - Execution configuration
- `.orchestrator/decomposition/tasks/*.yaml` - Task definitions

**Modified (via state manager, not directly):**
- `.orchestrator/state/execution.yaml` - Updated status, current_wave
- `.orchestrator/state/waves/wave-{N}.yaml` - Wave execution record
- `.orchestrator/state/tasks/{task-id}.yaml` - Task execution records
- `.orchestrator/state/manifest.yaml` - Artifact registry
- `.orchestrator/history/events.jsonl` - Event log

## Hooks Fired

The following hooks are invoked during execution:

| Hook | When | Actions |
|------|------|---------|
| pre-wave-start | Before first task dispatch | checkpoint_state, validate_dependencies |
| post-task-complete | After each task | log_event, update_state, register_artifacts |
| on-failure | When task fails | log_failure, check_cascade_policy |
| on-session-end | Execution ends | final_checkpoint, cleanup |

See `.claude/hooks/ptf/` for hook definitions.

</state_integration>

<error_handling>

## Common Error Conditions

### Prerequisites missing

```
ERROR: graph.yaml not found
→ Run /ptf:plan to generate execution plan first.

ERROR: execution.yaml not found
→ Run /ptf:plan to generate plan and initialize execution state.
```

### Wave not ready

```
Wave {N} blocked: dependency wave {M} not completed
→ Execute prior waves first using /ptf:execute.
```

### All waves complete

```
Execution already complete
→ No more waves to execute. Run /ptf:status for summary.
```

### Task failures

Handled via EXECUTION PAUSED return. Options presented to user.

### State manager unavailable

Critical error. Stop execution, return EXECUTION BLOCKED.
State consistency cannot be guaranteed without state manager.

</error_handling>

<success_criteria>
- [ ] Target wave identified (argument or next pending)
- [ ] Wave dependencies validated (all prior waves complete)
- [ ] Orchestrator dispatched with proper context
- [ ] Orchestrator return parsed correctly
- [ ] Results displayed with appropriate format based on outcome
- [ ] Next steps clearly communicated
- [ ] Error conditions handled with helpful messages
</success_criteria>
