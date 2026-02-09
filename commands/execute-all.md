---
description: Execute all waves with automatic progression
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
  - TeamCreate
  - TeamDelete
  - SendMessage
  - TodoWrite
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
@.claude/agents/ptf-team-lead.md
@.claude/agents/ptf-team-executor.md
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

## Phase 1.5: Determine Execution Mode

Read execution mode from config:

```bash
MODE=$(grep "mode:" .orchestrator/config.yaml 2>/dev/null | head -1 | awk '{print $2}')
MODE=${MODE:-classic}
```

**If MODE == "teams":**

Check that Agent Teams is enabled:
```bash
if [ -z "$CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS" ]; then
  echo "WARNING: Agent Teams not enabled. Set CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1"
  echo "Falling back to classic wave-based execution."
  MODE="classic"
fi
```

If teams mode is confirmed, skip to Phase 2T below.
Otherwise, continue to Phase 2 (classic).

## Phase 2: Execute All Waves (Classic Mode)

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

**CRITICAL: Event Logging Verification**
After EACH state-manager call:
1. Check return includes `events_logged` array
2. Verify event was written: `tail -1 .orchestrator/history/events.jsonl`
3. If missing, retry state-manager call once
Event log path: `.orchestrator/history/events.jsonl` (ONLY this path)

Return PLAN COMPLETE, EXECUTION PAUSED, or EXECUTION BLOCKED.
", subagent_type="ptf-orchestrator")
```

**2.3 Wait for orchestrator:**

The orchestrator handles the wave-by-wave loop internally.
Wait for final return indicating completion or stopping point.

Skip to Phase 3.

## Phase 2T: Execute All Tasks (Teams Mode)

Spawn team lead for dynamic scheduling via Agent Teams.

**2T.1 Read teams configuration:**

```bash
WORKER_COUNT=$(grep "worker_count:" .orchestrator/config.yaml 2>/dev/null | awk '{print $2}')
WORKER_COUNT=${WORKER_COUNT:-3}  # Default to 3 workers
```

**2T.2 Dispatch team lead:**

```
Task(prompt="
<mode>teams</mode>
<plan_id>{plan_id}</plan_id>

Execute all tasks using Agent Teams with dynamic scheduling.

Configuration:
- worker_count: {WORKER_COUNT}
- max_parallel_tasks: {from config.yaml or default 5}
- checkpoint_frequency: {from config.yaml or default task}

Required files:
- Graph: .orchestrator/decomposition/graph.yaml
- Tasks: .orchestrator/decomposition/tasks/*.yaml
- State: .orchestrator/state/execution.yaml
- Config: .orchestrator/config.yaml

Current state:
- Status: {status}
- Current wave: {current_wave}
- Waves total: {waves_total}
- Tasks remaining: {total_tasks}

Execution flow:
1. Create Agent Teams team (ptf-{plan_id})
2. Convert dependency graph to shared task list with dependencies
3. Spawn {WORKER_COUNT} executor teammates
4. Monitor teammate messages for task completion/failure
5. Handle failures per task on_failure policy
6. Log all events to .orchestrator/history/events.jsonl
7. Checkpoint state after each task completion
8. When all tasks complete: shutdown teammates, cleanup team

**CRITICAL: You are the SINGLE WRITER for all state files and events.jsonl.**
Teammates report via SendMessage. You process messages and write state.

Return PLAN COMPLETE, EXECUTION PAUSED, or EXECUTION BLOCKED.
", subagent_type="ptf-team-lead")
```

**2T.3 Wait for team lead:**

The team lead handles the full execution lifecycle internally:
team creation → task list population → worker dispatch → monitoring → shutdown → cleanup.
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

## /ptf:execute vs /ptf:execute-all vs /ptf:execute-all (teams)

| Aspect | /ptf:execute | /ptf:execute-all (classic) | /ptf:execute-all (teams) |
|--------|--------------|---------------------------|--------------------------|
| Scope | Single wave | All remaining waves | All remaining tasks |
| Scheduling | Wave boundaries | Wave boundaries | Dynamic (dependency-driven) |
| Parallelism | Within wave only | Within wave only | Across waves |
| Progression | Manual | Automatic | Automatic |
| Coordinator | ptf-orchestrator | ptf-orchestrator | ptf-team-lead |
| User control | After each wave | At completion/failure | At completion/failure |
| Best for | Step-by-step | Hands-off, smaller plans | Large plans, uneven tasks |

**When to use /ptf:execute:**
- Want to review results between waves
- Debugging task failures
- First-time execution (verify each step)
- Complex plans with potential issues

**When to use /ptf:execute-all (classic mode):**
- Confident in plan correctness
- Want unattended execution
- Resuming after interruption
- Smaller plans where wave overhead is minimal

**When to use /ptf:execute-all (teams mode):**
- Large plans (10+ tasks) with uneven task durations
- Want maximum parallelism and throughput
- Tasks have fine-grained dependencies (not just wave-level)
- Willing to use experimental Agent Teams feature

**Configuration for teams mode:**
```yaml
# .orchestrator/config.yaml
execution:
  mode: teams
  teams:
    worker_count: 3
```

Also requires: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

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
