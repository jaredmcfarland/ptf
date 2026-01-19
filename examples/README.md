# PTF Examples

This directory contains examples demonstrating PTF concepts, from basic task structure to complete domain adapter demonstrations.

## Task and Plan Examples

Basic building blocks:

| File | Description |
|------|-------------|
| `tasks/simple-task.yaml` | Basic task structure with inputs/outputs |
| `tasks/task-with-verification.yaml` | Task with multi-modal verification strategies |
| `plans/simple-plan.yaml` | Single-wave plan example |
| `plans/multi-wave-plan.yaml` | Multi-wave parallel execution with dependencies |

## Dependency Analysis

Examples showing dependency inference and wave computation:

| File | Description |
|------|-------------|
| `dependency-graph-example.yaml` | Full dependency graph with 5-pass inference output |
| `plan-output-example.md` | Human-readable plan output format |

## Domain Adapter Demonstrations

End-to-end examples showing adapters in action with complete orchestrator state.

### Software Development Demo

**Location:** `software-demo/`

Demonstrates the `software-development.yaml` adapter with an authentication API goal.

| Aspect | Details |
|--------|---------|
| Goal | Build a user authentication API with JWT tokens |
| Decomposition | by-layer heuristic (schema -> repo -> service -> API) |
| Wave structure | 5 waves with dependency chain |
| Key patterns | Type safety verification, test coverage tracking |

**Files:**
- `software-demo/README.md` - Walkthrough and explanation
- `software-demo/.orchestrator/config.yaml` - Project configuration
- `software-demo/.orchestrator/decomposition/analysis.yaml` - Goal analysis
- `software-demo/.orchestrator/decomposition/constitution.yaml` - Project principles

### Research Demo

**Location:** `research-demo/`

Demonstrates the `research.yaml` adapter with a literature review goal.

| Aspect | Details |
|--------|---------|
| Goal | Literature review: What are effective strategies for reducing technical debt? |
| Decomposition | by-question heuristic (sub-questions within main question) |
| Wave structure | 4 waves with finding -> synthesis flow |
| Key patterns | Source integrity, claim-evidence binding |

**Files:**
- `research-demo/README.md` - Walkthrough and explanation
- `research-demo/.orchestrator/config.yaml` - Project configuration
- `research-demo/.orchestrator/decomposition/analysis.yaml` - Goal analysis
- `research-demo/.orchestrator/decomposition/constitution.yaml` - Research principles

## Comparing Domains

| Aspect | Software Development | Research |
|--------|---------------------|----------|
| Primary heuristic | by-layer | by-question |
| Artifact types | source-code, test, config | finding, summary, synthesis |
| Dependency pattern | schema -> repo -> service | source -> finding -> synthesis |
| Atomicity focus | single-file, type-safe | bounded-sources, cited |
| Verification | type checking, tests pass | citations present, evidence linked |

## Using These Examples

These examples are for learning and reference. To create your own project:

1. **Initialize:** Run `/ptf:init` with your goal
2. **Answer:** Respond to adapter-driven clarification questions
3. **Decompose:** Run `/ptf:decompose` to generate tasks
4. **Execute:** Run `/ptf:execute-all` to run the plan

The adapter you select (explicitly or via auto-detection) shapes the entire workflow.
