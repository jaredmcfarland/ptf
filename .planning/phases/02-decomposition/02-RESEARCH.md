# Phase 2: Decomposition - Research

**Researched:** 2026-01-18
**Domain:** Goal Analysis, Task Decomposition, Slash Commands, Domain Adapters, File-Based State Persistence
**Confidence:** HIGH

## Summary

Phase 2 implements the decomposition process that transforms user goals into validated atomic tasks through a 5-step process. Research confirms the patterns, structures, and integration points needed for `/ptf:init` and `/ptf:decompose` commands, the decomposer subagent, domain adapter integration, and state persistence to `.orchestrator/decomposition/`.

Key findings:
- Claude Code slash commands use markdown files with YAML frontmatter in `.claude/commands/` directories
- The 5-step decomposition process (Goal Analysis -> Subgoal Identification -> Recursive Decomposition -> Validation -> Dependency Graph) is well-defined in PARALLEL-TASK-FRAMEWORK.md
- Domain adapters provide questioning patterns, decomposition heuristics, atomicity criteria, and constitution templates
- State persistence uses YAML files in `.orchestrator/decomposition/` with specific schemas for each step
- Subagents are defined in `.claude/agents/` with role, tools, and execution flow specifications

**Primary recommendation:** Implement commands and subagent following GSD patterns exactly. Use the detailed decomposition process from PARALLEL-TASK-FRAMEWORK.md Section 3. Create domain adapters as YAML files following the interface specification. Persist state incrementally after each decomposition step.

## Standard Stack

The established libraries/tools for this domain:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Claude Code Commands | - | Slash command interface | Native plugin format, required for `/ptf:*` commands |
| Claude Code Agents | - | Subagent definitions | Native format for spawning specialized agents |
| YAML | 1.2 | State files, adapters | Already established in Phase 1, human-readable |
| JSON Schema | Draft 7 | Schema validation | Already established in Phase 1 |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Task tool | - | Spawning subagents | When decomposer needs to run specialized analysis |
| AskUserQuestion | - | Goal clarification | During init command questioning phase |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| YAML state files | JSON state files | YAML more readable for complex nested structures |
| Single command | Multiple commands | Split `/ptf:init` and `/ptf:decompose` keeps concerns separate, matches GSD pattern |

**Installation:**
Not applicable - Claude Code plugin components are file-based, not npm dependencies.

## Architecture Patterns

### Recommended Project Structure
```
.claude/
+-- commands/ptf/
|   +-- init.md                    # /ptf:init [goal] command
|   +-- decompose.md               # /ptf:decompose command
|
+-- agents/
|   +-- ptf-decomposer.md          # Decomposer subagent definition

adapters/
+-- software-development.yaml      # Software domain adapter
+-- research.yaml                  # Research domain adapter
+-- template.yaml                  # Base template for custom adapters

.orchestrator/                     # Created per-project at runtime
+-- config.yaml                    # Project configuration
+-- goal.md                        # Original goal (immutable)
+-- decomposition/
|   +-- analysis.yaml              # Step 1: Analyzed goal
|   +-- constitution.yaml          # Domain-shaped principles
|   +-- subgoals.yaml              # Step 2: Identified subgoals
|   +-- tasks/                     # Step 3: Atomic tasks
|   |   +-- {task-id}.yaml         # Individual task definitions
|   +-- validation.yaml            # Step 4: Validation results
|   +-- graph.yaml                 # Step 5: Dependencies + waves
```

### Pattern 1: Slash Command with Argument Processing

**What:** Claude Code command that accepts arguments and follows a multi-phase process.

**When to use:** `/ptf:init [goal]` command implementation.

**Example:**
```markdown
---
name: ptf:init
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
asking clarifying questions, and generating a constitution.

Creates:
- .orchestrator/config.yaml
- .orchestrator/goal.md
- .orchestrator/decomposition/analysis.yaml
- .orchestrator/decomposition/constitution.yaml
</objective>

<context>
Goal: $ARGUMENTS

@.claude/skills/ptf/SKILL.md
</context>

<process>
[Multi-step process with phase gates]
</process>
```

### Pattern 2: Subagent with Structured Return

**What:** Specialized agent that performs focused work and returns structured results.

**When to use:** Decomposer subagent for executing the 5-step decomposition.

**Example:**
```markdown
---
name: ptf-decomposer
description: Executes 5-step decomposition process from analyzed goal to validated tasks
tools: Read, Write, Bash, Glob, Grep
---

<role>
You are a PTF decomposer. You transform analyzed goals into atomic tasks through
the 5-step decomposition process.
</role>

<execution_flow>
<step name="load_analysis">
Read .orchestrator/decomposition/analysis.yaml
Load domain adapter from adapters/{domain}.yaml
</step>

<step name="identify_subgoals">
Apply domain-specific heuristics from adapter
Write .orchestrator/decomposition/subgoals.yaml
</step>
...
</execution_flow>

<structured_returns>
## DECOMPOSITION COMPLETE
**Tasks:** {N} atomic tasks in {M} subgoals
**Coverage:** 100% of goal addressed
...
</structured_returns>
```

### Pattern 3: Domain Adapter Interface

**What:** YAML structure that customizes decomposition for specific domains.

**When to use:** Software development and research adapters.

**Example:**
```yaml
# adapters/software-development.yaml
name: software-development
version: "1.0"

questioning:
  init_questions:
    - category: scope
      question: "What's the core feature that must work first?"
      why: "Identifies critical path for decomposition"
    - category: constraints
      question: "Any existing code/patterns to follow?"
      why: "Shapes atomicity criteria"
    - category: integration
      question: "External services or APIs to integrate?"
      why: "Identifies potential dependency boundaries"

decomposition:
  subgoal_heuristics:
    - name: by-layer
      description: Split by architectural layer
      examples: [data, repository, service, api, ui]
    - name: by-feature
      description: Split by user-facing feature
      examples: [auth, content, search]
    - name: by-file-boundary
      description: One task per file/module

  atomicity_criteria:
    - criterion: single-file
      check: "Task produces at most one file (or tightly-coupled set of 2-3)"
    - criterion: fresh-context-completable
      check: "Can complete in <30% context window with fresh start"
    - criterion: verifiable
      check: "Has concrete verification command or check"
    - criterion: no-hidden-dependencies
      check: "All inputs explicitly declared"

constitution:
  template: |
    ## {project_name} Constitution

    These principles are IMMUTABLE for this project:

    ### Core Principles
    1. Every task produces artifacts that can be verified independently
    2. No task modifies files it doesn't declare in outputs
    3. Dependencies are inferred from inputs/outputs, not assumed

    ### Domain Principles
    {domain_principles}

    ### Project-Specific Constraints
    {project_constraints}

artifacts:
  types:
    - name: source-code
    - name: test
    - name: config
    - name: migration
    - name: documentation
```

### Pattern 4: Incremental State Persistence

**What:** Write state files after each decomposition step for resume capability.

**When to use:** After every step in the 5-step process.

**Example:**
```yaml
# .orchestrator/decomposition/analysis.yaml (Step 1 output)
step: 1-goal-analysis
created: 2026-01-18T10:30:00Z
status: complete

objective: |
  Build user authentication with email/password login,
  session management, and secure logout

scope:
  included:
    - User registration with email/password
    - Login endpoint with session token
    - Logout with session invalidation
    - Session expiry after 24 hours
  excluded:
    - OAuth providers
    - Two-factor authentication
    - Password reset flow

constraints:
  - Must use existing database (Prisma)
  - Must follow existing API patterns
  - Session tokens must be httpOnly cookies

success_criteria:
  - User can register with email/password
  - User can log in and receive valid session
  - User can log out and invalidate session
  - Expired sessions are rejected

domain: software-development
```

### Pattern 5: Validation with Gap Analysis

**What:** Check decomposition completeness and identify issues.

**When to use:** Step 4 validation before proceeding to dependency analysis.

**Example:**
```yaml
# .orchestrator/decomposition/validation.yaml
step: 4-validation
created: 2026-01-18T10:45:00Z
status: passed  # or: failed

checks:
  coverage:
    status: passed
    details: "All 4 success criteria have corresponding tasks"

  overlap:
    status: passed
    details: "No duplicate task outputs detected"

  atomicity:
    status: passed
    details: "All 8 tasks meet atomicity criteria"

  input_coverage:
    status: passed
    details: "All task inputs have producers or are external"

  output_usefulness:
    status: passed
    warnings:
      - "auth-tests outputs not consumed (final deliverable)"

issues: []  # Empty if passed

# If failed:
# issues:
#   - type: gap
#     description: "No task covers session expiry verification"
#     severity: error
#   - type: too_large
#     task: auth-service
#     reason: "Touches 5 files, breaks atomicity"
#     severity: error
```

### Anti-Patterns to Avoid

- **Monolithic command:** Don't put all logic in one command file. Split `/ptf:init` (goal analysis) from `/ptf:decompose` (task breakdown) for clarity.
- **Skipping state persistence:** Always write state files after each step. Resume depends on this.
- **Hardcoded domain logic:** Put all domain-specific logic in adapters, not in commands or agents.
- **Implicit dependencies:** Never assume task ordering. Always derive from inputs/outputs.
- **Vague atomicity:** Use concrete criteria (single-file, fresh-context-completable), not subjective assessment.

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Goal clarification UI | Custom prompting | AskUserQuestion tool | Native Claude Code tool, handles multi-option flows |
| Subagent spawning | Manual context loading | Task tool | Proper fresh context, tool isolation |
| Command frontmatter | Custom parsing | YAML frontmatter standard | Claude Code parses this natively |
| Domain adapter loading | Custom YAML parser | Read + parse | YAML is well-supported, no special handling needed |
| Decomposition heuristics | Custom rules engine | Adapter YAML + LLM | LLM applies heuristics naturally from examples |

**Key insight:** The decomposition process itself is LLM-native. Don't try to build a rules engine. Provide clear heuristics and examples in the adapter, let Claude apply them. The 5-step structure provides the guardrails.

## Common Pitfalls

### Pitfall 1: Goal Analysis Too Broad

**What goes wrong:** Goal analysis produces vague scope that leads to unbounded decomposition.

**Why it happens:** Not enough clarifying questions, accepting ambiguous descriptions.

**How to avoid:**
- Use adapter-driven questioning to probe specific areas
- Require explicit scope boundaries (included AND excluded)
- Derive success criteria that are testable

**Warning signs:** Success criteria like "system works well" or scope like "handle authentication."

### Pitfall 2: Atomicity Drift During Recursion

**What goes wrong:** Tasks get progressively larger as decomposition proceeds.

**Why it happens:** No consistent atomicity check at each recursion level.

**How to avoid:**
- Apply atomicity criteria from adapter at EVERY recursion step
- When in doubt, break further
- Use concrete checks: "Can this complete in fresh context? Does it produce one file?"

**Warning signs:** Tasks touching 4+ files, descriptions longer than a paragraph.

### Pitfall 3: Missing Input/Output Declarations

**What goes wrong:** Dependency inference fails because tasks don't declare what they read/write.

**Why it happens:** Focus on "what to do" without "what to use."

**How to avoid:**
- Task template requires inputs and outputs
- Validate every task has at least one output
- Flag tasks with only required: false inputs

**Warning signs:** Tasks with empty inputs array, outputs that don't match description.

### Pitfall 4: Constitution Ignored

**What goes wrong:** Later tasks violate constitutional principles established during init.

**Why it happens:** Constitution generated but not loaded into context.

**How to avoid:**
- Include constitution.yaml in every decomposer and executor context
- Validate tasks against constitution before finalizing
- Surface constitutional violations in validation step

**Warning signs:** Tasks that modify undeclared files, break domain conventions.

### Pitfall 5: State File Corruption

**What goes wrong:** Partial writes leave decomposition state inconsistent.

**Why it happens:** Interruption during multi-file writes.

**How to avoid:**
- Write each step's output file atomically
- Include `status: in_progress` at start, update to `complete` at end
- Resume protocol checks status field

**Warning signs:** Missing status fields, partial YAML files.

## Code Examples

Verified patterns from framework documentation and GSD reference implementation:

### /ptf:init Command Structure
```markdown
---
name: ptf:init
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
   [ -f .orchestrator/goal.md ] && echo "Project exists" && exit 1
   ```

## Phase 2: Domain Detection

1. Analyze goal text to determine domain
2. Load appropriate adapter from `adapters/{domain}.yaml`
3. If uncertain, ask user to confirm domain

## Phase 3: Goal Clarification

Use AskUserQuestion with adapter's `questioning.init_questions`:

For each question category:
- Present question with options derived from goal analysis
- Follow up on ambiguous responses
- Capture decisions

## Phase 4: Goal Analysis

Write `.orchestrator/goal.md` (immutable):
```markdown
---
received: {timestamp}
---

{original goal text}
```

Write `.orchestrator/decomposition/analysis.yaml`:
```yaml
step: 1-goal-analysis
created: {timestamp}
status: complete

objective: {refined objective}
scope:
  included: [...]
  excluded: [...]
constraints: [...]
success_criteria: [...]
domain: {detected domain}
```

## Phase 5: Constitution Generation

Apply adapter's constitution template with gathered constraints.

Write `.orchestrator/decomposition/constitution.yaml`

## Phase 6: Commit

```bash
git add .orchestrator/
git commit -m "ptf: initialize project

Goal: {one-liner}
Domain: {domain}
Ready for decomposition"
```

## Phase 7: Complete

Present summary and next steps.

</process>

<success_criteria>
- [ ] .orchestrator/ directory created
- [ ] goal.md contains original goal
- [ ] analysis.yaml has objective, scope, constraints, success_criteria
- [ ] constitution.yaml has domain-shaped principles
- [ ] All files committed to git
- [ ] User knows to run `/ptf:decompose` next
</success_criteria>
```

### /ptf:decompose Command Structure
```markdown
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
Execute the 5-step decomposition process to transform the analyzed goal
into validated atomic tasks with dependency graph.

**Requires:** `.orchestrator/decomposition/analysis.yaml` (from /ptf:init)

**Creates:**
- `.orchestrator/decomposition/subgoals.yaml` - Step 2 output
- `.orchestrator/decomposition/tasks/*.yaml` - Step 3 output
- `.orchestrator/decomposition/validation.yaml` - Step 4 output
- `.orchestrator/decomposition/graph.yaml` - Step 5 output

**After this command:** Run `/ptf:plan` to generate human-readable plan.
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
</execution_context>

<context>
@.orchestrator/decomposition/analysis.yaml
@.orchestrator/decomposition/constitution.yaml
</context>

<process>

## Phase 1: Validate Prerequisites

```bash
[ ! -f .orchestrator/decomposition/analysis.yaml ] && \
  echo "Run /ptf:init first" && exit 1
```

## Phase 2: Spawn Decomposer

Spawn ptf-decomposer subagent with full context:

```
Task(prompt="
<context>
Analysis: @.orchestrator/decomposition/analysis.yaml
Constitution: @.orchestrator/decomposition/constitution.yaml
Adapter: @adapters/{domain}.yaml
</context>

Execute the 5-step decomposition process:
1. (Already done - analysis exists)
2. Identify subgoals using adapter heuristics
3. Recursively decompose to atomic tasks
4. Validate coverage, overlap, atomicity
5. (Dependency graph built in Phase 3)

Write state files after each step.
Return DECOMPOSITION COMPLETE or DECOMPOSITION BLOCKED.
", subagent_type="ptf-decomposer")
```

## Phase 3: Handle Results

**If DECOMPOSITION COMPLETE:**
- Verify all state files exist
- Commit decomposition state
- Present summary

**If DECOMPOSITION BLOCKED:**
- Present blocking issues
- Offer resolution options

## Phase 4: Commit

```bash
git add .orchestrator/decomposition/
git commit -m "ptf: complete decomposition

{N} tasks in {M} subgoals
100% coverage verified"
```

</process>
```

### Decomposer Subagent Definition
```markdown
---
name: ptf-decomposer
description: Executes 5-step decomposition process transforming goals into atomic tasks
tools: Read, Write, Bash, Glob, Grep
---

<role>
You are a PTF decomposer. You transform analyzed goals into validated atomic tasks
through the framework's 5-step decomposition process.

Your job: Take a goal analysis and produce atomic tasks that together achieve
the goal with 100% coverage and no overlap.
</role>

<philosophy>

## Fresh Context is King

Every task you create must be completable by an agent starting fresh.
If a task requires "remembering" something not in its inputs, it's broken.

## Atomicity is Non-Negotiable

Apply atomicity criteria at every recursion level:
- Single file output (or tightly coupled 2-3)
- Fresh context completable (<30% context)
- Verifiable with concrete check
- All inputs explicit

When in doubt, break it down further.

## 100% Rule from WBS

Subgoals must COMPLETELY cover the goal - nothing missed, nothing extra.
Tasks must COMPLETELY cover their subgoal - same rule, recursively.

</philosophy>

<execution_flow>

<step name="load_context" priority="first">
Read and parse:
- .orchestrator/decomposition/analysis.yaml
- .orchestrator/decomposition/constitution.yaml
- adapters/{domain}.yaml

Extract:
- Objective, scope, success_criteria
- Domain-specific decomposition heuristics
- Atomicity criteria
</step>

<step name="step2_subgoals">
**Subgoal Identification**

Apply domain heuristics to break goal into major components.

For software-development:
- Consider by-layer cuts (data, service, api, ui)
- Consider by-feature cuts (auth, content, search)
- Choose cut that minimizes cross-subgoal coupling

Write `.orchestrator/decomposition/subgoals.yaml`:
```yaml
step: 2-subgoal-identification
created: {timestamp}
status: complete

subgoals:
  - id: {short-id}
    name: {descriptive name}
    description: |
      {what this subgoal accomplishes}
    outputs:
      - {file/artifact this produces}
    depends_on_outputs_from:
      - {other subgoal IDs if any}
```

Validate 100% coverage before proceeding.
</step>

<step name="step3_decompose">
**Recursive Decomposition**

For each subgoal, evaluate atomicity:
- Does it meet ALL criteria from adapter?
- YES -> Create task YAML
- NO -> Break into sub-subgoals, recurse

Create task files in `.orchestrator/decomposition/tasks/`:
```yaml
# tasks/{task-id}.yaml
id: {task-id}
name: {human-readable name}
from_subgoal: {parent subgoal id}

description: |
  {complete instructions for execution}

inputs:
  - path: {file path}
    description: {what it provides}
    required: {true|false}

outputs:
  - path: {file path}
    type: {artifact type from adapter}

verify:
  - type: {exists|contains|runs|syntax|custom}
    target: {what to check}
    expected: {expected result}

context_notes: |
  {additional guidance}

context_budget:
  estimated_input_tokens: {number}
  max_context_percentage: 30
```

Continue until all leaves are atomic.
</step>

<step name="step4_validate">
**Decomposition Validation**

Run validation checks:

1. **Coverage**: Do tasks completely cover all success_criteria?
2. **Overlap**: Any duplicate outputs between tasks?
3. **Atomicity**: Do all tasks meet all criteria?
4. **Input Coverage**: Does every input have a producer?
5. **Output Usefulness**: Is every output consumed or a final deliverable?

Write `.orchestrator/decomposition/validation.yaml`:
```yaml
step: 4-validation
created: {timestamp}
status: {passed|failed}

checks:
  coverage: {status, details}
  overlap: {status, details}
  atomicity: {status, details}
  input_coverage: {status, details}
  output_usefulness: {status, warnings}

issues: [...]  # If any
```

**If failed:** Return DECOMPOSITION BLOCKED with issues.
**If passed:** Continue to step 5.
</step>

<step name="step5_graph">
**Dependency Graph Construction**

Note: Full graph + wave computation is Phase 3 (Dependency Analysis).
Here we capture the data flow evident from task inputs/outputs.

Write `.orchestrator/decomposition/graph.yaml`:
```yaml
step: 5-dependency-graph
created: {timestamp}
status: complete

task_count: {N}
subgoal_count: {M}

# Dependencies inferred from inputs/outputs
dependencies:
  - from: {task-id}
    to: {task-id}
    type: artifact
    confidence: high
    reason: "{to} needs {artifact} produced by {from}"

# Note: Full inference (semantic, heuristic) done in Phase 3
```
</step>

</execution_flow>

<structured_returns>

## DECOMPOSITION COMPLETE

**Tasks:** {N} atomic tasks in {M} subgoals
**Coverage:** 100% of goal success criteria addressed
**Validation:** All checks passed

### Subgoal Structure

| Subgoal | Tasks | Outputs |
|---------|-------|---------|
| {name} | {count} | {key outputs} |

### Files Created

- .orchestrator/decomposition/subgoals.yaml
- .orchestrator/decomposition/tasks/*.yaml ({N} files)
- .orchestrator/decomposition/validation.yaml
- .orchestrator/decomposition/graph.yaml

### Ready for Phase 3

Run `/ptf:plan` to compute full dependency graph and wave assignments.

---

## DECOMPOSITION BLOCKED

**Blocked by:** {issue category}

### Issues Found

| Issue | Severity | Resolution |
|-------|----------|------------|
| {description} | {error/warning} | {how to fix} |

### Awaiting

{What's needed to continue - user decision, clarification, etc.}

</structured_returns>
```

### Domain Adapter: Software Development
```yaml
# adapters/software-development.yaml
name: software-development
description: |
  Domain adapter for software development tasks including
  web applications, APIs, databases, and infrastructure.
version: "1.0"

questioning:
  init_questions:
    - category: core_value
      question: "What's the ONE thing that must work perfectly?"
      why: "Identifies critical path, shapes verification priorities"
      options_template: ["The {main_feature}", "The {integration}", "The {user_flow}"]

    - category: existing_code
      question: "Any existing code patterns to follow?"
      why: "Constrains implementation approach"
      options_template: ["Follow existing {pattern}", "Greenfield", "Mixed"]

    - category: integration_points
      question: "External services or APIs to integrate?"
      why: "Identifies potential task boundaries"
      options_template: ["Database", "Auth provider", "Third-party API", "None"]

    - category: scale
      question: "Expected complexity level?"
      why: "Calibrates decomposition depth"
      options_template: ["Simple (1-5 files)", "Medium (5-15 files)", "Complex (15+ files)"]

decomposition:
  subgoal_heuristics:
    - name: by-layer
      description: Split by architectural layer
      examples:
        - "Data layer (schemas, migrations)"
        - "Repository layer (data access)"
        - "Service layer (business logic)"
        - "API layer (endpoints, controllers)"
        - "UI layer (components, pages)"
      when_to_use: "Feature spans multiple layers"

    - name: by-feature
      description: Split by user-facing feature
      examples:
        - "User authentication"
        - "Content management"
        - "Search functionality"
      when_to_use: "Multiple independent features"

    - name: by-file-boundary
      description: One task per file/module
      examples:
        - "Create userRepository.ts"
        - "Create authService.ts"
      when_to_use: "Clear module boundaries"

    - name: by-interface
      description: Cut at API/interface boundaries
      examples:
        - "Internal API vs external API"
        - "Service interfaces vs implementations"
      when_to_use: "Multiple integration points"

  atomicity_criteria:
    - criterion: single-file
      check: "Task produces at most one file (or tightly-coupled set of 2-3)"
      fail_signal: "outputs array has >3 files"

    - criterion: fresh-context-completable
      check: "Can complete in <30% context window starting fresh"
      fail_signal: "inputs require reading >50KB of context"

    - criterion: verifiable
      check: "Has concrete verification command or check"
      fail_signal: "verify array is empty or vague"

    - criterion: focused
      check: "Task touches one concept/concern"
      fail_signal: "description mentions multiple unrelated concepts"

    - criterion: explicit-inputs
      check: "All file dependencies declared in inputs"
      fail_signal: "description references files not in inputs"

  max_recursion_depth: 5

constitution:
  template: |
    # {project_name} Constitution

    These principles are IMMUTABLE for this project.
    Violating them means the task is incorrectly defined.

    ## Core Principles

    1. **Artifact Integrity**: Every task produces exactly the artifacts declared
       in its outputs. No undeclared file modifications.

    2. **Dependency Honesty**: All file dependencies are declared in inputs.
       Hidden dependencies break parallel execution.

    3. **Fresh Context**: Every task must be completable by an agent starting
       with empty context, loading only declared inputs.

    4. **Verification**: Every artifact has a concrete verification step.
       "Looks good" is not verification.

    ## Domain Principles (Software Development)

    5. **Type Safety**: Generated code must pass type checking.
       Include type check in verification.

    6. **Test Coverage**: Logic-containing code has corresponding tests.
       Tests are separate tasks (code -> test dependency).

    7. **API Contracts**: Endpoints follow existing patterns if present.
       New patterns require explicit documentation.

    ## Project-Specific Constraints

    {project_constraints}

    ---
    *Generated: {timestamp}*
    *Domain: software-development*

artifacts:
  types:
    - name: source-code
      extensions: [.ts, .js, .py, .go, .rs, .java]
      description: Application source code

    - name: migration
      extensions: [.sql, .prisma]
      description: Database schema changes

    - name: config
      extensions: [.yaml, .yml, .json, .toml, .env.example]
      description: Configuration files

    - name: test
      extensions: [.test.ts, .spec.ts, _test.go, _test.py]
      description: Test files

    - name: documentation
      extensions: [.md, .rst]
      description: Documentation files

  verification_strategies:
    source-code:
      - method: exists
        description: File exists at expected path
      - method: syntax
        description: File parses without syntax errors
      - method: runs
        command_template: "npx tsc --noEmit {path}"
        description: Type check passes

    migration:
      - method: exists
        description: Migration file exists
      - method: syntax
        description: Valid SQL/Prisma syntax
      - method: runs
        command_template: "npx prisma validate"
        description: Schema validates

    test:
      - method: exists
        description: Test file exists
      - method: runs
        command_template: "npm test -- {path}"
        description: Tests pass

    config:
      - method: exists
        description: Config file exists
      - method: syntax
        description: Valid YAML/JSON/TOML

dependencies:
  common_patterns:
    - name: schema-to-repository
      from_type: migration
      to_type: source-code
      description: Repositories depend on schemas they access
      confidence: high

    - name: repository-to-service
      from_type: source-code
      to_type: source-code
      description: Services depend on repositories they use
      confidence: high

    - name: service-to-api
      from_type: source-code
      to_type: source-code
      description: API endpoints depend on services they call
      confidence: high

    - name: code-to-test
      from_type: source-code
      to_type: test
      description: Tests depend on code they test
      confidence: high

  inference_hints:
    - pattern: "import.*from ['\"]\\./(.+)['\"]"
      implies: depends on imported module
    - pattern: "prisma\\.(.+)\\."
      implies: depends on prisma schema
```

### State File: subgoals.yaml
```yaml
# .orchestrator/decomposition/subgoals.yaml
step: 2-subgoal-identification
created: 2026-01-18T10:35:00Z
status: complete

goal_reference: .orchestrator/decomposition/analysis.yaml

subgoals:
  - id: auth-data
    name: Authentication Data Layer
    description: |
      Create database schema and data access for users and sessions.
      Foundation that all other auth components depend on.
    outputs:
      - prisma/schema.prisma (auth models)
      - src/repositories/user.ts
      - src/repositories/session.ts
    depends_on_outputs_from: []
    coverage_mapping:
      - "User can register" (partial - data storage)
      - "Sessions expire" (partial - data model)

  - id: auth-service
    name: Authentication Service Layer
    description: |
      Business logic for authentication: registration, login, logout,
      session management, password hashing.
    outputs:
      - src/services/auth.ts
    depends_on_outputs_from:
      - auth-data
    coverage_mapping:
      - "User can register" (logic)
      - "User can login" (logic)
      - "User can logout" (logic)
      - "Sessions expire" (enforcement)

  - id: auth-api
    name: Authentication API Layer
    description: |
      HTTP endpoints exposing authentication functionality.
      Input validation, response formatting, error handling.
    outputs:
      - src/routes/auth.ts
    depends_on_outputs_from:
      - auth-service
    coverage_mapping:
      - "User can register" (endpoint)
      - "User can login" (endpoint)
      - "User can logout" (endpoint)

  - id: auth-tests
    name: Authentication Tests
    description: |
      Integration tests for complete authentication flows.
      Exercises all success criteria end-to-end.
    outputs:
      - tests/auth.test.ts
    depends_on_outputs_from:
      - auth-api
    coverage_mapping:
      - All success criteria (verification)

validation:
  coverage: "All 4 success criteria mapped to subgoals"
  overlap: "None - clear layer boundaries"
  coupling: "Linear dependency chain (data -> service -> api -> tests)"
```

### State File: Individual Task
```yaml
# .orchestrator/decomposition/tasks/user-repository.yaml
id: user-repository
name: Implement User Repository
from_subgoal: auth-data

description: |
  Create repository layer for User model with data access operations.

  Required operations:
  - findByEmail(email: string): Promise<User | null>
  - findById(id: string): Promise<User | null>
  - create(data: { email: string, passwordHash: string }): Promise<User>

  Use Prisma client for database access. Follow existing repository
  patterns if present. Include proper error handling for database
  failures.

inputs:
  - path: prisma/schema.prisma
    description: User model definition
    required: true
  - path: src/lib/prisma.ts
    description: Prisma client instance
    required: true
  - path: src/repositories/
    description: Existing repository patterns (if any)
    required: false

outputs:
  - path: src/repositories/user.ts
    type: source-code

verify:
  - type: exists
    target: src/repositories/user.ts
  - type: contains
    target: src/repositories/user.ts
    expected: "findByEmail"
  - type: contains
    target: src/repositories/user.ts
    expected: "findById"
  - type: contains
    target: src/repositories/user.ts
    expected: "create"
  - type: runs
    target: "npx tsc --noEmit src/repositories/user.ts"
    expected: 0

context_notes: |
  Check for existing repository patterns in src/repositories/.
  If patterns exist, follow them exactly.
  If greenfield, use simple async/await pattern with Prisma.

  Do NOT include password hashing - that's auth-service responsibility.
  Repository is pure data access.

context_budget:
  estimated_input_tokens: 3000
  max_context_percentage: 25

on_failure:
  strategy: retry
  max_attempts: 2
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual task breakdown | LLM-assisted with heuristics | 2024+ | Consistent atomicity, better coverage |
| Single prompt decomposition | Multi-step with state persistence | 2024+ | Resume capability, incremental validation |
| Domain-agnostic rules | Domain adapters with specialized heuristics | 2024+ | Better decomposition quality per domain |
| Implicit dependencies | Explicit input/output declarations | 2024+ | Automatic dependency inference possible |

**Deprecated/outdated:**
- Single-shot decomposition: Multi-step with validation is more reliable
- Manual dependency declaration: Input/output inference is more accurate

## Open Questions

Things that couldn't be fully resolved:

1. **Optimal recursion depth per domain**
   - What we know: PARALLEL-TASK-FRAMEWORK.md suggests max 5 levels
   - What's unclear: Whether 5 is optimal or domain-dependent
   - Recommendation: Start with 5, adjust per adapter based on experience

2. **Constitution enforcement mechanism**
   - What we know: Constitution should constrain tasks
   - What's unclear: How to automatically validate tasks against constitution
   - Recommendation: Include constitution in decomposer context, rely on LLM for enforcement, add explicit checks for critical principles

3. **Partial decomposition resume**
   - What we know: State files enable resume
   - What's unclear: Exact protocol for resuming mid-step
   - Recommendation: Use status field (in_progress/complete), resume from last complete step

## Sources

### Primary (HIGH confidence)
- `/Users/jaredmcfarland/Developer/ptf/PARALLEL-TASK-FRAMEWORK.md` - Sections 3 (Decomposition), 9 (Domain Adapters)
- `/Users/jaredmcfarland/Developer/ptf/.planning/phases/01-foundation/01-RESEARCH.md` - Foundation patterns
- `/Users/jaredmcfarland/Developer/ptf/.claude/commands/gsd/new-project.md` - Command structure reference
- `/Users/jaredmcfarland/Developer/ptf/.claude/agents/gsd-planner.md` - Subagent structure reference
- `/Users/jaredmcfarland/Developer/ptf/schemas/task.schema.yaml` - Task schema from Phase 1

### Secondary (MEDIUM confidence)
- `/Users/jaredmcfarland/Developer/ptf/.claude/skills/ptf/SKILL.md` - Command reference, file locations
- `/Users/jaredmcfarland/Developer/ptf/.planning/REQUIREMENTS.md` - DECOMP-* requirements

### Tertiary (LOW confidence)
- Domain adapter examples in framework document (detailed but theoretical)

## Metadata

**Confidence breakdown:**
- Command structure: HIGH - Clear patterns from GSD implementation
- 5-step process: HIGH - Detailed in framework document
- Domain adapter interface: HIGH - Explicit specification in framework
- State persistence: HIGH - Documented structure in framework
- Subagent patterns: HIGH - Clear examples in GSD agents

**Research date:** 2026-01-18
**Valid until:** 2026-03-18 (60 days - patterns are stable)

---
*Phase: 02-decomposition*
*Research completed: 2026-01-18*
