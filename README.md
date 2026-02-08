# PTF - Parallel Task Framework

A Claude Code plugin that decomposes complex goals into atomic tasks with dependency-aware wave-based parallel execution. Each task executes with fresh context for optimal LLM quality.

## Installation

```bash
# From git
claude plugins add https://github.com/jaredmcfarland/ptf

# Local development
claude --plugin-dir /path/to/ptf
```

### Prerequisites

- Claude Code CLI installed
- Claude Max subscription (for agent execution)

## Quick Start

```
/ptf:init "Build a user authentication system with JWT tokens"
/ptf:decompose
/ptf:plan
/ptf:execute-all
/ptf:status
```

## Commands

| Command | Purpose |
|---------|---------|
| `/ptf:init [goal]` | Initialize project, run goal analysis |
| `/ptf:decompose` | Break goal into atomic tasks |
| `/ptf:plan` | Generate execution plan with dependencies and waves |
| `/ptf:execute` | Execute single wave or next pending |
| `/ptf:execute-all` | Execute all waves with automatic progression |
| `/ptf:status` | Show current execution state |
| `/ptf:verify [task]` | Run verification on specific task |
| `/ptf:resume` | Resume from interruption point |
| `/ptf:retry [task]` | Retry a failed task |
| `/ptf:abort` | Stop execution, preserve state |
| `/ptf:create-adapter` | Create a custom domain adapter |

## How It Works

1. **Init** - Analyzes your goal, determines domain, asks clarifying questions, generates project constitution
2. **Decompose** - Breaks goal into subgoals, then recursively into atomic tasks (each fits in fresh context)
3. **Plan** - Infers dependencies via multi-pass analysis, computes parallel execution waves
4. **Execute** - Dispatches tasks wave-by-wave with fresh context per task, checkpoints at boundaries
5. **Verify** - Independent verification of outputs against declared criteria

### Key Innovation: Fresh Context Per Task

Every task executes in a brand-new agent context with only its declared inputs loaded. This prevents context degradation that occurs when a single agent handles many sequential tasks.

### Execution Modes

- **Classic (default)**: Wave-based execution with checkpoints at wave boundaries
- **Teams**: Dynamic scheduling via Agent Teams - tasks execute as soon as dependencies are satisfied

## Domain Adapters

PTF ships with adapters for multiple domains:

- `software-development` - Code, tests, configs, deployments
- `research` - Literature review, experiments, analysis
- `system-design` - Architecture documents, ADRs, specs
- `spec-driven-development` - Specification-first workflows
- `prediction-market` - Analysis and forecasting
- `mixpanel-analytics` - Analytics pipeline design

Create custom adapters with `/ptf:create-adapter`.

## Agents

| Agent | Role |
|-------|------|
| `ptf:orchestrator` | Coordinates wave-by-wave execution |
| `ptf:executor` | Executes single task with fresh context |
| `ptf:decomposer` | Goal analysis and recursive task breakdown |
| `ptf:dependency-analyzer` | Multi-pass dependency inference and wave computation |
| `ptf:state-manager` | Checkpoints, event logging, artifact tracking |
| `ptf:verifier` | Independent output verification |
| `ptf:team-lead` | Teams mode coordinator (dynamic scheduling) |
| `ptf:team-executor` | Teams mode task dispatcher |

## Project State

Runtime state is stored in `.orchestrator/` in your project directory:

```
.orchestrator/
  config.yaml              # Project configuration
  goal.md                  # Original goal (immutable)
  adapters/                # Domain adapter copies
  decomposition/           # Tasks, dependencies, graph
  state/                   # Execution state, checkpoints
  artifacts/manifest.yaml  # Produced artifact registry
  history/events.jsonl     # Append-only audit log
```

## License

MIT
