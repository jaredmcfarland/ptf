# Parallel Task Framework (PTF)

## What This Is

A domain-agnostic system for decomposing complex goals into atomic tasks, analyzing dependencies, computing parallel execution waves, and orchestrating LLM agents with fresh context. Implemented as a Claude Code plugin providing slash commands, specialized subagents, an agent skill, and hooks.

## Core Value

**Fresh context execution for every task.** Context windows are the scarce resource in AI computation — optimizing for fresh context produces the highest quality output at the fastest speed. Every architectural decision serves this thesis.

## Requirements

### Validated

(None yet — ship to validate)

### Active

**Foundation**
- [ ] Task, Artifact, Dependency, Wave YAML schemas
- [ ] SKILL.md with framework concepts
- [ ] Plugin directory structure (commands/, agents/, skills/, hooks/, adapters/, schemas/)
- [ ] Example files demonstrating formats

**Decomposition Commands**
- [ ] `/ptf:init [goal]` command — initialize project, run goal analysis
- [ ] `/ptf:decompose` command — run full 5-step decomposition
- [ ] Decomposer subagent (goal analysis → subgoals → recursive breakdown → validation)
- [ ] Constitution generation (domain-shaped principles)
- [ ] Adapter-driven questioning during init

**Dependency Analysis**
- [ ] Dependency analyzer subagent
- [ ] Multi-pass inference algorithm (artifact, type, semantic, heuristic, resource)
- [ ] Cycle detection and reporting
- [ ] Wave computation via topological sort
- [ ] `/ptf:plan` command — generate human-readable plan

**Execution Engine**
- [ ] Orchestrator subagent — wave-by-wave execution control
- [ ] Task executor subagent — single task with fresh context
- [ ] `/ptf:execute [wave]` command — execute single wave
- [ ] `/ptf:execute-all` command — execute all waves
- [ ] Parallel subagent dispatch within waves
- [ ] Ralph-style execution mode (repeat until verified)

**State & Verification**
- [ ] File-based state persistence (.orchestrator/ directory)
- [ ] Verifier subagent (exists, contains, runs, syntax, custom)
- [ ] `/ptf:status` command — show execution state
- [ ] `/ptf:verify [task]` command — run verification
- [ ] Event logging hooks (post-task-complete, pre-wave-start)
- [ ] Checkpoint at wave boundaries

**Failure Handling**
- [ ] Retry strategies (retry, skip, escalate, replan)
- [ ] `/ptf:resume` command — resume from interruption
- [ ] `/ptf:retry [task]` command — retry failed task
- [ ] `/ptf:abort` command — stop execution, preserve state
- [ ] Cascade failure handling (blocked dependents)
- [ ] Failure record creation with context

**Domain Adapters**
- [ ] Software development adapter (complete)
- [ ] Research adapter (complete)
- [ ] Template adapter for customization
- [ ] Adapter loading and integration with decomposition
- [ ] Constitution templates per domain

**Polish**
- [ ] Plugin README with installation and usage
- [ ] Complete documentation
- [ ] Example projects (software goal, research goal)
- [ ] Error messages and edge case handling

### Out of Scope

- Standalone execution (not a separate runtime) — Claude Code is the execution environment
- Beads backend integration — file-based backend only for v1
- Additional domain adapters beyond software/research — template provided for extension
- MCP server implementation — plugin-first for v1
- Multi-framework orchestration — single PTF instance only

## Context

**Theoretical Foundations:**
- Miller's chunking and working memory limits (7±2)
- Simon's near-decomposability (weak inter-component coupling)
- Hierarchical Task Analysis (goals → subgoals → operations)
- Work Breakdown Structures (100% rule, no overlap, outcome-oriented)
- DAG execution models (topological sort, wave computation)

**Inspiration:**
- GSD (Get Shit Done) — spec-driven development patterns
- Ralph Wiggum Loop — fresh context via temporal iteration
- Beads — git-backed graph issue tracking for agents
- GitHub Spec Kit — constitution concept, immutable principles

**Key Insight:**
Context degradation is predictable (0-30% peak, 70%+ poor). The framework solves this through spatial decomposition (break into pieces) combined with temporal iteration (Ralph loops within tasks). Tasks within a wave are truly independent — no coordination needed, no distributed state problems.

## Constraints

- **Runtime**: Claude Code plugin — leverages existing Task tool, subagents, file system access
- **State**: File-based persistence in `.orchestrator/` — YAML for structure, JSONL for events, Markdown for human-readable
- **Adapters**: Two for v1 (software, research) — proves generalization without scope explosion
- **Execution**: Claude Code is both orchestrator and executor via subagents

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Claude Code plugin (not standalone) | Leverages existing agent capabilities, no separate runtime needed | — Pending |
| File-based state (not Beads for v1) | Simpler, no external dependency, proves core patterns first | — Pending |
| Two domain adapters for v1 | Software and research are sufficiently different to prove generalization | — Pending |
| Ralph-style execution as primitive | Temporal iteration complements spatial decomposition | — Pending |
| Constitution concept from SpecKit | Domain-shaped immutable principles constrain downstream decisions | — Pending |

---
*Last updated: 2025-01-18 after initialization*
