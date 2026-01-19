---
phase: 08-domain-adapters
plan: 03
subsystem: examples
tags: [domain-adapter, software-development, research, demo, walkthrough]

# Dependency graph
requires:
  - phase: 08-01
    provides: Adapter schema definitions and structure
provides:
  - End-to-end software development demo with auth API example
  - End-to-end research demo with literature review example
  - Examples README index for navigation
affects: [user-onboarding, documentation]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Demo READMEs explain adapter concepts through concrete examples"
    - "Complete .orchestrator state shows post-init structure"
    - "Side-by-side comparison tables highlight domain differences"

key-files:
  created:
    - examples/software-demo/README.md
    - examples/software-demo/.orchestrator/config.yaml
    - examples/software-demo/.orchestrator/decomposition/analysis.yaml
    - examples/software-demo/.orchestrator/decomposition/constitution.yaml
    - examples/research-demo/README.md
    - examples/research-demo/.orchestrator/config.yaml
    - examples/research-demo/.orchestrator/decomposition/analysis.yaml
    - examples/research-demo/.orchestrator/decomposition/constitution.yaml
    - examples/README.md
  modified: []

key-decisions:
  - "Auth API goal for software demo (realistic, matches research phase examples)"
  - "Tech debt literature review for research demo (practical research question)"
  - "Complete .orchestrator state rather than partial (shows real post-init structure)"
  - "Comparison tables in both READMEs and index (enables quick domain comparison)"

patterns-established:
  - "Demo README structure: What, How adapter shaped, Exploring files, Verification"
  - "Domain comparison: Decomposition, Artifacts, Dependencies, Atomicity, Verification"

# Metrics
duration: 2min
completed: 2026-01-18
---

# Phase 8 Plan 3: Domain Demo Examples Summary

**End-to-end demo projects for software-development and research adapters showing complete orchestrator state after /ptf:init**

## Performance

- **Duration:** 2 min
- **Started:** 2026-01-18T18:03:00Z
- **Completed:** 2026-01-18T18:05:00Z
- **Tasks:** 3
- **Files modified:** 9

## Accomplishments
- Software demo showing by-layer decomposition with auth API goal (5-wave structure)
- Research demo showing by-question decomposition with literature review goal (4-wave structure)
- Examples README index with navigation and domain comparison tables

## Task Commits

Each task was committed atomically:

1. **Task 1: Create software development demo** - `a0b0778` (feat)
2. **Task 2: Create research demo** - `ce4afe0` (feat)
3. **Task 3: Update examples README index** - `0abeb0b` (docs)

## Files Created/Modified

- `examples/software-demo/README.md` - Walkthrough explaining adapter usage and by-layer decomposition
- `examples/software-demo/.orchestrator/config.yaml` - Project config with software-development adapter
- `examples/software-demo/.orchestrator/decomposition/analysis.yaml` - Goal analysis with clarifications
- `examples/software-demo/.orchestrator/decomposition/constitution.yaml` - Domain principles
- `examples/research-demo/README.md` - Walkthrough explaining adapter usage and by-question decomposition
- `examples/research-demo/.orchestrator/config.yaml` - Project config with research adapter
- `examples/research-demo/.orchestrator/decomposition/analysis.yaml` - Goal analysis with methodology
- `examples/research-demo/.orchestrator/decomposition/constitution.yaml` - Research integrity principles
- `examples/README.md` - Index to all examples with domain comparison

## Decisions Made

- **Auth API for software demo:** Realistic goal matching previous examples, demonstrates complete dependency chain
- **Tech debt review for research:** Practical question, shows bounded-sources and synthesis flow
- **Complete orchestrator state:** Shows actual post-init structure users will see
- **Side-by-side comparisons:** Tables in README.md and demos enable quick domain understanding

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- Phase 8 (Domain Adapters) complete with all 3 plans executed
- All planned PTF components documented with examples
- Users can understand adapter system through concrete demonstrations

---
*Phase: 08-domain-adapters*
*Completed: 2026-01-18*
