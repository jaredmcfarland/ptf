# Phase 1: Foundation - Context

**Gathered:** 2025-01-18
**Status:** Ready for planning

<domain>
## Phase Boundary

Establish data contracts and plugin structure that all subsequent phases depend on. Deliverables: YAML schemas (Task, Artifact, Dependency, Wave, Plan), plugin directory structure, SKILL.md documentation, and example files demonstrating schema usage.

</domain>

<decisions>
## Implementation Decisions

### Context Budget Modeling
- Token estimation uses **heuristics** (not explicit author declaration)
- Heuristic inputs: file sizes + task complexity multipliers + output artifact estimates + retry/failure history context
- Limits expressed as **percentage of context window** (max_context_percentage field) — adapts to model's context size
- Overflow behavior: **warn and proceed** — log warning but attempt execution anyway

### Claude's Discretion
- Schema field naming conventions
- Validation strictness levels
- Plugin directory organization specifics
- SKILL.md structure and depth
- Example file complexity

</decisions>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches for schema design, plugin structure, and documentation.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-foundation*
*Context gathered: 2025-01-18*
