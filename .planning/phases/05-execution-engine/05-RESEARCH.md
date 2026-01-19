# Phase 5: Execution Engine - Research

**Researched:** 2026-01-18
**Domain:** Wave-based parallel execution, subagent dispatch, fresh context, Ralph-style iteration
**Confidence:** HIGH

## Summary

Phase 5 implements the core execution engine that runs tasks in parallel waves with fresh context per task. Research confirms the founding document's design is well-aligned with both the Ralph Wiggum Loop pattern (temporal iteration with fresh context) and structured parallelism (wave-based coordination without distributed state chaos).

Key findings:
- **Task tool dispatch** is the primitive for spawning fresh-context subagents in Claude Code
- **Wave-based execution** naturally provides synchronization points without complex coordination
- **Ralph-style iteration** (repeat until verified) should be configurable per-task with bounded iterations
- **Completion promise pattern** gates verification - agent must explicitly signal completion
- **State management integration** (from Phase 4) provides checkpoint and event logging primitives
- **JSONL event logging** enables audit trail and debugging

The execution engine is the orchestrator that coordinates all prior phases: it consumes waves from Phase 3, uses state management from Phase 4, and produces verified artifacts.

**Primary recommendation:** Implement execution engine as three components:
1. `/ptf:execute [wave]` command - single wave execution
2. `/ptf:execute-all` command - full plan execution
3. `ptf-executor` subagent - task execution with Ralph-style iteration

## Standard Stack

### Core Components
| Component | Source | Purpose | Why Standard |
|-----------|--------|---------|--------------|
| Task tool | Claude Code built-in | Spawn fresh-context subagents | Official mechanism for parallel dispatch |
| JSONL events | Phase 4 (event-log.schema.yaml) | Structured event logging | Append-only, greppable, replayable |
| YAML state files | Phase 4 schemas | Execution state persistence | Human-readable, existing patterns |
| Checksum validation | SHA-256 | Artifact integrity | Standard hash, shell-compatible |

### From Prior Phases
| Component | Phase | Purpose | Integration |
|-----------|-------|---------|-------------|
| graph.yaml | Phase 3 | Wave assignments, dependencies | Read waves to execute |
| execution.yaml | Phase 4 | Master state | Write status, read for resume |
| task-state.schema.yaml | Phase 4 | Per-task state | Track task lifecycle |
| wave-state.schema.yaml | Phase 4 | Per-wave state | Track wave lifecycle |
| ptf-state-manager | Phase 4 | Checkpoint operations | Invoke for state writes |

### Patterns from Founding Document
| Pattern | Section | Purpose | Implementation |
|---------|---------|---------|----------------|
| Wave execution loop | 5.1 | Execute wave-by-wave | for each wave in plan.waves |
| Fresh context dispatch | 5.3 | Load only declared inputs | Task tool with minimal prompt |
| Ralph-style iteration | 6.4 | Repeat until verified | while iterations < max |
| Completion promise | 6.5 | Agent signals completion | "VERIFICATION PASSED" phrase |
| Wave checkpoints | 5.5 | Persist state at boundaries | checkpoint_wave operation |

## Architecture Patterns

### Recommended Structure

```
.claude/
├── commands/ptf/
│   ├── execute.md              # /ptf:execute [wave] command
│   └── execute-all.md          # /ptf:execute-all command
│
├── agents/
│   ├── ptf-executor.md         # Task executor subagent
│   └── ptf-orchestrator.md     # Wave orchestrator subagent
│
└── skills/ptf/
    └── SKILL.md                # (already exists) Framework concepts
```

### Pattern 1: Wave Execution Loop

**What:** Execute plan wave by wave with parallel dispatch within each wave.

**When to use:** This is the core execution loop for `/ptf:execute-all`.

**Implementation (from founding document Section 5.1):**

```
execute_plan(plan: Plan):

  for each wave in plan.waves:

    # Wait for all dependencies (previous waves)
    if wave.depends_on_waves not all completed:
      continue  # Skip to next iteration or wait

    # Dispatch all tasks in this wave in parallel
    running_tasks = []
    for each task in wave.tasks:
      if task.status == 'ready':
        agent = spawn_fresh_agent(task)  # Task tool
        running_tasks.add(agent)

    # Wait for all tasks in wave to complete
    results = wait_all(running_tasks)

    # Handle failures according to policy
    for each result in results:
      if result.failed:
        handle_failure(result.task, result.error)

    # Checkpoint state
    invoke_state_manager("checkpoint_wave", wave.number, results)

    # Decide whether to continue
    if not can_proceed(wave, results):
      pause_execution("Wave {wave.number} blocked")
      return

  mark_plan_complete()
```

### Pattern 2: Fresh Context Dispatch with Task Tool

**What:** Spawn subagent with only declared inputs loaded.

**When to use:** Every task execution.

**Implementation:**

```markdown
# Dispatch a single task via Task tool
Task(prompt="
<task>
ID: {task.id}
Name: {task.name}

## Instructions
{task.description}

## Inputs
{for each input in task.inputs: load and include file content}

## Outputs Expected
{task.outputs with paths and types}

## Verification
{task.verify criteria}

## Completion Protocol
When complete:
1. Ensure all outputs exist at declared paths
2. Run verification steps
3. If all pass, output: VERIFICATION PASSED
4. If blocked, output: BLOCKED: [specific reason]

Do not output the completion phrase until verified.
</task>
", subagent_type="ptf-executor")
```

**Key properties:**
- Fresh context: No accumulated state from prior tasks
- Minimal loading: Only declared inputs, not entire project
- Explicit handoff: Information flows through files
- Clear contract: Completion promise gates verification

### Pattern 3: Ralph-Style Task Execution

**What:** Repeat task with fresh context until verification passes.

**When to use:** Tasks configured with `execution.mode: ralph`.

**Implementation (from founding document Section 6.4):**

```
execute_task_ralph(task):
  iterations = 0
  max_iterations = task.execution.max_iterations or config.ralph.max_iterations

  while iterations < max_iterations:
    iterations += 1

    # Fresh context every iteration
    result = dispatch_task(task)  # Task tool

    # Check for completion promise
    if result.output contains "VERIFICATION PASSED":
      # Agent claims completion - verify externally
      verification = run_verification(task)

      if verification.passed:
        return TaskResult(success=true, artifacts=result.files)
      else:
        # False promise - log and continue
        log_event("task_retry", task, "verification_failed")
        continue

    elif result.output contains "BLOCKED:":
      # Agent cannot proceed
      reason = extract_block_reason(result.output)
      return TaskResult(success=false, blocked=true, reason=reason)

    # No promise - agent didn't finish, continue
    log_event("task_retry", task, "no_completion_promise")

  # Max iterations reached
  return TaskResult(success=false, reason="max_iterations_exceeded")
```

### Pattern 4: Parallel Wave Dispatch

**What:** Dispatch all tasks in a wave simultaneously, respecting max_parallel config.

**When to use:** Wave execution.

**Implementation:**

```
execute_wave(wave, tasks):
  max_parallel = config.execution.max_parallel_tasks

  # Split into batches if needed
  batches = chunk(tasks, max_parallel)

  all_results = []
  for batch in batches:
    # Dispatch batch in parallel
    running = []
    for task in batch:
      # Log task_started event
      invoke_state_manager("task_started", task.id, 1)

      # Spawn via Task tool
      running.add(spawn_task_executor(task))

    # Wait for batch completion
    batch_results = wait_all(running)
    all_results.extend(batch_results)

  return all_results
```

### Pattern 5: Hook Integration

**What:** Execute hooks at defined points in execution lifecycle.

**When to use:** As specified in requirements (HOOK-01 through HOOK-04).

**Implementation:**

```yaml
# Hook points in execution flow
hooks:
  pre_wave_start:       # HOOK-02
    - checkpoint_state
    - validate_dependencies

  post_task_complete:   # HOOK-01
    - log_task_event
    - update_task_state
    - register_artifacts

  on_failure:           # HOOK-03
    - log_failure
    - check_retry_policy
    - update_failure_record

  on_session_end:       # HOOK-04
    - final_checkpoint
    - cleanup_temp_files
    - summary_report
```

### Anti-Patterns to Avoid

- **Shared state between parallel tasks:** Tasks in a wave must be independent. No in-memory communication.
- **Accumulated context:** Each task dispatch must start fresh. No "remember what happened before."
- **Unbounded iteration:** Ralph loops MUST have max_iterations to prevent infinite loops.
- **Implicit verification:** Always run external verification after completion promise. Don't trust agent claims.
- **Skip checkpoints:** Always checkpoint at wave boundaries. Never skip to "save time."

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Subagent dispatch | Custom process spawn | Task tool | Claude Code's official mechanism |
| State persistence | In-memory tracking | File-based state (Phase 4) | Survives interruption |
| Event logging | Custom logger | JSONL append (Phase 4) | Append-only, greppable |
| Checksum computation | Custom hash | shasum -a 256 | Shell-compatible, standard |
| Parallel coordination | Complex sync | Wave boundaries | Natural synchronization points |

**Key insight:** The Task tool IS the parallelism primitive. Wave boundaries ARE the synchronization mechanism. Don't add complexity.

## Common Pitfalls

### Pitfall 1: Context Accumulation

**What goes wrong:** Task subagent inherits context from orchestrator, degrading quality.

**Why it happens:** Including too much in the Task tool prompt or not using fresh dispatch.

**How to avoid:**
- Task tool prompt includes ONLY: task definition, declared inputs, verification criteria
- Never include "history" or "what we've done so far"
- Each task prompt is self-contained

**Warning signs:** Task prompts growing larger over execution, quality degrading in later waves.

### Pitfall 2: False Completion Promise

**What goes wrong:** Agent outputs "VERIFICATION PASSED" but verification actually fails.

**Why it happens:** Agent doesn't actually verify, just claims completion.

**How to avoid:**
- External verification after promise detection
- Log "false_completion_promise" events for debugging
- Continue Ralph loop on false promise
- Consider adjusting task description if persistent

**Warning signs:** High rate of verification failures after completion promises.

### Pitfall 3: Infinite Ralph Loop

**What goes wrong:** Task loops forever, never completing.

**Why it happens:** Task is impossible or agent keeps making same mistake.

**How to avoid:**
- max_iterations is REQUIRED (default: 10)
- Track iteration patterns in events
- After max_iterations: mark task BLOCKED, escalate
- Consider task re-decomposition if consistently failing

**Warning signs:** Same error message across multiple iterations.

### Pitfall 4: Lost Progress on Interruption

**What goes wrong:** Session interrupted, progress lost, must restart from beginning.

**Why it happens:** Not checkpointing at wave boundaries, state only in memory.

**How to avoid:**
- ALWAYS checkpoint after each wave completes
- Write state files before execution.yaml (atomic commit marker)
- Resume validates artifacts before skipping completed tasks
- Never skip checkpoint "for speed"

**Warning signs:** Duplicate work after resume, inconsistent state.

### Pitfall 5: Wave Blocking Without Escalation

**What goes wrong:** One failed task blocks entire plan, no path forward.

**Why it happens:** Strict failure policy without human escalation option.

**How to avoid:**
- Failure cascade policy per task (propagate_failure: true/false)
- After retry exhaustion: escalate to human with options
- Options: retry, skip, abort, replan
- Log detailed failure context for human decision

**Warning signs:** Execution stuck with no presented options.

### Pitfall 6: Exceeding max_parallel Limits

**What goes wrong:** Too many concurrent Task tool calls, resource exhaustion.

**Why it happens:** Not respecting max_parallel_tasks configuration.

**How to avoid:**
- Batch wave tasks into chunks of max_parallel
- Wait for batch completion before next batch
- Log actual parallelism in events for debugging

**Warning signs:** System slowdown, Task tool timeouts.

## Code Examples

### Task Executor Subagent Structure

```markdown
---
name: ptf-executor
description: Executes single PTF task with fresh context and verification
tools: Read, Write, Bash, Glob, Grep
---

<role>
You are a PTF task executor. You execute a single atomic task with fresh context.

You receive:
- Task definition (id, name, description)
- Input files (only what the task declared)
- Output expectations
- Verification criteria

Your job: Execute the task, produce outputs, verify, then signal completion.
</role>

<execution_flow>

<step name="understand">
Read and understand the task:
- What needs to be created/modified
- What inputs are available
- What outputs are expected
- How to verify success
</step>

<step name="execute">
Implement the task as described:
- Read input files as needed
- Create/modify output files
- Follow any patterns or conventions specified
- Stay within task boundaries (don't do extra work)
</step>

<step name="verify">
Run verification steps:
- Check all outputs exist
- Run any specified verification commands
- Confirm expected content/behavior
</step>

<step name="signal">
Signal completion:

If all verifications pass:
  Output: VERIFICATION PASSED

If blocked and cannot proceed:
  Output: BLOCKED: [specific reason why task cannot complete]

Do not output completion phrase until verification actually passes.
</step>

</execution_flow>

<structured_returns>

## VERIFICATION PASSED

Output this exact phrase when:
- All declared outputs exist at their paths
- All verification steps pass
- Task is complete

## BLOCKED: [reason]

Output this when:
- Cannot access required input
- Verification repeatedly fails
- Task requirements unclear
- External dependency missing

Include specific reason after "BLOCKED: "

</structured_returns>
```

### /ptf:execute Command Structure

```markdown
---
name: ptf:execute
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
- .orchestrator/decomposition/graph.yaml (waves computed)
- .orchestrator/state/execution.yaml (execution initialized)

**After this command:**
- Run again for next wave, OR
- Run `/ptf:execute-all` for automatic progression
</objective>

<process>

## Phase 1: Load State

Read execution state and determine target wave:

```bash
# Check prerequisites
if [ ! -f .orchestrator/state/execution.yaml ]; then
  echo "Execution not initialized. Run /ptf:init first."
  exit 1
fi

# Read current state
CURRENT_WAVE=$(grep "^current_wave:" .orchestrator/state/execution.yaml | awk '{print $2}')
EXECUTION_STATUS=$(grep "^status:" .orchestrator/state/execution.yaml | awk '{print $2}')

# Handle terminal states
if [ "$EXECUTION_STATUS" = "completed" ]; then
  echo "Execution already complete."
  exit 0
fi

if [ "$EXECUTION_STATUS" = "failed" ]; then
  echo "Execution failed. Use /ptf:resume to recover."
  exit 1
fi
```

Determine target wave:
- If wave argument provided: use that wave
- Otherwise: use current_wave from execution.yaml

## Phase 2: Validate Wave Ready

Check wave dependencies satisfied:

```bash
WAVE=${1:-$CURRENT_WAVE}
DEPENDS_ON=$(grep -A5 "- number: $WAVE" .orchestrator/decomposition/graph.yaml | grep "depends_on_waves:" | awk -F'[][]' '{print $2}')

# Check all dependency waves completed
for dep in $(echo $DEPENDS_ON | tr ',' ' '); do
  DEP_STATUS=$(grep "^  $dep:" .orchestrator/state/execution.yaml | awk '{print $2}')
  if [ "$DEP_STATUS" != "completed" ]; then
    echo "Wave $WAVE blocked: dependency wave $dep not completed"
    exit 1
  fi
done
```

## Phase 3: Dispatch Tasks

1. Load tasks for this wave from graph.yaml
2. Initialize state manager for wave start
3. Dispatch tasks in parallel (respecting max_parallel)

```
# Invoke state manager to start wave
Task(prompt="Start wave {WAVE}. Update execution.yaml, create wave state,
     mark tasks as ready.", subagent_type="ptf-state-manager")

# Get max_parallel from config
MAX_PARALLEL=$(grep "max_parallel_tasks:" .orchestrator/config.yaml | awk '{print $2}')

# Dispatch tasks in batches
for batch in chunk(wave.tasks, MAX_PARALLEL):
  # Spawn parallel executors
  for task in batch:
    Task(prompt="Execute task: {task.id}

         {task.description}

         Inputs: {load input files}

         Outputs expected: {task.outputs}

         Verification: {task.verify}

         Mode: {task.execution.mode or 'ralph'}
         Max iterations: {task.execution.max_iterations or 10}
         ", subagent_type="ptf-executor")

  # Wait for batch completion (implicit in Task tool)
```

## Phase 4: Collect Results

Parse executor returns:
- VERIFICATION PASSED -> mark task completed
- BLOCKED: reason -> mark task failed/blocked

For each task result:
```
Task(prompt="Record task result: {task.id}
     Status: {success | failed}
     Outputs: {produced files}
     Error: {if failed, error message}
     ", subagent_type="ptf-state-manager")
```

## Phase 5: Checkpoint Wave

Invoke state manager checkpoint:

```
Task(prompt="Checkpoint wave {WAVE} completion.
     Task results: {results summary}
     Follow checkpoint protocol.", subagent_type="ptf-state-manager")
```

## Phase 6: Report and Next Steps

Display wave completion summary:

```
Wave {N} Complete

| Task | Status | Duration | Artifacts |
|------|--------|----------|-----------|
| ... | ... | ... | ... |

{If all tasks succeeded}
Next: Run `/ptf:execute` for wave {N+1}

{If some tasks failed}
Failures:
- task-x: {reason}

Options:
- `/ptf:retry task-x` to retry
- `/ptf:execute` to continue (skipping failed)
- `/ptf:abort` to stop execution
```

</process>

<success_criteria>
- [ ] Target wave identified (argument or next pending)
- [ ] Wave dependencies validated (all prior waves complete)
- [ ] All tasks in wave dispatched with fresh context
- [ ] max_parallel_tasks configuration respected
- [ ] Task results collected and state updated
- [ ] Wave checkpoint completed (state files written)
- [ ] Events logged (wave_started, task_started, task_completed/failed, wave_completed)
- [ ] Summary displayed with next steps
</success_criteria>
```

### Event Logging Examples

```jsonl
{"ts":"2026-01-18T10:30:00Z","event":"session_started","session":"session-001"}
{"ts":"2026-01-18T10:30:01Z","event":"wave_started","wave":1}
{"ts":"2026-01-18T10:30:01Z","event":"task_started","task":"auth-schema","wave":1,"attempt":1}
{"ts":"2026-01-18T10:30:01Z","event":"task_started","task":"config-setup","wave":1,"attempt":1}
{"ts":"2026-01-18T10:32:00Z","event":"task_completed","task":"config-setup","wave":1,"duration_s":119,"outputs":["src/config/auth.ts"]}
{"ts":"2026-01-18T10:32:00Z","event":"artifact_produced","task":"config-setup","path":"src/config/auth.ts","checksum":"sha256:abc123..."}
{"ts":"2026-01-18T10:35:00Z","event":"task_completed","task":"auth-schema","wave":1,"duration_s":299,"outputs":["prisma/schema.prisma"]}
{"ts":"2026-01-18T10:35:00Z","event":"artifact_produced","task":"auth-schema","path":"prisma/schema.prisma","checksum":"sha256:def456..."}
{"ts":"2026-01-18T10:35:01Z","event":"checkpoint_started","wave":1}
{"ts":"2026-01-18T10:35:02Z","event":"wave_completed","wave":1,"duration_s":301}
{"ts":"2026-01-18T10:35:02Z","event":"checkpoint_completed","wave":1}
{"ts":"2026-01-18T10:35:03Z","event":"wave_started","wave":2}
```

### Configuration Schema

```yaml
# .orchestrator/config.yaml additions for execution

execution:
  default_mode: ralph           # ralph | single-shot
  max_parallel_tasks: 5         # Maximum concurrent tasks

  ralph:
    max_iterations: 10          # Default iterations per task
    completion_promise: "VERIFICATION PASSED"
    blocked_phrase: "BLOCKED:"
    iteration_delay: 0          # Seconds between iterations

  single_shot:
    retry_on_failure: false

  checkpoints:
    enabled: true
    at_wave_boundary: true      # Always checkpoint after wave
    on_task_complete: false     # Optional mid-wave checkpoints

hooks:
  enabled: true
  pre_wave_start:
    - checkpoint_state
  post_task_complete:
    - log_event
    - update_state
  on_failure:
    - log_failure
    - check_retry
  on_session_end:
    - final_checkpoint
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Single long conversation | Fresh context per task | 2024+ (Ralph pattern) | Eliminates context degradation |
| Sequential execution | Wave-based parallelism | Standard in orchestrators | 2-3x speedup for independent tasks |
| Hope-based completion | Completion promise + verification | 2025 (Ralph pattern) | Reliable task completion detection |
| Memory-only state | File-based with checkpoints | Standard practice | Survives interruption |

**Key insight from founding document:**
> "Ralph provides the execution primitive; the framework provides the coordination layer."

The framework orchestrates Ralph loops. Each task is a Ralph loop. Waves are synchronization points. This is structured parallelism, not microservices chaos.

## Integration Points

### Inputs from Prior Phases

| Artifact | Location | Used For |
|----------|----------|----------|
| Waves | .orchestrator/decomposition/graph.yaml | Determine what to execute |
| Task definitions | .orchestrator/decomposition/tasks/*.yaml | Task descriptions and verification |
| State schemas | schemas/*.schema.yaml | State file formats |
| State manager | .claude/agents/ptf-state-manager.md | Checkpoint operations |

### Outputs for Later Phases

| Artifact | Location | Consumed By |
|----------|----------|-------------|
| Task state files | .orchestrator/state/tasks/*.yaml | Phase 6 (Verification), Phase 7 (Failure) |
| Wave state files | .orchestrator/state/waves/*.yaml | Status reporting, resume |
| Event log | .orchestrator/history/events.jsonl | Debugging, audit |
| Produced artifacts | (various paths) | Verification, dependent tasks |

### Hook Integration Points

| Hook | When Invoked | Actions |
|------|--------------|---------|
| HOOK-01 post-task-complete | After task executor returns | Log event, update state, register artifacts |
| HOOK-02 pre-wave-start | Before first task in wave dispatched | Checkpoint current state |
| HOOK-03 on-failure | When task fails after retries | Log failure, check cascade policy |
| HOOK-04 on-session-end | Execution ends (complete/abort/interrupt) | Final checkpoint, cleanup |

## Open Questions

1. **Task tool timeout handling**
   - What we know: Task tool has implicit timeout
   - What's unclear: How to detect and handle timeout vs. normal completion?
   - Recommendation: Treat timeout as task failure, log as such, follow retry policy

2. **Partial wave completion semantics**
   - What we know: Wave can be "partial" (some succeeded, some failed)
   - What's unclear: Should partial wave block next wave or allow progression?
   - Recommendation: Configurable per wave; default: block unless task has `propagate_failure: false`

3. **Ralph iteration with accumulating context**
   - What we know: Each iteration should be fresh
   - What's unclear: Should previous attempt's files/errors be included?
   - Recommendation: Include produced artifacts (they persist), but NOT error history in prompt

4. **Max parallel vs. available resources**
   - What we know: Config specifies max_parallel_tasks
   - What's unclear: Does Claude Code have internal limits on concurrent Task calls?
   - Recommendation: Start with conservative default (5), allow user override

## Sources

### Primary (HIGH confidence)
- PTF Founding Document (PARALLEL-TASK-FRAMEWORK.md) - Sections 5 and 6
- Phase 4 Research (04-RESEARCH.md) - State management patterns
- Phase 4 Plans - State manager operations, checkpoint protocol
- Existing PTF commands (/ptf:decompose, /ptf:plan) - Task tool patterns

### Secondary (MEDIUM confidence)
- Ralph Wiggum Loop documentation (referenced in founding document)
- Claude Code Task tool behavior (from existing usage patterns)

### Tertiary (LOW confidence - training data)
- General workflow orchestration patterns (Airflow, Prefect, Dagster)
- Build system parallelism (Bazel, Make)

## Metadata

**Confidence breakdown:**
- Wave execution pattern: HIGH - Founding document specifies exactly
- Task tool dispatch: HIGH - Existing patterns in ptf commands
- Ralph-style iteration: HIGH - Founding document Section 6.4 specifies
- Completion promise: HIGH - Founding document Section 6.5 specifies
- State integration: HIGH - Phase 4 provides all primitives
- Hook points: MEDIUM - Requirements specify hooks, implementation details derived
- Parallel limits: MEDIUM - Config exists, actual behavior needs validation

**Research date:** 2026-01-18
**Valid until:** 2026-02-17 (30 days - execution patterns are stable once defined)
