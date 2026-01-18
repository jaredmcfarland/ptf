# Requirements: Parallel Task Framework (PTF)

**Defined:** 2025-01-18
**Core Value:** Fresh context execution for every task — context windows are the scarce resource in AI computation

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Foundation

- [ ] **FOUND-01**: YAML schema for Task (id, name, description, inputs, outputs, verify, context_notes, on_failure)
- [ ] **FOUND-02**: YAML schema for Artifact (path, type, produced_by, consumed_by, checksum, verified)
- [ ] **FOUND-03**: YAML schema for Dependency (from, to, type, confidence, reason)
- [ ] **FOUND-04**: YAML schema for Wave (number, tasks, status, depends_on_waves)
- [ ] **FOUND-05**: YAML schema for Plan (id, goal, analysis, tasks, dependencies, waves, execution_policy)
- [ ] **FOUND-06**: Context budget fields in Task schema (estimated tokens, max context percentage)
- [ ] **FOUND-07**: SKILL.md documenting framework concepts, core principles, command reference
- [ ] **FOUND-08**: Plugin directory structure (commands/, agents/, skills/, hooks/, adapters/, schemas/)
- [ ] **FOUND-09**: Example task definition files demonstrating schema usage
- [ ] **FOUND-10**: Example plan files demonstrating wave structure

### Decomposition

- [ ] **DECOMP-01**: `/ptf:init [goal]` command initializes project and runs goal analysis
- [ ] **DECOMP-02**: Goal analysis extracts objective, scope, constraints, success criteria, domain
- [ ] **DECOMP-03**: Adapter-driven questioning during init (domain shapes questions asked)
- [ ] **DECOMP-04**: Constitution generation (domain-shaped immutable principles)
- [ ] **DECOMP-05**: `/ptf:decompose` command runs full 5-step decomposition process
- [ ] **DECOMP-06**: Subgoal identification (Step 2) breaks analyzed goal into major components
- [ ] **DECOMP-07**: Recursive decomposition (Step 3) breaks subgoals into atomic tasks
- [ ] **DECOMP-08**: Atomicity evaluation using domain adapter criteria
- [ ] **DECOMP-09**: Decomposition validation (Step 4) checks coverage, overlap, atomicity
- [ ] **DECOMP-10**: Decomposer subagent executes the 5-step process
- [ ] **DECOMP-11**: Decomposition state persists to .orchestrator/decomposition/

### Dependencies

- [ ] **DEP-01**: Automatic dependency inference from task inputs/outputs (artifact matching)
- [ ] **DEP-02**: Type-based dependency inference (pattern/glob matching)
- [ ] **DEP-03**: Semantic dependency inference (LLM analysis of descriptions)
- [ ] **DEP-04**: Domain heuristic dependency inference (adapter-provided patterns)
- [ ] **DEP-05**: Resource conflict detection (tasks modifying same files)
- [ ] **DEP-06**: Cycle detection in dependency graph
- [ ] **DEP-07**: Cycle resolution guidance (suggest which dependency to break)
- [ ] **DEP-08**: Wave computation via topological sort
- [ ] **DEP-09**: Dependency analyzer subagent executes inference and wave computation
- [ ] **DEP-10**: `/ptf:plan` command generates human-readable plan from decomposition
- [ ] **DEP-11**: Dependency graph state persists to .orchestrator/decomposition/graph.yaml

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
- [ ] **CMD-03**: `/ptf:plan` — generate human-readable plan
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

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| FOUND-* | Phase 1 | Pending |
| DECOMP-* | Phase 2 | Pending |
| DEP-* | Phase 3 | Pending |
| STATE-* | Phase 4 | Pending |
| EXEC-* | Phase 5 | Pending |
| VERIFY-* | Phase 5 | Pending |
| FAIL-* | Phase 6 | Pending |
| ADAPT-* | Phase 7 | Pending |
| CMD-* | Phases 1-7 | Pending |
| HOOK-* | Phase 5 | Pending |

**Coverage:**
- v1 requirements: 88 total
- Mapped to phases: 88
- Unmapped: 0

---
*Requirements defined: 2025-01-18*
*Last updated: 2025-01-18 after initial definition*
