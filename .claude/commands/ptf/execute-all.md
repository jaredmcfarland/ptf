---
name: ptf:execute-all
description: Execute all waves with automatic progression
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Execute the entire plan, progressing through all waves automatically until completion or blocked.

**Usage:**
- `/ptf:execute-all` - Run entire plan from current position

**Requires:**
- `.orchestrator/decomposition/graph.yaml` (waves computed)
- `.orchestrator/state/execution.yaml` (execution initialized)

**Behavior:**
- Executes waves sequentially from current position
- Parallel task dispatch within each wave
- Automatic progression between waves
- Stops on failure or completion

**After this command:**
- `/ptf:status` for summary
- `/ptf:verify` for final verification
- `/ptf:resume` if interrupted
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

## Phase 1: Validate Prerequisites

Ensure everything is ready for full plan execution.

**1.1 Check required files exist:**

```bash
if [ ! -f .orchestrator/decomposition/graph.yaml ]; then
  echo "ERROR: graph.yaml not found. Run /ptf:plan first."
  exit 1
fi

if [ ! -f .orchestrator/state/execution.yaml ]; then
  echo "Execution state not initialized. Initializing..."
  # Could auto-initialize or require /ptf:init
fi
```

**1.2 Read current state:**

```bash
cat .orchestrator/state/execution.yaml
```

Extract:
- `status`: Current execution status
- `current_wave`: Where to resume from
- `waves_total`: Total waves in plan

**1.3 Handle terminal states:**

```
IF status == "completed":
  Display: "Plan already complete. Nothing to execute."
  Show completion summary
  Exit with success

IF status == "failed":
  Display: "Execution previously failed at wave {N}."
  Ask: "Resume from failure point or abort?"
  Options: /ptf:resume, /ptf:abort
  Exit
```

**1.4 Calculate work remaining:**

```
remaining_waves = waves_total - (current_wave - 1)
total_tasks = sum of tasks in remaining waves

Display:
"Ready to execute {remaining_waves} waves with {total_tasks} tasks."
"Starting from wave {current_wave}."
```

## Phase 2: Execute All Waves

Spawn orchestrator in full-plan mode.

**2.1 Prepare orchestrator context:**

Load configuration and prepare execution parameters:
- Start wave: current_wave from execution.yaml
- Mode: full-plan (execute all remaining waves)
- Configuration: max_parallel, execution mode, max_iterations

**2.2 Dispatch orchestrator:**

```
Task(prompt="
<mode>full-plan</mode>
<start_wave>{current_wave}</start_wave>
<waves_total>{waves_total}</waves_total>

Execute all remaining waves in the plan.

Current state:
- Status: {status}
- Current wave: {current_wave}
- Waves remaining: {remaining_waves}

Configuration:
- max_parallel_tasks: {from config.yaml or default 5}
- execution_mode: {ralph or single-shot}
- max_iterations: {from config.yaml or default 10}

Execution flow:
1. For each wave from {current_wave} to {waves_total}:
   a. Validate wave dependencies
   b. Invoke state manager to start wave
   c. Dispatch tasks in parallel (respecting max_parallel)
   d. Collect results
   e. Invoke state manager to checkpoint wave
   f. Decide: continue, pause, or blocked
2. Return final status

Return PLAN COMPLETE, EXECUTION PAUSED, or EXECUTION BLOCKED.
", subagent_type="ptf-orchestrator")
```

**2.3 Wait for orchestrator:**

The orchestrator handles the wave-by-wave loop internally.
Wait for final return indicating completion or stopping point.

## Phase 3: Handle Completion

Process orchestrator return and display final status.

**3.1 Parse orchestrator return:**

Look for structured markers:
- `PLAN COMPLETE` - All waves executed successfully
- `EXECUTION PAUSED` - Stopped due to failures
- `EXECUTION BLOCKED` - Cannot proceed, blocking issue

**3.2 Display results based on outcome:**

**On PLAN COMPLETE:**

```markdown
## Plan Complete

**Waves executed:** {total}
**Tasks completed:** {total}
**Total duration:** {time}

### Wave Summary

| Wave | Tasks | Status | Duration |
|------|-------|--------|----------|
| 1 | 3 | completed | 2m 15s |
| 2 | 5 | completed | 4m 30s |
| 3 | 2 | completed | 1m 45s |

### Artifacts Produced

{count} artifacts registered in manifest.yaml

Key artifacts:
- {path} - {description}
- {path} - {description}

### Success Criteria

All success criteria from analysis.yaml addressed.

---

**Next Steps:**

1. Run `/ptf:verify` for final verification
2. Run `/ptf:status` for detailed summary
3. Review artifacts in .orchestrator/state/manifest.yaml

Plan execution complete!
```

**On EXECUTION PAUSED:**

```markdown
## Execution Paused

**Progress:** Wave {N} of {total}
**Status:** paused
**Reason:** {reason}

### Completed

| Wave | Tasks | Status |
|------|-------|--------|
| 1 | 3 | completed |
| ... | ... | ... |

### Failed

**Wave {N}:**
| Task | Error | Attempts |
|------|-------|----------|
| {task-id} | {error} | {N} |

### Blocked

Tasks that cannot proceed:
- {task-id}: depends on {failed-task}

---

**Options:**

1. `/ptf:retry {task-id}` - Retry specific failed task
2. `/ptf:resume` - Resume from current position
3. `/ptf:skip {task-id}` - Skip failed task
4. `/ptf:abort` - Stop execution

State has been checkpointed. Safe to pause and investigate.
```

**On EXECUTION BLOCKED:**

```markdown
## Execution Blocked

**Progress:** Wave {N} of {total}
**Status:** blocked
**Reason:** {reason}

### Blocking Issue

{Detailed description}

### State Preserved

Checkpoint complete at wave {N-1}.
All completed work is preserved.

---

**Resolution:**

{Steps to resolve the blocking issue}

After resolving, run `/ptf:resume` to continue.
```

</process>

<comparison_with_execute>

## /ptf:execute vs /ptf:execute-all

| Aspect | /ptf:execute | /ptf:execute-all |
|--------|--------------|------------------|
| Scope | Single wave | All remaining waves |
| Progression | Manual (run again) | Automatic |
| Mode | single-wave | full-plan |
| User control | After each wave | At completion/failure |
| Best for | Step-by-step verification | Hands-off execution |

**When to use /ptf:execute:**
- Want to review results between waves
- Debugging task failures
- First-time execution (verify each step)
- Complex plans with potential issues

**When to use /ptf:execute-all:**
- Confident in plan correctness
- Want unattended execution
- Resuming after interruption
- Fast iteration on known-good patterns

Both commands use the same orchestrator, just with different mode parameters.
The orchestrator handles the actual execution logic.

</comparison_with_execute>

<state_integration>

## State Files Involved

**Read:**
- `.orchestrator/state/execution.yaml` - Overall execution state
- `.orchestrator/decomposition/graph.yaml` - Wave and task assignments
- `.orchestrator/config.yaml` - Execution configuration

**Modified (via orchestrator and state manager):**
- `.orchestrator/state/execution.yaml` - Updated through execution
- `.orchestrator/state/waves/*.yaml` - Wave execution records
- `.orchestrator/state/tasks/*.yaml` - Task execution records
- `.orchestrator/state/manifest.yaml` - Artifact registry
- `.orchestrator/history/events.jsonl` - Event log

## Hooks Fired

All hooks fire during execute-all as execution proceeds:

| Hook | When | Frequency |
|------|------|-----------|
| pre-wave-start | Before each wave | Once per wave |
| post-task-complete | After each task | Once per task |
| on-failure | When task fails | Per failure |
| on-session-end | Execution ends | Once at end |

</state_integration>

<error_handling>

## Error Conditions

### Missing prerequisites

```
ERROR: graph.yaml not found
→ Run /ptf:plan first to generate execution plan.
```

### Already complete

```
Plan already complete
→ Nothing to execute. Check /ptf:status for summary.
```

### Mid-execution failure

Handled via EXECUTION PAUSED return.
Options presented for recovery.

### Orchestrator failure

If orchestrator itself fails (not task failure):
- Treated as EXECUTION BLOCKED
- State preserved at last checkpoint
- Manual investigation needed

</error_handling>

<success_criteria>
- [ ] Prerequisites validated before starting
- [ ] Orchestrator dispatched with full-plan mode
- [ ] All waves executed in sequence
- [ ] Automatic progression between waves
- [ ] Appropriate handling of failures/completion
- [ ] Clear final status display
- [ ] Next steps provided based on outcome
</success_criteria>
