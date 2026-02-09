---
description: Initialize project and run goal analysis
argument-hint: "<goal description>"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Initialize a PTF project by analyzing the user's goal, determining domain,
asking domain-driven clarifying questions, and generating analysis and constitution.

**Creates:**
- `.orchestrator/config.yaml` - Project configuration
- `.orchestrator/goal.md` - Original goal (immutable)
- `.orchestrator/decomposition/analysis.yaml` - Structured goal analysis
- `.orchestrator/decomposition/constitution.yaml` - Immutable principles

**After this command:** Run `/ptf:decompose` to break goal into tasks.
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
</execution_context>

<context>
Goal: $ARGUMENTS
</context>

<process>

## Phase 1: Setup

1. **Create .orchestrator directory:**
   ```bash
   mkdir -p .orchestrator/decomposition
   ```

2. **Check for existing project:**
   ```bash
   if [ -f .orchestrator/goal.md ]; then
     echo "ERROR: Project already initialized. Use /ptf:status to check progress."
     exit 1
   fi
   ```

3. **Initialize git repo if needed:**
   ```bash
   if [ ! -d .git ] && [ ! -f .git ]; then
     git init
     echo "Initialized git repository"
   fi
   ```

## Phase 2: Domain Detection

1. **Analyze goal text to determine domain:**

   Examine the goal text for domain indicators:
   - Software keywords: build, create, implement, API, database, frontend, backend, app, service, code, deploy
   - Research keywords: analyze, study, investigate, literature, hypothesis, experiment, data analysis, paper

2. **Load appropriate adapter:**
   ```bash
   # Determine domain from goal analysis
   DOMAIN="software-development"  # or "research" based on analysis

   # Check adapter exists
   if [ ! -f "adapters/${DOMAIN}.yaml" ]; then
     echo "WARNING: Adapter not found for ${DOMAIN}, using software-development"
     DOMAIN="software-development"
   fi
   ```

3. **If domain is uncertain, ask user:**

   Use AskUserQuestion:
   - header: "Domain Detection"
   - question: "What type of work is this?"
   - options:
     - "Software Development" - Building code, APIs, applications
     - "Research" - Analysis, experiments, literature review
     - "Custom" - I'll describe my domain

4. **Read the domain adapter:**
   ```bash
   cat "adapters/${DOMAIN}.yaml"
   ```

   Extract:
   - `questioning.init_questions` - Questions to ask
   - `constitution.template` - Template for constitution
   - `decomposition.max_recursion_depth` - For config

## Phase 3: Goal Clarification

Use AskUserQuestion with questions from the adapter's `questioning.init_questions`:

**For each question category from the adapter:**

1. Present the question with context-aware options derived from the goal
2. Capture the user's response
3. Record the decision for later use

Example for software-development adapter:

**Question 1 (core_value):**
Use AskUserQuestion:
- header: "Core Value"
- question: "What's the ONE thing that must work perfectly?"
- options: (derived from goal analysis)
  - "[Primary feature from goal]"
  - "[Integration point from goal]"
  - "[User flow from goal]"
  - "Other - let me specify"

**Question 2 (existing_code):**
Use AskUserQuestion:
- header: "Code Patterns"
- question: "Any existing code patterns to follow?"
- options:
  - "Greenfield - starting fresh"
  - "Follow existing patterns in codebase"
  - "Mixed - some existing, some new"

**Question 3 (integration_points):**
Use AskUserQuestion:
- header: "Integrations"
- question: "External services or APIs to integrate?"
- multiSelect: true
- options:
  - "Database"
  - "Auth provider"
  - "Third-party API"
  - "None"

**Question 4 (scale):**
Use AskUserQuestion:
- header: "Scale"
- question: "Expected complexity level?"
- options:
  - "Simple (1-5 files)"
  - "Medium (5-15 files)"
  - "Complex (15+ files)"

Track all responses for use in analysis and constitution.

## Phase 4: Goal Analysis

**Write order: goal.md -> config.yaml -> analysis.yaml (dependencies flow downward)**

1. **Write goal.md (immutable original):**

   Write `.orchestrator/goal.md`:
   ```markdown
   ---
   received: {timestamp}
   ---

   {original goal text from $ARGUMENTS}
   ```

2. **Write config.yaml:**

   Write `.orchestrator/config.yaml`:
   ```yaml
   # PTF Project Configuration
   created: {timestamp}
   domain: {detected domain}
   adapter: adapters/{domain}.yaml
   goal_file: .orchestrator/goal.md

   decomposition:
     max_recursion_depth: 5  # From adapter

   execution:
     max_parallel_tasks: 3
     ralph_mode: true
     max_ralph_iterations: 3

   checkpoints:
     wave_boundary: true
     verification: true
   ```

3. **Write analysis.yaml:**

   Write `.orchestrator/decomposition/analysis.yaml`:
   ```yaml
   step: 1-goal-analysis
   created: {timestamp}
   status: complete

   objective: |
     {refined objective synthesized from goal and clarification}

   scope:
     included:
       - {item 1 from clarification}
       - {item 2 from clarification}
     excluded:
       - {explicitly out of scope item 1}
       - {explicitly out of scope item 2}

   constraints:
     - {constraint 1 from clarification}
     - {constraint 2 from patterns/integrations chosen}

   success_criteria:
     - {testable criterion 1}
     - {testable criterion 2}
     - {testable criterion 3}

   domain: {detected domain}

   clarification_summary:
     core_value: {response}
     code_patterns: {response}
     integrations: {response}
     scale: {response}
   ```

## Phase 5: Constitution Generation

1. **Load adapter's constitution template:**

   Read the `constitution.template` field from the loaded adapter.

2. **Apply gathered constraints:**

   Replace template placeholders:
   - `{project_name}` - Derived from goal
   - `{domain_principles}` - From adapter template
   - `{project_constraints}` - From clarification responses
   - `{timestamp}` - Current timestamp

3. **Write constitution.yaml:**

   Write `.orchestrator/decomposition/constitution.yaml`:
   ```yaml
   # Project Constitution
   # These principles are IMMUTABLE for this project

   generated: {timestamp}
   domain: {domain}

   principles: |
     {expanded constitution from template with constraints applied}

   enforcement:
     - Every task must be validated against these principles
     - Violations indicate incorrectly defined tasks
     - Constitution is loaded into every decomposer context
   ```

## Phase 6: Commit

Stage and commit all initialization files:

```bash
git add .orchestrator/
git commit -m "$(cat <<'EOF'
ptf: initialize project

Goal: {one-liner summary}
Domain: {domain}
Ready for decomposition
EOF
)"
```

## Phase 7: Complete

Present summary to user:

```
---

## PTF Project Initialized

**Goal:** {one-liner}
**Domain:** {domain}

### Files Created

| File | Purpose |
|------|---------|
| `.orchestrator/goal.md` | Original goal (immutable) |
| `.orchestrator/config.yaml` | Project configuration |
| `.orchestrator/decomposition/analysis.yaml` | Goal analysis |
| `.orchestrator/decomposition/constitution.yaml` | Immutable principles |

### Analysis Summary

**Objective:** {refined objective}

**Scope:**
- Included: {count} items
- Excluded: {count} items

**Success Criteria:** {count} testable criteria

---

## Next Step

Run `/ptf:decompose` to break the goal into atomic tasks.

---
```

</process>

<success_criteria>
- [ ] .orchestrator/ directory created
- [ ] goal.md contains original goal with timestamp
- [ ] config.yaml has domain, adapter path, execution settings
- [ ] analysis.yaml has objective, scope, constraints, success_criteria
- [ ] constitution.yaml has domain-shaped principles with project constraints
- [ ] All files committed to git
- [ ] User knows to run `/ptf:decompose` next
</success_criteria>
