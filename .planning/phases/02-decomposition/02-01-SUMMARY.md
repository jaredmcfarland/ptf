---
phase: 02-decomposition
plan: 01
subsystem: adapters
tags: [yaml, domain-adapters, decomposition, questioning, constitution]

# Dependency graph
requires:
  - phase: 01-foundation
    provides: Schema definitions, directory structure, SKILL.md framework concepts
provides:
  - Domain adapter loading interface
  - Software development decomposition heuristics
  - Research domain decomposition heuristics
  - Template for custom adapter creation
  - Constitution templates per domain
affects: [02-02, 02-03, 03-dependency, 04-execution]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Domain adapter YAML structure (questioning, decomposition, constitution, artifacts, dependencies)"
    - "Adapter-driven questioning for /ptf:init"
    - "Heuristic-based subgoal identification"
    - "Atomicity criteria checklist pattern"

key-files:
  created:
    - adapters/software-development.yaml
    - adapters/research.yaml
    - adapters/template.yaml
  modified: []

key-decisions:
  - "4 questioning categories per adapter (core domain, constraints, methodology, scope)"
  - "4+ subgoal heuristics per domain (by-layer, by-feature, by-interface for software)"
  - "5 atomicity criteria as checklist pattern (criterion, check, fail_signal)"
  - "Constitution template with {project_name}, {project_constraints}, {timestamp} placeholders"
  - "Template adapter uses [CUSTOMIZE] markers for self-documentation"

patterns-established:
  - "Adapter structure: header, questioning, decomposition, constitution, artifacts, dependencies"
  - "Heuristic pattern: name, description, examples, when_to_use"
  - "Atomicity pattern: criterion, check, fail_signal"
  - "Verification strategy per artifact type"

# Metrics
duration: 2m 51s
completed: 2026-01-18
---

# Phase 02 Plan 01: Domain Adapters Summary

**YAML domain adapters for software-development and research with questioning patterns, decomposition heuristics, atomicity criteria, and constitution templates**

## Performance

- **Duration:** 2m 51s
- **Started:** 2026-01-18T23:00:25Z
- **Completed:** 2026-01-18T23:03:16Z
- **Tasks:** 3
- **Files created:** 3

## Accomplishments

- Software development adapter with 4 heuristics, 5 atomicity criteria, and complete artifact verification
- Research adapter with domain-specific heuristics (by-question, by-source, by-stage, by-synthesis)
- Template adapter with 40 [CUSTOMIZE] markers for self-documenting custom adapter creation
- All adapters validate as proper YAML

## Task Commits

Each task was committed atomically:

1. **Task 1: Create software-development adapter** - `af5955e` (feat)
2. **Task 2: Create research adapter** - `7e91f0d` (feat)
3. **Task 3: Create template adapter** - `6ac6ff8` (docs)

## Files Created

- `adapters/software-development.yaml` - Full software domain adapter with by-layer, by-feature, by-file-boundary, by-interface heuristics
- `adapters/research.yaml` - Research domain adapter with finding, summary, synthesis artifact types
- `adapters/template.yaml` - Self-documenting template with [CUSTOMIZE] markers

## Decisions Made

1. **Questioning categories standardized** - Each adapter has 4 questioning categories aligned with domain-specific needs (software: core_value, existing_code, integration_points, scale; research: research_question, existing_knowledge, methodology, scope_bounds)

2. **Atomicity criteria as checklist** - Using criterion/check/fail_signal structure enables automated validation during decomposition

3. **Research-specific artifact types** - Added finding, summary, synthesis, bibliography types distinct from software's source-code, migration, test pattern

4. **40 CUSTOMIZE markers in template** - Ensures template is self-documenting and guides custom adapter creation

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None - all tasks completed without blockers.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

Ready for Phase 02-02 (/ptf:init command):
- Adapters provide `questioning.init_questions` for goal clarification
- Adapters provide `constitution.template` for principle generation
- Adapter loading pattern: `adapters/{domain}.yaml`

Blockers: None

---
*Phase: 02-decomposition*
*Completed: 2026-01-18*
