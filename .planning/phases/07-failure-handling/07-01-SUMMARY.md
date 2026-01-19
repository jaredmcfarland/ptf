---
phase: 07-failure-handling
plan: 01
subsystem: execution
tags: [retry, backoff, cascade, failure-handling, orchestration]

# Dependency graph
requires:
  - phase: 05-execution-engine
    provides: Base orchestrator and state manager agents
  - phase: 06-verification
    provides: Verification workflow foundation
provides:
  - Extended FailurePolicy schema with backoff configuration
  - Task state tracking for blocked/skipped status
  - Failure record creation for debugging
  - Cascade failure handling via propagate_failure
  - Human escalation workflow
affects: [07-02-recovery-commands, 08-human-escalation]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Exponential backoff with configurable base and cap"
    - "Cascade failure propagation to dependent tasks"
    - "Failure records in .orchestrator/failures/"

key-files:
  modified:
    - schemas/task.schema.yaml
    - schemas/task-state.schema.yaml
    - .claude/agents/ptf-state-manager.md
    - .claude/agents/ptf-orchestrator.md

key-decisions:
  - "Exponential backoff as default (2^(attempt-1) * base)"
  - "propagate_failure defaults to true (block dependents)"
  - "final_fallback determines action when max_attempts exhausted"
  - "Failure records stored in .orchestrator/failures/ for debugging"

patterns-established:
  - "BackoffState: attempts_remaining, next_retry_at, last_backoff_seconds"
  - "Cascade handling: mark_blocked with blocked_by array"
  - "ESCALATION REQUIRED structured return for human intervention"

# Metrics
duration: 3min
completed: 2026-01-18
---

# Phase 7 Plan 1: Core Failure Handling Summary

**Retry strategies with exponential backoff, skip/escalate fallbacks, and cascade failure propagation to dependent tasks**

## Performance

- **Duration:** 3 min
- **Started:** 2026-01-18T18:00:00Z
- **Completed:** 2026-01-18T18:03:00Z
- **Tasks:** 3
- **Files modified:** 4

## Accomplishments
- Extended FailurePolicy with backoff_type, base/max seconds, propagate_failure, final_fallback
- Added blocked/skipped status and BackoffState to task state schema
- Implemented mark_blocked, create_failure_record, mark_skipped state manager operations
- Added calculate_backoff function and handle_failure operation to orchestrator
- Created ESCALATION REQUIRED structured return for human intervention

## Task Commits

Each task was committed atomically:

1. **Task 1: Extend schemas for failure handling** - `e234e79` (feat)
2. **Task 2: Add state manager failure operations** - `1c1bc10` (feat)
3. **Task 3: Implement failure handling in orchestrator** - `9798032` (feat)

## Files Created/Modified
- `schemas/task.schema.yaml` - Extended FailurePolicy with backoff and cascade settings
- `schemas/task-state.schema.yaml` - Added blocked_by, failure_record, BackoffState
- `.claude/agents/ptf-state-manager.md` - Added mark_blocked, create_failure_record, mark_skipped
- `.claude/agents/ptf-orchestrator.md` - Added calculate_backoff, handle_failure, escalation

## Decisions Made
- Exponential backoff formula: 2^(attempt-1) * base_seconds, capped at max_seconds
- Default backoff: 2s base, 60s max (yields 2s, 4s, 8s, 16s, 32s, 60s sequence)
- propagate_failure: true by default (cascade to dependents)
- final_fallback: escalate by default (human intervention when retries exhausted)
- Failure records stored per-attempt: .orchestrator/failures/{task-id}-attempt-{N}.yaml

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness
- Failure handling infrastructure complete
- Ready for 07-02: Recovery commands (/ptf:retry, /ptf:skip, /ptf:abort)
- Escalation presentation ready for human interaction

---
*Phase: 07-failure-handling*
*Completed: 2026-01-18*
