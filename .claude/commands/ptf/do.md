---
name: ptf:do
description: One-shot ad-hoc development task - decompose, plan, and execute in a single command
argument-hint: "<task description>"
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
  - Task
  - AskUserQuestion
  - TodoWrite
  - TeamCreate
  - TeamDelete
  - SendMessage
---

<objective>
Execute an ad-hoc software development task end-to-end using the PTF framework.
Takes a natural language goal and runs the full pipeline: init, decompose, plan,
execute-all — automatically with the `dev-do` adapter. No manual phase transitions.

**Usage:**
- `/ptf:do Code review the current changes and fix any issues using TDD`
- `/ptf:do Refactor the authentication system to be more modular`
- `/ptf:do Implement a proper Settings screen in this app`
- `/ptf:do Study the codebase, design a tracking plan, integrate Mixpanel`

**Domain:** Always uses `dev-do` adapter (TDD-first, ad-hoc development)

**Creates:** Full `.orchestrator/` state tree, then executes all tasks

**After this command:** Check `/ptf:status` for detailed results
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
@adapters/dev-do.yaml
@.claude/agents/ptf-orchestrator.md
@.claude/agents/ptf-executor.md
@.claude/agents/ptf-team-lead.md
@.claude/agents/ptf-team-executor.md
</execution_context>

<context>
Goal: $ARGUMENTS
</context>

<process>

## Guard: Validate Input

```
IF "$ARGUMENTS" is empty or blank:
  Display: "Usage: /ptf:do <task description>"
  Display: ""
  Display: "Examples:"
  Display: "  /ptf:do Code review the current changes and fix issues with TDD"
  Display: "  /ptf:do Refactor the auth system for modularity and test coverage"
  Display: "  /ptf:do Implement a Settings screen with proper state management"
  Exit
```

## Guard: Check for Existing Project

```bash
if [ -f .orchestrator/goal.md ]; then
  echo "WARNING: .orchestrator/ already exists from a previous PTF session."
fi
```

If `.orchestrator/` exists:

Use AskUserQuestion:
- header: "Existing State"
- question: "A previous PTF session exists in .orchestrator/. How should we proceed?"
- options:
  - "Clean start" - Remove .orchestrator/ and start fresh
  - "Abort" - Keep existing state and exit

If "Clean start":
```bash
rm -rf .orchestrator/
echo "Cleared previous PTF state"
```

If "Abort":
```
Display: "Keeping existing state. Use /ptf:status to check progress or /ptf:resume to continue."
Exit
```

## Phase 1: Initialize (from /ptf:init, streamlined)

Use TodoWrite to set up progress tracking:

```
Todos:
1. [in_progress] Initialize project with dev-do adapter
2. [pending] Ask clarifying questions
3. [pending] Decompose goal into atomic tasks
4. [pending] Generate execution plan
5. [pending] Execute all waves
6. [pending] Report results
```

### 1.1 Create directory structure

```bash
mkdir -p .orchestrator/decomposition
mkdir -p .orchestrator/state/waves
mkdir -p .orchestrator/state/tasks
mkdir -p .orchestrator/artifacts
mkdir -p .orchestrator/history
```

### 1.2 Write goal.md (immutable)

Write `.orchestrator/goal.md`:
```markdown
---
received: {ISO 8601 timestamp}
source: /ptf:do
---

{$ARGUMENTS - the original goal text, verbatim}
```

### 1.3 Load dev-do adapter

Read `adapters/dev-do.yaml` to extract:
- `questioning.init_questions` - The 4 clarification questions
- `constitution.template` - Constitution template
- `decomposition.max_recursion_depth` - For config (4)

### 1.4 Write config.yaml

Write `.orchestrator/config.yaml`:
```yaml
# PTF Project Configuration (ad-hoc via /ptf:do)
created: {timestamp}
domain: dev-do
adapter: adapters/dev-do.yaml
goal_file: .orchestrator/goal.md
source_command: /ptf:do

decomposition:
  max_recursion_depth: 4

execution:
  mode: classic
  max_parallel_tasks: 3
  ralph_mode: true
  max_ralph_iterations: 3

checkpoints:
  wave_boundary: true
  verification: true
```

Mark todo 1 complete. Mark todo 2 in_progress.

## Phase 2: Ask Clarifying Questions

Present the 4 init questions from the dev-do adapter, adapted to the specific goal.

**Question 1 (task_nature):**

Use AskUserQuestion:
- header: "Task Nature"
- question: "Is this primarily investigation/analysis, implementation/modification, or both?"
- options:
  - "Investigation first, then implement" - Code review, debugging, optimization — explore before changing
  - "Mostly implementation" - Feature dev, integration — jump to building
  - "Both equally" - Refactoring, TDD, study-then-build — interleaved

**Question 2 (scope_awareness):**

Use AskUserQuestion:
- header: "Codebase Familiarity"
- question: "How familiar are you with the relevant parts of the codebase?"
- options:
  - "Very familiar" - Know exactly what to change
  - "Somewhat familiar" - Know the area but not details
  - "Not familiar" - Need to explore first

**Question 3 (quality_bar):**

Use AskUserQuestion:
- header: "Thoroughness"
- question: "What level of thoroughness is expected?"
- options:
  - "Quick and focused" - Minimal testing, ship fast
  - "Standard" - Reasonable test coverage, clean code
  - "Production-grade" - Comprehensive tests, docs, edge cases

**Question 4 (test_granularity):**

Use AskUserQuestion:
- header: "Test Coverage"
- question: "What level of test coverage does this task need?"
- options:
  - "Unit tests only" - Test individual functions and modules
  - "Unit + integration" - Test both units and their interactions
  - "Full stack" - Unit, integration, and end-to-end tests
  - "Match existing" - Follow the project's current test patterns

### 2.1 Write analysis.yaml

Synthesize goal text + clarification responses into structured analysis.

Write `.orchestrator/decomposition/analysis.yaml`:
```yaml
step: 1-goal-analysis
created: {timestamp}
status: complete

objective: |
  {refined objective synthesized from goal and clarification}

scope:
  included:
    - {item 1 derived from goal analysis}
    - {item 2 derived from goal analysis}
  excluded:
    - {explicitly out of scope}

constraints:
  - TDD: All code changes follow test-first development
  - {constraint from quality_bar response}
  - {constraint from scope_awareness response}

success_criteria:
  - {testable criterion 1}
  - {testable criterion 2}
  - {testable criterion 3}

domain: dev-do

clarification_summary:
  task_nature: {response}
  scope_awareness: {response}
  quality_bar: {response}
  test_granularity: {response}
```

### 2.2 Write constitution.yaml

Expand the dev-do adapter's constitution template with project-specific constraints.

Replace placeholders:
- `{project_name}` - Derived from goal (short descriptive name)
- `{project_constraints}` - Constraints from clarification responses
- `{timestamp}` - Current timestamp

Write `.orchestrator/decomposition/constitution.yaml`:
```yaml
generated: {timestamp}
domain: dev-do

principles: |
  {expanded constitution template with constraints applied}

enforcement:
  - Every task must be validated against these principles
  - Violations indicate incorrectly defined tasks
  - Constitution is loaded into every decomposer context
```

Mark todo 2 complete. Mark todo 3 in_progress.

## Phase 3: Decompose (from /ptf:decompose)

Spawn the decomposer subagent with full context.

```
Task(prompt="
<context>
Analysis: @.orchestrator/decomposition/analysis.yaml
Constitution: @.orchestrator/decomposition/constitution.yaml
Adapter: @adapters/dev-do.yaml
</context>

<instructions>
Execute the 5-step decomposition process:

1. Goal Analysis - SKIP (already done, analysis.yaml exists)
2. Identify subgoals using dev-do adapter heuristics:
   - by-phase: investigate → plan → write tests → implement → verify
   - by-concern: split by distinct responsibilities
   - by-file-scope: one task per file/module
   - by-test-boundary: red-green pairs (failing test → code to pass)
   - by-risk-level: safe changes first

3. Recursively decompose to atomic tasks. Apply atomicity criteria:
   - single-concern: one specific change per task
   - fresh-context-completable: <30% context window
   - verifiable: concrete verification method
   - bounded-scope: at most 3 tightly-coupled files
   - explicit-inputs: all dependencies declared
   - tdd-paired: every implementation has a preceding test task

4. Validate coverage, overlap, atomicity

5. Build initial dependency graph

CRITICAL TDD ENFORCEMENT:
- Every implementation task MUST have a corresponding test task that precedes it
- Test tasks produce FAILING tests that specify desired behavior
- Implementation tasks produce minimal code to make tests pass
- The dependency graph must reflect test → implementation ordering

Write all state files after each step.
Return DECOMPOSITION COMPLETE or DECOMPOSITION BLOCKED with details.
</instructions>
", subagent_type="ptf:decomposer")
```

### 3.1 Handle decomposer result

**If DECOMPOSITION COMPLETE:**

Verify state files exist:
```bash
test -f .orchestrator/decomposition/subgoals.yaml && echo "subgoals OK"
test -d .orchestrator/decomposition/tasks && echo "tasks OK"
test -f .orchestrator/decomposition/validation.yaml && echo "validation OK"
test -f .orchestrator/decomposition/graph.yaml && echo "graph OK"

TASK_COUNT=$(ls .orchestrator/decomposition/tasks/*.yaml 2>/dev/null | wc -l | tr -d ' ')
echo "Created ${TASK_COUNT} tasks"
```

Mark todo 3 complete. Mark todo 4 in_progress.
Continue to Phase 4.

**If DECOMPOSITION BLOCKED:**

Display the blocking reason and present options:

Use AskUserQuestion:
- header: "Decomposition Blocked"
- question: "The decomposer encountered a blocker: {reason}. How to proceed?"
- options:
  - "Refine goal" - Adjust the goal description and retry
  - "Force continue" - Skip blocked checks (with warnings)
  - "Abort" - Exit, preserving state for debugging

Handle accordingly. If "Abort": exit with state preserved.

## Phase 4: Plan (from /ptf:plan)

### 4.1 Run dependency analysis (if graph incomplete)

Check if graph.yaml needs dependency analysis:

```bash
if [ -f .orchestrator/decomposition/graph.yaml ]; then
  GRAPH_STATUS=$(grep "^status:" .orchestrator/decomposition/graph.yaml | awk '{print $2}')
else
  GRAPH_STATUS="missing"
fi
```

If graph needs analysis, spawn dependency analyzer:

```
Task(prompt="
<context>
Config: @.orchestrator/config.yaml
Tasks: .orchestrator/decomposition/tasks/*.yaml
Analysis: @.orchestrator/decomposition/analysis.yaml
Domain adapter: @adapters/dev-do.yaml
</context>

<instructions>
Analyze dependencies for all tasks in .orchestrator/decomposition/tasks/

Run multi-pass dependency inference:
1. Artifact matching (HIGH confidence) - exact input/output path matches
2. Type/pattern matching (MEDIUM confidence) - glob patterns
3. Semantic analysis (MEDIUM confidence) - description references
4. Domain heuristics from dev-do adapter:
   - analysis-to-plan (HIGH): plans depend on investigation findings
   - plan-to-source (HIGH): implementation follows plans
   - analysis-to-source (HIGH): implementation depends on review findings
   - test-to-source (HIGH): TDD - implementation depends on pre-written tests
   - source-to-test (MEDIUM): tests for existing code under review
   - config-to-source (MEDIUM): source depends on configuration
5. Resource conflicts (HIGH confidence) - overlapping outputs

Then:
- Detect cycles using Tarjan's algorithm
- Compute waves using Kahn's algorithm
- VERIFY: test tasks appear in earlier waves than their implementation tasks
- Write results to .orchestrator/decomposition/graph.yaml

Return ANALYSIS COMPLETE or ANALYSIS BLOCKED.
</instructions>
", subagent_type="ptf:dependency-analyzer")
```

Handle cycle detection same as /ptf:plan (see cycle_handling in plan.md).

### 4.2 Generate plan.md

Read graph.yaml, all task files, and analysis.yaml.
Generate `.orchestrator/plan.md` with:
- Header: goal summary, task/wave counts, timestamp
- Wave-by-wave breakdown with task tables
- Dependency graph visualization
- Dependency summary by type and confidence
- Warnings for low-confidence dependencies

### 4.3 Initialize execution state

Write `.orchestrator/state/execution.yaml`:
```yaml
version: "1.0"
plan_id: {from graph.yaml or generate}
status: pending
current_wave: 1
waves_total: {WAVE_COUNT}
progress:
  tasks_total: {TASK_COUNT}
  tasks_completed: 0
  tasks_failed: 0
  tasks_blocked: 0
started_at: null
completed_at: null
wave_summary: []
blockers: []
```

Initialize manifest and events log:
```bash
echo 'version: "1.0"' > .orchestrator/artifacts/manifest.yaml
echo 'artifacts: []' >> .orchestrator/artifacts/manifest.yaml
touch .orchestrator/history/events.jsonl
```

### 4.4 Display plan for review

Display the generated plan.md content to the user.

Mark todo 4 complete. Mark todo 5 in_progress.

### 4.5 Confirm execution

Use AskUserQuestion:
- header: "Execute Plan?"
- question: "Plan is ready with {TASK_COUNT} tasks in {WAVE_COUNT} waves. Proceed with execution?"
- options:
  - "Execute (classic mode)" - Wave-by-wave execution with automatic progression
  - "Execute (teams mode)" - Dynamic scheduling via Agent Teams (experimental)
  - "Review plan first" - Display the full plan before deciding
  - "Abort" - Exit without executing, plan is saved

If "Review plan first":
  Display `.orchestrator/plan.md` content.
  Then re-ask execution question.

If "Abort":
  ```
  Display: "Plan saved to .orchestrator/plan.md"
  Display: "Resume later with /ptf:execute-all"
  ```
  Commit state and exit.

If "Execute (teams mode)":
  Update config.yaml to set `execution.mode: teams`
  Check CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS env var

## Phase 5: Execute All (from /ptf:execute-all)

### 5.1 Commit all state before execution

Stage and commit everything generated so far:

```bash
git add .orchestrator/
git commit -m "$(cat <<'EOF'
ptf: initialize ad-hoc task via /ptf:do

Goal: {one-liner summary}
Domain: dev-do
{TASK_COUNT} tasks in {WAVE_COUNT} waves
Ready for execution

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 5.2 Determine execution mode

```bash
MODE=$(grep "mode:" .orchestrator/config.yaml 2>/dev/null | head -1 | awk '{print $2}')
MODE=${MODE:-classic}
```

### 5.3a Execute (Classic Mode)

Spawn the orchestrator for full-plan execution:

```
Task(prompt="
<mode>full-plan</mode>
<start_wave>1</start_wave>
<waves_total>{waves_total}</waves_total>

Execute all waves in the plan.

Current state:
- Status: pending
- Current wave: 1
- Waves total: {waves_total}

Configuration:
- max_parallel_tasks: 3
- execution_mode: ralph
- max_iterations: 3

Execution flow:
1. For each wave from 1 to {waves_total}:
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

**TDD ENFORCEMENT during execution:**
- Test tasks MUST produce failing tests before implementation tasks run
- Implementation tasks receive test files as inputs
- Verification checks that tests pass after implementation

Return PLAN COMPLETE, EXECUTION PAUSED, or EXECUTION BLOCKED.
", subagent_type="ptf:orchestrator")
```

### 5.3b Execute (Teams Mode)

Only if user selected teams mode and env var is set:

```
Task(prompt="
<mode>teams</mode>
<plan_id>{plan_id}</plan_id>

Execute all tasks using Agent Teams with dynamic scheduling.

Configuration:
- worker_count: 3
- max_parallel_tasks: 3
- checkpoint_frequency: task

Required files:
- Graph: .orchestrator/decomposition/graph.yaml
- Tasks: .orchestrator/decomposition/tasks/*.yaml
- State: .orchestrator/state/execution.yaml
- Config: .orchestrator/config.yaml

Execution flow:
1. Create Agent Teams team (ptf-{plan_id})
2. Convert dependency graph to shared task list with dependencies
3. Spawn 3 executor teammates
4. Monitor teammate messages for task completion/failure
5. Handle failures per task on_failure policy
6. Log all events to .orchestrator/history/events.jsonl
7. Checkpoint state after each task completion
8. When all tasks complete: shutdown teammates, cleanup team

**CRITICAL: You are the SINGLE WRITER for all state files and events.jsonl.**
Teammates report via SendMessage. You process messages and write state.

**TDD ENFORCEMENT:**
- Tasks with test dependencies must not be dispatched until test tasks complete
- Test tasks produce failing tests; implementation tasks make them pass

Return PLAN COMPLETE, EXECUTION PAUSED, or EXECUTION BLOCKED.
", subagent_type="ptf:team-lead")
```

### 5.4 Handle execution result

Wait for orchestrator/team-lead to return.

Mark todo 5 complete. Mark todo 6 in_progress.

## Phase 6: Report Results

### 6.1 Read final state

```bash
cat .orchestrator/state/execution.yaml
TASK_COUNT=$(ls .orchestrator/decomposition/tasks/*.yaml 2>/dev/null | wc -l | tr -d ' ')
EVENT_COUNT=$(wc -l < .orchestrator/history/events.jsonl 2>/dev/null | tr -d ' ')
ARTIFACT_COUNT=$(grep -c "^  - " .orchestrator/artifacts/manifest.yaml 2>/dev/null || echo 0)
```

### 6.2 Commit execution results

```bash
git add .orchestrator/
git commit -m "$(cat <<'EOF'
ptf: complete ad-hoc task execution

{COMPLETED_TASKS}/{TASK_COUNT} tasks completed
{ARTIFACT_COUNT} artifacts produced
{EVENT_COUNT} events logged

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### 6.3 Display final report

**On PLAN COMPLETE:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PTF:DO ► TASK COMPLETE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Goal:** {original goal}

## Results

| Metric | Value |
|--------|-------|
| Tasks completed | {completed}/{total} |
| Waves executed | {wave_count} |
| Artifacts produced | {artifact_count} |
| Events logged | {event_count} |

## Artifacts Produced

{list key artifacts from manifest.yaml}

## Verification

All task verifications passed.

---

**Next Steps:**

1. `/ptf:status` - Detailed execution summary
2. `/ptf:verify` - Re-run verification on specific tasks
3. Review artifacts in `.orchestrator/artifacts/manifest.yaml`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**On EXECUTION PAUSED:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PTF:DO ► EXECUTION PAUSED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Progress:** Wave {N} of {total}
**Reason:** {reason}

### Completed Tasks
{list completed tasks}

### Failed Tasks
| Task | Error | Attempts |
|------|-------|----------|
| {task-id} | {error} | {N} |

### Blocked Tasks
{list tasks blocked by failures}

---

**Recovery Options:**

1. `/ptf:retry {task-id}` - Retry a specific failed task
2. `/ptf:resume` - Resume from current position
3. `/ptf:status` - View detailed state
4. `/ptf:abort` - Stop execution, preserve state

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**On EXECUTION BLOCKED:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 PTF:DO ► EXECUTION BLOCKED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

**Progress:** Wave {N} of {total}
**Blocker:** {detailed description}

### Completed Work
All completed work is checkpointed and preserved.

---

**Resolution:**
{steps to resolve the blocking issue}

After resolving: `/ptf:resume`

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Mark todo 6 complete.

</process>

<comparison_with_manual_flow>

## /ptf:do vs Manual Flow

| Aspect | /ptf:do | Manual (/ptf:init → decompose → plan → execute-all) |
|--------|---------|------------------------------------------------------|
| Commands | 1 | 4-5 separate commands |
| Domain detection | Always `dev-do` | Auto-detected or specified |
| Git commits | 2 (pre-execution + post-execution) | 1 per phase (4-5 commits) |
| Phase transitions | Automatic | Manual with `/clear` between |
| Plan review | Optional (can skip to execute) | Always shown, manual proceed |
| Best for | Ad-hoc, well-understood tasks | Large projects, careful review |
| Context usage | Single conversation | Fresh context per phase |

**When to use /ptf:do:**
- Ad-hoc development tasks that benefit from decomposition
- When you want minimal ceremony and fast execution
- Tasks where you trust the framework to decompose well

**When to use the manual flow:**
- Large, multi-day projects
- When you want to review/edit the plan before execution
- When you need to customize decomposition or dependencies
- First time using PTF (learn each phase)

</comparison_with_manual_flow>

<success_criteria>
- [ ] $ARGUMENTS parsed as goal text
- [ ] Existing .orchestrator/ handled (clean or abort)
- [ ] dev-do adapter loaded (not auto-detected)
- [ ] 4 init questions asked and responses captured
- [ ] analysis.yaml, constitution.yaml written
- [ ] Decomposer subagent spawned and completed
- [ ] tasks/*.yaml created with TDD-paired structure
- [ ] Dependency analyzer run, graph.yaml complete
- [ ] plan.md generated and shown to user
- [ ] execution.yaml initialized
- [ ] User confirms execution (classic or teams)
- [ ] All state committed to git before execution
- [ ] Orchestrator/team-lead dispatched for full execution
- [ ] Final results committed to git
- [ ] Clear final report displayed
- [ ] TodoWrite progress tracked throughout
</success_criteria>
