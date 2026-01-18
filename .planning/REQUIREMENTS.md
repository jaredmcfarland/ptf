# Requirements: Parallel Task Framework (PTF)

**Defined:** 2025-01-18
**Core Value:** Fresh context execution for every task — context windows are the scarce resource in AI computation

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Foundation

- [x] **FOUND-01**: YAML schema for Task (id, name, description, inputs, outputs, verify, context_notes, on_failure)
- [x] **FOUND-02**: YAML schema for Artifact (path, type, produced_by, consumed_by, checksum, verified)
- [x] **FOUND-03**: YAML schema for Dependency (from, to, type, confidence, reason)
- [x] **FOUND-04**: YAML schema for Wave (number, tasks, status, depends_on_waves)
- [x] **FOUND-05**: YAML schema for Plan (id, goal, analysis, tasks, dependencies, waves, execution_policy)
- [x] **FOUND-06**: Context budget fields in Task schema (estimated tokens, max context percentage)
- [x] **FOUND-07**: SKILL.md documenting framework concepts, core principles, command reference
- [x] **FOUND-08**: Plugin directory structure (commands/, agents/, skills/, hooks/, adapters/, schemas/)
- [x] **FOUND-09**: Example task definition files demonstrating schema usage
- [x] **FOUND-10**: Example plan files demonstrating wave structure

### Decomposition

- [x] **DECOMP-01**: `/ptf:init [goal]` command initializes project and runs goal analysis
- [x] **DECOMP-02**: Goal analysis extracts objective, scope, constraints, success criteria, domain
- [x] **DECOMP-03**: Adapter-driven questioning during init (domain shapes questions asked)
- [x] **DECOMP-04**: Constitution generation (domain-shaped immutable principles)
- [x] **DECOMP-05**: `/ptf:decompose` command runs full 5-step decomposition process
- [x] **DECOMP-06**: Subgoal identification (Step 2) breaks analyzed goal into major components
- [x] **DECOMP-07**: Recursive decomposition (Step 3) breaks subgoals into atomic tasks
- [x] **DECOMP-08**: Atomicity evaluation using domain adapter criteria
- [x] **DECOMP-09**: Decomposition validation (Step 4) checks coverage, overlap, atomicity
- [x] **DECOMP-10**: Decomposer subagent executes the 5-step process
- [x] **DECOMP-11**: Decomposition state persists to .orchestrator/decomposition/

### Dependencies

- [x] **DEP-01**: Automatic dependency inference from task inputs/outputs (artifact matching)
- [x] **DEP-02**: Type-based dependency inference (pattern/glob matching)
- [x] **DEP-03**: Semantic dependency inference (LLM analysis of descriptions)
- [x] **DEP-04**: Domain heuristic dependency inference (adapter-provided patterns)
- [x] **DEP-05**: Resource conflict detection (tasks modifying same files)
- [x] **DEP-06**: Cycle detection in dependency graph
- [x] **DEP-07**: Cycle resolution guidance (suggest which dependency to break)
- [x] **DEP-08**: Wave computation via topological sort
- [x] **DEP-09**: Dependency analyzer subagent executes inference and wave computation
- [x] **DEP-10**: `/ptf:plan` command generates human-readable plan from decomposition
- [x] **DEP-11**: Dependency graph state persists to .orchestrator/decomposition/graph.yaml

### Execution

- [ ] **EXEC-01**: `/ptf:execute [wave]` command executes single wave or next pending wave
- [ ] **EXEC-02**: `/ptf:execute-all` command executes all waves with parallel dispatch
- [ ] **EXEC-03**: Wave-based parallel execution (all tasks in wave execute simultaneously)
- [ ] **EXEC-04**: Fresh context dispatch (each task runs in fresh subagent context)
- [ ] **EXEC-05**: Context loading limited to declared task inputs only
- [ ] **EXEC-06**: Task executor subagent executes single task with fresh context
- [ ] **EXEC-07**: Orchestrator subagent coordinates wave-by-wave execution
- [ ] **EXEC-08**: Ralph-style execution mode (repeat task until verified)
- [ ] **EXEC-09**: Completion promise pattern (agent must output specific phrase to signal done)
- [ ] **EXEC-10**: Configurable max iterations per task for Ralph-style execution
- [ ] **EXEC-11**: JSONL event logging (task_started, task_completed, wave_started, wave_completed)
- [ ] **EXEC-12**: Respect max_parallel_tasks configuration

### State

- [ ] **STATE-01**: File-based state persistence in .orchestrator/ directory
- [ ] **STATE-02**: execution.yaml as master state file (status, wave, progress, blockers)
- [ ] **STATE-03**: Per-task state files in state/tasks/ (status, attempts, outputs, verification)
- [ ] **STATE-04**: Per-wave state files in state/waves/ (status, tasks, completion time)
- [ ] **STATE-05**: Artifact manifest in artifacts/manifest.yaml (registry of produced files)
- [ ] **STATE-06**: Event log in history/events.jsonl (append-only, structured events)
- [ ] **STATE-07**: Wave boundary checkpoints (automatic state save after each wave)
- [ ] **STATE-08**: `/ptf:status` command shows current execution state
- [ ] **STATE-09**: `/ptf:resume` command resumes from interruption point
- [ ] **STATE-10**: Resume validates existing outputs before continuing

### Verification

- [ ] **VERIFY-01**: Verifier subagent runs verification checks independently
- [ ] **VERIFY-02**: Verification type: exists (file exists at expected path)
- [ ] **VERIFY-03**: Verification type: contains (file contains expected content)
- [ ] **VERIFY-04**: Verification type: runs (command executes with expected exit code)
- [ ] **VERIFY-05**: Verification type: syntax (file parses without syntax errors)
- [ ] **VERIFY-06**: Verification type: custom (user-defined verification logic)
- [ ] **VERIFY-07**: `/ptf:verify [task]` command runs verification for specific task
- [ ] **VERIFY-08**: Verification results recorded in task state
- [ ] **VERIFY-09**: Verification failure triggers retry or escalation per policy

### Failure Handling

- [ ] **FAIL-01**: Retry strategy with configurable max attempts
- [ ] **FAIL-02**: Exponential backoff between retry attempts
- [ ] **FAIL-03**: Skip strategy (mark task skipped, continue execution)
- [ ] **FAIL-04**: Escalate strategy (pause execution, present to human)
- [ ] **FAIL-05**: Failure cascade handling (block dependents when task fails)
- [ ] **FAIL-06**: Cascade policy per task (propagate_failure: true/false)
- [ ] **FAIL-07**: `/ptf:retry [task]` command retries failed task
- [ ] **FAIL-08**: `/ptf:abort` command stops execution and preserves state
- [ ] **FAIL-09**: Failure records in failures/ directory with full context
- [ ] **FAIL-10**: Human escalation with options (retry, skip, abort, replan)
- [ ] **FAIL-11**: Replan capability (re-decompose portion after failure)

### Domain Adapters

- [ ] **ADAPT-01**: Domain adapter interface (decomposition, artifacts, dependencies, context)
- [ ] **ADAPT-02**: Adapter loading and integration with all framework phases
- [ ] **ADAPT-03**: Software development adapter with complete implementation
- [ ] **ADAPT-04**: Software adapter: decomposition heuristics (by-layer, by-feature, by-file)
- [ ] **ADAPT-05**: Software adapter: atomicity criteria (single-file, testable, focused)
- [ ] **ADAPT-06**: Software adapter: artifact types (source-code, migration, config, test)
- [ ] **ADAPT-07**: Software adapter: verification strategies per artifact type
- [ ] **ADAPT-08**: Software adapter: dependency patterns (schema→repo, repo→service, code→test)
- [ ] **ADAPT-09**: Research adapter with complete implementation
- [ ] **ADAPT-10**: Research adapter: decomposition heuristics (by-question, by-source, by-stage)
- [ ] **ADAPT-11**: Research adapter: atomicity criteria (single-question, bounded-sources)
- [ ] **ADAPT-12**: Research adapter: artifact types (finding, summary, synthesis, data)
- [ ] **ADAPT-13**: Research adapter: verification strategies per artifact type
- [ ] **ADAPT-14**: Research adapter: dependency patterns (source→finding, finding→synthesis)
- [ ] **ADAPT-15**: Template adapter for creating custom adapters
- [ ] **ADAPT-16**: Adapter-specific constitution templates

### Commands & Interface

- [ ] **CMD-01**: `/ptf:init [goal]` — initialize project, run goal analysis
- [ ] **CMD-02**: `/ptf:decompose` — run full 5-step decomposition
- [x] **CMD-03**: `/ptf:plan` — generate human-readable plan
- [ ] **CMD-04**: `/ptf:execute [wave]` — execute single wave
- [ ] **CMD-05**: `/ptf:execute-all` — execute all waves
- [ ] **CMD-06**: `/ptf:status` — show execution state
- [ ] **CMD-07**: `/ptf:resume` — resume from interruption
- [ ] **CMD-08**: `/ptf:verify [task]` — verify specific task
- [ ] **CMD-09**: `/ptf:retry [task]` — retry failed task
- [ ] **CMD-10**: `/ptf:abort` — stop execution, preserve state

### Hooks

- [ ] **HOOK-01**: post-task-complete hook (log events, update state)
- [ ] **HOOK-02**: pre-wave-start hook (checkpoint state)
- [ ] **HOOK-03**: on-failure hook (failure handling trigger)
- [ ] **HOOK-04**: on-session-end hook (cleanup, final state save)

## v2 Requirements

Deferred to future release. Tracked but not in current roadmap.

### Advanced Features

- **ADV-01**: Partial wave execution (execute subset when some tasks ready)
- **ADV-02**: Visual execution dashboard (web UI for monitoring)
- **ADV-03**: Additional domain adapters (creative writing, music, etc.)
- **ADV-04**: Learning from past executions (improve decomposition over time)
- **ADV-05**: Multi-project orchestration (coordinate across goal hierarchies)
- **ADV-06**: MCP server mode (standalone service for non-Claude Code environments)

### Backend Integration

- **BACK-01**: Beads backend adapter (git-backed state persistence)
- **BACK-02**: SQLite backend adapter (complex query support)
- **BACK-03**: Backend interface abstraction

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Real-time agent communication | Violates wave independence, introduces distributed state problems |
| Autonomous replanning during execution | Unpredictable behavior, hard to debug; use checkpoint-based replan |
| Shared memory between parallel tasks | Defeats fresh context thesis; files are communication mechanism |
| Complex branching/conditional flows | Adds DAG complexity; most goals decompose to simple DAG |
| Multi-LLM provider orchestration | Coordination complexity; single provider (Claude) for v1 |
| Visual DAG editor | Development overhead; YAML/Markdown is sufficient |
| Persistent conversational memory | Scope creep; project-scoped state is enough |
| Learning from past executions | Requires ML infrastructure; explicit adapter updates instead |
| Agent-to-agent protocol (A2A) | Overkill for single framework; Task tool dispatch sufficient |
| Explicit dependency declaration only | Automatic inference is core differentiator |

## Traceability

Phase assignments for all v1 requirements.

### Phase 1: Foundation
| Requirement | Description | Status |
|-------------|-------------|--------|
| FOUND-01 | YAML schema for Task | Complete |
| FOUND-02 | YAML schema for Artifact | Complete |
| FOUND-03 | YAML schema for Dependency | Complete |
| FOUND-04 | YAML schema for Wave | Complete |
| FOUND-05 | YAML schema for Plan | Complete |
| FOUND-06 | Context budget fields in Task schema | Complete |
| FOUND-07 | SKILL.md documenting framework concepts | Complete |
| FOUND-08 | Plugin directory structure | Complete |
| FOUND-09 | Example task definition files | Complete |
| FOUND-10 | Example plan files | Complete |

### Phase 2: Decomposition
| Requirement | Description | Status |
|-------------|-------------|--------|
| DECOMP-01 | /ptf:init command | Complete |
| DECOMP-02 | Goal analysis extraction | Complete |
| DECOMP-03 | Adapter-driven questioning | Complete |
| DECOMP-04 | Constitution generation | Complete |
| DECOMP-05 | /ptf:decompose command | Complete |
| DECOMP-06 | Subgoal identification | Complete |
| DECOMP-07 | Recursive decomposition | Complete |
| DECOMP-08 | Atomicity evaluation | Complete |
| DECOMP-09 | Decomposition validation | Complete |
| DECOMP-10 | Decomposer subagent | Complete |
| DECOMP-11 | Decomposition state persistence | Complete |
| CMD-01 | /ptf:init command interface | Complete |
| CMD-02 | /ptf:decompose command interface | Complete |

### Phase 3: Dependency Analysis
| Requirement | Description | Status |
|-------------|-------------|--------|
| DEP-01 | Artifact-based dependency inference | Pending |
| DEP-02 | Type-based dependency inference | Pending |
| DEP-03 | Semantic dependency inference | Pending |
| DEP-04 | Domain heuristic inference | Pending |
| DEP-05 | Resource conflict detection | Pending |
| DEP-06 | Cycle detection | Pending |
| DEP-07 | Cycle resolution guidance | Pending |
| DEP-08 | Wave computation | Pending |
| DEP-09 | Dependency analyzer subagent | Pending |
| DEP-10 | /ptf:plan command | Pending |
| DEP-11 | Dependency graph persistence | Pending |
| CMD-03 | /ptf:plan command interface | Pending |

### Phase 4: State Management
| Requirement | Description | Status |
|-------------|-------------|--------|
| STATE-01 | File-based state in .orchestrator/ | Pending |
| STATE-02 | execution.yaml master state | Pending |
| STATE-03 | Per-task state files | Pending |
| STATE-04 | Per-wave state files | Pending |
| STATE-05 | Artifact manifest | Pending |
| STATE-06 | Event log (JSONL) | Pending |
| STATE-07 | Wave boundary checkpoints | Pending |
| STATE-08 | /ptf:status command | Pending |
| STATE-09 | /ptf:resume command | Pending |
| STATE-10 | Resume validation | Pending |
| CMD-06 | /ptf:status command interface | Pending |

### Phase 5: Execution Engine
| Requirement | Description | Status |
|-------------|-------------|--------|
| EXEC-01 | /ptf:execute [wave] command | Pending |
| EXEC-02 | /ptf:execute-all command | Pending |
| EXEC-03 | Wave-based parallel execution | Pending |
| EXEC-04 | Fresh context dispatch | Pending |
| EXEC-05 | Context loading from declared inputs | Pending |
| EXEC-06 | Task executor subagent | Pending |
| EXEC-07 | Orchestrator subagent | Pending |
| EXEC-08 | Ralph-style execution mode | Pending |
| EXEC-09 | Completion promise pattern | Pending |
| EXEC-10 | Configurable max iterations | Pending |
| EXEC-11 | JSONL event logging | Pending |
| EXEC-12 | max_parallel_tasks config | Pending |
| CMD-04 | /ptf:execute command interface | Pending |
| CMD-05 | /ptf:execute-all command interface | Pending |
| HOOK-01 | post-task-complete hook | Pending |
| HOOK-02 | pre-wave-start hook | Pending |
| HOOK-03 | on-failure hook | Pending |
| HOOK-04 | on-session-end hook | Pending |

### Phase 6: Verification
| Requirement | Description | Status |
|-------------|-------------|--------|
| VERIFY-01 | Verifier subagent | Pending |
| VERIFY-02 | Verification: exists | Pending |
| VERIFY-03 | Verification: contains | Pending |
| VERIFY-04 | Verification: runs | Pending |
| VERIFY-05 | Verification: syntax | Pending |
| VERIFY-06 | Verification: custom | Pending |
| VERIFY-07 | /ptf:verify [task] command | Pending |
| VERIFY-08 | Verification results in state | Pending |
| VERIFY-09 | Verification failure handling | Pending |
| CMD-08 | /ptf:verify command interface | Pending |

### Phase 7: Failure Handling
| Requirement | Description | Status |
|-------------|-------------|--------|
| FAIL-01 | Retry with max attempts | Pending |
| FAIL-02 | Exponential backoff | Pending |
| FAIL-03 | Skip strategy | Pending |
| FAIL-04 | Escalate strategy | Pending |
| FAIL-05 | Failure cascade handling | Pending |
| FAIL-06 | Cascade policy per task | Pending |
| FAIL-07 | /ptf:retry [task] command | Pending |
| FAIL-08 | /ptf:abort command | Pending |
| FAIL-09 | Failure records | Pending |
| FAIL-10 | Human escalation options | Pending |
| FAIL-11 | Replan capability | Pending |
| CMD-07 | /ptf:resume command interface | Pending |
| CMD-09 | /ptf:retry command interface | Pending |
| CMD-10 | /ptf:abort command interface | Pending |

### Phase 8: Domain Adapters
| Requirement | Description | Status |
|-------------|-------------|--------|
| ADAPT-01 | Domain adapter interface | Pending |
| ADAPT-02 | Adapter loading/integration | Pending |
| ADAPT-03 | Software adapter (complete) | Pending |
| ADAPT-04 | Software: decomposition heuristics | Pending |
| ADAPT-05 | Software: atomicity criteria | Pending |
| ADAPT-06 | Software: artifact types | Pending |
| ADAPT-07 | Software: verification strategies | Pending |
| ADAPT-08 | Software: dependency patterns | Pending |
| ADAPT-09 | Research adapter (complete) | Pending |
| ADAPT-10 | Research: decomposition heuristics | Pending |
| ADAPT-11 | Research: atomicity criteria | Pending |
| ADAPT-12 | Research: artifact types | Pending |
| ADAPT-13 | Research: verification strategies | Pending |
| ADAPT-14 | Research: dependency patterns | Pending |
| ADAPT-15 | Template adapter | Pending |
| ADAPT-16 | Constitution templates | Pending |

### Coverage Summary

| Phase | Requirements | Count |
|-------|--------------|-------|
| Phase 1 | FOUND-01 to FOUND-10 | 10 |
| Phase 2 | DECOMP-01 to DECOMP-11, CMD-01, CMD-02 | 13 |
| Phase 3 | DEP-01 to DEP-11, CMD-03 | 12 |
| Phase 4 | STATE-01 to STATE-10, CMD-06 | 11 |
| Phase 5 | EXEC-01 to EXEC-12, CMD-04, CMD-05, HOOK-01 to HOOK-04 | 18 |
| Phase 6 | VERIFY-01 to VERIFY-09, CMD-08 | 10 |
| Phase 7 | FAIL-01 to FAIL-11, CMD-07, CMD-09, CMD-10 | 14 |
| Phase 8 | ADAPT-01 to ADAPT-16 | 16 |
| **Total** | | **104** |

**Note:** Original count of 88 excluded CMD-* and HOOK-* as separate requirements (they were considered part of their functional categories). The detailed traceability above counts them separately for explicit tracking, resulting in 104 entries. All functionality is covered with no orphans.

---
*Requirements defined: 2025-01-18*
*Last updated: 2025-01-18 after roadmap creation*
