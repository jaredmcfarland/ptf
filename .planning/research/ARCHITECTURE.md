# Architecture Research

**Domain:** LLM Agent Orchestration and Task Decomposition Systems
**Researched:** 2025-01-18
**Confidence:** HIGH

## Standard Architecture

### System Overview

```
+------------------------------------------------------------------+
|                         USER INTERFACE                            |
|  +--------------+  +---------------+  +----------------------+    |
|  | Slash Cmds   |  | CLI/REPL      |  | Programmatic API     |    |
|  +--------------+  +---------------+  +----------------------+    |
+----------|-------------------|-------------------|-----------------+
           |                   |                   |
           v                   v                   v
+------------------------------------------------------------------+
|                       ORCHESTRATION LAYER                         |
|  +----------------+  +------------------+  +------------------+   |
|  | Goal Analyzer  |  | Plan Generator   |  | Execution Engine |   |
|  | (Decomposer)   |  | (DAG Builder)    |  | (Wave Executor)  |   |
|  +-------+--------+  +--------+---------+  +--------+---------+   |
|          |                    |                     |             |
|          v                    v                     v             |
|  +-----------------------------------------------------------+   |
|  |                    STATE MANAGER                           |   |
|  |  (Checkpoints, Progress, Artifacts, Event Log)            |   |
|  +-----------------------------------------------------------+   |
+----------|-------------------|-------------------|-----------------+
           |                   |                   |
           v                   v                   v
+------------------------------------------------------------------+
|                        AGENT LAYER                                |
|  +-------------+  +-------------+  +-------------+  +----------+  |
|  | Decomposer  |  | Dependency  |  | Task        |  | Verifier |  |
|  | Subagent    |  | Analyzer    |  | Executor    |  | Subagent |  |
|  +-------------+  +-------------+  +-------------+  +----------+  |
+----------|-------------------|-------------------|-----------------+
           |                   |                   |
           v                   v                   v
+------------------------------------------------------------------+
|                      DOMAIN LAYER                                 |
|  +------------------+  +------------------+  +------------------+ |
|  | Software Adapter |  | Research Adapter |  | Custom Adapters  | |
|  | (decomp rules,   |  | (decomp rules,   |  | (template-based) | |
|  |  verification)   |  |  verification)   |  |                  | |
|  +------------------+  +------------------+  +------------------+ |
+------------------------------------------------------------------+
           |                   |                   |
           v                   v                   v
+------------------------------------------------------------------+
|                     PERSISTENCE LAYER                             |
|  +------------------+  +------------------+  +------------------+ |
|  | .orchestrator/   |  | Artifact Files   |  | Event Log        | |
|  | (YAML state)     |  | (produced work)  |  | (JSONL history)  | |
|  +------------------+  +------------------+  +------------------+ |
+------------------------------------------------------------------+
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| **User Interface** | Accept commands, display progress, handle interrupts | Slash commands, CLI, SDK bindings |
| **Goal Analyzer (Decomposer)** | Transform goal into structured analysis, identify subgoals, produce atomic tasks | LLM-based reasoning with domain adapter heuristics |
| **Plan Generator (DAG Builder)** | Infer dependencies, detect cycles, compute execution waves | Multi-pass inference algorithm + topological sort |
| **Execution Engine (Wave Executor)** | Dispatch tasks, manage parallelism, handle failures, checkpoint state | State machine with event-driven transitions |
| **State Manager** | Persist progress, enable resume, track artifacts, log events | File-based (YAML/JSONL) or database-backed |
| **Subagents** | Execute specific functions with fresh context | Spawned via Task tool with scoped prompts |
| **Domain Adapters** | Customize decomposition, verification, dependency patterns per domain | Pluggable YAML + prompt templates |
| **Persistence Layer** | Durable storage for all state and artifacts | File system with structured directories |

## Recommended Project Structure

```
ptf/
+-- .claude/
|   +-- commands/ptf/           # Slash commands
|   |   +-- init.md             # /ptf:init - initialize project
|   |   +-- decompose.md        # /ptf:decompose - run decomposition
|   |   +-- plan.md             # /ptf:plan - generate readable plan
|   |   +-- execute.md          # /ptf:execute - run wave(s)
|   |   +-- status.md           # /ptf:status - show progress
|   |   +-- verify.md           # /ptf:verify - check task
|   |   +-- resume.md           # /ptf:resume - continue from interruption
|   |   +-- abort.md            # /ptf:abort - stop execution
|   |   +-- retry.md            # /ptf:retry - retry failed task
|   |
|   +-- agents/                 # Subagent definitions
|   |   +-- ptf-decomposer.md   # Goal analysis and task breakdown
|   |   +-- ptf-dependency-analyzer.md  # Dependency inference
|   |   +-- ptf-task-executor.md        # Single task execution
|   |   +-- ptf-verifier.md             # Artifact verification
|   |   +-- ptf-orchestrator.md         # Wave execution control
|   |
|   +-- skills/                 # Agent skills
|       +-- SKILL.md            # Framework knowledge for Claude
|
+-- adapters/                   # Domain adapters
|   +-- software-development.yaml
|   +-- research.yaml
|   +-- template.yaml           # For custom domains
|
+-- schemas/                    # YAML schema definitions
|   +-- task.schema.yaml
|   +-- dependency.schema.yaml
|   +-- wave.schema.yaml
|   +-- plan.schema.yaml
|   +-- adapter.schema.yaml
|
+-- hooks/                      # Event hooks (optional)
|   +-- post-task-complete.md
|   +-- pre-wave-start.md
|   +-- on-failure.md
|
+-- .orchestrator/              # Runtime state (created per project)
    +-- config.yaml             # Framework configuration
    +-- goal.md                 # Original goal (immutable)
    +-- decomposition/          # Decomposition outputs
    |   +-- analysis.yaml       # Step 1: Goal analysis
    |   +-- subgoals.yaml       # Step 2: Subgoal identification
    |   +-- tasks.yaml          # Step 3: Atomic tasks
    |   +-- validation.yaml     # Step 4: Validation results
    |   +-- graph.yaml          # Step 5: Dependencies + waves
    +-- plan.md                 # Human-readable plan
    +-- state/                  # Execution state
    |   +-- execution.yaml      # Master state
    |   +-- waves/              # Per-wave state
    |   +-- tasks/              # Per-task state
    +-- artifacts/              # Artifact tracking
    |   +-- manifest.yaml
    +-- history/                # Event history
    |   +-- events.jsonl
    |   +-- sessions/
    +-- failures/               # Failure records
```

### Structure Rationale

- **`.claude/commands/ptf/`:** Slash commands as user interface - follows Claude Code conventions, zero installation friction
- **`.claude/agents/`:** Subagents with focused responsibilities - each spawned with fresh context for peak quality
- **`adapters/`:** Domain-specific knowledge separate from core - enables extension without modifying framework
- **`schemas/`:** Explicit data contracts - enables validation, documentation, tooling
- **`.orchestrator/`:** Per-project state - everything survives interruption, enables resume from anywhere
- **Separation of decomposition/state/artifacts/history:** Each concern has clear ownership, no coupling

## Architectural Patterns

### Pattern 1: DAG-Based Task Graph

**What:** Represent work as a Directed Acyclic Graph where nodes are tasks and edges are dependencies. Compute execution waves via topological sort.

**When to use:** Any workflow with dependencies between steps that can be modeled as data flow.

**Trade-offs:**
- Pro: Enables automatic parallelization of independent tasks
- Pro: Clear visualization of execution plan
- Pro: Well-understood algorithms for scheduling
- Con: Requires upfront dependency analysis (not dynamic)
- Con: Cycles indicate decomposition problems, not features

**Example:**
```
Wave 1: [auth-schema, config-setup]     # No dependencies
           |              |
           v              v
Wave 2: [user-repo, session-repo]       # Depend on schema
              \          /
               \        /
                v      v
Wave 3:     [auth-service]              # Depends on repos
                   |
                   v
Wave 4:     [auth-tests]                # Depends on service
```

**Industry precedent:** Build systems (Make, Bazel), workflow engines (Airflow, Prefect), LangGraph, TDAG framework.

### Pattern 2: Fresh Context Execution (Ralph Pattern)

**What:** Each task executes in a completely fresh agent context, loading only declared inputs. Progress persists through files, not context window.

**When to use:** Any LLM-based execution where context degradation is a concern.

**Trade-offs:**
- Pro: Peak quality output (0-30% context usage)
- Pro: No accumulated confusion across tasks
- Pro: Failures are isolated
- Con: Some overhead per context switch
- Con: No "memory" across tasks except through artifacts

**Example:**
```python
def execute_task(task):
    agent = fresh_agent()               # Clean context
    context = load_task_inputs(task)    # Only declared deps
    result = agent.execute(task.description, context)
    verify(task.criteria, result)
    return result
```

**Industry precedent:** Ralph Wiggum Loop (Geoffrey Huntley), Claude Code's Task tool architecture.

### Pattern 3: Wave-Based Checkpointing

**What:** State is checkpointed at wave boundaries. All tasks in a wave are independent; wave completion is a safe pause point.

**When to use:** Long-running workflows that may be interrupted.

**Trade-offs:**
- Pro: Resume from any wave without re-execution
- Pro: Natural parallelism boundary
- Pro: Clear progress indication (wave N of M)
- Con: Must wait for all wave tasks before next wave
- Con: Unbalanced waves (1 slow task) can bottleneck

**Example:**
```yaml
# state/execution.yaml
status: running
current_wave: 2
waves_total: 5
wave_summary:
  1: completed
  2: running
  3: pending
```

**Industry precedent:** LangGraph checkpointing, AWS Step Functions state persistence.

### Pattern 4: Orchestrator-Worker with Event-Driven State

**What:** Central orchestrator maintains state machine, dispatches work to stateless workers (subagents), receives results via events.

**When to use:** Any multi-agent system with coordination requirements.

**Trade-offs:**
- Pro: Centralized control, easy to reason about
- Pro: Workers are simple, focused, replaceable
- Pro: Natural audit trail through event log
- Con: Orchestrator is single point of failure (mitigated by persistence)
- Con: Not fully decentralized (by design)

**Example:**
```
Orchestrator                    Workers
     |
     +-- dispatch(task-a) ---------> Agent 1
     +-- dispatch(task-b) ---------> Agent 2
     +-- dispatch(task-c) ---------> Agent 3
     |
     <-- result(task-a, success) ---+
     <-- result(task-b, success) ---+
     <-- result(task-c, failure) ---+
     |
     +-- checkpoint_wave()
     +-- handle_failure(task-c)
```

**Industry precedent:** Confluent's orchestrator-worker pattern, CrewAI's crew orchestration, AutoGen's coordinator pattern.

### Pattern 5: Domain Adapter Injection

**What:** Core framework is domain-agnostic. Domain-specific knowledge (decomposition rules, verification strategies, dependency patterns) loaded from pluggable adapters.

**When to use:** Framework that needs to support multiple domains without code changes.

**Trade-offs:**
- Pro: Single codebase, multiple applications
- Pro: Domain experts can contribute without framework knowledge
- Pro: Easy to extend to new domains
- Con: Adapter quality varies, affects results
- Con: Some domains may not fit adapter model

**Example:**
```yaml
# adapters/software-development.yaml
decomposition:
  subgoal_heuristics:
    - name: by-layer
      description: Split by architectural layer
    - name: by-feature
      description: Split by user-facing feature

artifacts:
  verification_strategies:
    typescript:
      - method: runs
        command: "npx tsc --noEmit {file}"
```

**Industry precedent:** CrewAI's role-based agents, LangChain's chain templates, Semantic Kernel's plugins.

## Data Flow

### Decomposition Flow

```
User Goal (natural language)
    |
    v
[Goal Analyzer Subagent]
    |
    +-> analysis.yaml (structured goal)
    |
    v
[Subgoal Identifier] + Domain Adapter
    |
    +-> subgoals.yaml (first-level breakdown)
    |
    v
[Recursive Decomposer] (loop until atomic)
    |
    +-> tasks.yaml (all atomic tasks)
    |
    v
[Validation Step]
    |
    +-> validation.yaml (issues flagged)
    |
    v
[Dependency Analyzer] (multi-pass inference)
    |
    +-> graph.yaml (dependencies + waves)
    |
    v
[Plan Generator]
    |
    +-> plan.md (human-readable)
```

### Execution Flow

```
Plan (graph.yaml)
    |
    v
[Orchestrator State Machine]
    |
    +-- for each wave:
    |       |
    |       +-- wait_for_dependencies()
    |       |
    |       +-- dispatch_parallel_tasks()
    |       |       |
    |       |       +-> [Task Executor 1] -> artifacts
    |       |       +-> [Task Executor 2] -> artifacts
    |       |       +-> [Task Executor 3] -> artifacts
    |       |
    |       +-- collect_results()
    |       |
    |       +-- [Verifier Subagent] -> verification results
    |       |
    |       +-- handle_failures()
    |       |
    |       +-- checkpoint_state()
    |       |
    |       +-- emit_events()
    |
    +-> execution.yaml (updated state)
    +-> events.jsonl (appended history)
    +-> artifacts/manifest.yaml (updated registry)
```

### Key Data Flows

1. **Goal -> Tasks:** Decomposition transforms unstructured goal into structured atomic tasks via recursive breakdown with domain adapter guidance

2. **Tasks -> Waves:** Dependency analyzer infers relationships from input/output declarations, computes parallel groups via topological sort

3. **Waves -> Execution:** Orchestrator dispatches each wave's tasks in parallel, waits for completion, checkpoints before next wave

4. **Tasks -> Artifacts:** Each task executor produces declared outputs, verifier confirms correctness, manifest tracks all produced artifacts

5. **Events -> History:** All state transitions append to JSONL log, enabling replay, debugging, and audit

## Claude Code Primitive Mapping

PTF components map directly to Claude Code's existing primitives:

| PTF Component | Claude Code Primitive | Notes |
|---------------|----------------------|-------|
| Slash commands | `.claude/commands/ptf/*.md` | Native support, zero friction |
| Subagents | `.claude/agents/*.md` + Task tool | Task tool spawns with fresh context |
| Skills | `.claude/skills/SKILL.md` | Framework knowledge for Claude |
| File-based state | Standard file I/O | No external dependencies |
| Parallel execution | Multiple Task tool calls | Up to 7 parallel subagents |
| Orchestrator | Main Claude Code session | Maintains control flow |
| Event hooks | Additional slash commands | Can be invoked from orchestrator |

**Critical Constraint:** Subagents cannot spawn subagents. This means:
- Orchestrator (main session) must dispatch all subagents
- Subagents return results, don't delegate further
- Hierarchical patterns require orchestrator mediation

```
Main Session (Orchestrator)
    |
    +-- Task tool --> Decomposer Subagent --> returns tasks
    |
    +-- Task tool --> Dependency Analyzer --> returns graph
    |
    +-- Task tool --> Task Executor 1 --> returns result
    +-- Task tool --> Task Executor 2 --> returns result
    +-- Task tool --> Task Executor 3 --> returns result
    |
    +-- Task tool --> Verifier Subagent --> returns verification
```

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| Single goal, <20 tasks | File-based state, single machine - PTF's sweet spot |
| Large goal, 20-100 tasks | Consider wave batching (max N parallel), memory for subagent prompts |
| Multi-project | Separate `.orchestrator/` per project, no cross-project state |
| Enterprise scale | Beyond PTF v1 scope - would need distributed state, external queue |

### Scaling Priorities

1. **First bottleneck: Context size for large decompositions**
   - Mitigation: Hierarchical decomposition (break into phases first, then tasks)
   - Mitigation: Streaming output during decomposition

2. **Second bottleneck: Parallel subagent limit (7 concurrent)**
   - Mitigation: Wave batching within parallel limit
   - Mitigation: Sequential fallback for large waves

3. **Third bottleneck: File I/O for large artifact sets**
   - Mitigation: Lazy loading of artifacts
   - Mitigation: Manifest-based verification without full reads

## Anti-Patterns

### Anti-Pattern 1: Monolithic Task Definition

**What people do:** Create tasks that are "implement entire feature" or "build complete module"

**Why it's wrong:** Tasks too large for fresh context, quality degrades, verification becomes vague

**Do this instead:** Decompose until tasks are single-file or single-concept. If a task needs more than 3 tightly-coupled inputs, it's probably too large.

### Anti-Pattern 2: Hidden Dependencies

**What people do:** Tasks reference work from other tasks through implicit knowledge rather than declared inputs

**Why it's wrong:** Breaks parallel execution (race conditions), prevents dependency inference, makes resume fragile

**Do this instead:** Explicitly declare all inputs in task definition. If Task B uses something Task A produces, list it in B's inputs.

### Anti-Pattern 3: Stateful Subagents

**What people do:** Try to maintain state across subagent invocations through context or memory

**Why it's wrong:** Violates fresh context principle, causes confusion accumulation, breaks isolation

**Do this instead:** All cross-task state flows through files. Subagents are stateless; the orchestrator and file system maintain state.

### Anti-Pattern 4: Deep Subagent Nesting

**What people do:** Design hierarchical agents where agents spawn agents spawn agents

**Why it's wrong:** Claude Code subagents cannot spawn subagents. The pattern fails at runtime.

**Do this instead:** Flat orchestration - main session dispatches all subagents directly, mediates any hierarchy needed.

### Anti-Pattern 5: Synchronous Blocking Within Waves

**What people do:** Have tasks within a wave wait on each other's partial results

**Why it's wrong:** Defeats parallelism, introduces coordination complexity, can cause deadlocks

**Do this instead:** Wave membership means complete independence. If tasks need to coordinate, they belong in different waves with explicit dependency.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Git | Hook for commits post-wave | Checkpoint commits |
| LLM APIs | Via Claude Code (abstracted) | No direct integration |
| File system | Native file I/O | Primary state mechanism |
| Terminal | Bash commands for verification | `runs` verification type |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Commands <-> Orchestrator | Slash command invocation | Commands trigger orchestrator functions |
| Orchestrator <-> Subagents | Task tool dispatch + return | Fresh context each dispatch |
| Orchestrator <-> State | File read/write | State persisted after each transition |
| Subagents <-> Domain Adapters | Adapter content in subagent prompt | Loaded at dispatch time |
| All <-> Event Log | Append-only writes | JSONL for audit trail |

## Build Order Implications

Based on component dependencies, recommended build sequence:

```
Phase 1: Foundation (no dependencies)
    +-- YAML schemas (task, artifact, dependency, wave)
    +-- SKILL.md with framework concepts
    +-- Directory structure (.orchestrator/, commands/, agents/)
    +-- Example files demonstrating formats

Phase 2: Decomposition (depends on Phase 1)
    +-- /ptf:init command (goal capture)
    +-- Decomposer subagent
    +-- Domain adapter structure + software adapter
    +-- /ptf:decompose command (orchestrates decomposition)

Phase 3: Dependency Analysis (depends on Phase 2)
    +-- Dependency analyzer subagent
    +-- Multi-pass inference algorithm
    +-- Cycle detection
    +-- Wave computation
    +-- /ptf:plan command (generate readable plan)

Phase 4: State Management (depends on Phase 1)
    +-- File-based state persistence
    +-- State schemas (execution, task, wave)
    +-- Checkpoint functions
    +-- Resume protocol

Phase 5: Execution Engine (depends on Phases 3 + 4)
    +-- Orchestrator subagent
    +-- Task executor subagent
    +-- Verifier subagent
    +-- /ptf:execute command
    +-- Parallel dispatch via Task tool

Phase 6: Failure Handling (depends on Phase 5)
    +-- Retry strategies
    +-- Failure records
    +-- Cascade handling
    +-- /ptf:resume, /ptf:retry, /ptf:abort commands

Phase 7: Polish (depends on all previous)
    +-- Research domain adapter
    +-- Complete documentation
    +-- Example projects
    +-- Error messages and edge cases
```

**Rationale:** Schemas and skills first (everything depends on data structures). Decomposition before dependency analysis (need tasks to analyze). State management can parallel with dependency work. Execution requires both plan and state. Failure handling layers on execution. Polish comes last.

## Sources

### Official Documentation & Frameworks
- [LangGraph Documentation](https://docs.langchain.com/) - Graph-based agent architecture, checkpointing, state management
- [CrewAI Documentation](https://docs.crewai.com/) - Role-based agents, crews, flows
- [Claude Code Subagents](https://code.claude.com/docs/en/sub-agents) - Task tool, agent spawning constraints
- [Microsoft AutoGen](https://github.com/microsoft/autogen) - Multi-agent orchestration patterns

### Research & Analysis
- [LLM Orchestration Frameworks 2025](https://orq.ai/blog/llm-orchestration) - Framework landscape survey
- [TDAG Framework](https://arxiv.org/abs/2402.10178) - Dynamic task decomposition and agent generation
- [Event-Driven Multi-Agent Systems](https://www.confluent.io/blog/event-driven-multi-agent-systems/) - Four design patterns
- [LLM Agent Architectures Core Components](https://futureagi.com/blogs/llm-agent-architectures-core-components) - Component analysis
- [Azure AI Agent Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) - Microsoft's pattern catalog

---
*Architecture research for: LLM Agent Orchestration and Task Decomposition Systems*
*Researched: 2025-01-18*
