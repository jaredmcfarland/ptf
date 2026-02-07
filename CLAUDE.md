# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PTF (Parallel Task Framework) is a Claude Code plugin that enables domain-agnostic decomposition of complex goals into atomic tasks, dependency analysis, and wave-based parallel execution. Each task executes with fresh context for optimal LLM quality.

This is a **specification and configuration repository** - not a traditional code project. It contains YAML schemas, agent definitions, slash commands, and domain adapters.

## Key Directories

| Directory | Purpose |
|-----------|---------|
| `.claude/agents/` | Subagent definitions (ptf-orchestrator, ptf-executor, ptf-decomposer, etc.) |
| `.claude/commands/ptf/` | Slash command implementations |
| `.claude/skills/ptf/` | The main PTF skill definition |
| `schemas/` | YAML schemas for tasks, plans, waves, artifacts, execution state |
| `adapters/` | Domain-specific adapters (software-development, research) |
| `.orchestrator/` | Runtime state directory (created per-project) |

## Commands

PTF is used via slash commands within Claude Code:

| Command | Purpose |
|---------|---------|
| `/ptf:init [goal]` | Initialize project, run goal analysis |
| `/ptf:decompose` | Break goal into atomic tasks |
| `/ptf:plan` | Generate execution plan with dependencies and waves |
| `/ptf:execute` | Execute single wave or next pending |
| `/ptf:execute-all` | Execute all waves with automatic progression (supports `--mode=teams`) |
| `/ptf:status` | Show current execution state |
| `/ptf:verify [task]` | Run verification on specific task |
| `/ptf:resume` | Resume from interruption point |
| `/ptf:retry [task]` | Retry a failed task |
| `/ptf:abort` | Stop execution, preserve state |

**Typical workflow:** `init` → `decompose` → `plan` → `execute-all` → `status`

## Architecture

### Subagents

PTF uses specialized subagents spawned via Claude Code's Task tool:

- **ptf-decomposer**: Goal analysis, subgoal identification, recursive task breakdown
- **ptf-dependency-analyzer**: Multi-pass dependency inference, cycle detection, wave computation
- **ptf-executor**: Execute single task with fresh context, verify outputs
- **ptf-verifier**: Independently verify task outputs against criteria
- **ptf-state-manager**: Checkpoint operations, event logging, artifact tracking
- **ptf-orchestrator**: Coordinate wave-by-wave execution, dispatch subagents
- **ptf-team-lead**: Coordinate dynamic execution via Agent Teams (teams mode)
- **ptf-team-executor**: Persistent teammate that claims tasks and spawns fresh executors (teams mode)

### Teams Mode (Experimental)

PTF supports dynamic scheduling via Claude Code's Agent Teams feature (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`). Configure with `execution.mode: teams` in `config.yaml`. Tasks execute as soon as their specific dependencies are satisfied (no wave boundary waiting). Uses a two-tier dispatch: persistent teammates claim tasks and spawn fresh `ptf-executor` subagents, preserving the fresh-context guarantee.

### Core Concepts

- **Task**: Atomic unit of work that fits in fresh context (target 10-30% of context window)
- **Wave**: Set of independent tasks that can execute in parallel
- **Artifact**: File produced/consumed by tasks; enables automatic dependency inference
- **Ralph-style execution**: Bounded retry loop with fresh context per iteration until verification passes

### Domain Adapters

Adapters in `adapters/` customize PTF for specific domains:
- Define clarification questions for `/ptf:init`
- Provide decomposition heuristics and atomicity criteria
- Specify artifact types and verification strategies
- Configure dependency inference patterns

## State Files

Runtime state lives in `.orchestrator/`:

| Path | Purpose |
|------|---------|
| `config.yaml` | Project configuration (domain, adapter, limits) |
| `goal.md` | Original goal statement |
| `decomposition/` | Goal analysis, tasks, dependencies |
| `state/` | Execution state, task results |
| `artifacts/manifest.yaml` | All artifact metadata |
| `history/events.jsonl` | Timestamped event stream (audit log) |

## Schema Validation

Validate YAML files against schemas using any JSON Schema validator. Schemas use `$schema` references:

```yaml
# yaml-language-server: $schema=../schemas/task.schema.yaml
```

Key schemas: `task.schema.yaml`, `plan.schema.yaml`, `execution-state.schema.yaml`, `adapter.schema.yaml`

## Event Logging

Events MUST be logged to `.orchestrator/history/events.jsonl` (not `.orchestrator/events/`). The state-manager handles this. Verify event counts match execution:
- `wave_started` count = waves executed
- `task_started` count = tasks attempted
- `task_completed` + `task_failed` = tasks finished
