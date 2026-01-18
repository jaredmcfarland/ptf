# Stack Research

**Domain:** Claude Code plugin for LLM agent orchestration
**Researched:** 2026-01-18
**Confidence:** HIGH (verified against official Claude Code documentation)

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **Claude Code Plugin** | Current | Runtime environment, agent execution | The plugin system IS the runtime. No separate framework needed — Claude Code provides Task tool, subagents, file access, and orchestration primitives. [Official docs](https://code.claude.com/docs/en/plugins) |
| **Markdown + YAML Frontmatter** | — | Commands, agents, skills, configuration | Claude Code's native format. Commands are `.md` files with YAML frontmatter. Skills are `SKILL.md`. Agents are markdown with frontmatter. This is non-negotiable. |
| **YAML** | 1.2 | Schema definitions, state files, plans | Human-readable, git-diffable, supported natively by Claude Code. Framework state persists in `.orchestrator/` as YAML. |
| **JSONL** | — | Event logs, execution history | Append-only format for event streams. Claude Code uses JSONL for conversation logs. Match the ecosystem pattern. |

### Plugin Directory Structure

This is the **required structure** for Claude Code plugins:

```
ptf/
├── .claude-plugin/
│   └── plugin.json          # REQUIRED: Plugin manifest (name, version, description)
├── commands/                 # Slash commands (ptf:init, ptf:decompose, etc.)
│   ├── init.md
│   ├── decompose.md
│   ├── plan.md
│   ├── execute.md
│   ├── status.md
│   ├── resume.md
│   └── ...
├── agents/                   # Subagents for specialized work
│   ├── ptf-decomposer.md
│   ├── ptf-dependency-analyzer.md
│   ├── ptf-orchestrator.md
│   ├── ptf-executor.md
│   └── ptf-verifier.md
├── skills/
│   └── ptf/
│       └── SKILL.md          # Auto-invoked framework knowledge
├── hooks/
│   └── hooks.json            # Event handlers (post-task, pre-wave, etc.)
├── adapters/                 # Domain adapters (software, research)
│   ├── software/
│   │   └── ADAPTER.md
│   └── research/
│       └── ADAPTER.md
├── schemas/                  # YAML schema definitions
│   ├── task.yaml
│   ├── artifact.yaml
│   ├── dependency.yaml
│   ├── wave.yaml
│   └── plan.yaml
└── README.md
```

**CRITICAL:** Only `plugin.json` goes inside `.claude-plugin/`. All other directories must be at plugin root.

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| **Zod** | 3.x | YAML schema validation (if TypeScript scripts needed) | Validating plan schemas, task definitions. TypeScript-first with excellent error messages. [Best for 2025](https://dev.to/dataformathub/zod-vs-yup-vs-typebox-the-ultimate-schema-validation-guide-for-2025-1l4l) |
| **yaml** | 2.x | YAML parsing in scripts | If hooks need to parse YAML programmatically |
| **chalk** | 5.x | Terminal output formatting | Status displays, progress indicators in hook scripts |

**Note:** Most PTF logic lives in markdown prompts, not TypeScript. Libraries are only needed for hook scripts.

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `claude --plugin-dir ./ptf` | Local plugin testing | Loads plugin without installation |
| `claude --debug` | Debug skill/command loading | Shows which files are loaded and errors |
| `/agents` command | Verify agent registration | Lists registered subagents |
| `git worktree` | Parallel testing | Test PTF with multiple Claude instances |

## File Format Specifications

### plugin.json (REQUIRED)

```json
{
  "name": "ptf",
  "description": "Parallel Task Framework - decompose goals into atomic tasks with dependency-aware parallel execution",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  },
  "repository": "https://github.com/user/ptf",
  "license": "MIT"
}
```

The `name` field becomes the command namespace: `/ptf:init`, `/ptf:execute`, etc.

### Slash Command Format

```markdown
---
description: Initialize a PTF project with goal analysis
---

# PTF Init

Initialize the Parallel Task Framework for a new goal.

## Arguments

`$ARGUMENTS` contains the user's goal description.

## Process

1. Create `.orchestrator/` directory structure
2. Run goal analysis (invoke ptf-decomposer agent)
3. Write `analysis.yaml` with structured goal
4. Report initialization status

## Example

User runs: `/ptf:init Build a REST API for user management`

## Output

Creates:
- `.orchestrator/config.yaml` (framework configuration)
- `.orchestrator/decomposition/analysis.yaml` (goal analysis)
```

### Subagent Format

```markdown
---
name: ptf-decomposer
description: Decomposes goals into atomic tasks using hierarchical analysis
tools: Read, Write, Bash, Grep, Glob
color: blue
---

<role>
You are the PTF Decomposer agent. Your job is to break down complex goals
into atomic, parallel-executable tasks.
</role>

<process>
1. Read the goal analysis from `.orchestrator/decomposition/analysis.yaml`
2. Identify subgoals using domain adapter heuristics
3. Recursively decompose until atomic
4. Validate completeness (100% rule)
5. Write tasks to `.orchestrator/decomposition/tasks.yaml`
</process>

<atomicity_criteria>
A task is atomic when:
- It can be fully understood from its description alone
- It fits comfortably in a fresh agent context
- It produces one or more coherent file artifacts
- It has clear, verifiable completion criteria
- It cannot be meaningfully subdivided further
</atomicity_criteria>
```

### SKILL.md Format

```yaml
---
name: ptf
description: Parallel Task Framework for decomposing complex goals into atomic tasks, computing dependency graphs, and orchestrating parallel execution. Use when building PTF commands, agents, or adapters.
allowed-tools: Read, Write, Bash, Grep, Glob
---

# Parallel Task Framework

## Core Concepts

### Task
The atomic unit of work...

### Wave
A set of tasks that can execute in parallel...

## File Locations

- `.orchestrator/` - All PTF state
- `.orchestrator/decomposition/` - Goal analysis, tasks, dependencies
- `.orchestrator/execution/` - Wave status, task results
- `.orchestrator/artifacts/` - Artifact manifest
```

### hooks.json Format

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "python scripts/register_artifact.py $FILE"
          }
        ]
      }
    ]
  }
}
```

**Available hook events:**
- `PreToolUse` - Before tool execution (can block)
- `PostToolUse` - After tool execution
- `PermissionRequest` - When permission dialog shown
- `Stop` - When agent stops

## State Persistence Pattern

### Directory Structure

```
.orchestrator/
├── config.yaml                    # Framework configuration
├── state.yaml                     # Current execution state
├── decomposition/
│   ├── analysis.yaml              # Goal analysis (Step 1)
│   ├── subgoals.yaml              # Subgoal identification (Step 2)
│   ├── tasks.yaml                 # Atomic tasks (Step 3-4)
│   └── dependencies.yaml          # Dependency graph (Step 5)
├── execution/
│   ├── plan.yaml                  # Execution plan with waves
│   ├── waves/
│   │   ├── wave-01.yaml           # Wave 1 status
│   │   └── wave-02.yaml           # Wave 2 status
│   └── tasks/
│       ├── task-001.yaml          # Individual task execution record
│       └── task-002.yaml
├── artifacts/
│   └── manifest.yaml              # Produced artifact registry
└── logs/
    └── events.jsonl               # Append-only event log
```

### state.yaml Schema

```yaml
# Current framework state
created: 2026-01-18T10:00:00Z
updated: 2026-01-18T10:30:00Z

goal: "Build a REST API for user management"
domain: software-development

phase: execution  # decomposition | planning | execution | complete
current_wave: 2
total_waves: 5

progress:
  tasks_total: 24
  tasks_complete: 8
  tasks_failed: 0
  tasks_pending: 16

last_checkpoint:
  wave: 1
  timestamp: 2026-01-18T10:25:00Z

resume_point:
  wave: 2
  task: null  # Start of wave
```

### events.jsonl Format

```jsonl
{"type":"wave_start","wave":1,"timestamp":"2026-01-18T10:00:00Z"}
{"type":"task_start","task":"auth-schema","wave":1,"timestamp":"2026-01-18T10:00:01Z"}
{"type":"task_complete","task":"auth-schema","wave":1,"duration_ms":45000,"timestamp":"2026-01-18T10:00:46Z"}
{"type":"task_start","task":"user-schema","wave":1,"timestamp":"2026-01-18T10:00:02Z"}
{"type":"task_complete","task":"user-schema","wave":1,"duration_ms":38000,"timestamp":"2026-01-18T10:00:40Z"}
{"type":"wave_complete","wave":1,"timestamp":"2026-01-18T10:00:46Z"}
```

## Claude Code Specific Capabilities

### Task Tool (Parallel Execution)

**Parallelism cap:** 10 concurrent tasks maximum. Additional tasks queue.

**Best practice:** Use Task tool for parallel **read** operations and independent work. Avoid parallel writes to overlapping files.

```markdown
## Parallel Execution Pattern

When executing Wave 2 with tasks [A, B, C, D]:

1. Spawn Task agents for each:
   - Task A: "Execute task A per spec in .orchestrator/execution/tasks/task-A.yaml"
   - Task B: "Execute task B per spec..."
   - Task C: ...
   - Task D: ...

2. Collect results from all tasks
3. Verify each task's outputs
4. Update wave status
5. Proceed to Wave 3 (if all passed)
```

**Token overhead:** Each Task spawns with ~20k token context overhead. Budget accordingly.

### Subagent Invocation

Subagents can be invoked:
1. **By keyword** - Claude auto-selects based on description match
2. **By explicit @mention** - `@ptf-decomposer`
3. **Via Task tool** - Spawn as parallel worker

**Subagent context:** Each subagent gets fresh context. This is the core mechanism enabling fresh context execution.

### Skills Auto-Loading

Skills auto-load based on description matching:

1. Claude loads only `name` and `description` at startup
2. When user request matches description, Claude asks to use skill
3. Full `SKILL.md` loads and Claude follows instructions

**Keep SKILL.md under 500 lines.** Use progressive disclosure — link to reference files for details.

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Claude Code Plugin | Standalone Python/Node runtime | Never for PTF. Claude Code IS the runtime. Standalone would duplicate execution logic. |
| YAML for schemas | JSON Schema | If you need strict validation with JSON Schema tooling (OpenAPI). YAML is more human-friendly for PTF's markdown-native ecosystem. |
| File-based state | SQLite / Beads | If you need complex queries across state. File-based is simpler, git-friendly, and sufficient for v1. Claude-Flow uses SQLite for persistent memory in v3. |
| Zod (if needed) | AJV + TypeBox | If you need 5-18x faster validation or JSON Schema interoperability. Zod's DX wins for plugin development. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **External orchestration frameworks** (LangGraph, AutoGen, CrewAI) | Claude Code already provides orchestration primitives. External frameworks add complexity, dependencies, and context overhead. | Native Claude Code Task tool + subagents |
| **In-memory state** | Lost on interruption. Claude Code sessions can end unexpectedly. | File-based state in `.orchestrator/` |
| **JSON for human-edited files** | Harder to read/edit, no comments, less git-diff friendly | YAML for configuration and state |
| **Deeply nested plugin directories** | Claude Code expects flat structure at plugin root | Flat structure: `commands/`, `agents/`, `skills/` at root |
| **`git add .` in hooks** | Stages unintended files, violates atomicity | Stage files individually |
| **Parallel writes to same file** | Race conditions, data loss | Partition files by task, merge in orchestrator |
| **Complex hook scripts** | Hooks should be fast and deterministic. Complex logic belongs in agents. | Move complexity to agents, keep hooks simple |

## Stack Patterns by Variant

**If building software adapter:**
- Include file-type verification (TypeScript, Python, etc.)
- Add test execution hooks
- Include common patterns (API, database, frontend)

**If building research adapter:**
- Include source type handlers (papers, websites, data)
- Add citation tracking
- Include synthesis patterns

**If extending with new domain:**
- Create adapter directory in `adapters/domain-name/`
- Provide `ADAPTER.md` with domain-specific decomposition heuristics
- Define artifact types and verification strategies

## Version Compatibility

| Component | Compatible With | Notes |
|-----------|-----------------|-------|
| Plugin format | Claude Code current | Plugin system in public beta as of late 2025 |
| YAML frontmatter | Markdown files | Standard format, no version constraints |
| hooks.json | PostToolUse, PreToolUse | Check official docs for new hook types |
| Task tool | 10 concurrent max | May increase in future versions |

## Confidence Assessment

| Recommendation | Confidence | Basis |
|----------------|------------|-------|
| Plugin directory structure | HIGH | [Official Claude Code docs](https://code.claude.com/docs/en/plugins) |
| YAML/Markdown formats | HIGH | Native Claude Code format, verified |
| Task tool parallelism cap | HIGH | [Multiple sources](https://claudelog.com/mechanics/task-agent-tools/) confirm 10-task limit |
| File-based state pattern | MEDIUM | Recommended pattern, but SQLite alternative exists (claude-flow) |
| Zod for validation | MEDIUM | Best for TypeScript DX, but may not need validation scripts |
| hooks.json format | HIGH | [Official docs](https://code.claude.com/docs/en/plugins) |

## Sources

- [Claude Code Plugins Documentation](https://code.claude.com/docs/en/plugins) — Plugin structure, manifest, commands, hooks
- [Claude Code Agent Skills](https://code.claude.com/docs/en/skills) — SKILL.md format, frontmatter fields, progressive disclosure
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices) — Agentic coding workflows, CLAUDE.md patterns
- [Task Tool Best Practices](https://claudelog.com/mechanics/task-agent-tools/) — Parallelism limits, token overhead
- [Claude Code Subagent Deep Dive](https://cuong.io/blog/2025/06/24-claude-code-subagent-deep-dive) — Subagent invocation patterns
- [Zod vs TypeBox Guide 2025](https://dev.to/dataformathub/zod-vs-yup-vs-typebox-the-ultimate-schema-validation-guide-for-2025-1l4l) — Schema validation comparison
- [Claude-Flow](https://github.com/ruvnet/claude-flow) — Reference implementation for multi-agent orchestration (alternative approach)

---
*Stack research for: PTF (Parallel Task Framework) - Claude Code Plugin*
*Researched: 2026-01-18*
