---
name: ptf:decompose
description: Run full 5-step decomposition process
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Execute the 5-step decomposition process to transform the analyzed goal into validated atomic tasks with dependency graph.

**Requires:** `.orchestrator/decomposition/analysis.yaml` (from /ptf:init)

**Creates:**
- `.orchestrator/decomposition/subgoals.yaml` - Step 2: Identified subgoals
- `.orchestrator/decomposition/tasks/*.yaml` - Step 3: Atomic task definitions
- `.orchestrator/decomposition/validation.yaml` - Step 4: Validation results
- `.orchestrator/decomposition/graph.yaml` - Step 5: Dependency graph

**After this command:** Run `/ptf:plan` to generate the execution plan.
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
</execution_context>

<context>
@.orchestrator/decomposition/analysis.yaml
@.orchestrator/decomposition/constitution.yaml
</context>

<process>

## Phase 1: Validate Prerequisites and Check Resume

1. **Check analysis exists:**
   ```bash
   if [ ! -f .orchestrator/decomposition/analysis.yaml ]; then
     echo "ERROR: analysis.yaml not found. Run /ptf:init first."
     exit 1
   fi
   ```

2. **Check constitution exists:**
   ```bash
   if [ ! -f .orchestrator/decomposition/constitution.yaml ]; then
     echo "ERROR: constitution.yaml not found. Run /ptf:init first."
     exit 1
   fi
   ```

3. **Load config for domain:**
   ```bash
   DOMAIN=$(grep "^domain:" .orchestrator/decomposition/analysis.yaml | awk '{print $2}')
   if [ ! -f "adapters/${DOMAIN}.yaml" ]; then
     echo "WARNING: Adapter not found for ${DOMAIN}, using software-development"
     DOMAIN="software-development"
   fi
   ```

4. **Check for partial decomposition:**
   ```bash
   RESUME_FROM=""

   # Check if decomposition was started but not completed
   if [ -f .orchestrator/decomposition/subgoals.yaml ]; then
     SUBGOALS_STATUS=$(grep "^status:" .orchestrator/decomposition/subgoals.yaml | awk '{print $2}')
     if [ "$SUBGOALS_STATUS" = "in_progress" ]; then
       echo "Partial decomposition found at step 2 (subgoal identification)"
       RESUME_FROM="step2"
     fi
   fi

   if [ -d .orchestrator/decomposition/tasks ] && \
      [ ! -f .orchestrator/decomposition/validation.yaml ]; then
     echo "Partial decomposition found at step 3 (task decomposition)"
     RESUME_FROM="step3"
   fi

   if [ -f .orchestrator/decomposition/validation.yaml ]; then
     VAL_STATUS=$(grep "^status:" .orchestrator/decomposition/validation.yaml | awk '{print $2}')
     if [ "$VAL_STATUS" = "in_progress" ]; then
       echo "Partial decomposition found at step 4 (validation)"
       RESUME_FROM="step4"
     elif [ "$VAL_STATUS" = "failed" ]; then
       echo "Previous decomposition failed validation"
       RESUME_FROM="fix_validation"
     fi
   fi

   # Check if already complete
   if [ -f .orchestrator/decomposition/graph.yaml ]; then
     GRAPH_STATUS=$(grep "^status:" .orchestrator/decomposition/graph.yaml | awk '{print $2}')
     if [ "$GRAPH_STATUS" = "complete" ]; then
       echo "Decomposition already complete. Use /ptf:plan to continue."
       exit 0
     fi
   fi
   ```

5. **Handle resume:**
   If partial decomposition detected (`RESUME_FROM` is set):

   Use AskUserQuestion:
   - header: "Resume Decomposition"
   - question: "Found partial decomposition at {RESUME_FROM}. Resume or restart?"
   - options:
     - "Resume" - Continue from detected step
     - "Restart" - Clear decomposition state and start fresh

   If "Restart":
   ```bash
   rm -f .orchestrator/decomposition/subgoals.yaml
   rm -rf .orchestrator/decomposition/tasks/
   rm -f .orchestrator/decomposition/validation.yaml
   rm -f .orchestrator/decomposition/graph.yaml
   echo "Cleared partial decomposition state"
   RESUME_FROM=""
   ```

## Phase 2: Spawn Decomposer

Build rich prompt for ptf-decomposer subagent with full context:

```
Task(prompt="
<context>
Analysis: @.orchestrator/decomposition/analysis.yaml
Constitution: @.orchestrator/decomposition/constitution.yaml
Adapter: @adapters/{DOMAIN}.yaml
</context>

<instructions>
Execute the 5-step decomposition process:

1. Goal Analysis - SKIP (already done, analysis.yaml exists)
2. Identify subgoals using adapter heuristics
3. Recursively decompose to atomic tasks
4. Validate coverage, overlap, atomicity
5. Build initial dependency graph

Resume from: {RESUME_FROM or 'beginning'}

Write state files after each step.
Return DECOMPOSITION COMPLETE or DECOMPOSITION BLOCKED.
</instructions>
", subagent_type="ptf-decomposer")
```

Wait for subagent to return.

## Phase 3: Handle Results

Parse the decomposer's return:

**If DECOMPOSITION COMPLETE:**

1. **Verify all state files exist:**
   ```bash
   test -f .orchestrator/decomposition/subgoals.yaml && echo "subgoals.yaml exists"
   test -d .orchestrator/decomposition/tasks && echo "tasks/ directory exists"
   test -f .orchestrator/decomposition/validation.yaml && echo "validation.yaml exists"
   test -f .orchestrator/decomposition/graph.yaml && echo "graph.yaml exists"
   ```

2. **Count tasks and subgoals:**
   ```bash
   TASK_COUNT=$(ls .orchestrator/decomposition/tasks/*.yaml 2>/dev/null | wc -l | tr -d ' ')
   SUBGOAL_COUNT=$(grep -c "^  - id:" .orchestrator/decomposition/subgoals.yaml 2>/dev/null || echo 0)
   echo "Created ${TASK_COUNT} tasks in ${SUBGOAL_COUNT} subgoals"
   ```

3. **Verify validation passed:**
   ```bash
   VAL_STATUS=$(grep "^status:" .orchestrator/decomposition/validation.yaml | awk '{print $2}')
   if [ "$VAL_STATUS" != "passed" ]; then
     echo "ERROR: Validation status is ${VAL_STATUS}, expected 'passed'"
     exit 1
   fi
   ```

**If DECOMPOSITION BLOCKED:**

Parse blocking issues from the return.

Present to user with resolution options:

Use AskUserQuestion:
- header: "Decomposition Blocked"
- question: "{Block reason from decomposer return}"
- options:
  - "Adjust goal" - Return to /ptf:init to refine the goal
  - "Force continue" - Skip blocked checks (with warnings logged)
  - "Abort" - Clean up and exit

If "Adjust goal":
```
Return message: "Goal needs refinement. Run /ptf:init to update the goal and try again."
```

If "Force continue":
```bash
# Log warning to validation.yaml
echo "forced_continue: true" >> .orchestrator/decomposition/validation.yaml
echo "forced_continue_reason: '{user reason}'" >> .orchestrator/decomposition/validation.yaml
```
Then proceed to Phase 4.

If "Abort":
```bash
# Keep partial state for debugging
echo "Aborted by user. Partial state preserved for debugging."
exit 1
```

## Phase 4: Commit

Stage and commit all decomposition state:

```bash
git add .orchestrator/decomposition/subgoals.yaml
git add .orchestrator/decomposition/tasks/
git add .orchestrator/decomposition/validation.yaml
git add .orchestrator/decomposition/graph.yaml

git commit -m "$(cat <<'EOF'
ptf: complete decomposition

{TASK_COUNT} tasks in {SUBGOAL_COUNT} subgoals
100% coverage verified
EOF
)"
```

## Phase 5: Complete

Read actual data from state files and present decomposition summary:

```bash
# Read objective from analysis
OBJECTIVE=$(grep -A1 "^objective:" .orchestrator/decomposition/analysis.yaml | tail -1 | sed 's/^  //')

# Read task count
TASK_COUNT=$(ls .orchestrator/decomposition/tasks/*.yaml 2>/dev/null | wc -l | tr -d ' ')

# Read subgoal count
SUBGOAL_COUNT=$(grep -c "^  - id:" .orchestrator/decomposition/subgoals.yaml 2>/dev/null || echo 0)

# Read validation checks
VAL_COVERAGE=$(grep -A2 "coverage:" .orchestrator/decomposition/validation.yaml | grep "status:" | awk '{print $2}')
VAL_OVERLAP=$(grep -A2 "overlap:" .orchestrator/decomposition/validation.yaml | grep "status:" | awk '{print $2}')
VAL_ATOMICITY=$(grep -A2 "atomicity:" .orchestrator/decomposition/validation.yaml | grep "status:" | awk '{print $2}')
VAL_INPUT=$(grep -A2 "input_coverage:" .orchestrator/decomposition/validation.yaml | grep "status:" | awk '{print $2}')
VAL_OUTPUT=$(grep -A2 "output_usefulness:" .orchestrator/decomposition/validation.yaml | grep "status:" | awk '{print $2}')

# Read success criteria count
CRITERIA_COUNT=$(grep -c "^  - " .orchestrator/decomposition/analysis.yaml | head -1 || echo "?")
```

Present formatted summary:

```
---

## PTF DECOMPOSITION COMPLETE

**Goal:** {OBJECTIVE}

### Subgoals

| # | Subgoal | Tasks | Key Outputs |
|---|---------|-------|-------------|
{Read from subgoals.yaml and format as table rows}

### Tasks

{TASK_COUNT} atomic tasks created:

| ID | Name | Outputs | Dependencies |
|----|------|---------|--------------|
{Read from tasks/*.yaml and format as table rows}

### Validation

- Coverage: {VAL_COVERAGE} - All {CRITERIA_COUNT} success criteria mapped
- Overlap: {VAL_OVERLAP} - No duplicate outputs
- Atomicity: {VAL_ATOMICITY} - All tasks meet criteria
- Inputs: {VAL_INPUT} - All inputs have producers
- Outputs: {VAL_OUTPUT} - All outputs consumed or final

### Files Created

- .orchestrator/decomposition/subgoals.yaml
- .orchestrator/decomposition/tasks/ ({TASK_COUNT} files)
- .orchestrator/decomposition/validation.yaml
- .orchestrator/decomposition/graph.yaml

---

## Next Up

**Dependency Analysis** - compute task dependencies and parallel waves

`/ptf:plan` - generate execution plan

<sub>`/clear` first - fresh context window</sub>

---
```

</process>

<success_criteria>
- [ ] .orchestrator/decomposition/analysis.yaml exists (prerequisite)
- [ ] .orchestrator/decomposition/constitution.yaml exists (prerequisite)
- [ ] ptf-decomposer subagent spawned with full context
- [ ] .orchestrator/decomposition/subgoals.yaml created with subgoal structure
- [ ] .orchestrator/decomposition/tasks/*.yaml created with atomic task definitions
- [ ] .orchestrator/decomposition/validation.yaml shows status: passed
- [ ] .orchestrator/decomposition/graph.yaml created with dependency data
- [ ] All decomposition files committed to git
- [ ] Summary shows subgoals table, tasks table, validation results
- [ ] User knows to run `/ptf:plan` next
</success_criteria>
