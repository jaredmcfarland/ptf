# Project Research Summary

**Project:** PTF (Parallel Task Framework)
**Domain:** LLM Agent Orchestration and Task Decomposition
**Researched:** 2026-01-18
**Confidence:** HIGH

## Executive Summary

PTF is a Claude Code plugin for decomposing complex goals into atomic tasks with dependency-aware parallel execution. Research across stack, features, architecture, and pitfalls reveals a clear path: build on Claude Code's native primitives (Task tool, subagents, markdown commands) rather than external orchestration frameworks. The core thesis — "fresh context execution for every task" — directly addresses the #1 documented failure mode in multi-agent systems: context degradation, which causes 11 of 12 tested models to drop below 50% performance at 32k tokens.

The recommended approach is a layered architecture with five core components: User Interface (slash commands), Orchestration Layer (decomposer, DAG builder, wave executor), State Manager (file-based persistence), Agent Layer (specialized subagents), and Domain Layer (pluggable adapters). Wave-based execution creates natural checkpoint boundaries, isolates tasks for parallel execution, and ensures each task runs with fresh context. All state persists to `.orchestrator/` as YAML/JSONL, making workflows resumable and git-diffable.

Key risks center on specification quality (41.77% of multi-agent failures) and coordination overhead (36.94%). Mitigations: rigorous YAML schemas for task definitions, explicit dependency declaration with multi-pass inference as backup, independent verification agents, and conservative defaults (when uncertain, serialize rather than parallelize). Loop detection, adaptive re-decomposition, and wave-boundary checkpointing must be built in from the start, not added later.

## Key Findings

### Recommended Stack

PTF runs entirely as a Claude Code plugin — no external orchestration framework needed. Claude Code provides the execution runtime (Task tool, subagents), file access, and orchestration primitives. All configuration uses Markdown with YAML frontmatter (Claude Code's native format). State persists as YAML files in `.orchestrator/`. Event logs use JSONL for append-only history.

**Core technologies:**
- **Claude Code Plugin System**: Runtime environment, agent execution — the plugin system IS the runtime
- **Markdown + YAML Frontmatter**: Commands, agents, skills — native Claude Code format, non-negotiable
- **YAML (1.2)**: Schemas, state files, plans — human-readable, git-diffable, native support
- **JSONL**: Event logs, execution history — append-only format matching Claude Code ecosystem

**Critical constraint:** Only `plugin.json` goes in `.claude-plugin/`. All other directories (commands/, agents/, skills/) must be at plugin root. Subagents cannot spawn subagents — orchestrator must dispatch all work directly.

### Expected Features

**Must have (table stakes):**
- Task definition schema with inputs, outputs, instructions
- State persistence with resume capability
- Basic error handling (retry with backoff)
- Task status tracking and logging
- Goal-to-task decomposition (PTF's 5-step process)
- Dependency declaration and verification/completion criteria

**Should have (differentiators):**
- Wave-based parallel execution with zero inter-task dependencies
- Fresh context per task (addresses context degradation)
- Automatic dependency inference (multi-pass: artifact, type, semantic, heuristic)
- Artifact-centric dependency model ("B needs what A produces")
- Domain adapters shaping decomposition (software, research)
- Decomposition validation (100% rule, no overlap, atomicity checks)

**Defer (v2+):**
- Visual execution dashboard
- Multi-project orchestration
- Learning from past executions
- MCP server mode
- A2A protocol support

### Architecture Approach

Layered architecture with orchestrator-worker pattern. Central orchestrator maintains state machine, dispatches work to stateless subagents via Task tool, receives results through events. Wave-based checkpointing at natural boundaries. DAG-based task graph enables automatic parallelization via topological sort. Domain adapters inject specialized decomposition rules and verification strategies without modifying core framework.

**Major components:**
1. **User Interface Layer** — Slash commands (`/ptf:init`, `/ptf:execute`, etc.)
2. **Orchestration Layer** — Goal Analyzer, Plan Generator (DAG Builder), Wave Executor
3. **State Manager** — File-based persistence, checkpoints, artifact registry, event log
4. **Agent Layer** — Decomposer, Dependency Analyzer, Task Executor, Verifier subagents
5. **Domain Layer** — Software Adapter, Research Adapter, custom adapter templates

### Critical Pitfalls

1. **Context Degradation** — LLM performance drops dramatically as context fills (>50% drop at 32k tokens). **Avoid:** PTF's fresh context per task is the core mitigation; design tasks to complete within 0-30% of context window.

2. **Infinite Loop Trap** — Agents retry failed approaches repeatedly without progress. **Avoid:** Explicit loop detection with configurable thresholds, "tried actions" log outside LLM context, escape hatch mechanisms.

3. **Task Granularity Mismatch** — Too coarse fails execution, too fine creates coordination overhead. **Avoid:** Adaptive decomposition (decompose further only when execution fails), clear atomicity criteria, re-decomposition at execution time.

4. **Cascading Failure Propagation** — Early mistakes compound through workflow. **Avoid:** Wave boundary verification, independent judge agents, fail-fast design, structured success/failure signals.

5. **Verification Theater** — System reports success but output doesn't meet requirements. **Avoid:** Multi-modal verification (exists + contains + runs), independent verifier agent, explicit success criteria in task specs.

## Implications for Roadmap

Based on research, suggested phase structure follows component dependencies: schemas first (everything depends on data structures), decomposition before dependency analysis (need tasks to analyze), state management can parallel with dependency work, execution requires both plan and state, failure handling layers on execution.

### Phase 1: Foundation
**Rationale:** Everything depends on data structures and framework knowledge. YAML schemas define the contract for all subsequent work.
**Delivers:** Task, Artifact, Dependency, Wave schemas; SKILL.md with framework concepts; `.orchestrator/` directory structure; plugin.json manifest
**Addresses:** Task definition schema (table stakes), structured data contracts
**Avoids:** String-based task specs (technical debt), parsing errors

### Phase 2: Decomposition
**Rationale:** Must decompose goals before analyzing dependencies or executing. This implements PTF's core 5-step process.
**Delivers:** `/ptf:init`, `/ptf:decompose` commands; Decomposer subagent; Software development adapter; Goal analysis, subgoal identification, recursive decomposition, validation
**Addresses:** Goal-to-task decomposition (table stakes), decomposition validation (differentiator)
**Avoids:** Task granularity mismatch, hallucinated plans (validation catches non-existent resources)

### Phase 3: Dependency Analysis
**Rationale:** With atomic tasks defined, can now infer dependencies and compute execution waves. Depends on Phase 2 outputs.
**Delivers:** Dependency Analyzer subagent; Multi-pass inference algorithm; Cycle detection; Wave computation; `/ptf:plan` command
**Addresses:** Dependency declaration (table stakes), automatic dependency inference (differentiator), artifact-centric dependencies
**Avoids:** Dependency inference failures (multi-pass with conservative defaults), cycles in graph

### Phase 4: State Management
**Rationale:** Can parallel with Phase 3. Required before execution can work. Enables resume capability.
**Delivers:** File-based state persistence; State schemas (execution, task, wave); Checkpoint functions; Resume protocol
**Addresses:** State persistence (table stakes), resume capability
**Avoids:** State persistence failures, lost progress on interruption

### Phase 5: Execution Engine
**Rationale:** Requires both dependency graph (Phase 3) and state management (Phase 4) before tasks can execute.
**Delivers:** Orchestrator subagent; Task Executor subagent; Verifier subagent; `/ptf:execute` command; Parallel dispatch via Task tool
**Addresses:** Wave-based parallel execution (differentiator), fresh context per task (differentiator), basic verification
**Avoids:** Context degradation (fresh context), coordination overhead (wave isolation), infinite loops (loop detection)

### Phase 6: Failure Handling
**Rationale:** Layers on execution engine. Requires working execution to test failure scenarios.
**Delivers:** Retry strategies; Failure records; Cascade handling; `/ptf:resume`, `/ptf:retry`, `/ptf:abort` commands
**Addresses:** Basic error handling (table stakes), failure cascade handling (differentiator)
**Avoids:** Cascading failures, verification theater (multi-modal verification)

### Phase 7: Polish
**Rationale:** Depends on all previous phases being stable. Extends framework to new domains.
**Delivers:** Research domain adapter; Complete documentation; Example projects; Error messages and edge cases
**Addresses:** Research adapter (v1.x feature), extensibility proof
**Avoids:** Domain adapter quality issues (stabilized API)

### Phase Ordering Rationale

- **Schemas before behavior:** All components share data structures. Defining schemas first prevents contract drift.
- **Decomposition before dependency:** Cannot analyze task dependencies without tasks. Decomposition must precede graph construction.
- **State in parallel with dependency:** No dependency between these. Both needed for execution but not for each other.
- **Execution after plan + state:** Requires both dependency graph and state persistence. Cannot meaningfully execute without them.
- **Failure handling last in core:** Must have working execution to test failure scenarios. Layered approach.
- **Polish after stability:** Domain extensions and documentation require stable APIs. Premature documentation becomes maintenance burden.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 2 (Decomposition):** Atomicity criteria and 100% rule validation are heuristic. May need iteration based on real decomposition attempts.
- **Phase 3 (Dependency Analysis):** Multi-pass inference algorithm design. Conservative defaults important but may need tuning.
- **Phase 5 (Execution Engine):** Loop detection thresholds, parallel dispatch patterns. Claude Code's 10-task concurrency limit shapes design.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Foundation):** YAML schema design is well-documented. Claude Code plugin structure is in official docs.
- **Phase 4 (State Management):** File-based state with YAML/JSONL is straightforward. Wave-boundary checkpointing is a known pattern.
- **Phase 6 (Failure Handling):** Retry/backoff patterns well-established. Circuit breaker patterns documented.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Verified against official Claude Code documentation, plugin structure confirmed |
| Features | HIGH | Cross-referenced multiple framework comparisons, academic papers, production post-mortems |
| Architecture | HIGH | Patterns validated against LangGraph, CrewAI, Microsoft Agent Framework, academic research |
| Pitfalls | HIGH | Corroborated by academic papers (41-86.7% failure rates), production post-mortems, framework docs |

**Overall confidence:** HIGH

### Gaps to Address

- **Atomicity criteria calibration:** Research provides principles but specific thresholds (e.g., "fits in fresh context") need validation during decomposition implementation
- **Dependency inference accuracy:** Multi-pass algorithm design is conceptual. May need iteration on conservative defaults vs. parallelization gains
- **Context budget measurement:** "0-30% of context window" guidance exists but measuring actual context consumption per task needs implementation
- **Adapter API stability:** Software adapter will inform API. Research adapter implementation may require API adjustments

## Sources

### Primary (HIGH confidence)
- [Claude Code Plugins Documentation](https://code.claude.com/docs/en/plugins) — Plugin structure, manifest, commands, hooks
- [Claude Code Agent Skills](https://code.claude.com/docs/en/skills) — SKILL.md format, frontmatter fields
- [Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/html/2503.13657v1) — MAST taxonomy, failure rates (41-86.7%)
- [Context Rot Research](https://research.trychroma.com/context-rot) — Context degradation quantification

### Secondary (MEDIUM confidence)
- [LangGraph Documentation](https://docs.langchain.com/) — Graph-based agent architecture, checkpointing patterns
- [CrewAI Documentation](https://docs.crewai.com/) — Role-based agents, task concepts
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Agentic coding workflows
- [Task Tool Best Practices](https://claudelog.com/mechanics/task-agent-tools/) — Parallelism limits (10 concurrent), token overhead

### Tertiary (LOW confidence)
- Framework comparisons (Turing, Langflow, AIMultiple) — Landscape surveys, feature matrices
- Individual production post-mortems — Anecdotal but consistent patterns

---
*Research completed: 2026-01-18*
*Ready for roadmap: yes*
