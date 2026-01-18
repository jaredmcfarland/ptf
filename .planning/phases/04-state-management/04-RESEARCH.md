# Phase 4: State Management - Research

**Researched:** 2026-01-18
**Domain:** File-based state persistence, checkpoints, resume capability
**Confidence:** HIGH

## Summary

State management in PTF enables reliable persistence of execution state, session resumption, and audit trails. The framework uses file-based persistence following established patterns from event sourcing and workflow orchestration systems.

Research confirms the founding document's design is well-aligned with industry patterns:
- **Event sourcing** via JSONL append-only logs (events.jsonl) matches Temporal.io, Dapr Workflows, and Durable Task Framework patterns
- **Checkpoint-at-wave-boundaries** aligns with durable execution best practices
- **Master state file** (execution.yaml) provides single-source-of-truth for "where are we?"
- **Artifact manifest** enables resume validation and integrity checking

The key insight from modern orchestration systems: **deterministic replay from events** is the foundation of reliable resume. Rather than complex snapshot logic, simply replay the event log to reconstruct state.

**Primary recommendation:** Implement state management as described in the founding document (Section 7), with minor enhancements for idempotency validation and structured status output.

## Standard Stack

The established approach for this domain (file-based state in orchestration systems):

### Core
| Technology | Purpose | Why Standard |
|------------|---------|--------------|
| YAML | Structured state files (execution.yaml, task states, wave states) | Human-readable, editable, established in PTF |
| JSONL | Event log (events.jsonl) | Append-only, line-level atomicity, easy to parse/grep |
| SHA-256 | Artifact checksums | Standard integrity verification |
| ISO 8601 | Timestamps | Unambiguous, sortable, timezone-aware |

### Supporting
| Technology | Purpose | When to Use |
|------------|---------|-------------|
| YAML frontmatter | Metadata in .md files | For human-readable state documents |
| File locking (flock) | Concurrent write protection | If multiple agents write same file |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| JSONL events | SQLite | DB adds complexity, JSONL simpler for append-only |
| YAML state | JSON state | JSON less readable, YAML already used throughout |
| File-based | Beads | Beads deferred to v2 per prior decision |

**No additional dependencies needed.** PTF already uses YAML; JSONL is just text files with JSON lines.

## Architecture Patterns

### Recommended Directory Structure

The founding document specifies this structure (Section 7.2):

```
.orchestrator/
├── config.yaml                    # Framework configuration
├── goal.md                        # Original goal (immutable)
│
├── decomposition/
│   ├── analysis.yaml              # Step 1: Analyzed goal
│   ├── subgoals.yaml              # Step 2: Identified subgoals
│   ├── tasks.yaml                 # Step 3: All atomic tasks
│   ├── validation.yaml            # Step 4: Validation results
│   └── graph.yaml                 # Step 5: Dependencies + waves
│
├── plan.md                        # Human-readable plan (generated)
│
├── state/
│   ├── execution.yaml             # Master state file (STATUS SOURCE)
│   ├── waves/
│   │   ├── wave-1.yaml            # Per-wave state
│   │   └── wave-{N}.yaml
│   └── tasks/
│       ├── {task-id}.yaml         # Per-task state
│       └── ...
│
├── artifacts/
│   └── manifest.yaml              # Registry of produced artifacts
│
├── history/
│   ├── events.jsonl               # Append-only event log
│   └── sessions/
│       └── session-{N}.yaml       # Session metadata
│
└── failures/
    └── {task-id}-attempt-{N}.yaml # Detailed failure records
```

### Pattern 1: Event Sourcing for History

**What:** All state changes are recorded as timestamped events in append-only log.

**When to use:** Always - this is the foundation of reliable resume and audit.

**Example:**
```jsonl
{"ts":"2026-01-18T10:30:00Z","event":"goal_received","goal_hash":"abc123"}
{"ts":"2026-01-18T10:32:00Z","event":"execution_started","session":"session-001"}
{"ts":"2026-01-18T10:32:00Z","event":"wave_started","wave":1}
{"ts":"2026-01-18T10:32:01Z","event":"task_started","task":"auth-schema","agent":"agent-001"}
{"ts":"2026-01-18T10:35:00Z","event":"task_completed","task":"auth-schema","duration_s":179}
{"ts":"2026-01-18T10:35:00Z","event":"artifact_produced","path":"prisma/schema.prisma","checksum":"sha256:..."}
{"ts":"2026-01-18T10:35:00Z","event":"wave_completed","wave":1}
```

**Key properties:**
- Append-only: Never modify existing lines
- Line-level atomicity: Each line is complete event
- Replayable: Can reconstruct state from events
- Greppable: Easy to filter and analyze

### Pattern 2: Master State File for "Where Are We?"

**What:** Single file (execution.yaml) is authoritative source for current state.

**When to use:** Reading state for /ptf:status or resume decisions.

**Example:**
```yaml
# .orchestrator/state/execution.yaml
status: running  # pending | running | paused | completed | failed

current_wave: 2
waves_total: 5

progress:
  tasks_total: 12
  tasks_completed: 3
  tasks_running: 3
  tasks_pending: 6
  tasks_failed: 0
  tasks_blocked: 0

wave_summary:
  1: completed
  2: running
  3: pending
  4: pending
  5: pending

session:
  id: session-003
  started: 2026-01-18T10:32:00Z
  last_update: 2026-01-18T10:35:22Z

blockers: []

recent_events:
  - timestamp: 2026-01-18T10:35:22Z
    event: task_completed
    task: auth-schema
```

### Pattern 3: Checkpoint at Wave Boundaries

**What:** Persist all state atomically after each wave completes.

**When to use:** Wave completion, before starting next wave.

**Protocol:**
1. All tasks in wave complete (success/fail)
2. Write all task state files
3. Write wave state file
4. Update artifact manifest
5. Append events to log
6. Update execution.yaml (last, serves as commit marker)

**Why atomic update order matters:** If interrupted mid-checkpoint:
- If execution.yaml not updated: Resume will re-checkpoint (idempotent)
- If execution.yaml updated: State is consistent

### Pattern 4: Artifact Manifest for Resume Validation

**What:** Track all produced artifacts with checksums for integrity verification.

**When to use:** Resume validation - verify artifacts before skipping completed tasks.

**Example:**
```yaml
# .orchestrator/artifacts/manifest.yaml
artifacts:
  - path: prisma/schema.prisma
    type: source-code
    produced_by: auth-schema
    produced_at: 2026-01-18T10:35:00Z
    verified: true
    checksum: sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
    consumed_by: [user-repository, session-repository]

  - path: src/repositories/user.ts
    type: source-code
    produced_by: user-repository
    produced_at: 2026-01-18T10:40:00Z
    verified: true
    checksum: sha256:abc123...
    consumed_by: [auth-service]

pending:
  - path: src/services/auth.ts
    expected_from: auth-service
    needed_by: [auth-routes]
```

### Anti-Patterns to Avoid

- **Memory-only state:** All state must persist to files. Claude Code may restart at any time.
- **Snapshot-only (no events):** Without event log, debugging and audit are impossible.
- **Global file locks:** Use task-partitioned writes instead (each agent owns its task file).
- **Mutable event log:** Events are append-only. Never edit/delete existing events.
- **Complex recovery logic:** Rely on idempotent operations and checkpoint replay.

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Timestamp formatting | Custom date strings | ISO 8601 | Sortable, unambiguous, standard |
| File integrity | Custom hashing | SHA-256 | Battle-tested, universal support |
| Concurrent writes | Custom locking | Task-partitioned ownership | Simpler, no deadlocks |
| State reconstruction | Complex snapshot merge | Event replay | Deterministic, debuggable |
| Progress display | Custom formatting | Structured YAML + presenter | Separation of concerns |

**Key insight:** File-based state with event sourcing is a well-understood pattern. The main complexity is getting the write order correct for atomic checkpoints.

## Common Pitfalls

### Pitfall 1: Incomplete Checkpoint Writes

**What goes wrong:** Interruption during checkpoint leaves state inconsistent.

**Why it happens:** Writing multiple files is not atomic.

**How to avoid:**
- Write files in dependency order (details before summary)
- Update execution.yaml last (serves as commit marker)
- On resume, detect incomplete checkpoint and re-run

**Warning signs:** execution.yaml shows wave N but task files show wave N-1 states.

### Pitfall 2: Resume Without Validation

**What goes wrong:** Resume assumes artifacts exist but they don't (or are corrupted).

**Why it happens:** Task marked complete but output was deleted/modified.

**How to avoid:**
- Validate artifact checksums before skipping completed tasks
- Re-execute tasks with missing/invalid outputs
- Log validation failures as events

**Warning signs:** Downstream tasks fail because inputs don't exist.

### Pitfall 3: Non-Idempotent Operations

**What goes wrong:** Retry/resume causes duplicate side effects.

**Why it happens:** Operation wasn't designed for re-execution.

**How to avoid:**
- Design all state writes as idempotent (write full state, not delta)
- Check existing state before writing
- Use artifact checksums as idempotency markers

**Warning signs:** Duplicate entries in manifest, duplicate events.

### Pitfall 4: Status Output Clutters CI Logs

**What goes wrong:** Progress bars and spinners create unreadable logs.

**Why it happens:** Interactive formatting in non-interactive context.

**How to avoid:**
- Detect TTY vs pipe/redirect
- Use static output format for CI (no animations)
- Support --plain flag for machine-readable output

**Warning signs:** Christmas tree logs, broken escape sequences.

### Pitfall 5: Events Without Sufficient Context

**What goes wrong:** Event log exists but can't diagnose issues.

**Why it happens:** Events too minimal to reconstruct what happened.

**How to avoid:**
- Include relevant IDs (task, wave, session)
- Include duration for timing analysis
- Include error details for failures
- Include artifact paths for productions

**Warning signs:** "Something failed" without knowing what or why.

## Code Examples

### Event Log Writing (JSONL)

```typescript
// Source: Event sourcing pattern - append-only log
interface Event {
  ts: string;         // ISO 8601 timestamp
  event: string;      // Event type
  [key: string]: any; // Event-specific data
}

function appendEvent(logPath: string, event: Omit<Event, 'ts'>): void {
  const entry: Event = {
    ts: new Date().toISOString(),
    ...event
  };
  // Append single line - atomic at filesystem level
  fs.appendFileSync(logPath, JSON.stringify(entry) + '\n');
}

// Usage
appendEvent('.orchestrator/history/events.jsonl', {
  event: 'task_completed',
  task: 'auth-schema',
  duration_s: 179,
  outputs: ['prisma/schema.prisma']
});
```

### Checkpoint Protocol

```typescript
// Source: Founding document Section 7
async function checkpointWave(wave: number, results: TaskResult[]): Promise<void> {
  // 1. Write task state files (details first)
  for (const result of results) {
    await writeTaskState(result.task.id, {
      status: result.success ? 'completed' : 'failed',
      attempts: result.attempts,
      outputs_produced: result.artifacts,
      verification: result.verification
    });
  }

  // 2. Write wave state file
  await writeWaveState(wave, {
    status: results.every(r => r.success) ? 'completed' : 'partial',
    completed_at: new Date().toISOString(),
    tasks: results.map(r => ({
      id: r.task.id,
      status: r.success ? 'completed' : 'failed'
    }))
  });

  // 3. Update artifact manifest
  for (const result of results.filter(r => r.success)) {
    await registerArtifacts(result.artifacts);
  }

  // 4. Append events to log
  appendEvent(eventsPath, { event: 'wave_completed', wave });

  // 5. Update execution.yaml LAST (commit marker)
  await updateExecutionState({
    current_wave: wave + 1,
    wave_summary: { [wave]: 'completed' },
    last_update: new Date().toISOString()
  });
}
```

### Resume Protocol

```typescript
// Source: Founding document Section 7.6
async function resume(): Promise<ResumeAction> {
  // 1. Read master state
  const execution = await readYaml('.orchestrator/state/execution.yaml');

  // 2. Handle completed/failed states
  if (execution.status === 'completed') {
    return { action: 'none', reason: 'Already completed' };
  }

  if (execution.status === 'failed') {
    return { action: 'prompt_user', reason: execution.blockers };
  }

  // 3. Validate artifacts for "completed" tasks
  const completedTasks = await getCompletedTasks(execution.current_wave);
  for (const task of completedTasks) {
    const valid = await validateArtifacts(task);
    if (!valid) {
      // Mark for re-execution
      await markTaskReady(task.id);
      appendEvent(eventsPath, {
        event: 'artifact_validation_failed',
        task: task.id,
        reason: 'Checksum mismatch or missing file'
      });
    }
  }

  // 4. Handle interrupted tasks (were "running")
  const interruptedTasks = await getTasksByStatus('running');
  for (const task of interruptedTasks) {
    const outputsValid = await validateArtifacts(task);
    if (outputsValid) {
      await markTaskCompleted(task.id);
    } else {
      await markTaskReady(task.id);
    }
  }

  // 5. Continue from current wave
  appendEvent(eventsPath, {
    event: 'session_resumed',
    from_wave: execution.current_wave,
    session: generateSessionId()
  });

  return {
    action: 'continue',
    wave: execution.current_wave
  };
}
```

### Status Command Output

```typescript
// Source: CLI UX best practices
interface StatusOutput {
  status: string;
  progress: Progress;
  wave_summary: WaveSummary[];
  recent_events: Event[];
  blockers: string[];
}

function formatStatus(state: StatusOutput, isTTY: boolean): string {
  if (!isTTY) {
    // Machine-readable for CI
    return JSON.stringify(state, null, 2);
  }

  // Human-readable with visual elements
  const progressBar = renderProgressBar(
    state.progress.tasks_completed,
    state.progress.tasks_total
  );

  return `
PTF Status: ${state.status.toUpperCase()}

Progress: ${progressBar} ${state.progress.tasks_completed}/${state.progress.tasks_total}

Waves:
${state.wave_summary.map(w =>
  `  ${w.number}. ${w.status} [${w.tasks.join(', ')}]`
).join('\n')}

Recent Activity:
${state.recent_events.slice(0, 5).map(e =>
  `  ${e.ts.slice(11, 19)} ${e.event} ${e.task || ''}`
).join('\n')}

${state.blockers.length > 0 ?
  `Blockers:\n${state.blockers.map(b => `  - ${b}`).join('\n')}` :
  'No blockers.'}
`;
}

function renderProgressBar(complete: number, total: number): string {
  const width = 20;
  const filled = Math.round((complete / total) * width);
  const empty = width - filled;
  return '[' + '='.repeat(filled) + ' '.repeat(empty) + ']';
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Snapshot-only persistence | Event sourcing + snapshots | 2020+ | Better debugging, audit, replay |
| Complex rollback logic | Idempotent operations | Industry standard | Simpler resume, fewer edge cases |
| Custom state formats | YAML/JSON standards | - | Better tooling, human-readable |
| Progress bars everywhere | TTY-aware formatting | CLI UX guidelines | Clean CI logs |

**Industry alignment:**
- Temporal.io uses event sourcing for workflow state
- Dapr Workflows uses append-only history log
- LangGraph uses checkpoint-based durable execution
- All major orchestrators checkpoint at task/step boundaries

## Schema Recommendations

### execution.yaml Schema (execution-state.schema.yaml)

```yaml
$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://ptf.dev/schemas/execution-state.schema.yaml"
title: ExecutionState
description: Master state file for PTF execution
type: object
required:
  - status
  - current_wave
  - waves_total
  - progress
properties:
  status:
    type: string
    enum: [pending, running, paused, completed, failed]
  current_wave:
    type: integer
    minimum: 1
  waves_total:
    type: integer
    minimum: 1
  progress:
    type: object
    required: [tasks_total, tasks_completed, tasks_pending]
    properties:
      tasks_total:
        type: integer
      tasks_completed:
        type: integer
      tasks_running:
        type: integer
      tasks_pending:
        type: integer
      tasks_failed:
        type: integer
      tasks_blocked:
        type: integer
  wave_summary:
    type: object
    additionalProperties:
      type: string
      enum: [pending, running, completed, partial, failed]
  session:
    type: object
    properties:
      id:
        type: string
      started:
        type: string
        format: date-time
      last_update:
        type: string
        format: date-time
  blockers:
    type: array
    items:
      type: string
```

### task-state.schema.yaml

```yaml
$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://ptf.dev/schemas/task-state.schema.yaml"
title: TaskState
description: Per-task execution state
type: object
required:
  - task_id
  - wave
  - status
properties:
  task_id:
    type: string
  wave:
    type: integer
  status:
    type: string
    enum: [pending, ready, running, completed, failed, blocked, skipped]
  attempts:
    type: array
    items:
      type: object
      properties:
        attempt:
          type: integer
        started:
          type: string
          format: date-time
        completed:
          type: string
          format: date-time
        status:
          type: string
  outputs_produced:
    type: array
    items:
      type: object
      properties:
        path:
          type: string
        checksum:
          type: string
  verification:
    type: object
    properties:
      status:
        type: string
        enum: [pending, passed, failed]
      results:
        type: array
```

### artifact-manifest.schema.yaml

```yaml
$schema: "http://json-schema.org/draft-07/schema#"
$id: "https://ptf.dev/schemas/artifact-manifest.schema.yaml"
title: ArtifactManifest
description: Registry of all produced artifacts
type: object
required:
  - artifacts
properties:
  artifacts:
    type: array
    items:
      type: object
      required: [path, type, produced_by, checksum]
      properties:
        path:
          type: string
        type:
          type: string
          enum: [source-code, config, test, migration, documentation, data, schema]
        produced_by:
          type: string
        produced_at:
          type: string
          format: date-time
        verified:
          type: boolean
        checksum:
          type: string
          pattern: "^sha256:[a-f0-9]{64}$"
        consumed_by:
          type: array
          items:
            type: string
  pending:
    type: array
    items:
      type: object
      properties:
        path:
          type: string
        expected_from:
          type: string
        needed_by:
          type: array
          items:
            type: string
```

## Open Questions

Things that couldn't be fully resolved:

1. **Concurrent Multi-Agent Writes**
   - What we know: Task-partitioned writes avoid contention on task files
   - What's unclear: Does orchestrator need locking for execution.yaml updates?
   - Recommendation: Single orchestrator model (Claude Code) means no contention; defer multi-agent complexity

2. **Event Log Rotation**
   - What we know: JSONL files grow unbounded
   - What's unclear: When/how to rotate for long-running projects?
   - Recommendation: Archive old events to `events-{date}.jsonl` if file exceeds threshold (e.g., 10MB)

3. **Checksum Algorithm Portability**
   - What we know: SHA-256 is standard
   - What's unclear: How to compute consistently across tools (Node.js, shell)?
   - Recommendation: Use `shasum -a 256` in shell, crypto.createHash('sha256') in Node

## Sources

### Primary (HIGH confidence)
- PTF Founding Document (PARALLEL-TASK-FRAMEWORK.md) - Sections 7 and 8
- PTF Existing Schemas (task.schema.yaml, wave.schema.yaml, artifact.schema.yaml)
- GSD State Template (.claude/get-shit-done/templates/state.md)
- GSD Checkpoints Reference (.claude/get-shit-done/references/checkpoints.md)

### Secondary (MEDIUM confidence)
- [Event Sourcing Pattern - Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Durable Execution - LangGraph](https://docs.langchain.com/oss/python/langgraph/durable-execution)
- [Temporal.io Durable Execution](https://temporal.io/)
- [CLI UX Best Practices - Evil Martians](https://evilmartians.com/chronicles/cli-ux-best-practices-3-patterns-for-improving-progress-displays)
- [Command Line Interface Guidelines](https://clig.dev/)

### Tertiary (LOW confidence - WebSearch only)
- [QCon SF 2025: Database-Backed Workflow Orchestration](https://www.infoq.com/news/2025/11/database-backed-workflow/) - Confirms checkpoint patterns

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - Using established PTF patterns (YAML, JSONL)
- Architecture: HIGH - Founding document specifies directory structure and protocols
- Pitfalls: HIGH - Well-documented in event sourcing and workflow literature
- Schema design: MEDIUM - Based on founding document examples, not yet validated

**Research date:** 2026-01-18
**Valid until:** 2026-02-17 (30 days - stable domain, established patterns)
