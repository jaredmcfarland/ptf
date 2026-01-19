# Software Development Demo

This example demonstrates the `software-development` domain adapter in action with a complete authentication API project.

## What This Example Demonstrates

1. **Adapter-shaped questioning** - How `init_questions` from the adapter drive clarification
2. **By-layer decomposition** - The `by-layer` heuristic splitting work across architectural layers
3. **Dependency patterns** - Schema -> Repository -> Service -> API flow
4. **Verification strategies** - Type checking, existence, syntax validation

## The Goal

```
Build a user authentication API with JWT tokens
```

This is a typical software development goal that the adapter shapes into a structured execution plan.

## How the Adapter Shaped This Project

### 1. Clarification Questions (from adapter)

The adapter's `init_questions` gathered:

| Category | Question | Response |
|----------|----------|----------|
| core_value | What's the ONE thing that must work perfectly? | Token validation and refresh flow |
| existing_code | Any existing code patterns to follow? | Greenfield |
| integration_points | External services or APIs to integrate? | Database, Auth provider |
| scale | Expected complexity level? | Medium (5-15 files) |

### 2. Decomposition Strategy

The adapter's `by-layer` heuristic decomposed the goal into subgoals:

```
Subgoal 1: Data Layer
  - Task: auth-schema (Prisma schema with User, Session models)

Subgoal 2: Repository Layer
  - Task: user-repository (CRUD operations)
  - Task: session-repository (token management)

Subgoal 3: Service Layer
  - Task: auth-service (login, logout, refresh logic)

Subgoal 4: API Layer
  - Task: auth-routes (POST /login, /logout, /refresh)

Subgoal 5: Tests
  - Task: auth-tests (integration tests)
```

### 3. Resulting Wave Structure

Dependencies form a 5-wave execution plan:

```
Wave 1: [auth-schema]                    # No dependencies
Wave 2: [user-repository, session-repository]  # Depend on schema
Wave 3: [auth-service]                   # Depends on repositories
Wave 4: [auth-routes]                    # Depends on service
Wave 5: [auth-tests]                     # Depends on routes
```

Parallelism factor: 1.2x (6 tasks / 5 waves)

### 4. Atomicity Applied

Each task follows the adapter's atomicity criteria:

| Criterion | How Applied |
|-----------|-------------|
| single-file | Each task produces 1-2 files maximum |
| fresh-context-completable | Inputs declared explicitly |
| verifiable | Each has concrete verification |
| focused | One architectural concept per task |
| explicit-inputs | All dependencies declared |

## Exploring the Files

```
software-demo/
  README.md              # This walkthrough
  .orchestrator/
    config.yaml          # Project configuration
    decomposition/
      analysis.yaml      # Goal analysis with clarifications
      constitution.yaml  # Immutable project principles
```

### Key Files to Examine

**config.yaml** - References the software-development adapter:
```yaml
domain_adapter: software-development
```

**analysis.yaml** - Shows clarification responses matching adapter questions:
```yaml
clarifications:
  core_value: "Token validation and refresh flow"
  existing_code: "Greenfield"
```

**constitution.yaml** - Generated from adapter template with domain principles:
- Type Safety: Generated code must pass type checking
- Test Coverage: Logic-containing code has corresponding tests
- API Contracts: Endpoints follow existing patterns

## Dependency Graph Visualization

```
auth-schema
    |
    +---> user-repository ----+
    |                         |
    +---> session-repository --+--> auth-service --> auth-routes --> auth-tests
```

The `schema-to-repository` and `repository-to-service` patterns from the adapter's `common_patterns` section drive this structure.

## Verification Example

For the `auth-service` task, verification strategies from the adapter:

```yaml
verify:
  - type: exists
    target: src/services/auth.ts
  - type: runs
    target: "npx tsc --noEmit src/services/auth.ts"
    expected: 0
```

This follows the adapter's `source-code` verification strategy priority.

## Running This Example

This is a demonstration of adapter behavior, not a runnable project. To create a similar project:

1. Run `/ptf:init` with a software development goal
2. Answer clarification questions from the adapter
3. Run `/ptf:decompose` to generate subgoals and tasks
4. Run `/ptf:execute-all` to execute the plan

---

*Generated to demonstrate software-development.yaml adapter*
