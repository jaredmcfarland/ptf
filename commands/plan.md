---
description: Generate human-readable execution plan from decomposition
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Generate a human-readable execution plan showing waves and task ordering.

**Requires:**
- `.orchestrator/decomposition/tasks/*.yaml` (from /ptf:decompose)

**Creates/Updates:**
- `.orchestrator/decomposition/graph.yaml` (if not exists)
- `.orchestrator/plan.md` (human-readable output)

**After this command:** Run `/ptf:execute` to begin execution.
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
</execution_context>

<context>
@.orchestrator/decomposition/analysis.yaml
@.orchestrator/decomposition/constitution.yaml
@.orchestrator/config.yaml
</context>

<process>

## Phase 1: Validate Prerequisites

Check for decomposition output:

```bash
[ ! -d .orchestrator/decomposition/tasks ] && \
  echo "ERROR: Tasks directory not found. Run /ptf:decompose first." && exit 1

TASK_COUNT=$(ls .orchestrator/decomposition/tasks/*.yaml 2>/dev/null | wc -l | tr -d ' ')
if [ "$TASK_COUNT" -eq 0 ]; then
  echo "ERROR: No task files found. Run /ptf:decompose first."
  exit 1
fi

echo "Found ${TASK_COUNT} task files"
```

Load configuration:
```bash
if [ -f .orchestrator/config.yaml ]; then
  DOMAIN=$(grep "^domain:" .orchestrator/config.yaml | awk '{print $2}')
  echo "Domain: ${DOMAIN}"
else
  echo "WARNING: config.yaml not found, using software-development"
  DOMAIN="software-development"
fi
```

## Phase 2: Run Dependency Analysis (if needed)

Check if graph.yaml already exists:

```bash
if [ -f .orchestrator/decomposition/graph.yaml ]; then
  GRAPH_STATUS=$(grep "^status:" .orchestrator/decomposition/graph.yaml | awk '{print $2}')
  if [ "$GRAPH_STATUS" = "complete" ]; then
    echo "graph.yaml exists and is complete - skipping analysis"
  else
    echo "graph.yaml exists but status is ${GRAPH_STATUS} - re-running analysis"
    NEED_ANALYSIS=true
  fi
else
  echo "graph.yaml not found - running dependency analysis"
  NEED_ANALYSIS=true
fi
```

If analysis needed, spawn the dependency analyzer:

```
Task(prompt="
<context>
Config: @.orchestrator/config.yaml
Tasks: .orchestrator/decomposition/tasks/*.yaml
Analysis: @.orchestrator/decomposition/analysis.yaml

Domain adapter: @adapters/{DOMAIN}.yaml
</context>

<instructions>
Analyze dependencies for all tasks in .orchestrator/decomposition/tasks/

Run multi-pass dependency inference:
1. Artifact matching (HIGH confidence) - exact input/output path matches
2. Type/pattern matching (MEDIUM confidence) - glob patterns
3. Semantic analysis (MEDIUM confidence) - description references
4. Domain heuristics (LOW confidence) - adapter patterns
5. Resource conflicts (HIGH confidence) - overlapping outputs

Then:
- Detect cycles using Tarjan's algorithm
- Compute waves using Kahn's algorithm
- Write results to .orchestrator/decomposition/graph.yaml

Return ANALYSIS COMPLETE or ANALYSIS BLOCKED.
</instructions>
", subagent_type="ptf-dependency-analyzer")
```

Wait for analyzer to complete.

## Phase 3: Handle Analyzer Results

Parse the analyzer's structured return.

**If ANALYSIS COMPLETE:**

Read and verify graph.yaml:
```bash
test -f .orchestrator/decomposition/graph.yaml && echo "graph.yaml created"
WAVE_COUNT=$(grep -c "^  - number:" .orchestrator/decomposition/graph.yaml || echo 0)
DEP_COUNT=$(grep "^  total_dependencies:" .orchestrator/decomposition/graph.yaml | awk '{print $2}' || echo 0)
echo "Dependencies: ${DEP_COUNT}, Waves: ${WAVE_COUNT}"
```

Continue to Phase 4.

**If ANALYSIS BLOCKED:**

See <cycle_handling> section for detailed resolution flow.

## Phase 4: Generate Plan Output

Read graph.yaml and all task files.
Generate .orchestrator/plan.md using the format in <plan_template>.

Steps:
1. Read analysis.yaml for goal context (objective, success criteria)
2. Read graph.yaml for dependencies and waves
3. Read each task file for details (description, outputs, context budget)
4. Generate markdown document with all sections

Template sections to include:
- Header with goal summary, counts, timestamp
- Wave-by-wave breakdown with task tables
- ASCII dependency graph visualization
- Dependency summary by type and confidence
- Warnings for low-confidence dependencies
- Next steps with /ptf:execute instruction

See <plan_template> for exact format specification.

## Phase 5: Initialize Execution State

Initialize `.orchestrator/state/execution.yaml` so `/ptf:execute` can proceed without a separate initialization step.

**5.1 Create state directories if needed:**

```bash
mkdir -p .orchestrator/state/waves
mkdir -p .orchestrator/state/tasks
mkdir -p .orchestrator/artifacts
mkdir -p .orchestrator/history
```

**5.2 Read wave count from graph.yaml:**

```bash
WAVE_COUNT=$(grep -c "^  - number:" .orchestrator/decomposition/graph.yaml || echo 1)
TASK_COUNT=$(ls .orchestrator/decomposition/tasks/*.yaml 2>/dev/null | wc -l | tr -d ' ')
```

**5.3 Create execution.yaml:**

Write `.orchestrator/state/execution.yaml`:

```yaml
# Auto-generated by /ptf:plan
version: "1.0"
plan_id: {from graph.yaml or generate UUID}
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

**5.4 Create empty manifest and events log:**

```bash
# Create manifest.yaml if not exists
if [ ! -f .orchestrator/artifacts/manifest.yaml ]; then
  echo "version: \"1.0\"" > .orchestrator/artifacts/manifest.yaml
  echo "artifacts: []" >> .orchestrator/artifacts/manifest.yaml
fi

# Create events.jsonl if not exists
touch .orchestrator/history/events.jsonl
```

## Phase 6: Commit and Present

Stage and commit the files:

```bash
# Stage graph.yaml if new or modified
git add .orchestrator/decomposition/graph.yaml

# Stage plan.md
git add .orchestrator/plan.md

# Stage execution state files
git add .orchestrator/state/
git add .orchestrator/artifacts/
git add .orchestrator/history/

# Get counts for commit message
TASK_COUNT=$(ls .orchestrator/decomposition/tasks/*.yaml 2>/dev/null | wc -l | tr -d ' ')
WAVE_COUNT=$(grep -c "^  - number:" .orchestrator/decomposition/graph.yaml || echo 1)
PARALLELISM=$(echo "scale=1; ${TASK_COUNT} / ${WAVE_COUNT}" | bc)

git commit -m "$(cat <<EOF
ptf: generate execution plan

${TASK_COUNT} tasks in ${WAVE_COUNT} waves
Parallelism factor: ${PARALLELISM}x
EOF
)"
```

Display plan.md content to user:
```bash
cat .orchestrator/plan.md
```

Show completion message:
```
---
Plan generation complete.

Next: /ptf:execute to begin execution
Tip: /clear first for fresh context window
---
```

</process>

<plan_template>
**Plan.md Output Format**

Generate .orchestrator/plan.md with this structure:

```markdown
# Execution Plan: {goal_name from analysis.yaml}

**Generated:** {timestamp ISO 8601}
**Domain:** {domain from config.yaml}
**Tasks:** {N} tasks in {M} waves
**Estimated parallel speedup:** {parallelism_factor}x (tasks / waves)

---

## Overview

{objective from analysis.yaml}

### Success Criteria
{list from analysis.yaml success_criteria, each as bullet point}

---

## Wave 1 ({task_count} tasks - parallel)

| Task | Description | Outputs | Est. Context |
|------|-------------|---------|--------------|
| {task_id} | {task.description truncated to 60 chars} | {task.outputs[0].path} | {task.context_budget.estimated_input_tokens} |

**No dependencies - can start immediately**

---

## Wave 2 ({task_count} tasks - parallel)

| Task | Description | Outputs | Est. Context |
|------|-------------|---------|--------------|
| {task_id} | {task.description truncated to 60 chars} | {task.outputs[0].path} | {task.context_budget.estimated_input_tokens} |

**Depends on:** Wave 1 ({list of task_ids from wave 1 that these depend on})

---

{Repeat for each wave...}

---

## Dependency Graph

```
{ASCII art visualization showing task flow}

task-a ──────────┬──> task-c ──────> task-e
                 │                       │
task-b ──────────┘                       └──> task-f
```

For simpler representation, use wave-based layout:

```
Wave 1: [task-a, task-b]
   │
   └──> Wave 2: [task-c]
           │
           └──> Wave 3: [task-d, task-e]
                   │
                   └──> Wave 4: [task-f]
```

---

## Dependency Summary

| Type | Count | Confidence |
|------|-------|------------|
| artifact | {count} | HIGH |
| semantic | {count} | MEDIUM |
| implicit | {count} | LOW |
| resource | {count} | HIGH |
| **Total** | {total} | |

---

## Warnings

{For each dependency with confidence: low}
- Task `{to}` depends on `{from}` via {type} ({confidence} confidence): {reason}

{For each warning in graph.yaml validation.warnings}
- {warning}

{If no warnings: "No warnings - all dependencies are high or medium confidence."}

---

## Ready for Execution

All dependencies resolved. No cycles detected.

**Next:** Run `/ptf:execute` to begin execution

<sub>`/clear` first -> fresh context window recommended</sub>
```

</plan_template>

<cycle_handling>
**Handling Dependency Cycles**

When analyzer returns ANALYSIS BLOCKED with cycle details:

## Step 1: Parse Cycle Information

From analyzer output, extract:
- cycles: [[task-ids in each cycle]]
- resolution_hints: [
    {
      cycle: [task-ids],
      suggested_break: {from, to, type, confidence, reason},
      reason: "explanation"
    }
  ]

## Step 2: Present to User

Use structured prompt with options:

```
## Dependency Cycle Detected

A cycle was detected in the dependency graph:

**Cycle:** {cycle_tasks joined by " -> "} -> {first_task}

This means these tasks have circular dependencies and cannot be executed.

**Suggested resolution:**
Break the dependency from `{suggested_break.from}` to `{suggested_break.to}`
- Type: {suggested_break.type}
- Confidence: {suggested_break.confidence}
- Reason: {suggested_break.reason}

How would you like to resolve this?

**Options:**
1. **Break suggested dependency** - Remove this dependency and re-run analysis
2. **Edit tasks manually** - Open task files to fix the cycle yourself
3. **Abort** - Exit without generating plan
```

Use AskUserQuestion with these options.

## Step 3: Handle User Choice

**If "Break suggested dependency":**

1. Read graph.yaml (if exists) or initialize empty
2. Add to graph.yaml `ignored_dependencies` list:
   ```yaml
   ignored_dependencies:
     - from: {task-id}
       to: {task-id}
       reason: "User chose to break cycle"
       original_type: {type}
       original_confidence: {confidence}
   ```
3. Re-run dependency analyzer with context noting ignored dependencies:
   ```
   Task(prompt="
   <context>
   Previous analysis found cycles.
   User has chosen to ignore these dependencies:
   {list from ignored_dependencies}

   Re-run analysis excluding these dependencies.
   </context>
   ", subagent_type="ptf-dependency-analyzer")
   ```
4. Handle new result (may find more cycles or succeed)

**If "Edit tasks manually":**

1. List task files involved in cycle:
   ```
   The following task files need editing to resolve the cycle:

   - .orchestrator/decomposition/tasks/{task-a}.yaml
   - .orchestrator/decomposition/tasks/{task-b}.yaml
   - .orchestrator/decomposition/tasks/{task-c}.yaml

   Common fixes:
   - Remove an input that creates the dependency
   - Split a task into two independent tasks
   - Combine cyclic tasks into one atomic task

   After editing, re-run /ptf:plan
   ```
2. Exit with status: "Awaiting manual task edits"

**If "Abort":**

1. Exit with message: "Plan generation aborted due to dependency cycle"
2. Leave state files unchanged for debugging

## Multiple Cycles

If multiple cycles detected:
- Address one at a time
- After resolving first, re-run analysis
- New cycles may be revealed or previous ones may resolve
- Repeat until all cycles resolved or user aborts

## Prevented Cycles

If an ignored dependency would prevent plan generation:
- Warn user that the original dependency may have been correct
- Suggest re-evaluating the task structure
- Option to restore the dependency and try a different break point

</cycle_handling>

<success_criteria>
- [ ] .orchestrator/decomposition/tasks/ exists with at least one task
- [ ] graph.yaml exists with dependencies and waves (created or pre-existing)
- [ ] No cycles in dependency graph (or user resolved them)
- [ ] plan.md generated with human-readable format
- [ ] plan.md includes: header, wave tables, dependency graph, summary, warnings
- [ ] execution.yaml created with status: pending (bridges gap to /ptf:execute)
- [ ] All files committed to git
- [ ] User knows to run `/ptf:execute` next
</success_criteria>
