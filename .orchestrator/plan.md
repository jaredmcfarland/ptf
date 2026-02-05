# Execution Plan: Data Codex System Architecture

**Generated:** 2026-02-04
**Domain:** system-design
**Tasks:** 27 tasks in 12 waves
**Dependencies:** 81 (all artifact-based, high confidence)
**Estimated parallel speedup:** 2.25x (27 tasks / 12 waves)

---

## Overview

Design the complete system architecture for Data Codex, a desktop analytics
application built on Electron that functions as a serverless data warehouse.
The design covers the embedded database trio (DuckDB, LanceDB, RxDB),
Apache Arrow data plane for zero-copy IPC, local/remote data source connectors,
GPU-accelerated visualization (Perspective.js, Observable Plot, ECharts),
AI chat integration (Claude Agent SDK, MCP, ACP), and a publishing engine
for exporting interactive web applications.

### Success Criteria

- System design document covers all 6 key systems with component diagrams
- Data model specification defines all entities, relationships, and storage locations
- IPC/API specification defines all message types between main/renderer/worker processes
- Technology decisions documented with alternatives considered and rationale
- Each roadmap phase has a corresponding architecture section enabling independent implementation
- Arrow data flow documented end-to-end from ingestion through visualization
- All design documents cross-reference each other consistently
- Verification checks confirm document completeness and cross-reference integrity

---

## Wave 1 (8 tasks - 8-way parallel)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| research-electron-process-model | Research Electron Process Model | research/electron-process-model.md | 8K peak |
| research-duckdb-electron | Research DuckDB in Electron | research/duckdb-electron.md | 8K peak |
| research-lancedb-electron | Research LanceDB in Electron | research/lancedb-electron.md | 8K peak |
| research-rxdb-electron | Research RxDB with SQLite | research/rxdb-electron.md | 8K peak |
| research-arrow-ipc-electron | Research Arrow IPC in Electron | research/arrow-ipc-electron.md | 8K peak |
| research-perspective-electron | Research Perspective.js Stack | research/perspective-electron.md | 8K peak |
| research-embedding-models | Research Embedding Models | research/embedding-models.md | 8K peak |
| research-claude-sdk-mcp-acp | Research Claude SDK/MCP/ACP | research/claude-sdk-mcp-acp.md | 8K peak |

**No dependencies - can start immediately**

---

## Wave 2 (5 tasks - 5-way parallel)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| adr-database-trio | ADR-001: Database Trio | decisions/adr-001-database-trio.md | 25K good |
| adr-arrow-data-plane | ADR-002: Arrow Data Plane | decisions/adr-002-arrow-data-plane.md | 18K good |
| adr-visualization-stack | ADR-003: Visualization Stack | decisions/adr-003-visualization-stack.md | 12K peak |
| adr-ai-integration | ADR-004: AI Integration | decisions/adr-004-ai-integration.md | 20K good |
| adr-process-architecture | ADR-005: Process Architecture | decisions/adr-005-process-architecture.md | 12K peak |

**Depends on:** Wave 1 research findings (each ADR consumes 1-3 research outputs)

---

## Wave 3 (1 task)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| design-system-overview | System Overview and Component Topology | design/01-system-overview.md | 40K good |

**Depends on:** All 5 ADRs from Wave 2

---

## Wave 4 (1 task)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| design-electron-process-model | Electron Process Model Design | design/02-electron-process-model.md | 35K good |

**Depends on:** design-system-overview (W3), adr-005, adr-001

---

## Wave 5 (2 tasks - 2-way parallel)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| design-arrow-data-plane | Arrow Data Plane Design | design/03-arrow-data-plane.md | 30K good |
| design-visualization-engine | Visualization Engine Architecture | design/08-visualization-engine.md | 30K good |

**Depends on:** design-electron-process-model (W4), respective ADRs

---

## Wave 6 (1 task)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| design-database-integration | Database Integration Layer Design | design/04-database-integration.md | 35K good |

**Depends on:** design-electron-process-model (W4), design-arrow-data-plane (W5)

---

## Wave 7 (2 tasks - 2-way parallel)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| design-vector-search-fts | Vector Search and FTS Design | design/05-vector-search-fts.md | 30K good |
| design-local-ingestion | Local Ingestion Pipeline Design | design/06-local-ingestion.md | 35K good |

**Depends on:** design-database-integration (W6), respective ADRs and design docs

---

## Wave 8 (2 tasks - 2-way parallel)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| design-remote-connectors | Remote Connector Framework Design | design/07-remote-connectors.md | 35K good |
| design-ai-integration | AI Integration Architecture Design | design/09-ai-integration.md | 40K good |

**Depends on:** design-database-integration (W6), design-local-ingestion (W7), design-vector-search-fts (W7)

---

## Wave 9 (2 tasks - 2-way parallel)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| design-publishing-engine | Publishing Engine Design | design/10-publishing-engine.md | 30K good |
| spec-data-model | Data Model Specification | data-model/data-model-spec.md | 50K acceptable |

**Depends on:** design-visualization-engine (W5), design-ai-integration (W8), design-remote-connectors (W8)

---

## Wave 10 (1 task)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| spec-ipc-api | IPC and API Specification | api-spec/ipc-api-spec.md | 55K acceptable |

**Depends on:** design-electron-process-model (W4), design-arrow-data-plane (W5), design-database-integration (W6), design-visualization-engine (W5), spec-data-model (W9)

---

## Wave 11 (1 task)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| integration-architecture-overview | Master Architecture Overview | design/00-architecture-overview.md | 55K acceptable |

**Depends on:** All 10 design documents, spec-data-model, spec-ipc-api

---

## Wave 12 (1 task)

| Task | Name | Output | Est. Tokens |
|------|------|--------|-------------|
| integration-cross-reference-validation | Cross-Reference Validation | validation/cross-reference-report.md | 55K acceptable |

**Depends on:** integration-architecture-overview (W11), all design docs, all ADRs, both specs

---

## Dependency Graph

```
Wave 1:  [8 research tasks] ─────────────────────────────────────────────────────
           │
           ▼
Wave 2:  [5 ADR tasks] ──────────────────────────────────────────────────────────
           │
           ▼
Wave 3:  [system-overview] ──────────────────────────────────────────────────────
           │
           ▼
Wave 4:  [electron-process-model] ───────────────────────────────────────────────
           │                          │
           ▼                          ▼
Wave 5:  [arrow-data-plane]     [visualization-engine] ──────────────────────────
           │                          │
           ▼                          │
Wave 6:  [database-integration] ──────┤──────────────────────────────────────────
           │                          │
           ▼                          │
Wave 7:  [vector-search-fts]  [local-ingestion] ─────────────────────────────────
           │                    │     │
           ▼                    ▼     │
Wave 8:  [ai-integration] [remote-connectors] ───────────────────────────────────
           │                    │     │
           ▼                    ▼     ▼
Wave 9:  [publishing-engine]  [data-model-spec] ─────────────────────────────────
                                │
                                ▼
Wave 10: [ipc-api-spec] ────────────────────────────────────────────────────────
                                │
                                ▼
Wave 11: [architecture-overview] ───────────────────────────────────────────────
                                │
                                ▼
Wave 12: [cross-reference-validation] ──────────────────────────────────────────
```

---

## Dependency Summary

| Type | Count | Confidence |
|------|-------|------------|
| artifact | 81 | HIGH |
| semantic | 0 | - |
| implicit | 0 | - |
| resource | 0 | - |
| **Total** | **81** | |

All dependencies are artifact-based with high confidence. Every dependency
is traceable to a specific input/output file path match between tasks.

---

## Context Budget Summary

| Category | Tasks | Token Range | Quality |
|----------|-------|-------------|---------|
| Peak | 12 | 8K-12K | Optimal |
| Good | 11 | 18K-40K | High |
| Acceptable | 4 | 50K-55K | Adequate |

No tasks exceed the 60K token (30% of 200K) quality threshold.

---

## Warnings

No warnings - all dependencies are high confidence artifact matches.

One minor note: `cross-reference-report.md` (Wave 12) is a terminal deliverable
not consumed by any other task. This is expected as it satisfies success criterion 8
(verification checks confirm document completeness).

---

## Execution Configuration

From `.orchestrator/config.yaml`:
- Max parallel tasks: 3
- Ralph mode: enabled (bounded retry)
- Max Ralph iterations: 3
- Checkpoints: at wave boundaries

**Note:** Wave 1 has 8 tasks but max_parallel is 3, so Wave 1 will execute
in 3 batches (3 + 3 + 2). Wave 2 will execute in 2 batches (3 + 2).

---

## Ready for Execution

All dependencies resolved. No cycles detected. 27 tasks across 12 waves.

**Artifact outputs:** All files written to `.orchestrator/artifacts/` subdirectories:
- `research/` (8 files)
- `decisions/` (5 files)
- `design/` (11 files, numbered 00-10)
- `data-model/` (1 file)
- `api-spec/` (1 file)
- `validation/` (1 file)

**Next:** Run `/ptf:execute-all` to begin execution of all waves.
