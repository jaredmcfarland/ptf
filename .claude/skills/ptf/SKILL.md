---
name: ptf
description: Parallel Task Framework for decomposing complex goals into atomic tasks, computing dependency graphs, and orchestrating parallel execution with fresh context per task. Use when working with PTF commands, building decomposition plans, or understanding wave-based execution.
---

# Parallel Task Framework

The Parallel Task Framework (PTF) enables domain-agnostic decomposition of complex goals into atomic tasks, dependency analysis, and wave-based parallel execution. Each task executes with fresh context for optimal LLM quality.

## Core Concepts

### Task

The atomic unit of work in PTF. A task:

- Has a unique ID and human-readable name
- Declares explicit inputs (files/artifacts it needs to read)
- Declares explicit outputs (files/artifacts it will produce)
- Has verification criteria (how to confirm completion)
- Fits comfortably in a fresh agent context (target 10-30% of context window)
- Includes optional context_notes for execution guidance

A well-formed task is self-contained: given its inputs, an agent can complete it without additional context.

### Wave

A set of tasks with no interdependencies. All tasks in a wave:

- Can execute in parallel (no coordination needed)
- Have no data flow between them
- Complete before the next wave starts

Waves are computed via topological sort of the dependency graph. Tasks in the same wave are truly independent—no distributed state problems.

### Artifact

A file produced or consumed by tasks. Artifacts enable:

- **Dependency inference**: If Task B reads a file that Task A produces, B depends on A
- **Verification**: Output exists, has correct content, passes validation
- **Resume**: Know which artifacts exist, skip completed tasks

Artifact types: `source-code`, `config`, `test`, `migration`, `documentation`, `data`, `schema`

### Dependency

A relationship where one task must complete before another can start.

**Dependency types:**

| Type | Description | Confidence |
|------|-------------|------------|
| artifact | Task B reads output of Task A | High |
| semantic | Task B references concepts from Task A | Medium |
| resource | Tasks compete for same resource | Medium |
| implicit | Logical ordering (e.g., init before configure) | Low |

Dependencies are inferred via multi-pass analysis: artifact matching, type inference, semantic analysis, heuristics, resource detection.

### Plan

A complete execution blueprint containing:

- Goal analysis (objective, scope, constraints, success criteria)
- Task definitions (all atomic tasks)
- Dependencies (relationships between tasks)
- Waves (computed parallel execution groups)
- Execution policy (max parallel, failure strategy, checkpoints)

### Context Budget

Token limits to maintain LLM quality. Based on research showing quality degrades as context fills:

| Context Usage | Quality Level | Recommendation |
|--------------|---------------|----------------|
| 0-30% | Peak quality | Target zone |
| 30-50% | Good quality | Acceptable |
| 50-70% | Degrading | Caution |
| 70%+ | Poor quality | Avoid |

**Heuristic**: ~4 characters = 1 token (varies by content type)

**Context window targets:**

| Model | Window | Target (30%) | Max Safe (50%) |
|-------|--------|--------------|----------------|
| Claude Sonnet | 200K | 60K | 100K |
| Claude Sonnet (Enterprise) | 500K | 150K | 250K |

## File Locations

PTF stores runtime state in `.orchestrator/` (created per project):

| Path | Purpose |
|------|---------|
| `.orchestrator/config.yaml` | Project configuration |
| `.orchestrator/decomposition/` | Goal analysis, tasks, dependencies |
| `.orchestrator/decomposition/goal.yaml` | Original goal and analysis |
| `.orchestrator/decomposition/tasks/` | Individual task definitions |
| `.orchestrator/decomposition/dependencies.yaml` | Dependency graph |
| `.orchestrator/decomposition/waves.yaml` | Computed wave assignments |
| `.orchestrator/execution/` | Wave status, task results |
| `.orchestrator/execution/state.yaml` | Current execution state |
| `.orchestrator/execution/results/` | Task completion records |
| `.orchestrator/artifacts/` | Artifact manifest and checksums |
| `.orchestrator/artifacts/manifest.yaml` | All artifact metadata |
| `.orchestrator/history/` | Event log |
| `.orchestrator/history/events.jsonl` | Timestamped event stream |

## Commands

PTF provides slash commands for each stage of the workflow:

| Command | Purpose |
|---------|---------|
| `/ptf:init [goal]` | Initialize project, run goal analysis |
| `/ptf:decompose` | Run full decomposition (goal → subgoals → tasks) |
| `/ptf:plan` | Generate human-readable execution plan |
| `/ptf:execute [wave]` | Execute single wave |
| `/ptf:execute-all` | Execute all waves sequentially |
| `/ptf:status` | Show current execution state |
| `/ptf:verify [task]` | Run verification on specific task |
| `/ptf:resume` | Resume from interruption point |
| `/ptf:retry [task]` | Retry a failed task |
| `/ptf:abort` | Stop execution, preserve state |

**Typical workflow:**

1. `/ptf:init "Build user authentication"` - Analyze goal
2. `/ptf:decompose` - Break into atomic tasks
3. `/ptf:plan` - Review generated plan
4. `/ptf:execute-all` - Run all waves
5. `/ptf:status` - Monitor progress

## Schema Reference

PTF schemas are defined in `schemas/` using JSON Schema Draft 7:

### Task Schema (`task.schema.yaml`)

```yaml
id: string          # Unique identifier (lowercase, alphanumeric, hyphens)
name: string        # Human-readable name (max 100 chars)
description: string # Complete execution instructions
inputs: array       # Input artifacts/files
  - path: string
    description: string
    required: boolean
outputs: array      # Output artifacts/files (min 1)
  - path: string
    type: enum [source-code, config, test, migration, documentation, data]
verify: array       # Verification steps (min 1)
  - type: enum [exists, contains, runs, syntax, custom]
    target: string
    expected: any
context_notes: string     # Additional guidance
context_budget: object    # Token estimates
  estimated_input_tokens: integer
  max_context_percentage: number (default 30)
on_failure: object        # Failure handling
  strategy: enum [retry, skip, escalate]
  max_attempts: integer (default 3)
```

### Artifact Schema (`artifact.schema.yaml`)

```yaml
path: string        # File path relative to project root
type: enum          # Artifact type for verification routing
produced_by: string # Task ID that created this artifact
produced_at: datetime
consumed_by: array  # Task IDs that read this artifact
checksum: string    # SHA-256 hash
verified: boolean   # Passed verification
```

### Dependency Schema (`dependency.schema.yaml`)

```yaml
from: string        # Prerequisite task ID
to: string          # Dependent task ID
type: enum          # artifact, semantic, resource, implicit
confidence: enum    # high, medium, low
reason: string      # Human-readable explanation
```

### Wave Schema (`wave.schema.yaml`)

```yaml
number: integer     # Wave sequence (1-based)
tasks: array        # Task IDs in this wave (min 1)
status: enum        # pending, running, completed, partial, failed
depends_on_waves: array  # Wave numbers that must complete first
started_at: datetime
completed_at: datetime
```

## Subagents

PTF uses specialized subagents for different responsibilities:

| Agent | Responsibility |
|-------|----------------|
| `ptf-decomposer` | Goal analysis, subgoal identification, recursive task breakdown |
| `ptf-dependency-analyzer` | Multi-pass dependency inference, cycle detection, wave computation |
| `ptf-task-executor` | Execute single task with fresh context, produce outputs |
| `ptf-verifier` | Independent verification of task outputs against criteria |
| `ptf-orchestrator` | Coordinate wave-by-wave execution, dispatch subagents |

**Execution flow:**

1. Orchestrator dispatches all Wave N tasks in parallel
2. Task executors run independently with fresh context
3. Verifier checks each task's outputs
4. On success: mark complete, proceed to Wave N+1
5. On failure: apply failure policy (retry/skip/escalate)

## Domain Adapters

Adapters customize PTF for specific domains. Located in `adapters/`:

- `software-development.yaml` - Code, tests, configs, deployments
- `research.yaml` - Literature review, experiments, analysis, writing
- `template.yaml` - Base for custom adapters

Adapters provide:

- **Questioning patterns**: Domain-relevant goal clarification
- **Decomposition heuristics**: How to break down domain tasks
- **Verification templates**: Domain-appropriate checks
- **Constitution principles**: Immutable domain constraints

## Verification Types

Tasks specify verification criteria. PTF supports:

| Type | Description | Example |
|------|-------------|---------|
| `exists` | File exists at path | `target: src/auth.ts` |
| `contains` | File contains pattern | `target: src/auth.ts, expected: "export function login"` |
| `runs` | Command exits 0 | `target: "npm test", expected: 0` |
| `syntax` | File is syntactically valid | `target: prisma/schema.prisma` |
| `custom` | Custom verification script | `target: "./scripts/verify-auth.sh"` |

## Failure Handling

PTF provides configurable failure strategies:

| Strategy | Behavior |
|----------|----------|
| `retry` | Retry task up to max_attempts (default 3) |
| `skip` | Mark failed, continue with dependents blocked |
| `escalate` | Stop execution, surface for human decision |
| `replan` | Trigger re-decomposition from current state |

**Cascade behavior:** When a task fails and is not retried successfully, all dependent tasks are marked `blocked`.

## Best Practices

### Task Design

- **One output focus**: Each task should have a clear primary output
- **Explicit inputs**: List all files the task needs to read
- **Specific verification**: Avoid "looks good" — use concrete checks
- **Context-appropriate**: Keep estimated tokens under 30% of window

### Dependency Declaration

- **Trust artifact inference**: High confidence for input/output matches
- **Document implicit deps**: Add `reason` for non-obvious dependencies
- **Avoid cycles**: PTF will detect and report cycles

### Wave Optimization

- **Balance parallelism**: Many small tasks in early waves feed later work
- **Critical path awareness**: Some task chains cannot be parallelized
- **Resource contention**: Tasks modifying same file should be sequenced
