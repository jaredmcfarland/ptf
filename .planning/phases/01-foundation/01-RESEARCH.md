# Phase 1: Foundation - Research

**Researched:** 2026-01-18
**Domain:** YAML Schema Design, Claude Code Plugin Structure, Agent Skill Documentation
**Confidence:** HIGH

## Summary

Phase 1 establishes the data contracts and plugin structure for the Parallel Task Framework. Research confirms YAML with JSON Schema validation is the standard approach for configuration schemas, Claude Code has well-documented plugin and SKILL.md formats, and context budget estimation can use simple character-to-token heuristics.

Key findings:
- YAML schemas should use JSON Schema Draft 7 for validation, which provides IDE support via Schema Store
- Claude Code plugins have a specific directory structure with plugin.json in `.claude-plugin/` and all other components at root
- SKILL.md files require YAML frontmatter with `name` and `description` fields, kept under 500 lines
- Context budget estimation uses ~4 characters per token heuristic, targeting 10-30% of context window for peak performance

**Primary recommendation:** Use YAML 1.2 for all schema definitions with JSON Schema for validation. Structure the plugin following Claude Code conventions exactly. Implement context budget as heuristic-based percentage limits.

## Standard Stack

The established libraries/tools for this domain:

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| YAML | 1.2 | Schema definitions, state files | Human-readable, git-diffable, Claude Code native format |
| JSON Schema | Draft 7 | Schema validation | IDE integration via Schema Store, well-documented |
| Markdown | CommonMark | Commands, agents, skills | Claude Code native format for all prompts |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| js-yaml | 4.x | YAML parsing (if hooks need scripting) | Hook scripts parsing YAML |
| ajv | 8.x | JSON Schema validation (if runtime validation needed) | Validating plan files programmatically |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| JSON Schema | Yamale | Yamale is Python-native and simpler, but loses IDE integration |
| JSON Schema | Zod | TypeScript-first with better DX, but requires runtime code |
| YAML | JSON | More strict, but harder to read/edit for humans |

**Installation:**
Not applicable - YAML and Markdown are file formats, not dependencies. Schema validation is optional.

## Architecture Patterns

### Recommended Project Structure
```
ptf/
+-- .claude/
|   +-- commands/ptf/           # Slash commands (/ptf:init, etc.)
|   |   +-- init.md
|   |   +-- decompose.md
|   |   +-- plan.md
|   |   +-- execute.md
|   |   +-- status.md
|   |   +-- verify.md
|   |   +-- resume.md
|   |   +-- retry.md
|   |   +-- abort.md
|   |
|   +-- agents/                 # Subagent definitions
|   |   +-- ptf-decomposer.md
|   |   +-- ptf-dependency-analyzer.md
|   |   +-- ptf-task-executor.md
|   |   +-- ptf-verifier.md
|   |   +-- ptf-orchestrator.md
|   |
|   +-- skills/
|       +-- ptf/
|           +-- SKILL.md        # Framework knowledge
|
+-- adapters/                   # Domain adapters
|   +-- software-development.yaml
|   +-- research.yaml
|   +-- template.yaml
|
+-- schemas/                    # YAML schema definitions
|   +-- task.schema.yaml
|   +-- artifact.schema.yaml
|   +-- dependency.schema.yaml
|   +-- wave.schema.yaml
|   +-- plan.schema.yaml
|
+-- examples/                   # Example files
|   +-- tasks/
|   |   +-- simple-task.yaml
|   |   +-- task-with-verification.yaml
|   +-- plans/
|       +-- simple-plan.yaml
|       +-- multi-wave-plan.yaml
|
+-- hooks/
|   +-- hooks.json              # Event handlers (optional for v1)
|
+-- .orchestrator/              # Runtime state (created per project)
    +-- config.yaml
    +-- decomposition/
    +-- execution/
    +-- artifacts/
    +-- history/
```

### Pattern 1: YAML Schema with JSON Schema Validation

**What:** Define YAML schemas that can be validated against JSON Schema Draft 7.

**When to use:** All data structures that need validation (Task, Artifact, Dependency, Wave, Plan).

**Example:**
```yaml
# schemas/task.schema.yaml
$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://ptf.dev/schemas/task.schema.yaml"
title: Task
description: Atomic unit of work in the Parallel Task Framework
type: object
required:
  - id
  - name
  - description
  - outputs
  - verify
properties:
  id:
    type: string
    pattern: "^[a-z0-9-]+$"
    description: Unique identifier (lowercase, alphanumeric, hyphens)
  name:
    type: string
    maxLength: 100
    description: Human-readable name
  description:
    type: string
    description: Complete instructions for execution
  inputs:
    type: array
    items:
      $ref: "#/definitions/TaskInput"
  outputs:
    type: array
    minItems: 1
    items:
      $ref: "#/definitions/TaskOutput"
  verify:
    type: array
    minItems: 1
    items:
      $ref: "#/definitions/VerificationStep"
  context_notes:
    type: string
    description: Additional guidance for executing agent
  context_budget:
    $ref: "#/definitions/ContextBudget"
  on_failure:
    $ref: "#/definitions/FailurePolicy"

definitions:
  TaskInput:
    type: object
    required:
      - path
    properties:
      path:
        type: string
      description:
        type: string
      required:
        type: boolean
        default: true

  TaskOutput:
    type: object
    required:
      - path
    properties:
      path:
        type: string
      type:
        type: string
        enum: [source-code, config, test, migration, documentation, data]

  VerificationStep:
    type: object
    required:
      - type
      - target
    properties:
      type:
        type: string
        enum: [exists, contains, runs, syntax, custom]
      target:
        type: string
      expected:
        description: Expected result (type depends on verification type)

  ContextBudget:
    type: object
    properties:
      estimated_input_tokens:
        type: integer
        description: Estimated tokens for input files
      max_context_percentage:
        type: number
        minimum: 0
        maximum: 100
        default: 30
        description: Maximum percentage of context window to use

  FailurePolicy:
    type: object
    properties:
      strategy:
        type: string
        enum: [retry, skip, escalate]
        default: retry
      max_attempts:
        type: integer
        minimum: 1
        default: 3
```

### Pattern 2: SKILL.md with YAML Frontmatter

**What:** Agent skill documentation in Markdown with structured YAML frontmatter.

**When to use:** SKILL.md file for the PTF framework.

**Example:**
```markdown
---
name: ptf
description: Parallel Task Framework for decomposing complex goals into atomic tasks, computing dependency graphs, and orchestrating parallel execution. Use when working with PTF commands, building decomposition plans, or understanding wave-based execution.
---

# Parallel Task Framework

## Core Concepts

### Task
The atomic unit of work. A task:
- Has a unique ID and human-readable name
- Declares explicit inputs (files it needs to read)
- Declares explicit outputs (files it will produce)
- Has verification criteria (how to confirm completion)
- Fits comfortably in a fresh agent context

### Wave
A set of tasks with no interdependencies. All tasks in a wave:
- Can execute in parallel
- Have no data flow between them
- Complete before the next wave starts

### Artifact
A file produced or consumed by tasks. Artifacts enable:
- Dependency inference (B needs what A produces)
- Verification (output exists and is correct)
- Resume (know what is already done)

## File Locations

| Path | Purpose |
|------|---------|
| `.orchestrator/` | All PTF runtime state |
| `.orchestrator/decomposition/` | Goal analysis, tasks, dependencies |
| `.orchestrator/execution/` | Wave status, task results |
| `.orchestrator/artifacts/` | Artifact manifest |
| `.orchestrator/history/` | Event log (JSONL) |

## Context Budget

Tasks should target 10-30% of context window for peak quality.
Heuristic: ~4 characters = 1 token.

| Context Usage | Quality Level |
|--------------|---------------|
| 0-30% | Peak quality |
| 30-50% | Good quality |
| 50-70% | Degrading |
| 70%+ | Poor quality |

## Commands

| Command | Purpose |
|---------|---------|
| `/ptf:init [goal]` | Initialize project, run goal analysis |
| `/ptf:decompose` | Run full decomposition process |
| `/ptf:plan` | Generate human-readable plan |
| `/ptf:execute [wave]` | Execute single wave |
| `/ptf:execute-all` | Execute all waves |
| `/ptf:status` | Show execution state |
| `/ptf:verify [task]` | Verify specific task |
| `/ptf:resume` | Resume from interruption |
| `/ptf:retry [task]` | Retry failed task |
| `/ptf:abort` | Stop execution, preserve state |
```

### Pattern 3: Context Budget Heuristics

**What:** Estimate token consumption without calling the API.

**When to use:** Task schema context_budget field, planning decisions.

**Heuristic formula:**
```
estimated_tokens = file_size_chars / 4
```

**Complexity multipliers:**
| Task Complexity | Multiplier | Description |
|-----------------|------------|-------------|
| Simple | 1.0x | Read file, make small change |
| Medium | 1.5x | Multiple files, moderate reasoning |
| Complex | 2.0x | Many files, complex reasoning |

**Output artifact estimates:**
| Artifact Type | Estimated Tokens |
|---------------|------------------|
| Small file (<1KB) | 250 |
| Medium file (1-5KB) | 1,000 |
| Large file (5-20KB) | 4,000 |
| Very large (>20KB) | 8,000+ |

**Context window targets:**
| Model | Context Window | Target (30%) | Max Safe (50%) |
|-------|----------------|--------------|----------------|
| Claude Sonnet 4.5 | 200K | 60K | 100K |
| Claude Sonnet 4.5 (Enterprise) | 500K | 150K | 250K |
| Claude Sonnet 4.5 (1M beta) | 1M | 300K | 500K |

### Anti-Patterns to Avoid

- **Deeply nested YAML:** Keep nesting to 3 levels max for readability
- **Untyped fields:** Always specify types in schemas for validation
- **Vague descriptions:** Be specific in task descriptions and verification criteria
- **Hardcoded token limits:** Use percentages instead of absolute numbers
- **Missing required fields:** All schemas should clearly mark required vs optional

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| YAML validation | Custom parser | JSON Schema + ajv | IDE integration, error messages, community support |
| Schema documentation | Manual docs | JSON Schema $description | Auto-generates from schema |
| File type detection | Extension parsing | Output type enum in schema | Consistent, validated |
| Token counting (exact) | Character heuristic | Anthropic API `/messages/count_tokens` | Exact counts when needed |
| Directory structure | Custom scanner | Claude Code plugin conventions | Already implemented, documented |

**Key insight:** YAML + JSON Schema is a solved problem with excellent tooling. Focus effort on the domain-specific aspects (task decomposition, dependency inference) not basic validation.

## Common Pitfalls

### Pitfall 1: Schema Versioning Neglect

**What goes wrong:** Schemas evolve but old files become invalid.

**Why it happens:** No version field in schema, breaking changes without migration path.

**How to avoid:**
- Include `schema_version` field in all schemas
- Use semantic versioning for schema changes
- Document migration paths for breaking changes

**Warning signs:** Validation errors on previously valid files.

### Pitfall 2: Overly Strict Validation

**What goes wrong:** Valid use cases rejected by overly restrictive schemas.

**Why it happens:** Premature optimization of allowed values.

**How to avoid:**
- Start permissive, tighten based on actual problems
- Use `additionalProperties: true` initially
- Add constraints based on real validation needs

**Warning signs:** Users bypassing validation, complaints about flexibility.

### Pitfall 3: SKILL.md Bloat

**What goes wrong:** SKILL.md exceeds 500 lines, becomes ineffective.

**Why it happens:** Putting all documentation in one file.

**How to avoid:**
- Keep SKILL.md as index/overview only
- Link to reference files for details
- Use progressive disclosure pattern

**Warning signs:** Skill loading slow, Claude missing key instructions.

### Pitfall 4: Context Budget Ignored

**What goes wrong:** Tasks exceed context budget, quality degrades.

**Why it happens:** No tracking of estimated vs actual token usage.

**How to avoid:**
- Include context_budget in every task
- Warn (but proceed) when budget exceeded
- Track actual usage for heuristic refinement

**Warning signs:** Later tasks in execution showing quality degradation.

### Pitfall 5: Implicit Dependencies in Examples

**What goes wrong:** Example files show patterns that create hidden dependencies.

**Why it happens:** Examples don't make input/output declarations explicit.

**How to avoid:**
- Every example file includes full input/output declarations
- Examples demonstrate both minimal and complete patterns
- Include dependency inference examples

**Warning signs:** Users confused about what to declare.

## Code Examples

Verified patterns from official sources:

### Task Definition (Complete Example)
```yaml
# examples/tasks/complete-task.yaml
id: auth-schema
name: Create Authentication Schema
description: |
  Create the Prisma schema for user authentication including:
  - User model with email, password hash, created_at
  - Session model with token, user_id, expires_at
  - Proper indexes and relations

inputs:
  - path: prisma/schema.prisma
    description: Existing Prisma schema to extend
    required: false
  - path: docs/auth-requirements.md
    description: Authentication requirements document
    required: true

outputs:
  - path: prisma/schema.prisma
    type: source-code

verify:
  - type: exists
    target: prisma/schema.prisma
  - type: contains
    target: prisma/schema.prisma
    expected: "model User"
  - type: contains
    target: prisma/schema.prisma
    expected: "model Session"
  - type: runs
    target: "npx prisma validate"
    expected: 0

context_notes: |
  Use Prisma best practices. Include proper indexes for
  email lookups and session token queries.

context_budget:
  estimated_input_tokens: 2000
  max_context_percentage: 25

on_failure:
  strategy: retry
  max_attempts: 2
```

### Plan with Waves (Complete Example)
```yaml
# examples/plans/multi-wave-plan.yaml
id: auth-system-plan
goal: Build user authentication system with login, logout, and session management
created: 2026-01-18T10:00:00Z

analysis:
  objective: Implement complete authentication flow
  scope:
    included:
      - User registration
      - Login/logout
      - Session management
      - Password hashing
    excluded:
      - OAuth providers
      - Two-factor authentication
  constraints:
    - Must use Prisma for database
    - Must use bcrypt for password hashing
  success_criteria:
    - User can register with email/password
    - User can login and receive session token
    - User can logout and invalidate session
    - Sessions expire after 24 hours
  domain: software-development

tasks:
  - id: auth-schema
    name: Create Authentication Schema
    # ... (full task definition)

  - id: user-repository
    name: Implement User Repository
    # ... (full task definition)

  - id: auth-service
    name: Implement Auth Service
    # ... (full task definition)

  - id: auth-routes
    name: Implement Auth Routes
    # ... (full task definition)

  - id: auth-tests
    name: Write Auth Tests
    # ... (full task definition)

dependencies:
  - from: auth-schema
    to: user-repository
    type: artifact
    confidence: high
    reason: Repository needs schema to exist

  - from: user-repository
    to: auth-service
    type: artifact
    confidence: high
    reason: Service uses repository

  - from: auth-service
    to: auth-routes
    type: artifact
    confidence: high
    reason: Routes call service methods

  - from: auth-routes
    to: auth-tests
    type: artifact
    confidence: high
    reason: Tests exercise routes

waves:
  - number: 1
    tasks: [auth-schema]
    status: pending
    depends_on_waves: []

  - number: 2
    tasks: [user-repository]
    status: pending
    depends_on_waves: [1]

  - number: 3
    tasks: [auth-service]
    status: pending
    depends_on_waves: [2]

  - number: 4
    tasks: [auth-routes]
    status: pending
    depends_on_waves: [3]

  - number: 5
    tasks: [auth-tests]
    status: pending
    depends_on_waves: [4]

execution_policy:
  max_parallel: 5
  failure_strategy: escalate
  checkpoint_frequency: wave
```

### Artifact Schema
```yaml
# schemas/artifact.schema.yaml
$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://ptf.dev/schemas/artifact.schema.yaml"
title: Artifact
description: A file produced or consumed by tasks
type: object
required:
  - path
  - type
  - produced_by
properties:
  path:
    type: string
    description: Canonical file path relative to project root
  type:
    type: string
    enum: [source-code, config, test, migration, documentation, data, schema]
    description: Artifact type for verification routing
  produced_by:
    type: string
    description: Task ID that created this artifact
  produced_at:
    type: string
    format: date-time
    description: When the artifact was created
  consumed_by:
    type: array
    items:
      type: string
    description: Task IDs that read this artifact
  checksum:
    type: string
    description: SHA-256 hash of file contents
  verified:
    type: boolean
    default: false
    description: Whether artifact has passed verification
```

### Dependency Schema
```yaml
# schemas/dependency.schema.yaml
$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://ptf.dev/schemas/dependency.schema.yaml"
title: Dependency
description: Relationship where one task must complete before another
type: object
required:
  - from
  - to
  - type
properties:
  from:
    type: string
    description: Prerequisite task ID
  to:
    type: string
    description: Dependent task ID
  type:
    type: string
    enum: [artifact, semantic, resource, implicit]
    description: How the dependency was determined
  confidence:
    type: string
    enum: [high, medium, low]
    default: medium
    description: Inference confidence level
  reason:
    type: string
    description: Human-readable explanation
```

### Wave Schema
```yaml
# schemas/wave.schema.yaml
$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://ptf.dev/schemas/wave.schema.yaml"
title: Wave
description: Set of tasks that can execute in parallel
type: object
required:
  - number
  - tasks
  - status
properties:
  number:
    type: integer
    minimum: 1
    description: Wave sequence number
  tasks:
    type: array
    items:
      type: string
    minItems: 1
    description: Task IDs in this wave
  status:
    type: string
    enum: [pending, running, completed, partial, failed]
    default: pending
  depends_on_waves:
    type: array
    items:
      type: integer
    description: Wave numbers that must complete first
  started_at:
    type: string
    format: date-time
  completed_at:
    type: string
    format: date-time
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| JSON configs | YAML configs | 2020+ | Human readability, comments |
| Manual validation | JSON Schema | 2018+ | IDE integration, automation |
| Hardcoded token limits | Percentage-based | 2024+ | Model-agnostic scaling |
| Single agent context | Fresh context per task | 2024+ | Quality improvement |

**Deprecated/outdated:**
- YAML 1.1: Use YAML 1.2 (fixes boolean parsing issues like "Norway problem")
- Custom schema languages: JSON Schema has won the ecosystem
- tiktoken for Claude: Different tokenizer, use Anthropic's API or heuristics

## Open Questions

Things that couldn't be fully resolved:

1. **Exact token overhead per subagent spawn**
   - What we know: ~20k token overhead mentioned in community sources
   - What's unclear: Exact number, whether it varies by model
   - Recommendation: Budget conservatively (25k overhead), measure actual usage

2. **Schema Store integration for custom schemas**
   - What we know: JSON Schema Store provides IDE integration
   - What's unclear: Whether custom schemas can be registered
   - Recommendation: Use local schema references in VS Code settings

3. **Hook script execution context**
   - What we know: hooks.json supports command execution
   - What's unclear: Environment variables available, timeout behavior
   - Recommendation: Keep hooks simple, test thoroughly in development

## Sources

### Primary (HIGH confidence)
- [Claude Code Plugins Documentation](https://code.claude.com/docs/en/plugins) - Plugin structure, manifest, commands
- [Claude Code Agent Skills](https://code.claude.com/docs/en/skills) - SKILL.md format, frontmatter fields
- [Anthropic Token Counting API](https://platform.claude.com/docs/en/build-with-claude/token-counting) - Official token counting endpoint
- [Claude Context Windows](https://platform.claude.com/docs/en/build-with-claude/context-windows) - Context window sizes, management

### Secondary (MEDIUM confidence)
- [JSON Schema Everywhere - YAML](https://json-schema-everywhere.github.io/yaml) - JSON Schema for YAML validation
- [YAML 1.2.2 Specification](https://yaml.org/spec/1.2.2/) - Current YAML specification
- [Claude Skills Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/) - Community analysis of skill system
- [Token-Budget-Aware LLM Reasoning (ACL 2025)](https://aclanthology.org/2025.findings-acl.1274/) - Research on token budgeting

### Tertiary (LOW confidence, needs validation)
- [Kiro Feature Request for Context Monitoring](https://github.com/kirodotdev/Kiro/issues/4162) - Heuristics discussion (2-6K tokens for inline, 8-16K for chat)
- Community estimates of 4 chars per token (varies by content type)

## Metadata

**Confidence breakdown:**
- Schema design patterns: HIGH - JSON Schema well-documented, widely used
- Plugin structure: HIGH - Official Claude Code documentation
- SKILL.md format: HIGH - Official documentation + verified examples
- Context budget heuristics: MEDIUM - Based on research papers and community practice, not official guidance

**Research date:** 2026-01-18
**Valid until:** 2026-03-18 (60 days - stable domain, schemas don't change frequently)

---
*Phase: 01-foundation*
*Research completed: 2026-01-18*
