# Parallel Task Framework

## A Generalized Framework for Task Decomposition and Parallel Work Orchestration

**Version:** 0.1.0-draft  
**Status:** Founding Document  
**Created:** January 17, 2025

---

## Executive Summary

The Parallel Task Framework is a work-agnostic system for decomposing complex goals into atomic tasks, analyzing dependencies, computing optimal parallel execution sequences, and orchestrating execution across multiple agents. It provides the conceptual foundation, data schemas, prompt templates, and orchestration patterns that enable any domain-specific agentic system to reliably break down and execute complex work.

This framework abstracts the patterns observed in successful spec-driven development systems (such as GSD, SpecKit, and others) into reusable primitives that can be applied to software development, research, creative writing, music composition, or any domain where complex work can be decomposed into parallel-executable units.

The framework treats LLM agents as a new class of computational resource with specific constraints (context window limits, probabilistic output, quality degradation over time) and capabilities (natural language understanding, emergent reasoning, flexible problem-solving). It applies principles from parallel computing, cognitive science, systems theory, and project management to create a rigorous yet practical approach to agent orchestration.

**V1 Implementation:** The framework is implemented as a **Claude Code plugin** providing slash commands, specialized subagents, an agent skill, and hooks. Claude Code serves as both the orchestrator and the executing agent, with the plugin providing the structure and patterns for reliable parallel execution.

---

## Table of Contents

1. [Philosophy and Principles](#1-philosophy-and-principles)
2. [Core Concepts](#2-core-concepts)
3. [The Decomposition Process](#3-the-decomposition-process)
4. [Dependency Analysis](#4-dependency-analysis)
5. [Wave Computation and Parallel Execution](#5-wave-computation-and-parallel-execution)
6. [Ralph-Style Execution](#6-ralph-style-execution)
7. [State Persistence Model](#7-state-persistence-model)
8. [Failure Handling and Recovery](#8-failure-handling-and-recovery)
9. [Domain Adapters](#9-domain-adapters)
10. [File Formats and Schemas](#10-file-formats-and-schemas)
11. [Framework Architecture](#11-framework-architecture)
12. [Implementation Roadmap](#12-implementation-roadmap)

---

## 1. Philosophy and Principles

### 1.1 The Problem This Framework Solves

Large Language Model agents are powerful but suffer from a fundamental constraint: **context degradation**. As an agent fills its context window, quality degrades predictably:

| Context Usage | Quality Level | Symptoms |
|--------------|---------------|----------|
| 0-30% | Peak quality | Full attention, complete reasoning |
| 30-50% | Good quality | Reliable output, minor oversights |
| 50-70% | Degrading quality | Increasing shortcuts, missed details |
| 70%+ | Poor quality | Significant errors, incomplete work |

This degradation manifests as:
- "Due to context limits, I'll be more concise now" (cutting corners)
- Drift from original requirements
- Inconsistent implementation across a project
- Half-completed features
- Loss of coherence in long-running tasks

The Parallel Task Framework solves this through:

1. **Aggressive decomposition** — Breaking work into context-appropriate atomic units
2. **Fresh context execution** — Each task runs in a fresh agent context
3. **Structured state persistence** — Progress survives context boundaries
4. **Parallel orchestration** — Independent tasks execute simultaneously

### 1.2 Core Principles

#### Principle 1: Everything is a File

All artifacts, state, configuration, and communication happen through files. This provides:
- Universal compatibility across tools and environments
- Natural persistence and version control
- Human-readable inspection and debugging
- Simple concurrency model (file-level locking/partitioning)

The framework produces files, consumes files, and tracks files. There are no in-memory-only constructs that would be lost on interruption.

#### Principle 2: Context is a Scarce Resource

Context window capacity is treated as a first-class resource to be budgeted and allocated, not an infinite pool to be filled carelessly. The framework:
- Sizes tasks to fit comfortably in fresh context
- Loads only relevant context for each task
- Never accumulates garbage across task boundaries
- Measures and respects context budgets

#### Principle 3: Atomicity Enables Parallelism

Tasks must be truly independent to parallelize. The framework enforces:
- Clear input/output declarations for every task
- Explicit dependency relationships
- No hidden shared state between parallel tasks
- Verification that confirms task isolation

#### Principle 4: Dependencies are Data Flow

A dependency is not merely "do A before B" — it is "B needs something A produces." The framework models what flows between tasks, enabling:
- Automatic dependency inference from artifact declarations
- Clear understanding of what each task needs to start
- Precise identification of what blocks progress
- Potential for partial execution when some dependencies are met

#### Principle 5: Verification Closes the Loop

Without verification, parallel execution is blind. Every task must have:
- Observable completion criteria
- Automatable verification steps (where possible)
- Clear pass/fail determination
- Recorded verification results

#### Principle 6: State Enables Resume

Work can stop at any point (agent failure, human interruption, context exhaustion) and resume later. The framework maintains:
- Durable state that survives any interruption
- Clear checkpoint semantics
- Resume protocols for any state
- Complete history for debugging and audit

#### Principle 7: Domain-Agnostic Core, Domain-Specific Adapters

The core framework provides universal primitives for decomposition, dependency analysis, and orchestration. Domain-specific knowledge (how to decompose software vs. research vs. creative writing) lives in pluggable adapters that customize behavior without modifying the core.

### 1.3 Theoretical Foundations

The framework draws from established fields:

**From Cognitive Science: Chunking and Working Memory**

George Miller's research on cognitive limits (the "7±2" principle) establishes that humans process information in chunks, with limits on simultaneous chunk handling. LLM agents exhibit analogous behavior — they work best when tasks are "single chunks" that fit entirely in working attention.

Hierarchical Task Analysis (HTA) from cognitive psychology provides a model for recursive decomposition: goals break into subgoals, which break into operations (atomic executable units). The framework adopts this hierarchy.

**From Systems Theory: Near-Decomposability**

Herbert Simon's concept of "near-decomposability" states that in well-structured systems:
- Interactions *within* components are strong
- Interactions *between* components are weak

This suggests a decomposition heuristic: find natural seams where coupling is lowest. Tightly coupled work stays together; loosely coupled work separates into parallel-executable units.

**From Software Architecture: Bounded Contexts**

Domain-Driven Design's "bounded context" pattern applies directly. Each task should have clear boundaries around what it knows about and operates on. A task that requires understanding the entire system is poorly decomposed; a task with narrow, well-defined scope is properly decomposed.

**From Project Management: Work Breakdown Structures**

Work Breakdown Structure (WBS) methodology provides practical heuristics:
- **100% Rule**: Children of any parent must completely account for the parent
- **No Overlap**: Each piece of work appears exactly once
- **Outcome-Oriented**: Tasks are defined by deliverables, not activities
- **Appropriate Granularity**: Neither too coarse nor too fine

**From Parallel Computing: Dataflow and DAGs**

The framework's execution model is fundamentally a Directed Acyclic Graph (DAG) where:
- Nodes represent tasks
- Edges represent dependencies (data flow)
- Topological sorting produces execution waves

This is the same model used in build systems (Make, Bazel), workflow engines (Airflow, Prefect), and dataflow programming. The novelty is adapting it for LLM agent constraints.

---

## 2. Core Concepts

### 2.1 Task

A **Task** is the atomic unit of work in the framework. It represents a single operation that:
- Can be fully understood from its description
- Can be executed in a single fresh agent context
- Produces one or more coherent file artifacts
- Has clear, verifiable completion criteria
- Cannot be meaningfully subdivided further

```yaml
Task:
  id: string                    # Unique identifier (e.g., "auth-schema")
  name: string                  # Human-readable name
  description: string           # Complete instructions for execution
  
  inputs:                       # What this task needs to read
    - path: string              # File path
      description: string       # What it's used for
      required: boolean         # Can task proceed without it?
  
  outputs:                      # What this task produces
    - path: string              # File path
      type: string              # File type for routing to handlers
  
  verify:                       # How to confirm completion
    - type: string              # exists | contains | runs | custom
      target: string            # What to check
      expected: any             # Expected result
  
  context_notes: string         # Additional guidance for executing agent
  
  on_failure:                   # Failure handling policy
    strategy: string            # retry | skip | escalate
    max_attempts: number
```

### 2.2 Artifact

An **Artifact** is a file produced or consumed by tasks. The framework tracks artifacts to:
- Infer dependencies (Task B needs file that Task A produces)
- Verify completion (expected output exists)
- Enable resume (know what's already done)
- Provide audit trail (what was produced, when, by whom)

```yaml
Artifact:
  path: string                  # Canonical file path
  type: string                  # File type (for verification routing)
  produced_by: TaskId           # Which task created this
  produced_at: timestamp        # When it was created
  consumed_by: TaskId[]         # Which tasks read this
  checksum: string              # Content hash for integrity
  verified: boolean             # Has it passed verification?
```

Since everything is a file, artifacts have a beautifully simple model. Complex objects are serialized to files. Communication between tasks happens through files. The artifact manifest tracks the complete state of produced work.

### 2.3 Dependency

A **Dependency** represents a relationship where one task must complete before another can start.

```yaml
Dependency:
  from: TaskId                  # The prerequisite task
  to: TaskId                    # The dependent task
  type: string                  # artifact | semantic | resource | implicit
  confidence: string            # high | medium | low
  reason: string                # Explanation for human review
```

**Dependency Types:**

| Type | Description | Inference Method |
|------|-------------|------------------|
| `artifact` | Task B needs a file Task A produces | Match outputs to inputs |
| `semantic` | Task B's description references Task A's work | NLP analysis of descriptions |
| `resource` | Tasks A and B both modify the same file | Detect overlapping outputs |
| `implicit` | Logical ordering (tests need code to exist) | Domain adapter heuristics |

### 2.4 Wave

A **Wave** is a set of tasks that can execute in parallel because they have no dependencies on each other. Waves are computed by topological sorting of the dependency graph.

```yaml
Wave:
  number: integer               # Wave sequence (1, 2, 3, ...)
  tasks: TaskId[]               # Tasks in this wave
  status: string                # pending | running | completed | partial | failed
  depends_on_waves: integer[]   # Which waves must complete first
```

**Wave Properties:**
- All tasks in a wave can start simultaneously
- A wave cannot start until all previous waves complete
- Within a wave, tasks are independent (no data flow between them)
- Wave boundaries are natural checkpoints for state persistence

### 2.5 Plan

A **Plan** is a complete specification for executing a goal, containing:
- The analyzed goal (what we're trying to accomplish)
- All decomposed tasks
- The dependency graph
- Wave assignments
- Execution policies

```yaml
Plan:
  id: string                    # Unique plan identifier
  goal: string                  # Original user goal
  created: timestamp            # When plan was created
  
  analysis:                     # Structured goal analysis
    objective: string
    scope: { included: [], excluded: [] }
    constraints: []
    success_criteria: []
    domain: string
  
  tasks: Task[]                 # All tasks in the plan
  dependencies: Dependency[]    # All dependency relationships
  waves: Wave[]                 # Computed parallel execution groups
  
  execution_policy:             # How to run this plan
    max_parallel: number
    failure_strategy: string
    checkpoint_frequency: string
```

### 2.6 Orchestrator

The **Orchestrator** is the execution engine that:
- Loads plans and tracks state
- Dispatches tasks to agents
- Monitors completion and verification
- Handles failures according to policy
- Maintains persistent state
- Enables resume from any point

The orchestrator is conceptually a state machine that processes the plan wave by wave, managing the lifecycle of each task.

### 2.7 Domain Adapter

A **Domain Adapter** provides domain-specific knowledge for:
- How to identify subgoals in this domain
- What constitutes an atomic task
- What artifact types exist
- How to verify different artifact types
- Common dependency patterns
- Shared context that should load for all tasks

```yaml
DomainAdapter:
  name: string                  # e.g., "software-development"
  
  decomposition:
    subgoal_heuristics: []      # How to break down goals
    atomicity_criteria: []      # What makes a task atomic
  
  artifacts:
    types: []                   # File types this domain produces
    verification_strategies: {} # How to verify each type
  
  dependencies:
    common_patterns: []         # Typical dependency shapes
    inference_hints: []         # Domain-specific inference rules
  
  context:
    shared_files: []            # Files to load for all tasks
    conventions: string         # Domain conventions document
```

---

## 3. The Decomposition Process

Decomposition transforms a high-level goal into a set of atomic, executable tasks. The framework uses a multi-step process that separates concerns and allows for iteration at each stage.

### 3.1 Overview

```
User Goal
    │
    ▼
┌─────────────────────────┐
│  Step 1: Goal Analysis  │
│  Extract structure      │
└───────────┬─────────────┘
            │ Analyzed Goal
            ▼
┌─────────────────────────┐
│  Step 2: Subgoal        │
│  Identification         │
└───────────┬─────────────┘
            │ Subgoal List
            ▼
┌─────────────────────────┐
│  Step 3: Recursive      │◄────┐
│  Decomposition          │     │ Not yet atomic
└───────────┬─────────────┘     │
            │                   │
            ├───────────────────┘
            │ Atomic Tasks
            ▼
┌─────────────────────────┐
│  Step 4: Validation     │
│  Check completeness     │
└───────────┬─────────────┘
            │ Validated Tasks
            ▼
┌─────────────────────────┐
│  Step 5: Dependency     │
│  Graph Construction     │
└───────────┬─────────────┘
            │
            ▼
    Tasks + Dependencies + Waves
    (Ready for Execution)
```

### 3.2 Step 1: Goal Analysis

**Purpose:** Transform raw user input into structured goal specification.

**Input:** Raw goal from user (natural language, potentially ambiguous)

**Output:** Structured goal with clear scope, constraints, and success criteria

**Process:**

The goal analysis prompt extracts:

1. **Core Objective** — What is actually being asked for? Strip away ambiguity and identify the essential deliverable.

2. **Scope Boundaries** — What's explicitly included? What's explicitly excluded? Undefined scope leads to drift.

3. **Constraints** — Technical limitations, preferences, requirements that shape implementation.

4. **Success Criteria** — Measurable conditions that define "done." These become the basis for final verification.

5. **Domain Classification** — What type of work is this? This selects the appropriate domain adapter.

**Output Schema:**

```yaml
# .orchestrator/decomposition/analysis.yaml

created: timestamp
step: 1-goal-analysis

objective: |
  Clear statement of what needs to be accomplished
  
scope:
  included:
    - Explicit inclusions
  excluded:
    - Explicit exclusions
    
constraints:
  - Technical or process constraints
  
success_criteria:
  - Measurable completion criterion
  
domain: domain-identifier
```

**Quality Criteria:**

- Objective is unambiguous (one interpretation)
- Scope has no gaps (everything relevant is classified)
- Constraints are actionable (can be checked against)
- Success criteria are verifiable (can determine pass/fail)

### 3.3 Step 2: Subgoal Identification

**Purpose:** Break the analyzed goal into major components (first-level decomposition).

**Input:** Analyzed goal + domain adapter

**Output:** List of subgoals that together accomplish the goal

**Process:**

Using domain-specific heuristics, identify natural divisions in the work. The domain adapter provides guidance on how to identify subgoals in this particular domain.

**Decomposition Heuristics by Domain:**

| Domain | Cut Points |
|--------|------------|
| Software | By layer, by feature, by file boundary, by API boundary |
| Research | By question, by source type, by analysis stage |
| Writing | By section, by narrative arc, by character |
| Music | By track, by section, by voice, by parameter |

**Output Schema:**

```yaml
# .orchestrator/decomposition/subgoals.yaml

created: timestamp
step: 2-subgoal-identification

subgoals:
  - id: short-identifier
    name: Descriptive name
    description: |
      What this subgoal accomplishes
    outputs:
      - What this produces (files/artifacts)
    depends_on_outputs_from:
      - Other subgoal IDs if any
```

**Critical Requirement:** 

At this stage, we capture `depends_on_outputs_from` — early identification of likely dependencies. This seeds the dependency inference process and catches obvious relationships before they're obscured by further decomposition.

**Validation at this Stage:**

- **100% Rule**: Subgoals completely cover the goal (no gaps)
- **No Overlap**: Each piece of work appears once
- **Natural Boundaries**: Cuts occur at low-coupling points

### 3.4 Step 3: Recursive Decomposition

**Purpose:** Recursively break subgoals into atomic tasks (operations).

**Input:** A single subgoal + domain adapter

**Output:** Either a Task (if atomic) or more subgoals (if further breakdown needed)

**Process:**

For each subgoal, evaluate whether it meets the **Operation Criteria**:

#### Operation Criteria (Is This Task Atomic?)

A subgoal is an "operation" (atomic task) when it meets ALL of these criteria:

1. **Single Session Completion**
   - Could a focused agent complete this in one sitting without losing track?
   - If the task requires "coming back to it," it's too big.

2. **Fits in Working Memory**
   - Can the task be held entirely in attention at once?
   - Description, context, execution approach, verification — all simultaneously.
   - If constant reference back to notes is needed, it's too big.

3. **One Coherent Artifact**
   - Does this produce one thing? A file, a component, a module.
   - Multiple unrelated outputs suggest multiple tasks.

4. **Clear Done State**
   - Can completion be stated in one sentence?
   - Complex, multi-part criteria suggest the task is too big.

5. **Minimal Context Loading**
   - Does this require understanding the entire system, or just a small part?
   - Tasks needing global knowledge are often too big.
   - Well-decomposed tasks have narrow, local scope.

**Decision Tree:**

```
Is this subgoal an operation?
    │
    ├─ YES → Create Task definition
    │         - Specify inputs (files needed)
    │         - Specify outputs (files produced)
    │         - Write verification steps
    │         - Document context notes
    │
    └─ NO → Further decompose
              - Identify sub-subgoals
              - Apply same criteria recursively
              - Continue until all leaves are operations
```

**Domain Adapter Influence:**

The domain adapter can provide domain-specific answers to "is this atomic?" For example:
- Software: "Does it touch more than 3 tightly-coupled files?"
- Research: "Does it require synthesizing more than one major source?"
- Writing: "Does it span multiple scenes or perspectives?"

**Output Schema (for atomic task):**

```yaml
task:
  id: identifier
  name: Descriptive name
  description: |
    Clear instructions for execution
  inputs:
    - path: /path/to/input/file
      description: What it's used for
  outputs:
    - path: /path/to/output/file
      type: file-type
  verify:
    - type: exists
      target: /path/to/output/file
    - type: runs
      target: verification-command
  context_notes: |
    Additional guidance for executing agent
```

**Recursion Termination:**

The process terminates when all leaf nodes are operations. Unbounded recursion is prevented by:
- Domain adapter providing floor on granularity
- Explicit detection of "cannot meaningfully subdivide"
- Maximum recursion depth (safety limit)

### 3.5 Step 4: Decomposition Validation

**Purpose:** Verify the decomposition is complete, correct, and ready for execution.

**Input:** All tasks from recursive decomposition

**Output:** Validation result with any issues flagged

**Validation Checks:**

1. **Coverage**: Do tasks completely accomplish the goal?
   - Compare task outputs against goal requirements
   - Flag any work needed but not covered by a task

2. **Overlap**: Is any work duplicated?
   - Check for multiple tasks producing same artifact
   - Check for redundant operations

3. **Atomicity**: Is each task truly an operation?
   - Re-verify against operation criteria
   - Flag any that seem too large

4. **Input Coverage**: Does every required input have a producer?
   - For each task input, verify some task produces it
   - Or input is external/pre-existing
   - Flag orphan inputs

5. **Output Usefulness**: Is every output consumed?
   - For each task output, verify some task needs it
   - Or output is a final deliverable
   - Flag orphan outputs (warning, not error)

6. **Coherence**: Do tasks make logical sense together?
   - Check for contradictions
   - Check for impossible orderings
   - Flag logical issues

**Output Schema:**

```yaml
# .orchestrator/decomposition/validation.yaml

created: timestamp
step: 4-validation

valid: boolean

issues:
  gaps:
    - description: Work needed but no task covers it
      severity: error
      
  overlaps:
    - description: Duplicate work detected
      tasks: [task-a, task-b]
      severity: warning
      
  too_large:
    - task: task-id
      reason: Why it should be further decomposed
      severity: error
      
  missing_producers:
    - input: /path/to/file
      needed_by: task-id
      severity: error
      
  orphan_outputs:
    - output: /path/to/file
      produced_by: task-id
      severity: warning
      
  other:
    - description: Other issue
      severity: warning

recommendations:
  - How to fix each error
  - Suggestions for warnings
```

**Iteration:**

If validation fails, the process returns to earlier steps:
- Gaps → Return to subgoal identification
- Too large → Return to recursive decomposition for specific tasks
- Missing producers → Add new task or correct input specification
- Overlaps → Merge tasks or clarify boundaries

### 3.6 Step 5: Dependency Graph Construction

**Purpose:** Analyze tasks to build complete dependency graph and compute execution waves.

**Input:** Validated task list

**Output:** Dependencies + wave assignments

This step is detailed in Section 4.

---

## 4. Dependency Analysis

### 4.1 Philosophy

The framework aims to **infer** dependencies automatically from task definitions rather than requiring manual declaration. This is possible because:

1. Tasks have explicit input/output declarations
2. The framework controls the task format
3. Domain adapters provide inference heuristics
4. LLM analysis can catch semantic dependencies

Human override is always available but shouldn't be the primary mode.

### 4.2 Inference Algorithm

The dependency inference algorithm runs multiple passes, from highest to lowest confidence:

```
infer_dependencies(tasks: Task[]) → Dependency[]:
  
  dependencies = []
  
  # Pass 1: Explicit artifact matching (highest confidence)
  for each task B in tasks:
    for each input in B.inputs:
      for each task A in tasks where A ≠ B:
        if A.outputs contains artifact matching input.path:
          dependencies.add(Dependency {
            from: A,
            to: B,
            type: "artifact",
            confidence: "high",
            reason: "B.input matches A.output"
          })
  
  # Pass 2: Type-based matching
  for each task B in tasks:
    for each input in B.inputs where input.path is pattern/glob:
      for each task A in tasks where A ≠ B:
        if A.outputs matches input pattern or type:
          dependencies.add(Dependency {
            from: A,
            to: B,
            type: "artifact",
            confidence: "medium",
            reason: "Type/pattern match"
          })
  
  # Pass 3: Semantic analysis (LLM-assisted)
  semantic_deps = llm_analyze_semantic_dependencies(tasks)
  # LLM looks for references in descriptions like
  # "uses the X created by..." or "after Y is complete..."
  for each dep in semantic_deps:
    if not already_exists(dep, dependencies):
      dependencies.add(dep with type: "semantic")
  
  # Pass 4: Domain heuristics
  domain_deps = domain_adapter.suggest_dependencies(tasks)
  # e.g., "test tasks depend on code tasks"
  for each suggested in domain_deps:
    if not already_exists(suggested, dependencies):
      dependencies.add(suggested with {
        type: "heuristic",
        confidence: "low"
      })
  
  # Pass 5: Resource conflict detection
  for each pair of tasks (A, B):
    if A.outputs ∩ B.outputs is not empty:
      # Both modify same file - can't parallelize
      dependencies.add(Dependency {
        from: A,  # or B, based on other ordering
        to: B,
        type: "resource",
        confidence: "high",
        reason: "Both modify {overlapping files}"
      })
  
  # Validation
  validate_no_cycles(dependencies)
  dependencies = deduplicate(dependencies)
  
  return dependencies
```

### 4.3 Inference Sources by Confidence

| Confidence | Source | Reliability | Override Frequency |
|------------|--------|-------------|-------------------|
| High | Explicit path match (A outputs X, B inputs X) | Very reliable | Rare |
| High | Resource conflict (both modify same file) | Very reliable | Rare |
| Medium | Type/pattern match | Usually correct | Occasional |
| Medium | Semantic analysis | Good but not perfect | Sometimes |
| Low | Domain heuristics | Educated guesses | Often |

Low-confidence dependencies are included but flagged. During debugging, if something fails, low-confidence dependencies are the first place to look for incorrect assumptions.

### 4.4 Cycle Detection

A dependency cycle would cause deadlock (A waits for B, B waits for A). The framework detects cycles using standard algorithms (DFS-based cycle detection).

When a cycle is detected:
1. Report the cycle to the user
2. Identify which dependencies form the cycle
3. Suggest which dependency might be incorrect
4. Request human decision on how to break the cycle

Cycles typically indicate:
- Incorrect dependency inference (most common)
- Poorly decomposed tasks that should be merged
- Missing intermediate task that breaks the cycle

### 4.5 Wave Computation

Once dependencies are validated (acyclic), waves are computed by topological sorting:

```
compute_waves(tasks: Task[], dependencies: Dependency[]) → Wave[]:
  
  # Build adjacency list
  dependents = {}  # task → tasks that depend on it
  dependencies_count = {}  # task → number of unmet dependencies
  
  for each task in tasks:
    dependents[task] = []
    dependencies_count[task] = 0
  
  for each dep in dependencies:
    dependents[dep.from].add(dep.to)
    dependencies_count[dep.to] += 1
  
  # Kahn's algorithm for topological sort into waves
  waves = []
  remaining = set(tasks)
  
  while remaining is not empty:
    # Find all tasks with no unmet dependencies
    ready = [t for t in remaining if dependencies_count[t] == 0]
    
    if ready is empty and remaining is not empty:
      error("Cycle detected")  # Should have been caught earlier
    
    # This is the next wave
    wave = Wave {
      number: len(waves) + 1,
      tasks: ready
    }
    waves.add(wave)
    
    # Remove from remaining, update dependency counts
    for each task in ready:
      remaining.remove(task)
      for each dependent in dependents[task]:
        dependencies_count[dependent] -= 1
  
  return waves
```

### 4.6 Dependency Graph Output

```yaml
# .orchestrator/decomposition/graph.yaml

created: timestamp
step: 5-dependency-graph

dependencies:
  - from: auth-schema
    to: user-repository
    type: artifact
    confidence: high
    reason: user-repository lists /src/db/schema.sql as input

  - from: user-repository
    to: auth-service
    type: semantic
    confidence: medium
    reason: auth-service description mentions "uses UserRepository"

waves:
  1:
    tasks: [auth-schema, config-setup]
    rationale: No dependencies, can start immediately
    
  2:
    tasks: [user-repository, session-repository]
    rationale: Depend only on wave 1 outputs
    
  3:
    tasks: [auth-service]
    rationale: Depends on wave 2 outputs

validation:
  has_cycles: false
  all_tasks_assigned: true
  orphan_tasks: []
  low_confidence_count: 2
  warnings:
    - "auth-service → auth-tests dependency is heuristic-based"
```

---

## 5. Wave Computation and Parallel Execution

### 5.1 Execution Model

The orchestrator executes the plan wave by wave:

```
execute_plan(plan: Plan):
  
  for each wave in plan.waves:
    
    # Wait for all dependencies to be satisfied
    wait_until(all_previous_waves_complete(wave))
    
    # Dispatch all tasks in this wave in parallel
    running_tasks = []
    for each task in wave.tasks:
      agent = allocate_fresh_agent()
      running_tasks.add(dispatch(task, agent))
    
    # Wait for all tasks in wave to complete
    results = wait_all(running_tasks)
    
    # Handle any failures according to policy
    for each result in results:
      if result.failed:
        handle_failure(result.task, result.error)
    
    # Update state
    update_wave_state(wave, results)
    checkpoint_state()
    
    # Decide whether to continue
    if not can_proceed(wave, results):
      pause_execution("Wave {wave.number} blocked")
      return
  
  mark_plan_complete()
```

### 5.2 Parallelism Control

The orchestrator controls parallelism through configuration:

```yaml
execution:
  max_parallel_tasks: 5      # Never more than 5 tasks at once
  max_parallel_per_wave: 10  # Cap within a single wave
  agent_pool_size: 10        # Total available agents
```

**Parallelism Strategies:**

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Unlimited | All ready tasks run simultaneously | Fast machines, independent tasks |
| Capped | Maximum N tasks at once | Resource-constrained environments |
| Sequential | One task at a time | Debugging, deterministic replay |
| Adaptive | Adjust based on resource usage | Production with variable load |

### 5.3 Agent Allocation

Each task receives a fresh agent context:

```
dispatch(task: Task, agent: Agent):
  
  # Load only the context this task needs
  context = []
  context.add(load_file(plan.analysis))  # Goal context
  context.add(load_file(domain_adapter.conventions))  # Domain conventions
  
  for each input in task.inputs:
    context.add(load_file(input.path))
  
  # Add task-specific context notes
  context.add(task.context_notes)
  
  # Execute with fresh context (no accumulated garbage)
  result = agent.execute(task.description, context)
  
  # Verify outputs
  verification = verify_task(task, result)
  
  return TaskResult {
    task: task,
    artifacts: result.files_produced,
    verification: verification,
    success: verification.passed
  }
```

### 5.4 Context Management

The key insight driving the framework is that **fresh context produces peak quality**. The orchestrator ensures:

1. **No Context Accumulation**: Each task starts with empty context
2. **Minimal Context Loading**: Only load what the task declares as inputs
3. **No Cross-Task Pollution**: Parallel tasks have no shared context
4. **Explicit Handoff**: Information flows only through files

This differs from traditional approaches where one long conversation handles everything. That approach suffers from:
- Quality degradation as context fills
- Accumulated irrelevant information
- Difficulty isolating failures
- No parallelization opportunity

### 5.5 Wave Boundaries as Checkpoints

After each wave completes:

1. **Persist State**: Write all state to disk
2. **Verify Artifacts**: Confirm all expected outputs exist
3. **Update Progress**: Mark completed tasks/waves
4. **Log Events**: Append to event log

This means execution can safely stop at any wave boundary and resume later with no loss of progress.

### 5.6 Dynamic Scheduling (Teams Mode)

Wave-based execution has a structural limitation: **all tasks in wave N must complete before any task in wave N+1 starts**. If one task in a wave takes 10 minutes while others take 2 minutes, the fast tasks' dependents wait unnecessarily.

**Dynamic scheduling** eliminates wave boundary waiting. Tasks become eligible the instant their specific dependencies are satisfied, not when the entire wave completes.

#### Timing Example

Consider a 3-wave plan:

```
Wave 1: [A (2min), B (8min), C (3min)]
Wave 2: [D depends on A, E depends on B, F depends on C]
Wave 3: [G depends on D and F]
```

| Scheduling | Timeline | Total |
|-----------|----------|-------|
| Wave-based | Wave 1: 8min (waits for B) → Wave 2: start at 8min → Wave 3: after Wave 2 | ~20min |
| Dynamic | D starts at 2min (A done), F starts at 3min (C done), G starts when D+F done (~5min) | ~14min |

The improvement grows with plan size and task duration variance.

#### Two-Tier Architecture

Dynamic scheduling uses Claude Code's Agent Teams feature with a two-tier dispatch pattern:

```
┌─────────────────────────────────────────────────┐
│  Team Lead (ptf:team-lead)                      │
│  - Manages shared task list with dependencies   │
│  - Single writer for events.jsonl + state       │
│  - Receives completion messages from workers    │
│  - Handles failures per task policy             │
└────────┬──────────┬──────────┬──────────────────┘
         │          │          │
    ┌────▼───┐ ┌───▼────┐ ┌──▼─────┐
    │Worker 1│ │Worker 2│ │Worker 3│  ← Tier 1: Persistent
    │  claim │ │  claim │ │  claim │    (lightweight dispatchers)
    │  tasks │ │  tasks │ │  tasks │
    └───┬────┘ └───┬────┘ └───┬────┘
        │          │          │
    ┌───▼────┐ ┌───▼────┐ ┌──▼─────┐
    │Executor│ │Executor│ │Executor│  ← Tier 2: Fresh
    │ (fresh)│ │ (fresh)│ │ (fresh)│    (identical to classic)
    └────────┘ └────────┘ └────────┘
```

**Tier 1 workers** are persistent teammates that accumulate minimal context (~500 tokens per task cycle). They claim tasks from the shared list, read task definitions, spawn fresh Tier 2 executors, parse results, and report back to the lead.

**Tier 2 executors** are identical to classic mode's `ptf:executor` — same agent, same prompt format, same fresh context guarantee.

#### Configuration

```yaml
# .orchestrator/config.yaml
execution:
  mode: teams           # "classic" (default) | "teams"
  max_parallel_tasks: 3
  teams:
    worker_count: 3     # Persistent teammates
```

Requires: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`

#### Comparison: Wave-Based vs Dynamic

| Aspect | Wave-Based (Classic) | Dynamic (Teams) |
|--------|---------------------|-----------------|
| Task eligibility | All deps in prior wave | Specific deps satisfied |
| Parallelism | Within wave only | Across wave boundaries |
| Scheduling | Precomputed waves | Runtime dependency resolution |
| Coordinator | ptf:orchestrator (accumulates context) | ptf:team-lead (message-based) |
| Worker lifetime | Fresh per task | Persistent dispatcher + fresh executor |
| Resume | Wave boundary checkpoints | Per-task checkpoints |
| Best for | Small plans, step-by-step review | Large plans, uneven task durations |

---

## 6. Ralph-Style Execution

### 6.1 The Ralph Wiggum Loop

The Ralph Wiggum Loop, created by Geoffrey Huntley in 2025, is a technique for running AI coding agents that addresses context rot through **temporal iteration** rather than spatial decomposition:

```bash
while :; do cat PROMPT.md | claude-code ; done
```

The core insight: progress doesn't need to persist in an LLM's context window—it can live in files and git history instead. Each iteration starts with fresh context but inherits all *outcomes* (modified files, commits, test results) without inheriting any *confusion* that accumulated during the previous iteration.

The technique relies on a **completion promise**—the agent must output a specific phrase (e.g., "COMPLETE") to signal genuine completion. Until that promise appears, the loop continues.

### 6.2 Relationship to the Parallel Task Framework

Both techniques solve the same fundamental problem—context rot—but from different angles:

| Ralph Wiggum Loop | Parallel Task Framework |
|-------------------|------------------------|
| Solves context rot through **temporal iteration** | Solves context rot through **spatial decomposition** |
| "Keep trying with fresh context until done" | "Break into pieces that each fit in fresh context" |
| Progress lives in filesystem/git | Progress lives in state files/artifacts |
| Single task, repeated attempts | Multiple tasks, single attempt each (ideally) |

**These aren't competing approaches—they're complementary layers.**

Huntley's critique of multi-agent systems targets **chaotic coordination**—microservices-style agents that need to communicate, negotiate, and handle distributed state with non-deterministic results. But the Parallel Task Framework uses **structured parallelism**:

- Tasks within a wave are **truly independent**—no coordination needed
- Dependencies are **explicit and computed upfront**—no runtime negotiation
- State is **file-based**—no distributed consensus problems
- Wave boundaries are **synchronization points**—clear checkpoints

This is closer to dataflow parallelism (like a GPU executing independent shader invocations) than microservices chaos. Each agent operates in isolation, producing artifacts that later tasks consume through the filesystem—exactly the pattern Ralph endorses.

### 6.3 Ralph as Execution Primitive

The cleanest synthesis: **Ralph provides the execution primitive; the framework provides the coordination layer.**

```
Level 1: Single LLM call
           ↓
Level 2: Ralph Loop (repeat until verified)
           ↓  
Level 3: Parallel Task Framework (decompose + parallel Ralph loops)
           ↓
Level 4: Multi-framework orchestration (future)
```

Each level uses the one below as a primitive. The framework doesn't replace Ralph—it **orchestrates Ralph loops**.

Within the framework:

- **Within a task**: Vertical. One agent, one task, fresh context. This IS Ralph.
- **Across tasks in a wave**: Horizontal parallelism with no coordination. Each task runs independently. They don't communicate or share context. They're parallel Ralph loops that happen to run simultaneously.
- **Across waves**: Sequential. Wave 2 doesn't start until Wave 1 completes. This is the synchronization that prevents chaos.

### 6.4 Task Execution as Ralph Loop

Each task execution can operate in Ralph mode—repeating with fresh context until verification passes:

```yaml
task:
  id: auth-schema
  # ... other fields ...
  
  execution:
    mode: ralph          # or "single-shot" for simple tasks
    max_iterations: 10
    completion_promise: "VERIFICATION PASSED"
```

The task executor becomes:

```
execute_task(task):
  iterations = 0
  
  while iterations < task.execution.max_iterations:
    # Fresh context every iteration
    agent = allocate_fresh_agent()
    
    # Load only declared inputs
    context = load_task_inputs(task)
    
    # Execute
    output = agent.execute(task.description, context)
    
    # Check for completion promise
    if output contains task.execution.completion_promise:
      # Agent claims completion - verify the claim
      verification = verify(task.criteria)
      
      if verification.passed:
        return TaskResult(success=true, artifacts=output.files)
      else:
        # False promise - agent thought done but wasn't
        # Verification errors become context for next iteration
        log("False completion promise, verification failed")
    
    iterations += 1
  
  # Max iterations reached without success
  return TaskResult(success=false, reason="max_iterations_exceeded")
```

**Key mechanics:**

1. **Fresh context each iteration**: No accumulated confusion
2. **Completion promise as gate**: Agent must claim completion before verification runs
3. **Verification validates the claim**: Prevents false positives
4. **Failure feeds forward**: Each iteration's outcomes (files, errors) inform the next
5. **Bounded iterations**: Prevents infinite loops on impossible tasks

### 6.5 Completion Promise Pattern

The completion promise is a critical mechanism. The agent must explicitly output a specific phrase to signal it believes the work is complete. This:

- Prevents premature exits where the agent stops mid-task
- Creates a clear verification trigger
- Enables detection of "false promises" (agent claims done, verification fails)
- Provides a consistent contract between agent and orchestrator

**Standard completion promises:**

| Situation | Promise |
|-----------|---------|
| Task completion | `TASK COMPLETE` |
| Verification passed | `VERIFICATION PASSED` |
| Decomposition complete | `DECOMPOSITION COMPLETE` |
| Cannot proceed | `BLOCKED: [reason]` |

The agent's prompt includes instructions to output the appropriate promise:

```markdown
## Completion Protocol

When you have completed this task:
1. Ensure all outputs exist at their declared paths
2. Run all verification steps
3. If all verifications pass, output: VERIFICATION PASSED
4. If you cannot complete the task, output: BLOCKED: [specific reason]

Do not output the completion promise until you have verified your work.
```

### 6.6 Decomposition as Ralph Loop

The decomposition process itself benefits from Ralph-style execution:

```
decompose(goal):
  iterations = 0
  
  while iterations < max_decomposition_attempts:
    agent = fresh_agent()
    
    # Run decomposition steps
    agent.execute(decomposition_prompt)
    
    # Validate the decomposition
    validation = validate_decomposition()
    
    if validation.passed:
      return load_tasks_from_files()
    
    # Validation errors become context for next iteration
    # (they're written to files that next iteration reads)
    iterations += 1
  
  return DecompositionResult(success=false)
```

The agent doesn't need to remember its failed decomposition attempts—it just needs to see the validation errors written to files, adjust, and try again.

### 6.7 Tuning and Guardrails

Huntley describes "tuning Ralph like a guitar"—adding signs when Ralph fails. When the loop makes mistakes, you add notes that future iterations will see and follow.

This maps directly to **domain adapter evolution**:

```yaml
# Initial adapter
adapter:
  name: software-development
  guardrails: []

# After observing auth tasks failing on type errors
adapter:
  name: software-development
  guardrails:
    - "Always run type check before claiming completion"
    
# After observing config file deletions
adapter:
  name: software-development
  guardrails:
    - "Always run type check before claiming completion"
    - "Never delete configuration files without explicit instruction"
    
# After observing infinite retry loops
adapter:
  name: software-development
  guardrails:
    - "Always run type check before claiming completion"
    - "Never delete configuration files without explicit instruction"
    - "If the same test fails 3 consecutive times, output BLOCKED with diagnosis"
```

Guardrails are loaded into every task's context, providing the "signs" that Ralph learns from.

**Complementary constraint patterns from the Ralph ecosystem:**

| Ralph Pattern | Framework Equivalent |
|---------------|---------------------|
| **Marge** (constraints Ralph must respect) | Domain adapter guardrails + task constraints |
| **Principal Skinner** (structural harness, deterministic lanes) | Orchestrator validation + hook enforcement |
| **Signs** (notes for future iterations) | Guardrails + context notes in task definitions |

### 6.8 Mitigating Ralph's Failure Modes

Ralph has documented failure modes. The framework's structure helps mitigate several:

**Overbaking** (agent can't exit impossible task):
- Framework mitigation: Bounded iterations per task + BLOCKED promise
- If agent outputs `BLOCKED: [reason]`, orchestrator escalates rather than loops forever

**Meaning decay** (understanding drifts over iterations):
- Framework mitigation: Atomic tasks are small enough to complete in few iterations
- Clear task descriptions encode purpose explicitly
- Domain adapter conventions provide consistent interpretation

**Sycophancy loops** (agent overrides safety to satisfy completion):
- Framework mitigation: Verification is external to the agent
- Orchestrator validates artifacts independent of agent claims
- Hook enforcement (Skinner pattern) prevents dangerous actions

**Context exhaustion on complex tasks**:
- Framework mitigation: Decomposition breaks complex work into context-appropriate pieces
- Each task is sized to fit comfortably in fresh context
- Ralph handles iteration within a task; framework handles task sizing

### 6.9 Economic Implications

Ralph's economics are compelling—documented cases of $50K contracts completed for ~$300 in API costs. The framework can amplify this through parallelism:

**Single Ralph loop**: Tasks execute sequentially. Total time = sum of all task times.

**Framework with parallel waves**: Independent tasks run simultaneously. Total time = sum of wave times (where wave time = max task time in wave).

For a plan with 12 tasks across 5 waves, with 3 tasks parallelizable in wave 2:

| Approach | Execution Pattern | Relative Time |
|----------|------------------|---------------|
| Sequential Ralph | 12 tasks × avg time | 12 units |
| Framework parallel | 5 waves, parallel within | ~6-7 units |

Token costs are similar (same total work), but wall-clock time improves significantly.

**Trade-off**: The framework adds overhead (decomposition, dependency analysis, orchestration). For simple single-task work, raw Ralph is more efficient. For complex multi-part work, the framework's structure prevents thrashing and wasted iterations that occur when Ralph tackles something too large for its context.

### 6.10 Configuration

Task execution mode is configurable:

```yaml
# .orchestrator/config.yaml

execution:
  default_mode: ralph           # ralph | single-shot
  
  ralph:
    max_iterations: 10          # Per-task iteration limit
    completion_promise: "VERIFICATION PASSED"
    iteration_delay: 0          # Seconds between iterations (for rate limiting)
    
  single_shot:
    # No iteration, fail immediately if verification fails
    retry_on_failure: false

# Per-task override
task:
  id: simple-config-task
  execution:
    mode: single-shot           # Override default for simple tasks
    
task:
  id: complex-algorithm-task
  execution:
    mode: ralph
    max_iterations: 25          # More iterations for harder tasks
```

---

## 7. State Persistence Model

### 6.1 Design Goals

The state model supports:

| Goal | How Achieved |
|------|--------------|
| Resume from anywhere | All state on disk, nothing memory-only |
| Understand status | Single file shows current state |
| Debug failures | Detailed failure records with context |
| Audit history | Append-only event log |
| Inspect artifacts | Manifest of what was produced |

### 6.2 Directory Structure

```
.orchestrator/
├── config.yaml                    # Framework configuration
├── goal.md                        # Original goal (immutable)
│
├── decomposition/
│   ├── analysis.yaml              # Step 1: Analyzed goal
│   ├── subgoals.yaml              # Step 2: Identified subgoals  
│   ├── tasks.yaml                 # Step 3: All atomic tasks
│   ├── validation.yaml            # Step 4: Validation results
│   └── graph.yaml                 # Step 5: Dependencies + waves
│
├── plan.md                        # Human-readable plan (generated)
│
├── state/
│   ├── execution.yaml             # Current execution state (master)
│   ├── waves/
│   │   ├── wave-1.yaml            # Per-wave state
│   │   ├── wave-2.yaml
│   │   └── ...
│   └── tasks/
│       ├── task-a.yaml            # Per-task state
│       ├── task-b.yaml
│       └── ...
│
├── artifacts/
│   └── manifest.yaml              # Registry of produced artifacts
│
├── history/
│   ├── events.jsonl               # Append-only event log
│   └── sessions/
│       ├── session-001.yaml       # Session metadata
│       └── session-002.yaml
│
└── failures/
    ├── task-a-attempt-1.yaml      # Detailed failure records
    └── task-a-attempt-2.yaml
```

### 6.3 File Schemas

#### config.yaml

Framework configuration for this project.

```yaml
version: "1.0"
created: 2025-01-17T10:30:00Z

domain_adapter: software-development

execution:
  max_parallel_tasks: 5
  max_task_attempts: 3
  default_failure_strategy: retry_then_escalate

paths:
  artifacts_root: /src
  logs: .orchestrator/history

session:
  current: session-003
  auto_save_interval: 30s
```

#### goal.md

Original goal, preserved exactly as received. **Never modified after creation.**

```markdown
---
received: 2025-01-17T10:30:00Z
source: user
---

Build a user authentication system with email/password login,
session management, and password reset functionality.
```

#### state/execution.yaml

Master execution state — the primary file for understanding "where are we?"

```yaml
status: running  # pending | running | paused | completed | failed

current_wave: 2
waves_total: 5

progress:
  tasks_total: 12
  tasks_completed: 3
  tasks_running: 3
  tasks_pending: 6
  tasks_failed: 0
  tasks_blocked: 0

wave_summary:
  1: completed
  2: running
  3: pending
  4: pending
  5: pending

session:
  id: session-003
  started: 2025-01-17T10:32:00Z
  last_update: 2025-01-17T10:35:22Z

next_actions:
  - Complete wave 2 (3 tasks running)
  - Begin wave 3 when wave 2 completes

blockers: []

recent_events:
  - timestamp: 2025-01-17T10:35:22Z
    event: task_completed
    task: auth-schema
```

#### state/tasks/task-id.yaml

Per-task state with full detail.

```yaml
task_id: user-repository
wave: 2

status: running  # pending | ready | running | completed | failed | blocked | skipped

attempts:
  - attempt: 1
    started: 2025-01-17T10:35:05Z
    completed: null
    status: running
    agent_id: agent-wave2-001

dependencies:
  - task: auth-schema
    status: satisfied
    artifact: /src/db/migrations/001_auth_schema.sql
    verified: 2025-01-17T10:35:00Z

outputs_expected:
  - path: /src/repositories/userRepository.ts
    produced: false

outputs_produced: []

verification:
  status: pending  # pending | passed | failed
  results: []

blocked_by: []
blocking: [auth-service]

context_loaded:
  - .orchestrator/decomposition/analysis.yaml
  - /src/db/migrations/001_auth_schema.sql

notes: []
```

#### artifacts/manifest.yaml

Registry of all produced artifacts.

```yaml
artifacts:
  - path: /src/db/migrations/001_auth_schema.sql
    type: sql-migration
    produced_by: auth-schema
    produced_at: 2025-01-17T10:35:00Z
    verified: true
    checksum: sha256:abc123...
    consumed_by: [user-repository, session-repository]

pending:
  - path: /src/repositories/userRepository.ts
    expected_from: user-repository
    needed_by: [auth-service]
```

#### history/events.jsonl

Append-only event log for replay and debugging.

```jsonl
{"ts":"2025-01-17T10:30:00Z","event":"goal_received","goal_hash":"abc123"}
{"ts":"2025-01-17T10:30:15Z","event":"decomposition_started","step":1}
{"ts":"2025-01-17T10:31:30Z","event":"decomposition_completed","tasks":12,"waves":5}
{"ts":"2025-01-17T10:32:00Z","event":"execution_started","session":"session-003"}
{"ts":"2025-01-17T10:32:00Z","event":"wave_started","wave":1}
{"ts":"2025-01-17T10:32:01Z","event":"task_started","task":"auth-schema","agent":"agent-001"}
{"ts":"2025-01-17T10:35:00Z","event":"task_completed","task":"auth-schema","duration_s":179}
{"ts":"2025-01-17T10:35:00Z","event":"wave_completed","wave":1}
```

#### failures/task-attempt.yaml

Detailed failure records for debugging.

```yaml
task_id: auth-service
attempt: 1
wave: 3

started: 2025-01-17T10:40:00Z
failed: 2025-01-17T10:42:30Z
duration_s: 150

failure_mode: verification_failure

error:
  type: verification_failed
  message: Type check failed
  details: |
    npx tsc --noEmit src/services/authService.ts
    
    src/services/authService.ts:45:3 - error TS2345: 
    Argument of type 'string' is not assignable to parameter of type 'Buffer'.

context_snapshot:
  files_read:
    - /src/repositories/userRepository.ts
    - /src/repositories/sessionRepository.ts
  files_written:
    - /src/services/authService.ts (partial)

recovery_action: retry
next_attempt: 2

agent_logs: |
  [10:40:05] Loading task context...
  [10:41:00] Writing authService.ts...
  [10:42:30] Verification failed.
```

### 6.4 State Transitions

#### Task Lifecycle

```
         ┌──────────────────────────────────────────┐
         │                                          │
         ▼                                          │
    ┌─────────┐     dependencies      ┌─────────┐   │
    │ pending │ ──────satisfied─────► │  ready  │   │
    └─────────┘                       └────┬────┘   │
         ▲                                 │        │
         │                            dispatched    │
         │                                 │        │
    dependency                             ▼        │
      failed                          ┌─────────┐   │
         │                            │ running │   │
         │                            └────┬────┘   │
         │                                 │        │
    ┌─────────┐                    ┌───────┴───────┐
    │ blocked │                    │               │
    └─────────┘               succeeded         failed
                                   │               │
                                   ▼               ▼
                             ┌──────────┐    ┌─────────┐
                             │completed │    │  retry? │
                             └──────────┘    └────┬────┘
                                                  │
                                          ┌───────┴───────┐
                                          │               │
                                        yes              no
                                          │               │
                                          ▼               ▼
                                     ┌─────────┐    ┌─────────┐
                                     │  ready  │    │ failed  │
                                     └─────────┘    └─────────┘
```

#### Wave Lifecycle

```
    ┌─────────┐     all dependencies      ┌─────────┐
    │ pending │ ───────satisfied────────► │ running │
    └─────────┘                           └────┬────┘
                                               │
                                       ┌───────┴───────┐
                                       │               │
                                  all tasks       some tasks
                                  completed         failed
                                       │               │
                                       ▼               ▼
                                 ┌───────────┐   ┌─────────┐
                                 │ completed │   │ partial │
                                 └───────────┘   └─────────┘
```

### 6.5 Concurrency Model

Multiple agents may run in parallel. The framework uses **task-partitioned writes**:

- Each agent only writes to its own task state file
- The orchestrator owns execution.yaml and wave state files
- Event log is append-only with line-level concurrent safety
- Artifact manifest is orchestrator-owned, updated after task completion

This eliminates contention while maintaining consistent global state.

### 6.6 Resume Protocol

```
resume():
  
  1. READ state/execution.yaml
     → Determine current status, wave, progress
  
  2. IF status == "completed":
     → Report completion, nothing to resume
  
  3. IF status == "failed":
     → Show failure summary
     → Request human decision
  
  4. IF status == "running" or "paused":
     
     a. READ current wave state
        → Identify tasks that were running (may need restart)
        → Identify tasks ready but not started
     
     b. For tasks that were "running" when interrupted:
        → Check if outputs exist and verify
        → If verified: mark completed
        → If not: mark ready for retry
     
     c. UPDATE state files to reflect current reality
     
     d. CONTINUE execution from current wave
  
  5. LOG session resume event
```

---

## 8. Failure Handling and Recovery

### 8.1 Failure Modes

| Mode | Description | Detection |
|------|-------------|-----------|
| **Verification Failure** | Task completed but output doesn't meet criteria | Verification step returns failure |
| **Execution Failure** | Task couldn't complete (error, crash) | Agent reports error or times out |
| **Partial Output** | Task produced some but not all expected artifacts | Artifact check finds missing files |
| **Dependency Failure** | A prerequisite task failed | Cascade from upstream failure |
| **Timeout** | Task exceeded time limit | Timeout timer expired |

### 8.2 Recovery Strategies

```yaml
RetryStrategy:
  max_attempts: number
  backoff: none | linear | exponential
  backoff_base_seconds: number
  retry_condition: all | verification_only | execution_only

SkipStrategy:
  propagate_failure: boolean  # Do dependents also fail?
  reason: string

EscalateStrategy:
  message: string
  pause_execution: boolean
  options:
    - action: retry
      description: "Try again"
    - action: skip
      description: "Skip this task"
    - action: abort
      description: "Stop execution"

ReplanStrategy:
  scope: task | wave | phase | full
  preserve_completed: boolean
```

### 8.3 Failure Policy per Task

Each task can specify its failure handling:

```yaml
task:
  id: example-task
  # ... other fields ...
  
  on_failure:
    verification_failure:
      strategy: retry
      max_attempts: 2
      
    execution_failure:
      strategy: escalate
      message: "Task crashed, need human review"
      
    timeout:
      strategy: retry
      max_attempts: 1
      
    final_fallback:
      strategy: escalate
      pause_execution: true
```

### 8.4 Cascade Behavior

When Task A fails and Task B depends on A:

**Default: Cascade Failure**
```
A fails → B marked "blocked:dependency_failure"
```

Dependent tasks don't attempt execution. This is predictable and safe.

**Alternative: Explicit Override**

A task can declare it can proceed without a dependency:
```yaml
inputs:
  - path: /optional/file
    required: false  # Task will attempt even if this is missing
```

### 8.5 Failure Recovery Workflow

```
handle_failure(task, error):
  
  1. LOG failure event with full context
  
  2. CREATE failure record in failures/
  
  3. DETERMINE strategy from task.on_failure
  
  4. IF strategy == "retry" AND attempts < max_attempts:
     → Mark task as "ready"
     → Increment attempt counter
     → Return (will retry on next dispatch)
  
  5. IF strategy == "skip":
     → Mark task as "skipped"
     → IF propagate_failure:
         → Mark all dependents as "blocked:dependency_failure"
     → Continue execution with remaining tasks
  
  6. IF strategy == "escalate":
     → Pause execution
     → Present failure to human with options
     → Wait for human decision
     → Execute chosen option
  
  7. UPDATE state files
```

---

## 9. Domain Adapters

### 9.1 Purpose

Domain adapters customize the framework for specific types of work without modifying core logic. They provide:

- **Decomposition guidance**: How to break down goals in this domain
- **Atomicity criteria**: What makes a task properly sized
- **Artifact types**: What files this domain produces
- **Verification strategies**: How to verify each artifact type
- **Dependency patterns**: Common dependency shapes
- **Shared context**: Files to load for all tasks

### 9.2 Adapter Interface

```yaml
DomainAdapter:
  name: string
  description: string
  version: string
  
  decomposition:
    subgoal_heuristics:
      - name: string
        description: string
        examples: []
        
    atomicity_criteria:
      - criterion: string
        check: string  # How to verify this criterion
        
    max_recursion_depth: number
    
  artifacts:
    types:
      - name: string
        extensions: []
        description: string
        
    verification_strategies:
      type_name:
        - method: exists | contains | runs | syntax | custom
          description: string
          
  dependencies:
    common_patterns:
      - name: string
        from_type: string
        to_type: string
        description: string
        
    inference_hints:
      - pattern: string
        implies: string
        
  context:
    shared_files:
      - path: string
        purpose: string
        
    conventions_doc: string  # Path to conventions document
```

### 9.3 Example: Software Development Adapter

```yaml
name: software-development
description: |
  Domain adapter for software development tasks including
  web applications, APIs, databases, and infrastructure.
version: "1.0"

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
        
    - name: by-feature
      description: Split by user-facing feature
      examples:
        - "User authentication"
        - "Payment processing"
        - "Notification system"
        
    - name: by-file-boundary
      description: One task per file/module
      examples:
        - "Create userRepository.ts"
        - "Create authService.ts"
        
    - name: by-interface
      description: Cut at API/interface boundaries
      examples:
        - "Internal API vs external API"
        - "Service interfaces vs implementations"
        
  atomicity_criteria:
    - criterion: single-file
      check: Task produces at most one file (or tightly-coupled set of 2-3)
      
    - criterion: testable
      check: Can write a verification command for the output
      
    - criterion: focused
      check: Task touches one concept/concern
      
    - criterion: context-bounded
      check: Needs only local knowledge, not entire system
      
  max_recursion_depth: 5

artifacts:
  types:
    - name: source-code
      extensions: [.ts, .js, .py, .go, .rs, .java]
      description: Application source code
      
    - name: sql-migration
      extensions: [.sql]
      description: Database schema migrations
      
    - name: configuration
      extensions: [.yaml, .yml, .json, .toml, .env]
      description: Configuration files
      
    - name: documentation
      extensions: [.md, .rst, .txt]
      description: Documentation files
      
    - name: test
      extensions: [.test.ts, .spec.ts, _test.go, _test.py]
      description: Test files
      
  verification_strategies:
    source-code:
      - method: exists
        description: File exists at expected path
      - method: syntax
        description: File parses without syntax errors
      - method: runs
        description: Type check / lint passes
        
    sql-migration:
      - method: exists
        description: Migration file exists
      - method: syntax
        description: Valid SQL syntax
      - method: runs
        description: Dry-run against database succeeds
        
    test:
      - method: exists
        description: Test file exists
      - method: runs
        description: Tests pass

dependencies:
  common_patterns:
    - name: schema-to-repository
      from_type: sql-migration
      to_type: source-code
      description: Repositories depend on schemas they access
      
    - name: repository-to-service
      from_type: source-code
      to_type: source-code
      description: Services depend on repositories they use
      
    - name: code-to-test
      from_type: source-code
      to_type: test
      description: Tests depend on code they test
      
  inference_hints:
    - pattern: "imports {X} from"
      implies: depends on file that exports X
      
    - pattern: "extends {X}"
      implies: depends on file that defines X
      
    - pattern: "uses {X}Repository"
      implies: depends on X repository task

context:
  shared_files:
    - path: .orchestrator/decomposition/analysis.yaml
      purpose: Goal and requirements context
      
    - path: docs/architecture.md
      purpose: System architecture reference
      
    - path: docs/conventions.md
      purpose: Coding conventions and patterns
      
  conventions_doc: docs/conventions.md
```

### 9.4 Example: Research Adapter

```yaml
name: research
description: |
  Domain adapter for research and analysis tasks including
  literature review, data analysis, and synthesis.
version: "1.0"

decomposition:
  subgoal_heuristics:
    - name: by-question
      description: Split by research question/sub-question
      examples:
        - "What is the current state of X?"
        - "What are the main approaches to Y?"
        - "How do A and B compare?"
        
    - name: by-source-type
      description: Split by type of source to analyze
      examples:
        - "Academic literature review"
        - "Industry reports analysis"
        - "Case study examination"
        
    - name: by-analysis-stage
      description: Split by stage in research process
      examples:
        - "Source gathering"
        - "Data extraction"
        - "Synthesis"
        - "Conclusion formulation"
        
  atomicity_criteria:
    - criterion: single-question
      check: Task addresses one specific question
      
    - criterion: bounded-sources
      check: Task analyzes a manageable number of sources (1-5)
      
    - criterion: clear-output
      check: Task produces one finding document
      
  max_recursion_depth: 4

artifacts:
  types:
    - name: finding
      extensions: [.md]
      description: Research finding or analysis result
      
    - name: summary
      extensions: [.md]
      description: Summary of sources or literature
      
    - name: synthesis
      extensions: [.md]
      description: Synthesized conclusions from multiple findings
      
    - name: data
      extensions: [.csv, .json, .yaml]
      description: Extracted or processed data
      
  verification_strategies:
    finding:
      - method: exists
        description: Finding document exists
      - method: contains
        description: Contains required sections (question, evidence, conclusion)
        
    synthesis:
      - method: exists
        description: Synthesis document exists
      - method: contains
        description: References underlying findings

dependencies:
  common_patterns:
    - name: source-to-finding
      from_type: summary
      to_type: finding
      description: Findings depend on source analysis
      
    - name: finding-to-synthesis
      from_type: finding
      to_type: synthesis
      description: Synthesis depends on individual findings
      
  inference_hints:
    - pattern: "based on {X}"
      implies: depends on X finding/summary
      
    - pattern: "synthesizing {X} and {Y}"
      implies: depends on X and Y findings

context:
  shared_files:
    - path: .orchestrator/decomposition/analysis.yaml
      purpose: Research goals and scope
      
  conventions_doc: null
```

### 9.5 Creating Custom Adapters

To create a domain adapter:

1. **Identify decomposition patterns**: How is work naturally divided in this domain?
2. **Define atomicity**: What makes a task the right size?
3. **Catalog artifact types**: What files does this domain produce?
4. **Specify verification**: How do you know each artifact type is correct?
5. **Map dependency patterns**: What are typical dependency relationships?
6. **Identify shared context**: What background knowledge helps all tasks?

The framework provides a template adapter that can be customized:

```yaml
# adapters/template.yaml
name: my-domain
description: |
  Adapter for [describe your domain]
version: "1.0"

decomposition:
  subgoal_heuristics:
    - name: [heuristic-name]
      description: [how to apply this heuristic]
      examples: []
      
  atomicity_criteria:
    - criterion: [criterion-name]
      check: [how to verify]
      
  max_recursion_depth: 4

artifacts:
  types:
    - name: [artifact-type]
      extensions: []
      description: [what this artifact type is]
      
  verification_strategies:
    [artifact-type]:
      - method: [exists | contains | runs | custom]
        description: [verification description]

dependencies:
  common_patterns: []
  inference_hints: []

context:
  shared_files: []
  conventions_doc: null
```

---

## 10. File Formats and Schemas

### 10.1 Format Philosophy

The framework uses:
- **Markdown** for human-readable documents with optional YAML frontmatter
- **YAML** for structured data and configuration
- **JSON Lines** for append-only logs

This provides:
- Human readability without tooling
- Easy parsing in any language
- Git-friendly diffing
- Clear separation of metadata (YAML) and content (Markdown)

### 10.2 Task Definition Format

Tasks are defined in markdown with YAML frontmatter:

```markdown
---
id: auth-schema
name: Create authentication database schema
wave: 1

inputs:
  - path: .orchestrator/decomposition/analysis.yaml
    description: Requirements and constraints

outputs:
  - path: /src/db/migrations/001_auth_schema.sql
    type: sql-migration

verify:
  - type: exists
    target: /src/db/migrations/001_auth_schema.sql
  - type: contains
    target: /src/db/migrations/001_auth_schema.sql
    expected: [CREATE TABLE users, CREATE TABLE sessions]
  - type: runs
    target: psql -f /src/db/migrations/001_auth_schema.sql --dry-run

on_failure:
  strategy: retry
  max_attempts: 2
---

Create the PostgreSQL schema for user authentication.

## Context

The system needs email/password authentication with session tokens.
Sessions expire after 24 hours of inactivity.

## Requirements

- `users` table with id, email, password_hash, created_at
- `sessions` table with id, user_id, token, expires_at
- Appropriate indexes for lookup by email and token
- Foreign key from sessions to users

## Done When

The schema file exists and passes dry-run validation against PostgreSQL.
```

### 10.3 Plan Format

Plans collect tasks with dependency and wave information:

```markdown
---
plan_id: phase-01-auth
goal: Implement user authentication system
created: 2025-01-17T10:30:00Z
status: pending
domain: software-development

summary:
  total_tasks: 12
  total_waves: 5
  estimated_parallel_time: "45 minutes"
---

# Phase 1: Authentication System

## Overview

This phase implements email/password authentication with session management.

## Dependency Graph

```
Wave 1: [auth-schema] [rate-limiter-config]
           │                    │
           ▼                    ▼
Wave 2: [user-repo] [session-repo] [rate-limiter-middleware]
              │          │
              ▼          ▼
Wave 3:    [auth-service]
                 │
                 ▼
Wave 4: [login-endpoint] [register-endpoint] [logout-endpoint]
                              │
                              ▼
Wave 5:              [auth-tests]
```

## Wave 1

No dependencies — can start immediately.

### Task: auth-schema

[task definition as above]

### Task: rate-limiter-config

[task definition]

## Wave 2

Depends on Wave 1 completion.

### Task: user-repository

[task definition]

[... remaining waves and tasks ...]
```

### 10.4 State File Formats

See Section 6.3 for complete state file schemas.

### 10.5 Verification Result Format

```yaml
verification:
  task_id: auth-schema
  timestamp: 2025-01-17T10:35:00Z
  status: passed  # passed | failed
  
  checks:
    - type: exists
      target: /src/db/migrations/001_auth_schema.sql
      passed: true
      
    - type: contains
      target: /src/db/migrations/001_auth_schema.sql
      expected: [CREATE TABLE users, CREATE TABLE sessions]
      passed: true
      found: [CREATE TABLE users, CREATE TABLE sessions, CREATE INDEX]
      
    - type: runs
      target: psql -f /src/db/migrations/001_auth_schema.sql --dry-run
      passed: true
      exit_code: 0
      stdout: |
        CREATE TABLE
        CREATE TABLE
        CREATE INDEX
      stderr: ""
      
  summary: "All 3 verification checks passed"
```

---

## 11. Framework Architecture

### 11.1 Component Overview

The v1 implementation is a Claude Code plugin. Claude Code serves as the execution environment, with the plugin providing structure and orchestration:

```
┌─────────────────────────────────────────────────────────────────┐
│                      User (via Claude Code)                     │
│                                                                 │
│  /ptf:init   /ptf:decompose   /ptf:execute   /ptf:status       │
└─────────────────────────────────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
┌─────────────────────────────┐ ┌─────────────────────────────────┐
│      Slash Commands         │ │         Agent Skill             │
│                             │ │                                 │
│  - Parse user intent        │ │  - Framework concepts           │
│  - Invoke appropriate flow  │ │  - Decomposition patterns       │
│  - Present results          │ │  - Best practices               │
└─────────────────────────────┘ └─────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Orchestrator Subagent                       │
│                                                                 │
│  - Coordinate execution flow                                    │
│  - Manage wave progression                                      │
│  - Handle failures                                              │
│  - Dispatch to specialized subagents                            │
└─────────────────────────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┬───────────┐
        ▼           ▼           ▼           ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ Decomposer  │ │ Dependency  │ │   Task      │ │  Verifier   │
│  Subagent   │ │  Analyzer   │ │  Executor   │ │  Subagent   │
│             │ │  Subagent   │ │  Subagent   │ │             │
│ - 5-step    │ │             │ │             │ │ - Check     │
│   process   │ │ - Inference │ │ - Fresh     │ │   outputs   │
│ - Recursive │ │ - Waves     │ │   context   │ │ - Run tests │
│   breakdown │ │ - Cycles    │ │ - Single    │ │ - Report    │
│             │ │             │ │   task      │ │   results   │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
        │               │               │               │
        └───────────────┴───────────────┴───────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Domain Adapter                             │
│                                                                 │
│  - Decomposition heuristics for this domain                     │
│  - Atomicity criteria                                           │
│  - Verification strategies                                      │
│  - Common dependency patterns                                   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    State Persistence (Files)                    │
│                                                                 │
│  .orchestrator/                                                 │
│  ├── goal.md              (immutable)                           │
│  ├── decomposition/       (decomposition outputs)               │
│  ├── state/               (execution state)                     │
│  ├── artifacts/           (artifact registry)                   │
│  └── history/             (event log)                           │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Hooks                                  │
│                                                                 │
│  - post-task-complete     (log events, update state)            │
│  - pre-wave-start         (checkpoint)                          │
│  - on-failure             (failure handling)                    │
└─────────────────────────────────────────────────────────────────┘
```

**Key Architectural Decisions:**

1. **Claude Code as Runtime**: The plugin leverages Claude Code's existing capabilities (Task tool, subagents, file system access) rather than implementing its own execution environment.

2. **Subagent Isolation**: Each task executes in a fresh subagent context, ensuring no context degradation across tasks.

3. **File-Based State**: All state persists to files, enabling resume from any point and providing human-readable debugging.

4. **Pluggable Adapters**: Domain-specific knowledge lives in adapters, keeping the core orchestration logic domain-agnostic.

### 11.2 Plugin File Structure

The v1 implementation is a Claude Code plugin:

```
parallel-task-framework/
├── README.md                      # Plugin documentation
├── SPECIFICATION.md               # This document (theory/concepts)
├── LICENSE                        # License file
├── package.json                   # Plugin metadata
│
├── commands/                      # Slash commands
│   ├── init.md                    # /ptf:init [goal]
│   ├── decompose.md               # /ptf:decompose
│   ├── plan.md                    # /ptf:plan
│   ├── execute.md                 # /ptf:execute [wave]
│   ├── execute-all.md             # /ptf:execute-all
│   ├── status.md                  # /ptf:status
│   ├── resume.md                  # /ptf:resume
│   ├── verify.md                  # /ptf:verify [task]
│   ├── retry.md                   # /ptf:retry [task]
│   └── abort.md                   # /ptf:abort
│
├── agents/                        # Subagent definitions
│   ├── decomposer.md              # Decomposition specialist
│   ├── dependency-analyzer.md     # Dependency inference
│   ├── task-executor.md           # Single task execution
│   ├── verifier.md                # Verification specialist
│   └── orchestrator.md            # Execution coordination
│
├── skills/
│   └── SKILL.md                   # Framework knowledge for Claude
│
├── hooks/
│   ├── post-task-complete.sh      # After task completion
│   ├── pre-wave-start.sh          # Before wave execution
│   ├── on-failure.sh              # On task failure
│   └── on-session-end.sh          # On session end
│
├── adapters/                      # Domain adapters
│   ├── software-development.yaml
│   ├── research.yaml
│   └── template.yaml
│
├── schemas/                       # YAML schemas
│   ├── task.schema.yaml
│   ├── plan.schema.yaml
│   ├── state.schema.yaml
│   ├── dependency.schema.yaml
│   └── adapter.schema.yaml
│
├── prompts/                       # Reusable prompt templates
│   ├── analyze-goal.md
│   ├── identify-subgoals.md
│   ├── evaluate-atomicity.md
│   ├── decompose-subgoal.md
│   ├── validate-decomposition.md
│   ├── infer-dependencies.md
│   └── compute-waves.md
│
└── examples/                      # Example projects
    ├── auth-system/               # Software example
    │   ├── goal.md
    │   └── expected-output/
    └── literature-review/         # Research example
        ├── goal.md
        └── expected-output/
```

### 11.3 What the Framework Is (and Isn't)

The Parallel Task Framework is fundamentally about orchestrating **LLM agents**. Every step requires an LLM:

| Step | Executor |
|------|----------|
| Goal Analysis | LLM prompt |
| Subgoal Identification | LLM prompt |
| Recursive Decomposition | LLM prompt |
| Dependency Inference | LLM prompt (semantic analysis) |
| Task Execution | LLM agent |
| Verification | Automated checks + LLM verification |

This means the framework is not standalone executable software like a build system. It is:

**A specification + prompt library + schema definitions + orchestration patterns**

The framework provides the "how to think about decomposition and parallel execution." A host system provides the actual LLM integration and execution environment.

### 11.4 V1 Implementation: Claude Code Plugin

The primary implementation of the framework is a **Claude Code plugin** that provides:

- **Slash Commands** — User-facing commands that invoke framework workflows
- **Subagents** — Specialized agents for decomposition, execution, and verification
- **Agent Skill** — SKILL.md that teaches Claude the framework's concepts and patterns
- **Hooks** — Automation for state management and event logging

This is the natural fit because Claude Code **is** the agent. Claude Code serves as both the orchestrator AND the executing agent (via Task tool / subagents).

#### Plugin Structure

```
ptf-plugin/
├── README.md                      # Plugin documentation
├── package.json                   # Plugin metadata
│
├── commands/
│   ├── init.md                    # /ptf:init [goal]
│   ├── decompose.md               # /ptf:decompose
│   ├── execute.md                 # /ptf:execute [wave]
│   ├── status.md                  # /ptf:status
│   ├── resume.md                  # /ptf:resume
│   └── ...
│
├── agents/
│   ├── decomposer.md              # Decomposition subagent
│   ├── dependency-analyzer.md     # Dependency inference subagent
│   ├── task-executor.md           # Task execution subagent
│   ├── verifier.md                # Verification subagent
│   └── orchestrator.md            # Main orchestration agent
│
├── skills/
│   └── SKILL.md                   # Framework knowledge for Claude
│
├── hooks/
│   ├── post-task-complete.sh      # Log events, update state
│   ├── pre-wave-start.sh          # Checkpoint state
│   └── on-failure.sh              # Failure handling
│
├── adapters/
│   ├── software-development.yaml  # Software domain adapter
│   ├── research.yaml              # Research domain adapter
│   └── template.yaml              # Template for custom adapters
│
└── schemas/
    ├── task.schema.yaml
    ├── plan.schema.yaml
    └── state.schema.yaml
```

#### Slash Commands

| Command | Purpose |
|---------|---------|
| `/ptf:init [goal]` | Initialize new project, run goal analysis |
| `/ptf:decompose` | Run full decomposition process (Steps 1-5) |
| `/ptf:plan` | Generate human-readable plan from decomposition |
| `/ptf:execute [wave]` | Execute specified wave (or next pending wave) |
| `/ptf:execute-all` | Execute all waves with parallel subagents |
| `/ptf:status` | Show current execution state |
| `/ptf:resume` | Resume from interruption |
| `/ptf:verify [task]` | Run verification for a task |
| `/ptf:retry [task]` | Retry a failed task |
| `/ptf:abort` | Stop execution, preserve state |

#### Subagents

**Decomposer Agent**

Runs the 5-step decomposition process. Spawned by `/ptf:decompose`.

```markdown
# agents/decomposer.md

You are a decomposition specialist. Your role is to break down
complex goals into atomic, parallelizable tasks.

## Process

1. Run goal analysis (Step 1)
2. Identify subgoals (Step 2)
3. Recursively decompose until atomic (Step 3)
4. Validate decomposition (Step 4)
5. Hand off to dependency analyzer

## Inputs
- Goal from .orchestrator/goal.md
- Domain adapter from config

## Outputs
- .orchestrator/decomposition/analysis.yaml
- .orchestrator/decomposition/subgoals.yaml
- .orchestrator/decomposition/tasks.yaml
- .orchestrator/decomposition/validation.yaml
```

**Dependency Analyzer Agent**

Infers dependencies and computes waves. Spawned after decomposition.

```markdown
# agents/dependency-analyzer.md

You are a dependency analysis specialist. Your role is to
analyze tasks and determine execution order.

## Process

1. Load all tasks from decomposition
2. Run multi-pass dependency inference
3. Detect and report any cycles
4. Compute wave assignments
5. Generate dependency graph

## Outputs
- .orchestrator/decomposition/graph.yaml
```

**Task Executor Agent**

Executes a single task. Spawned per-task during wave execution.

```markdown
# agents/task-executor.md

You are a task execution specialist. Execute exactly one task
with fresh context.

## Process

1. Load task definition
2. Load declared inputs only
3. Execute task instructions
4. Produce declared outputs
5. Run verification steps
6. Report results

## Context Loading
ONLY load files declared in task.inputs. Do not load
additional context. Fresh context = peak quality.
```

**Verifier Agent**

Runs verification checks. Can be spawned independently.

```markdown
# agents/verifier.md

You are a verification specialist. Confirm task outputs
meet their criteria.

## Verification Methods

- exists: Check file exists
- contains: Check file contains expected content
- runs: Execute command, check exit code
- syntax: Validate file syntax/format
- custom: Run custom verification logic
```

**Orchestrator Agent**

Coordinates wave execution. The main control loop.

```markdown
# agents/orchestrator.md

You are the execution orchestrator. Manage wave-by-wave
execution of the plan.

## Process

1. Load execution state
2. Identify next wave to execute
3. Spawn task-executor subagents for all tasks in wave
4. Collect results
5. Handle any failures
6. Update state
7. Checkpoint
8. Continue to next wave or pause

## Parallel Execution
Use Task tool to spawn parallel subagents for all tasks
in a wave. Monitor completion. Respect max_parallel config.
```

#### Agent Skill (SKILL.md)

The skill file teaches Claude the framework's concepts:

```markdown
# skills/SKILL.md

# Parallel Task Framework Skill

## Overview

This skill enables systematic decomposition of complex goals
into atomic, parallelizable tasks with automatic dependency
analysis and wave-based parallel execution.

## When to Use

- User has a complex goal requiring multiple steps
- Work can be parallelized for efficiency
- Context management is important (large projects)
- Reliable, verifiable execution is needed

## Core Concepts

[Condensed version of Section 2 from this document]

## Decomposition Process

[Condensed version of Section 3]

## Key Files

- .orchestrator/goal.md — Original goal (immutable)
- .orchestrator/decomposition/ — Decomposition outputs
- .orchestrator/state/ — Execution state
- .orchestrator/plan.md — Human-readable plan

## Commands Reference

[Command table]
```

#### Hooks

**post-task-complete.sh**

Runs after each task completes:

```bash
#!/bin/bash
# Append event to log
echo "{\"ts\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"event\":\"task_completed\",\"task\":\"$TASK_ID\"}" >> .orchestrator/history/events.jsonl

# Update task state
# (Claude handles YAML updates)
```

**pre-wave-start.sh**

Runs before starting a wave:

```bash
#!/bin/bash
# Checkpoint current state
cp .orchestrator/state/execution.yaml ".orchestrator/state/checkpoints/$(date +%s).yaml"
```

**on-failure.sh**

Runs when a task fails:

```bash
#!/bin/bash
# Log failure event
echo "{\"ts\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"event\":\"task_failed\",\"task\":\"$TASK_ID\"}" >> .orchestrator/history/events.jsonl
```

### 11.6 Backend Adapters

The framework's state persistence can be implemented by different backends. The default is a lightweight file-based system (the `.orchestrator/` directory described in Section 7). However, the framework supports pluggable backends that provide the same capabilities through different infrastructure.

```
┌─────────────────────────────────────────────────────────────────┐
│              Parallel Task Framework Core                       │
│     (Decomposition, Wave Computation, Orchestration)            │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌───────────────────────┐         ┌───────────────────────┐
│    File Backend       │         │    Beads Backend      │
│   (.orchestrator/)    │         │      (.beads/)        │
│                       │         │                       │
│ - YAML state files    │         │ - bd CLI              │
│ - Framework schemas   │         │ - JSONL + SQLite      │
│ - Event log (JSONL)   │         │ - Git sync daemon     │
│ - Simple, portable    │         │ - Merge driver        │
└───────────────────────┘         └───────────────────────┘
```

**Backend Interface:**

```yaml
StateBackend:
  # Task management
  create_task(task: Task) → TaskId
  get_task(task_id: TaskId) → Task
  update_task(task_id: TaskId, updates: TaskUpdates)
  close_task(task_id: TaskId, reason: string)
  
  # Dependency management
  add_dependency(from: TaskId, to: TaskId, type: DependencyType)
  get_dependencies(task_id: TaskId) → Dependency[]
  
  # Ready work (Wave 1)
  get_ready_tasks(filter: WorkFilter) → Task[]
  
  # State queries
  get_blocked_tasks() → Task[]
  get_task_status(task_id: TaskId) → Status
  
  # Execution state
  save_execution_state(state: ExecutionState)
  load_execution_state() → ExecutionState
  
  # Events
  log_event(event: Event)
  get_events(filter: EventFilter) → Event[]
```

### 11.7 Beads Backend Integration

**Beads** is a distributed, git-backed graph issue tracker designed for AI agent workflows, created by Steve Yegge. It provides persistent, structured memory for coding agents through a dependency-aware graph that allows agents to handle long-horizon tasks without losing context.

Beads is an excellent candidate for a framework backend because it solves many of the same problems with mature, battle-tested infrastructure.

#### Concept Mapping

| Parallel Task Framework | Beads Equivalent |
|------------------------|------------------|
| Task | Issue |
| Artifact | Git-tracked file (implicit) |
| Dependency (blocks) | `blocks` dependency type |
| Dependency (parent-child) | `parent-child` dependency type |
| Wave 1 (ready tasks) | `bd ready` output |
| Plan / Milestone | Molecule (epic with children) |
| Checkpoint:human-verify | Gate (`human` type) |
| Checkpoint:decision | Gate (`human` type) |
| Checkpoint:external | Gate (`gh:run`, `gh:pr`, `timer` types) |
| State persistence | `.beads/` directory (JSONL + SQLite) |
| Event log | Git history + issue audit trail |
| Completion verification | `bd close --reason` |

#### What Beads Provides

**Storage Architecture:**
- **JSONL file** (`.beads/issues.jsonl`) — Git-tracked source of truth, merge-friendly
- **SQLite database** (`.beads/beads.db`) — Local cache for fast queries, git-ignored
- **Automatic sync** — Bidirectional sync between JSONL and SQLite
- **Background daemon** — Auto-sync, file watching, RPC interface

**Dependency System:**
- `blocks` — Hard dependency, affects ready work calculation
- `parent-child` — Hierarchical relationships (epics with children)
- `waits-for` — Fan-in gates, wait for all children to complete
- `conditional-blocks` — Task runs only if another fails
- `related` — Soft link for reference, doesn't block

**Ready Work Calculation:**
- `bd ready` returns tasks with no open blockers
- Respects dependency types that affect ready work
- Filters by priority, assignee, labels, status
- Handles deferred tasks (future start dates)

**Git Integration:**
- Hash-based IDs (`bd-a1b2`) prevent merge collisions
- Custom merge driver for `.beads/issues.jsonl`
- Protected branch workflow via sync branches
- Full audit trail through git history

**AI Agent Integration:**
- Claude Code plugin with hooks and slash commands
- `bd prime` for dynamic context injection
- MCP server for tool-based access
- Session management for multi-conversation workflows

**Advanced Coordination:**
- **Molecules** — Epics with workflow semantics, up to 3 levels of nesting
- **Gates** — Async wait conditions (human approval, CI completion, timers)
- **Wisps** — Ephemeral issues for temporary work

#### What the Framework Still Owns

Even with Beads as the backend, the framework provides:

| Capability | Why Framework Owns It |
|------------|----------------------|
| **Decomposition** | Beads doesn't break down goals; we do |
| **Wave computation** | We pre-compute full wave structure; Beads computes incrementally |
| **Domain adapters** | Beads has no concept of domain-specific decomposition |
| **Task format** | Our markdown+YAML with verification criteria |
| **Ralph-style execution** | Beads doesn't loop with fresh context; we do |
| **Verification orchestration** | Beads tracks completion; we verify outputs |
| **Subagent dispatch** | Beads is storage; we're the execution engine |

#### Beads Backend Implementation

```python
class BeadsBackend(StateBackend):
    """
    State backend using Beads (bd) for persistence.
    Requires: bd CLI installed and initialized in repository.
    """
    
    def __init__(self, use_daemon: bool = True):
        self.daemon_flag = "" if use_daemon else "--no-daemon"
    
    # ─────────────────────────────────────────────────────────
    # Task Management
    # ─────────────────────────────────────────────────────────
    
    def create_task(self, task: Task) -> TaskId:
        """Create a task as a Beads issue."""
        
        # Build command
        cmd = f'bd create "{task.name}" -t task'
        
        # Map priority (our 1-5 to Beads P0-P4)
        if task.priority:
            cmd += f' -p {task.priority - 1}'
        
        # Add labels from task metadata
        if task.labels:
            cmd += f' -l "{",".join(task.labels)}"'
        
        # Store full task definition in notes
        task_yaml = serialize_task_to_yaml(task)
        cmd += f' --notes "PTF_TASK_DEF:\n{task_yaml}"'
        
        # If this is a child of a molecule/epic
        if task.parent_id:
            cmd += f' --parent {task.parent_id}'
        
        result = shell(cmd)
        return parse_issue_id(result)
    
    def get_task(self, task_id: TaskId) -> Task:
        """Retrieve task from Beads issue."""
        result = shell(f'bd show {task_id} --json')
        issue = json.loads(result)
        
        # Extract our task definition from notes
        task_def = extract_ptf_task_def(issue['notes'])
        if task_def:
            return deserialize_task_from_yaml(task_def)
        
        # Fallback: construct task from issue fields
        return Task(
            id=issue['id'],
            name=issue['title'],
            status=map_beads_status(issue['status']),
            priority=issue['priority'] + 1,
            labels=issue.get('labels', [])
        )
    
    def update_task(self, task_id: TaskId, updates: TaskUpdates):
        """Update task in Beads."""
        cmd = f'bd update {task_id}'
        
        if updates.status:
            cmd += f' --status {map_to_beads_status(updates.status)}'
        
        if updates.notes:
            cmd += f' --notes "{updates.notes}"'
        
        if updates.assignee:
            cmd += f' --assignee {updates.assignee}'
        
        shell(cmd)
    
    def close_task(self, task_id: TaskId, reason: str):
        """Close task with completion reason."""
        shell(f'bd close {task_id} --reason "{reason}"')
    
    # ─────────────────────────────────────────────────────────
    # Dependency Management
    # ─────────────────────────────────────────────────────────
    
    def add_dependency(self, from_id: TaskId, to_id: TaskId, 
                       dep_type: DependencyType = "blocks"):
        """
        Add dependency: to_id depends on from_id.
        
        Beads: bd dep add <child> <parent>
        Meaning: child is blocked by parent
        """
        beads_type = map_to_beads_dep_type(dep_type)
        shell(f'bd dep add {to_id} {from_id} --type {beads_type}')
    
    def get_dependencies(self, task_id: TaskId) -> List[Dependency]:
        """Get all dependencies for a task."""
        result = shell(f'bd dep tree {task_id} --json')
        tree = json.loads(result)
        return parse_dependency_tree(tree)
    
    # ─────────────────────────────────────────────────────────
    # Ready Work (Wave 1)
    # ─────────────────────────────────────────────────────────
    
    def get_ready_tasks(self, filter: WorkFilter = None) -> List[Task]:
        """
        Get tasks ready for execution (no open blockers).
        
        This is effectively "Wave 1" - tasks that can start now.
        """
        cmd = 'bd ready --json'
        
        if filter:
            if filter.priority:
                cmd += f' --priority {filter.priority - 1}'
            if filter.assignee:
                cmd += f' --assignee {filter.assignee}'
            if filter.labels:
                cmd += f' --label {",".join(filter.labels)}'
        
        result = shell(cmd)
        issues = json.loads(result)
        
        return [self.get_task(issue['id']) for issue in issues]
    
    def get_blocked_tasks(self) -> List[Task]:
        """Get tasks that are blocked by dependencies."""
        result = shell('bd blocked --json')
        issues = json.loads(result)
        return [self.get_task(issue['id']) for issue in issues]
    
    # ─────────────────────────────────────────────────────────
    # Gates (Human Checkpoints)
    # ─────────────────────────────────────────────────────────
    
    def create_human_gate(self, task_id: TaskId, description: str) -> GateId:
        """
        Create a human verification gate for a task.
        
        Maps to checkpoint:human-verify in our task types.
        The task will not be considered complete until the gate is closed.
        """
        # Gates in Beads are typically created as part of molecules/formulas
        # For ad-hoc gates, we create a blocking issue
        gate_id = shell(f'bd create "GATE: {description}" -t gate')
        shell(f'bd dep add {task_id} {gate_id} --type blocks')
        return gate_id
    
    def close_gate(self, gate_id: GateId, approved: bool, reason: str):
        """Close a human verification gate."""
        status = "closed" if approved else "wont_fix"
        shell(f'bd update {gate_id} --status {status}')
        shell(f'bd close {gate_id} --reason "{reason}"')
    
    # ─────────────────────────────────────────────────────────
    # Molecules (Plans/Epics)
    # ─────────────────────────────────────────────────────────
    
    def create_molecule(self, plan: Plan) -> MoleculeId:
        """
        Create a Beads molecule from a plan.
        
        Molecules are epics with children - perfect for our plans.
        """
        # Create the epic (molecule parent)
        epic_id = shell(
            f'bd create "{plan.goal}" -t epic -p 0 '
            f'--notes "PTF_PLAN:\n{serialize_plan_metadata(plan)}"'
        )
        
        # Create child issues for each task
        for task in plan.tasks:
            task_id = self.create_task(task)
            # Parent-child relationship created automatically if --parent used
            # Otherwise, add explicitly:
            shell(f'bd dep add {task_id} {epic_id} --type parent-child')
        
        # Add inter-task dependencies
        for dep in plan.dependencies:
            self.add_dependency(dep.from_id, dep.to_id, dep.type)
        
        return epic_id
    
    def get_molecule_status(self, molecule_id: MoleculeId) -> MoleculeStatus:
        """Get status of a molecule (plan) and its children."""
        result = shell(f'bd dep tree {molecule_id} --json')
        tree = json.loads(result)
        return parse_molecule_status(tree)
    
    # ─────────────────────────────────────────────────────────
    # Execution State
    # ─────────────────────────────────────────────────────────
    
    def save_execution_state(self, state: ExecutionState):
        """
        Save execution state.
        
        With Beads, most state is implicit in issue statuses.
        We store framework-specific state (wave assignments, etc.)
        in a special tracking issue or in notes.
        """
        # Find or create the execution tracking issue
        tracker_id = self._get_or_create_execution_tracker()
        
        # Store state as YAML in notes
        state_yaml = serialize_execution_state(state)
        shell(f'bd update {tracker_id} --notes "{state_yaml}"')
    
    def load_execution_state(self) -> ExecutionState:
        """Load execution state from Beads."""
        tracker_id = self._get_execution_tracker()
        if not tracker_id:
            return ExecutionState(status="pending")
        
        result = shell(f'bd show {tracker_id} --json')
        issue = json.loads(result)
        return deserialize_execution_state(issue['notes'])
    
    # ─────────────────────────────────────────────────────────
    # Events and History
    # ─────────────────────────────────────────────────────────
    
    def log_event(self, event: Event):
        """
        Log an event.
        
        Beads tracks events implicitly via issue updates and git history.
        For explicit event logging, we append to issue notes.
        """
        if event.task_id:
            timestamp = event.timestamp.isoformat()
            note = f"[{timestamp}] {event.type}: {event.description}"
            shell(f'bd update {event.task_id} --notes "{note}"')
    
    def get_events(self, filter: EventFilter) -> List[Event]:
        """
        Get events from Beads.
        
        Events are reconstructed from issue history and git log.
        """
        # For task-specific events, parse issue audit trail
        if filter.task_id:
            result = shell(f'bd show {filter.task_id} --json')
            issue = json.loads(result)
            return parse_events_from_issue(issue)
        
        # For global events, would need to query git history
        # This is a limitation of the Beads backend
        return []
```

#### Synthesis: How They Work Together

When using the Beads backend, the framework workflow becomes:

```
┌─────────────────────────────────────────────────────────────────┐
│                     User Provides Goal                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Framework: Decomposition                           │
│                                                                 │
│  Goal → Subgoals → Recursive breakdown → Atomic tasks           │
│  (This is framework logic - Beads doesn't do decomposition)     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Framework: Dependency Analysis                     │
│                                                                 │
│  Infer dependencies from task inputs/outputs                    │
│  Compute wave structure (full DAG analysis)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Beads Backend: Write Plan                          │
│                                                                 │
│  bd create "Goal" -t epic           → Create molecule           │
│  bd create "Task 1" --parent epic   → Create child issues       │
│  bd dep add task-2 task-1           → Add dependencies          │
│                                                                 │
│  Tasks stored with PTF metadata in notes                        │
│  Wave assignments stored in execution tracker issue             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Framework: Orchestration Loop                      │
│                                                                 │
│  while not complete:                                            │
│      ready_tasks = beads.get_ready_tasks()  # bd ready          │
│                                                                 │
│      for task in ready_tasks (parallel):                        │
│          # Ralph-style execution with fresh context             │
│          result = execute_task_ralph_loop(task)                 │
│                                                                 │
│          if result.success:                                     │
│              beads.close_task(task.id, result.reason)           │
│          else:                                                  │
│              handle_failure(task, result)                       │
│                                                                 │
│      # Beads automatically updates ready work                   │
│      # Next iteration picks up newly unblocked tasks            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              Beads: Automatic Git Sync                          │
│                                                                 │
│  Daemon watches for changes                                     │
│  JSONL updated, committed to git                                │
│  Full audit trail preserved                                     │
│  Other agents/sessions see updates                              │
└─────────────────────────────────────────────────────────────────┘
```

**Key Integration Points:**

1. **Decomposition outputs to Beads** — Framework creates issues via `bd create`, storing full task definitions in issue notes with a `PTF_TASK_DEF:` marker

2. **Dependencies flow to Beads** — Framework's inferred dependencies become `bd dep add` calls with appropriate types

3. **Wave computation is framework-side** — We pre-compute full wave structure; Beads' `bd ready` gives us Wave 1 dynamically

4. **Execution uses Beads for state** — Task status updates via `bd update`, completions via `bd close`

5. **Human checkpoints become Gates** — `checkpoint:human-verify` tasks create Beads gates that block until human closes them

6. **Git sync is automatic** — Beads daemon handles persistence; framework doesn't manage git directly

7. **Multi-agent coordination** — Hash-based IDs and merge driver handle parallel agent work automatically

#### Benefits of Beads Integration

| Benefit | How Beads Provides It |
|---------|----------------------|
| **Git-native persistence** | JSONL format, automatic commits, merge driver |
| **Multi-agent safety** | Hash-based IDs prevent collisions |
| **Resume across sessions** | `bd prime` restores context |
| **Human coordination** | Gates for approvals and checkpoints |
| **Audit trail** | Git history + issue audit trail |
| **Ecosystem** | MCP server, Claude plugin, community tools |
| **Battle-tested** | Production use by Yegge and community |

#### Limitations and Trade-offs

| Consideration | Impact |
|---------------|--------|
| **Additional dependency** | Requires `bd` CLI installed |
| **Schema divergence** | Our task format stored in notes, not native |
| **Event log fidelity** | Less structured than our JSONL event log |
| **Wave pre-computation** | Beads computes incrementally; we'd maintain parallel tracking |
| **Verification criteria** | Stored in notes, not native to Beads |

#### Configuration

```yaml
# .orchestrator/config.yaml

backend:
  type: beads                    # or "file" for default
  
  beads:
    use_daemon: true             # Use bd daemon for sync
    molecule_prefix: "ptf"       # Prefix for PTF-created issues
    store_task_defs: true        # Store full task YAML in notes
    create_gates_for_checkpoints: true  # Use Beads gates for human checkpoints
    
    # Status mapping
    status_map:
      pending: open
      ready: open
      running: in_progress
      completed: closed
      failed: open              # Failed tasks stay open for retry
      blocked: open             # Blocked by dependencies
```

### 11.8 Future Integration Possibilities

While the Claude Code plugin is the v1 implementation, the framework's patterns could be adapted to:

- **Other agent IDEs** (Cursor, Windsurf) via their configuration systems
- **Custom orchestration tools** that implement the patterns using LLM APIs
- **MCP servers** that expose framework capabilities to any MCP client

These would be separate projects that consume the framework's concepts, not part of the core framework.

---

## 12. Implementation Roadmap

The v1 implementation is a Claude Code plugin. Development proceeds in phases:

### 12.1 Phase 1: Foundation

**Goal:** Establish core schemas and skill documentation

**Deliverables:**
- Task, Artifact, Dependency, Wave YAML schemas
- SKILL.md with framework concepts
- Example files demonstrating formats
- Plugin directory structure

**Success Criteria:**
- Schemas validate correctly
- SKILL.md is comprehensive and usable
- Example task/plan files are valid

### 12.2 Phase 2: Decomposition Commands

**Goal:** Implement decomposition slash commands and subagent

**Deliverables:**
- `/ptf:init` command
- `/ptf:decompose` command
- Decomposer subagent (all 5 steps)
- Goal analysis, subgoal identification, recursive decomposition
- Decomposition validation

**Success Criteria:**
- Can take natural language goal
- Produces valid atomic tasks
- Handles recursive breakdown correctly
- Validates coverage and no overlap
- Writes correct state files

### 12.3 Phase 3: Dependency Analysis

**Goal:** Implement dependency inference and wave computation

**Deliverables:**
- Dependency analyzer subagent
- Multi-pass inference algorithm implementation
- Cycle detection
- Wave computation
- `/ptf:plan` command for human-readable output

**Success Criteria:**
- Correctly infers artifact dependencies
- Detects and reports cycles
- Computes valid wave assignments
- Generates readable plan document

### 12.4 Phase 4: Execution Engine

**Goal:** Implement orchestrator and task execution

**Deliverables:**
- Orchestrator subagent
- Task executor subagent
- `/ptf:execute` command (single wave)
- `/ptf:execute-all` command (all waves)
- Parallel subagent dispatch
- Result collection

**Success Criteria:**
- Executes tasks in correct order
- Respects wave boundaries
- Parallel execution within waves works
- Results are collected and recorded

### 12.5 Phase 5: State and Verification

**Goal:** Implement state persistence and verification

**Deliverables:**
- State file writers for all schemas
- Verifier subagent
- `/ptf:status` command
- `/ptf:verify` command
- Event logging hooks
- Checkpoint hooks

**Success Criteria:**
- All state survives interruption
- `/ptf:status` accurately reports progress
- Verification correctly identifies pass/fail
- Event log captures complete history

### 12.6 Phase 6: Failure Handling and Resume

**Goal:** Implement failure recovery and resume capability

**Deliverables:**
- Failure handling logic in orchestrator
- `/ptf:resume` command
- `/ptf:retry` command
- Failure record creation
- Cascade failure handling

**Success Criteria:**
- Failed tasks are handled per policy
- Can resume from any interruption point
- Retry works correctly
- Dependent tasks are blocked appropriately

### 12.7 Phase 7: Domain Adapters

**Goal:** Create initial domain adapters

**Deliverables:**
- Software development adapter (complete)
- Research adapter (complete)
- Template adapter for customization
- Adapter loading and integration

**Success Criteria:**
- Adapters customize decomposition behavior
- Verification strategies work for each artifact type
- Easy to create new adapters from template

### 12.8 Phase 8: Polish and Documentation

**Goal:** Production-ready plugin

**Deliverables:**
- Complete plugin README
- Usage documentation
- Example projects (software, research)
- Error messages and edge case handling
- Performance optimization

**Success Criteria:**
- Plugin installs cleanly
- Documentation covers all features
- Examples work end-to-end
- Handles edge cases gracefully

---

## Appendix A: Glossary

| Term | Definition |
|------|------------|
| **Artifact** | A file produced or consumed by tasks |
| **Atomicity** | Property of a task being indivisible |
| **Backend** | Storage implementation for framework state (file-based or Beads) |
| **Beads** | Git-backed graph issue tracker for AI agents (optional backend) |
| **Cascade Failure** | When a task failure causes dependent tasks to fail |
| **Completion Promise** | Explicit output phrase signaling task completion (Ralph pattern) |
| **Context Degradation** | Quality loss as LLM context window fills |
| **DAG** | Directed Acyclic Graph (the dependency structure) |
| **Decomposition** | Breaking a goal into atomic tasks |
| **Dependency** | Relationship requiring one task to complete before another |
| **Domain Adapter** | Plugin providing domain-specific knowledge |
| **Fresh Context** | Empty LLM context window (peak quality) |
| **Gate** | Async wait condition in Beads (human approval, timer, CI) |
| **Goal** | High-level user intent to be accomplished |
| **Guardrails** | Constraints added to prevent repeated failure modes |
| **HTA** | Hierarchical Task Analysis (cognitive science method) |
| **Marge Pattern** | Constraints that the agent must respect (Ralph ecosystem) |
| **Molecule** | Beads term for epic with workflow semantics |
| **Near-Decomposability** | Systems theory concept about weak inter-component coupling |
| **Operation** | Atomic unit of work (leaf node in HTA) |
| **Orchestrator** | Engine that manages plan execution |
| **Plan** | Complete specification for executing a goal |
| **Ralph Loop** | Execution pattern: repeat with fresh context until verified |
| **Skinner Pattern** | Structural harness enforcing deterministic lanes (Ralph ecosystem) |
| **Subgoal** | Intermediate decomposition unit (may need further breakdown) |
| **Task** | Atomic, executable unit of work |
| **Topological Sort** | Algorithm for ordering DAG nodes by dependencies |
| **Verification** | Checking that task output meets criteria |
| **Wave** | Set of tasks that can execute in parallel |
| **WBS** | Work Breakdown Structure (project management method) |
| **Wisp** | Beads term for ephemeral/temporary issue |

---

## Appendix B: References

### Theoretical Foundations

- Miller, G.A. (1956). "The Magical Number Seven, Plus or Minus Two"
- Simon, H.A. (1962). "The Architecture of Complexity"
- Annett, J. & Duncan, K.D. (1967). "Task Analysis and Training Design"
- Evans, E. (2003). "Domain-Driven Design"

### Related Systems and Techniques

- **GSD (Get Shit Done)** - https://github.com/glittercowboy/get-shit-done - Spec-driven development system for Claude Code by TÂCHES
- **Ralph Wiggum Loop** - Stateless resampling technique by Geoffrey Huntley (2025) - Execution primitive using temporal iteration with fresh context
- **Gas Town** - Multi-agent orchestrator by Steve Yegge - "Kubernetes for agents"
- **Beads** - https://github.com/steveyegge/beads - Distributed, git-backed graph issue tracker for AI agents by Steve Yegge. Provides dependency-aware persistent memory with auto-sync, hash-based IDs, and Claude Code integration. Excellent candidate for backend adapter.
- SpecKit - Spec-driven development for Claude
- BMAD - Build Measure Analyze Decide framework
- Apache Airflow - Workflow orchestration
- Bazel - Build system with DAG execution

### LLM Agent Patterns

- Anthropic Claude Documentation
- LangChain Agent Patterns
- AutoGPT Architecture

---

## Appendix C: Version History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0-draft | 2025-01-17 | Initial founding document |

---

*This document represents the founding specification for the Parallel Task Framework. It captures the vision, principles, architecture, and detailed design that will guide implementation. As the project evolves, this document should be updated to reflect learnings and refinements while preserving the core philosophy.*
