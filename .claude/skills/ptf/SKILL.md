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
- Fits in fresh agent context (target 10-30% of context window)

A well-formed task is self-contained: given its inputs, an agent can complete it without additional context.

### Wave

A set of tasks with no interdependencies. All tasks in a wave:
- Can execute in parallel (no coordination needed)
- Have no data flow between them
- Complete before the next wave starts

Waves are computed via topological sort. Tasks in the same wave are truly independent.

### Artifact

A file produced or consumed by tasks. Artifacts enable:
- **Dependency inference**: If Task B reads a file that Task A produces, B depends on A
- **Verification**: Output exists, has correct content, passes validation
- **Resume**: Know which artifacts exist, skip completed tasks

Artifact types: `source-code`, `config`, `test`, `migration`, `documentation`, `data`, `schema`

### Dependency

A relationship where one task must complete before another can start.

| Type | Description | Confidence |
|------|-------------|------------|
| artifact | Task B reads output of Task A | High |
| semantic | Task B references concepts from Task A | Medium |
| resource | Tasks compete for same resource | Medium |
| implicit | Logical ordering (e.g., init before configure) | Low |

### Plan

A complete execution blueprint containing:
- Goal analysis (objective, scope, constraints, success criteria)
- Task definitions (all atomic tasks)
- Dependencies (relationships between tasks)
- Waves (computed parallel execution groups)
- Execution policy (max parallel, failure strategy, checkpoints)

### Context Budget

Token limits to maintain LLM quality:

| Context Usage | Quality Level | Recommendation |
|--------------|---------------|----------------|
| 0-30% | Peak quality | Target zone |
| 30-50% | Good quality | Acceptable |
| 50-70% | Degrading | Caution |
| 70%+ | Poor quality | Avoid |

**Heuristic**: ~4 characters = 1 token. Target 60K tokens (30% of 200K) per task.

## Execution Concepts

### Fresh Context Dispatch

Each task runs in complete isolation with only its declared inputs loaded. No accumulated state from prior tasks means:
- No context degradation from pollution
- No implicit assumptions from "what happened before"
- Tasks are reproducible and parallelizable

The executor reads ONLY files in the task's input declaration.

### Completion Promise

The executor signals task outcome with explicit phrases:
- `VERIFICATION PASSED`: All outputs exist, all verifications pass
- `BLOCKED: [reason]`: Cannot proceed, with specific category

Reason categories: `missing_input`, `verification_failed`, `execution_error`, `max_iterations`

### Ralph-Style Iteration

Tasks execute with bounded retries:
1. Orchestrator spawns executor with fresh context
2. Executor attempts task, runs verification
3. If verification fails: orchestrator spawns NEW executor (fresh context)
4. Repeat until verification passes or max_iterations reached

**Key insight**: Each iteration is independent. No memory of prior attempts.

### Max Iterations

Bounded retry attempts prevent infinite loops. Default: 10 iterations.
After exhaustion: task escalates to human or marks as blocked.

## Verification Concepts

### Independent Verification

Verification runs in a separate context from execution. The verifier:
- Does not trust executor claims
- Loads only task definition and expected outputs
- Runs all declared verification checks
- Reports pass/fail with details

Why independent? Executor may hallucinate success. Fresh context prevents bias.

### Verification Types

| Type | What it checks | Example |
|------|----------------|---------|
| exists | File exists at path | `[ -f "src/auth.ts" ]` |
| contains | File contains pattern | `grep -q "export function" file` |
| runs | Command exits successfully | `npm test -- auth.test.ts` |
| syntax | File parses correctly | `npx tsc --noEmit file.ts` |
| custom | User-defined check | Any command returning 0 on success |

### Multi-Modal Verification

Combine checks for comprehensive validation:

```yaml
verify:
  - type: exists
    target: src/auth.ts
  - type: syntax
    target: src/auth.ts
  - type: contains
    target: src/auth.ts
    expected: "export function authenticate"
  - type: runs
    target: "npm test -- auth"
```

Execution order: exists -> syntax -> contains -> runs -> custom (fail-fast).

### Verification Results

Results are recorded in task state:

```yaml
verification:
  status: passed | failed
  results:
    - type: exists
      passed: true
      message: "File exists"
```

Use `/ptf:verify task-id` to run verification manually.

## File Locations

PTF stores runtime state in `.orchestrator/`:

| Path | Purpose |
|------|---------|
| `.orchestrator/config.yaml` | Project configuration |
| `.orchestrator/decomposition/` | Goal analysis, tasks, dependencies |
| `.orchestrator/state/` | Execution state, task results |
| `.orchestrator/artifacts/manifest.yaml` | All artifact metadata |
| `.orchestrator/history/events.jsonl` | Timestamped event stream |

## Commands

| Command | Purpose |
|---------|---------|
| `/ptf:init [goal]` | Initialize project, run goal analysis |
| `/ptf:decompose` | Run full decomposition (goal -> subgoals -> tasks) |
| `/ptf:plan` | Generate execution plan with dependencies and waves |
| `/ptf:execute [wave]` | Execute single wave or next pending |
| `/ptf:execute-all` | Execute all waves with automatic progression |
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

## Subagents

| Agent | Responsibility |
|-------|----------------|
| `ptf-decomposer` | Goal analysis, subgoal identification, recursive task breakdown |
| `ptf-dependency-analyzer` | Multi-pass dependency inference, cycle detection, wave computation |
| `ptf-executor` | Execute single task with fresh context, verify outputs |
| `ptf-verifier` | Independently verify task outputs against criteria |
| `ptf-state-manager` | Checkpoint operations, event logging, artifact tracking |
| `ptf-orchestrator` | Coordinate wave-by-wave execution, dispatch subagents |

## Dependency Analysis

PTF automatically infers task dependencies through multi-pass analysis:

1. **Artifact Matching (HIGH)**: Exact input/output path matches
2. **Type/Pattern Matching (MEDIUM)**: Glob patterns match outputs to inputs
3. **Semantic Analysis (MEDIUM)**: LLM finds implicit references in descriptions
4. **Domain Heuristics (LOW)**: Adapter-provided patterns
5. **Resource Conflicts (HIGH)**: Tasks modifying same file serialized

### Wave Computation

Kahn's algorithm groups tasks into parallel execution waves:
- Wave 1: Tasks with no dependencies
- Wave N: Tasks whose dependencies completed in wave N-1

### Cycle Detection

Tarjan's algorithm detects circular dependencies. Reports exact tasks involved and suggests which dependency to break (lowest confidence).

## Verification Types

| Type | Description | Example |
|------|-------------|---------|
| `exists` | File exists at path | `target: src/auth.ts` |
| `contains` | File contains pattern | `target: src/auth.ts, expected: "export function"` |
| `runs` | Command exits 0 | `target: "npm test"` |
| `syntax` | File is syntactically valid | `target: prisma/schema.prisma` |
| `custom` | Custom verification script | `target: "./scripts/verify.sh"` |

## Failure Handling

PTF provides graceful failure recovery instead of terminating on first error.

### Failure Strategies

Each task can define an `on_failure` policy:

| Strategy | Behavior | Use When |
|----------|----------|----------|
| **retry** | Retry with exponential backoff up to max_attempts | Transient failures, flaky operations |
| **skip** | Mark task skipped, continue execution | Non-critical tasks, optional features |
| **escalate** | Pause execution, present options to human | Critical failures, need human decision |
| **replan** | Re-decompose portion of plan | Wrong approach, need different breakdown |

### Backoff Configuration

Retry strategy supports configurable backoff:

```yaml
on_failure:
  strategy: retry
  max_attempts: 3
  backoff_type: exponential  # none | linear | exponential
  backoff_base_seconds: 2
  backoff_max_seconds: 60
  final_fallback: escalate   # escalate | skip (after retries exhausted)
```

| Backoff Type | Attempt 1 | Attempt 2 | Attempt 3 | Attempt 4 |
|--------------|-----------|-----------|-----------|-----------|
| none | 0s | 0s | 0s | 0s |
| linear (base=2) | 2s | 4s | 6s | 8s |
| exponential (base=2) | 2s | 4s | 8s | 16s |

### Cascade Policy

When a task fails, its dependents are affected:

```yaml
on_failure:
  propagate_failure: true   # Default: block dependents
```

- **propagate_failure: true** (default): Dependent tasks are marked "blocked"
- **propagate_failure: false**: Dependents attempt anyway (will fail on missing input)
- **dependency.required: false**: Optional dependency - dependent can proceed without

### Failure Records

Full debugging context is saved to `.orchestrator/failures/`:

```
.orchestrator/failures/
  auth-service-attempt-1.yaml
  auth-service-attempt-2.yaml
  data-migration-attempt-1.yaml
```

Each record contains:
- Task identification (id, name, wave)
- Timing (started, failed, duration)
- Error details (type, category, message)
- Context (inputs loaded, files written)
- Recovery action taken

### Recovery Commands

| Command | Purpose |
|---------|---------|
| `/ptf:retry task-id` | Reset task state, retry with fresh context |
| `/ptf:abort` | Stop execution cleanly, preserve state |
| `/ptf:resume` | Continue from last checkpoint |
| `/ptf:status` | View current state including failures |

### Human Escalation

When escalation triggers, options are presented: retry, skip, abort, or replan. Review `.orchestrator/failures/` before choosing.

**Best practices**: Set max_attempts 2-3 for quick failures, 5-10 for flaky ops. Use skip for non-critical tasks. Escalate by default for unknowns.

## Domain Adapters

Adapters customize PTF for specific domains (`adapters/`):
- `software-development.yaml` - Code, tests, configs, deployments
- `research.yaml` - Literature review, experiments, analysis, writing
- `template.yaml` - Base for custom adapters

## Key Terms

| Term | Definition |
|------|------------|
| Executor | Subagent that runs a single task with fresh context |
| Completion promise | Explicit phrase signaling task success or failure |
| Ralph loop | Iteration pattern: fresh context execution until verified |
| Wave | Set of independent tasks that can run in parallel |
| Artifact | File produced/consumed by tasks, enables dependency inference |
| Checkpoint | State save at wave boundary for resume capability |
| Verification | Independent check that task outputs meet declared criteria |
| Verifier | Subagent that runs verification checks in fresh context |
| Multi-modal verification | Combining multiple check types (exists + contains + runs) |
