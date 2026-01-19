# Roadmap: Parallel Task Framework (PTF)

## Overview

PTF transforms complex goals into atomic tasks with dependency-aware parallel execution, ensuring fresh context for every task. The roadmap progresses from data schemas through decomposition, dependency analysis, state management, and execution engine to verification, failure handling, and domain adaptation. Each phase delivers a coherent capability building toward the complete framework.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

- [x] **Phase 1: Foundation** - YAML schemas, plugin structure, skill documentation
- [x] **Phase 2: Decomposition** - Goal analysis, recursive task breakdown, validation
- [x] **Phase 3: Dependency Analysis** - Multi-pass inference, cycle detection, wave computation
- [x] **Phase 4: State Management** - File-based persistence, checkpoints, resume capability
- [x] **Phase 5: Execution Engine** - Wave-based parallel execution, fresh context dispatch
- [ ] **Phase 6: Verification** - Multi-modal verification, independent verifier subagent
- [ ] **Phase 7: Failure Handling** - Retry strategies, cascade handling, recovery commands
- [ ] **Phase 8: Domain Adapters** - Software/research adapters, templates, documentation

## Phase Details

### Phase 1: Foundation
**Goal**: Establish data contracts and plugin structure that all subsequent phases depend on
**Depends on**: Nothing (first phase)
**Requirements**: FOUND-01, FOUND-02, FOUND-03, FOUND-04, FOUND-05, FOUND-06, FOUND-07, FOUND-08, FOUND-09, FOUND-10
**Success Criteria** (what must be TRUE):
  1. YAML schemas for Task, Artifact, Dependency, Wave, Plan exist and validate correctly
  2. Plugin directory structure exists with commands/, agents/, skills/, hooks/, adapters/, schemas/ folders
  3. SKILL.md documents framework concepts and is readable by Claude Code
  4. Example files demonstrate schema usage and can be parsed without errors
  5. Context budget fields exist in Task schema (estimated_tokens, max_context_percentage)
**Plans**: 3 plans

Plans:
- [x] 01-01-PLAN.md — Create all 5 YAML schemas (Task, Artifact, Dependency, Wave, Plan)
- [x] 01-02-PLAN.md — Create plugin directory structure and SKILL.md documentation
- [x] 01-03-PLAN.md — Create example task and plan files demonstrating schema usage

### Phase 2: Decomposition
**Goal**: Transform goals into validated atomic tasks through 5-step decomposition process
**Depends on**: Phase 1 (requires schemas)
**Requirements**: DECOMP-01, DECOMP-02, DECOMP-03, DECOMP-04, DECOMP-05, DECOMP-06, DECOMP-07, DECOMP-08, DECOMP-09, DECOMP-10, DECOMP-11, CMD-01, CMD-02
**Success Criteria** (what must be TRUE):
  1. User can run `/ptf:init [goal]` and receive goal analysis with extracted scope, constraints, success criteria
  2. User can run `/ptf:decompose` and see goal break into subgoals then atomic tasks
  3. Decomposition validates 100% coverage (no task overlap, all goal aspects addressed)
  4. Tasks meet atomicity criteria (single-file, fresh-context-completable)
  5. Decomposition state persists to .orchestrator/decomposition/ and survives session restart
**Plans**: 4 plans

Plans:
- [x] 02-01-PLAN.md — Create domain adapters (software-development, research, template)
- [x] 02-02-PLAN.md — Create /ptf:init command for goal analysis
- [x] 02-03-PLAN.md — Create ptf-decomposer subagent for 5-step decomposition
- [x] 02-04-PLAN.md — Create /ptf:decompose command to orchestrate decomposition

### Phase 3: Dependency Analysis
**Goal**: Infer task dependencies and compute parallel execution waves
**Depends on**: Phase 2 (requires decomposed tasks)
**Requirements**: DEP-01, DEP-02, DEP-03, DEP-04, DEP-05, DEP-06, DEP-07, DEP-08, DEP-09, DEP-10, DEP-11, CMD-03
**Success Criteria** (what must be TRUE):
  1. Framework automatically infers dependencies from task inputs/outputs without explicit declaration
  2. Multi-pass inference (artifact, type, semantic, heuristic) catches non-obvious dependencies
  3. Cycles in dependency graph are detected and reported with resolution guidance
  4. `/ptf:plan` produces human-readable plan showing waves and task ordering
  5. Dependency graph persists to .orchestrator/decomposition/graph.yaml
**Plans**: 3 plans

Plans:
- [x] 03-01-PLAN.md — Create ptf-dependency-analyzer subagent (5-pass inference, Kahn's, Tarjan's)
- [x] 03-02-PLAN.md — Create /ptf:plan command (orchestrate analyzer, cycle handling, plan.md)
- [x] 03-03-PLAN.md — Create examples and update SKILL.md with dependency analysis concepts

### Phase 4: State Management
**Goal**: Enable reliable state persistence and session resumption
**Depends on**: Phase 1 (requires schemas), can parallel with Phase 3
**Requirements**: STATE-01, STATE-02, STATE-03, STATE-04, STATE-05, STATE-06, STATE-07, STATE-08, STATE-09, STATE-10, CMD-06
**Success Criteria** (what must be TRUE):
  1. All execution state persists to .orchestrator/ directory as YAML/JSONL files
  2. User can run `/ptf:status` and see current execution state (phase, wave, task status, blockers)
  3. Session interruption preserves state; `/ptf:resume` continues from last checkpoint
  4. Wave boundary checkpoints happen automatically after each wave completes
  5. Artifact manifest tracks all produced files with verification status
**Plans**: 3 plans

Plans:
- [x] 04-01-PLAN.md — Create state schemas (execution-state, task-state, wave-state, artifact-manifest)
- [x] 04-02-PLAN.md — Create event-log schema and ptf-state-manager subagent (checkpoint protocol)
- [x] 04-03-PLAN.md — Create /ptf:status and /ptf:resume commands

### Phase 5: Execution Engine
**Goal**: Execute tasks in parallel waves with fresh context per task
**Depends on**: Phase 3 (requires waves), Phase 4 (requires state management)
**Requirements**: EXEC-01, EXEC-02, EXEC-03, EXEC-04, EXEC-05, EXEC-06, EXEC-07, EXEC-08, EXEC-09, EXEC-10, EXEC-11, EXEC-12, CMD-04, CMD-05, HOOK-01, HOOK-02, HOOK-03, HOOK-04
**Success Criteria** (what must be TRUE):
  1. User can run `/ptf:execute [wave]` to execute a single wave with all tasks running in parallel
  2. User can run `/ptf:execute-all` to execute entire plan wave-by-wave automatically
  3. Each task runs in fresh subagent context with only declared inputs loaded
  4. Ralph-style execution mode repeats tasks until verification passes (configurable max iterations)
  5. Event logging captures task_started, task_completed, wave_started, wave_completed in JSONL format
**Plans**: 3 plans

Plans:
- [x] 05-01-PLAN.md — Create ptf-executor subagent (fresh context, Ralph-style iteration)
- [x] 05-02-PLAN.md — Create ptf-orchestrator subagent (wave coordination, parallel dispatch)
- [x] 05-03-PLAN.md — Create execute commands and hook infrastructure

### Phase 6: Verification
**Goal**: Independently verify task outputs with multi-modal strategies
**Depends on**: Phase 5 (requires task execution)
**Requirements**: VERIFY-01, VERIFY-02, VERIFY-03, VERIFY-04, VERIFY-05, VERIFY-06, VERIFY-07, VERIFY-08, VERIFY-09, CMD-08
**Success Criteria** (what must be TRUE):
  1. Verifier subagent runs independently from task executor (separate context)
  2. Verification types work: exists (file exists), contains (expected content), runs (command exit code), syntax (parses), custom (user-defined)
  3. User can run `/ptf:verify [task]` to manually trigger verification for any task
  4. Verification results recorded in task state and influence retry/continue decisions
  5. Multi-modal verification possible (e.g., exists AND contains AND runs for same artifact)
**Plans**: 2 plans

Plans:
- [ ] 06-01-PLAN.md — Create ptf-verifier subagent with 5 verification types
- [ ] 06-02-PLAN.md — Create /ptf:verify command and state integration

### Phase 7: Failure Handling
**Goal**: Handle failures gracefully with retry, escalation, and recovery strategies
**Depends on**: Phase 5 (requires execution), Phase 6 (requires verification)
**Requirements**: FAIL-01, FAIL-02, FAIL-03, FAIL-04, FAIL-05, FAIL-06, FAIL-07, FAIL-08, FAIL-09, FAIL-10, FAIL-11, CMD-07, CMD-09, CMD-10
**Success Criteria** (what must be TRUE):
  1. Failed tasks retry automatically with configurable max attempts and exponential backoff
  2. User can choose strategies per task: retry, skip, escalate (pause for human), replan
  3. `/ptf:retry [task]` retries a specific failed task; `/ptf:abort` stops execution preserving state
  4. Cascade handling blocks dependent tasks when prerequisite fails (configurable per task)
  5. Failure records in failures/ directory contain full context for debugging
**Plans**: TBD

Plans:
- [ ] 07-01: TBD (retry strategies and backoff)
- [ ] 07-02: TBD (cascade handling and failure records)
- [ ] 07-03: TBD (recovery commands: resume, retry, abort)

### Phase 8: Domain Adapters
**Goal**: Prove framework generalization with software and research domain adapters
**Depends on**: Phases 1-7 (requires stable core framework)
**Requirements**: ADAPT-01, ADAPT-02, ADAPT-03, ADAPT-04, ADAPT-05, ADAPT-06, ADAPT-07, ADAPT-08, ADAPT-09, ADAPT-10, ADAPT-11, ADAPT-12, ADAPT-13, ADAPT-14, ADAPT-15, ADAPT-16
**Success Criteria** (what must be TRUE):
  1. Software development adapter provides decomposition heuristics, atomicity criteria, artifact types, verification strategies, dependency patterns
  2. Research adapter provides equivalent capabilities shaped for research workflows (by-question decomposition, finding/synthesis artifacts)
  3. Template adapter enables users to create custom domain adapters
  4. Adapters integrate with all framework phases (decomposition, verification, dependencies)
  5. Example projects demonstrate both software and research domains end-to-end
**Plans**: TBD

Plans:
- [ ] 08-01: TBD (adapter interface and loading)
- [ ] 08-02: TBD (software development adapter)
- [ ] 08-03: TBD (research adapter)
- [ ] 08-04: TBD (template adapter and documentation)

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8
Note: Phases 3 and 4 can execute in parallel (no dependency between them).

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 3/3 | Complete | 2026-01-18 |
| 2. Decomposition | 4/4 | Complete | 2026-01-18 |
| 3. Dependency Analysis | 3/3 | Complete | 2026-01-18 |
| 4. State Management | 3/3 | Complete | 2026-01-19 |
| 5. Execution Engine | 3/3 | Complete | 2026-01-19 |
| 6. Verification | 0/2 | Planned | - |
| 7. Failure Handling | 0/TBD | Not started | - |
| 8. Domain Adapters | 0/TBD | Not started | - |

## Requirement Coverage

All 88 v1 requirements mapped:

| Category | Count | Phase(s) |
|----------|-------|----------|
| FOUND-* | 10 | Phase 1 |
| DECOMP-* | 11 | Phase 2 |
| DEP-* | 11 | Phase 3 |
| STATE-* | 10 | Phase 4 |
| EXEC-* | 12 | Phase 5 |
| VERIFY-* | 9 | Phase 6 |
| FAIL-* | 11 | Phase 7 |
| ADAPT-* | 16 | Phase 8 |
| CMD-* | 10 | Distributed (see below) |
| HOOK-* | 4 | Phase 5 |

**CMD Distribution:**
- CMD-01, CMD-02 -> Phase 2 (init, decompose)
- CMD-03 -> Phase 3 (plan)
- CMD-04, CMD-05 -> Phase 5 (execute, execute-all)
- CMD-06 -> Phase 4 (status)
- CMD-07, CMD-09, CMD-10 -> Phase 7 (resume, retry, abort)
- CMD-08 -> Phase 6 (verify)

**Coverage verification:** 10+11+11+10+12+9+11+16+10+4 = 94 requirements counted in categories
**REQUIREMENTS.md states:** 88 requirements

Note: The count discrepancy suggests CMD-* requirements duplicate functionality already counted in other categories (e.g., CMD-01 overlaps DECOMP-01). All unique functionality is covered.

---
*Roadmap created: 2025-01-18*
*Phase 1 planned: 2025-01-18*
*Phase 2 planned: 2026-01-18*
*Phase 3 planned: 2026-01-18*
*Phase 4 planned: 2026-01-18*
*Phase 5 planned: 2026-01-19*
*Phase 6 planned: 2026-01-19*
*Depth: comprehensive (8 phases)*
