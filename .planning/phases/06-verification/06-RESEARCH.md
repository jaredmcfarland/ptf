# Phase 6: Verification - Research

**Researched:** 2026-01-18
**Domain:** Multi-modal verification, independent verifier subagent, verification strategies
**Confidence:** HIGH

## Summary

Phase 6 implements independent verification of task outputs with multi-modal strategies. Research confirms the PTF founding document provides clear specifications for verification types (exists, contains, runs, syntax, custom), the verification result format, and the integration with domain adapters for artifact-type-specific verification strategies.

Key findings:
- **Verifier independence** is crucial - verification runs in separate context from execution to catch "false promises"
- **Five verification types** are defined: exists, contains, runs, syntax, custom - each with specific implementation patterns
- **Multi-modal verification** combines multiple checks (e.g., exists AND contains AND runs) for comprehensive artifact validation
- **Adapter integration** provides artifact-type-specific verification strategies (source-code uses tsc, migrations use prisma validate, etc.)
- **State integration** records verification results in task-state.yaml's `verification` field
- **Existing executor implementation** already includes internal verification step - Phase 6 adds independent external verification

The verification subsystem is the quality gate that closes the loop on Ralph-style execution. Without reliable verification, the completion promise pattern cannot be trusted.

**Primary recommendation:** Implement verification as three components:
1. `ptf-verifier` subagent - independent verification execution
2. `/ptf:verify [task]` command - manual verification trigger
3. Verification integration with executor/orchestrator (verification results influence retry decisions)

## Standard Stack

### Core Components
| Component | Source | Purpose | Why Standard |
|-----------|--------|---------|--------------|
| Shell verification commands | Bash | exists/contains/runs checks | Universal, no dependencies |
| Syntax validators | Language-specific | syntax checks (tsc, prisma, eslint, python -m py_compile) | Language-native validation |
| SHA-256 checksums | shasum -a 256 | Artifact integrity verification | Standard hash algorithm |
| Exit codes | Unix convention | runs verification pass/fail | 0 = pass, non-0 = fail |

### From Prior Phases
| Component | Phase | Purpose | Integration |
|-----------|-------|---------|-------------|
| task.schema.yaml | Phase 1 | Defines verify array structure | VerificationStep type |
| task-state.schema.yaml | Phase 4 | Records verification results | verification.results array |
| ptf-state-manager | Phase 4 | Updates verification state | Invoke for state writes |
| ptf-executor | Phase 5 | Internal verification step | Verifier runs AFTER executor |
| Domain adapters | Phase 2 | Verification strategies per artifact type | Software/research adapters |

### Verification Types from Founding Document
| Type | Description | Implementation |
|------|-------------|----------------|
| `exists` | File exists at expected path | `[ -f "$target" ]` |
| `contains` | File contains expected content | `grep -q "$expected" "$target"` |
| `runs` | Command exits with expected code | Run command, check `$?` |
| `syntax` | File parses without errors | Language-specific parser/linter |
| `custom` | User-defined verification logic | Execute provided command/script |

## Architecture Patterns

### Recommended Project Structure

```
.claude/
├── commands/ptf/
│   └── verify.md                # /ptf:verify [task] command
│
├── agents/
│   └── ptf-verifier.md          # Verification subagent
│
└── skills/ptf/
    └── SKILL.md                 # (update with verification concepts)
```

### Pattern 1: Independent Verification

**What:** Verifier runs in separate context from executor, validating outputs independently.

**When to use:** After task executor claims VERIFICATION PASSED.

**Why independent:**
- Executor may have accumulated context affecting judgment
- Independent verification catches "false promises"
- Separate context ensures fresh evaluation
- Different agent role (auditor vs implementer)

**Implementation:**
```
execute_task():
  # Executor runs task
  result = dispatch_executor(task)

  if result.output contains "VERIFICATION PASSED":
    # DON'T TRUST - verify independently
    verification = dispatch_verifier(task)

    if verification.passed:
      mark_task_completed(task)
    else:
      # False promise detected
      log_event("false_completion_promise", task, verification.failures)
      mark_task_failed(task, "verification_failed")
```

### Pattern 2: Multi-Modal Verification

**What:** Combine multiple verification checks for comprehensive validation.

**When to use:** Tasks with complex outputs requiring multiple validations.

**Implementation (from task.schema.yaml):**
```yaml
verify:
  - type: exists
    target: src/auth/login.ts
  - type: contains
    target: src/auth/login.ts
    expected: "export function authenticate"
  - type: runs
    target: "npx tsc --noEmit src/auth/login.ts"
  - type: syntax
    target: src/auth/login.ts
```

**Execution order:**
1. exists - fastest, fail-fast if file missing
2. syntax - quick structural validation
3. contains - content pattern matching
4. runs - slowest, full execution check
5. custom - user-defined last

All checks must pass for verification to succeed.

### Pattern 3: Adapter-Driven Verification Strategies

**What:** Domain adapter specifies verification strategies per artifact type.

**When to use:** Determining which verifications to apply to an artifact.

**From software-development adapter:**
```yaml
verification_strategies:
  source-code:
    - method: exists
    - method: syntax
    - method: runs
      command_template: "npx tsc --noEmit {path}"

  migration:
    - method: exists
    - method: syntax
    - method: runs
      command_template: "npx prisma validate"

  test:
    - method: exists
    - method: runs
      command_template: "npm test -- {path}"
```

**Implementation:**
```
get_verification_steps(artifact_type, artifact_path):
  adapter = load_domain_adapter()
  strategies = adapter.verification_strategies[artifact_type]

  steps = []
  for strategy in strategies:
    step = {
      type: strategy.method,
      target: artifact_path
    }
    if strategy.command_template:
      step.expected = strategy.command_template.replace("{path}", artifact_path)
    steps.append(step)

  return steps
```

### Pattern 4: Verification Result Recording

**What:** Record detailed verification results in task state.

**When to use:** After every verification run.

**From task-state.schema.yaml:**
```yaml
verification:
  status: passed | failed | pending | skipped
  results:
    - type: exists
      target: src/auth/login.ts
      passed: true
      message: "File exists at path"
    - type: contains
      target: src/auth/login.ts
      passed: true
      message: "Pattern 'export function authenticate' found"
    - type: runs
      target: "npx tsc --noEmit src/auth/login.ts"
      passed: true
      message: "Command exited with code 0"
```

**Event logging:**
```jsonl
{"ts":"2026-01-18T10:35:00Z","event":"verification_started","task":"auth-login","checks":3}
{"ts":"2026-01-18T10:35:05Z","event":"verification_completed","task":"auth-login","status":"passed","duration_s":5}
```

### Pattern 5: Verification-Driven Retry Decisions

**What:** Verification failure triggers retry or escalation per task policy.

**When to use:** After verification fails.

**Implementation:**
```
handle_verification_failure(task, verification):
  # Check task's failure policy
  policy = task.on_failure or default_policy

  if policy.strategy == "retry":
    attempts = get_attempt_count(task)
    if attempts < policy.max_attempts:
      # Retry with fresh context
      schedule_retry(task)
    else:
      # Max retries exhausted
      escalate(task, "max_attempts_exceeded")

  elif policy.strategy == "skip":
    mark_task_skipped(task)
    continue_execution()

  elif policy.strategy == "escalate":
    pause_execution()
    present_to_human(task, verification)
```

### Anti-Patterns to Avoid

- **Trusting executor's self-verification:** Always run independent external verification
- **Single-mode verification:** Complex outputs need multiple verification types
- **Ignoring verification results:** Results must influence retry/continue decisions
- **Blocking verification in Ralph loop:** Verification runs BETWEEN iterations, not during
- **Hardcoding verification commands:** Use adapter-provided strategies when available

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| File existence check | Custom file check | `[ -f "$path" ]` | Shell built-in, reliable |
| Pattern matching | Custom parser | `grep -q "$pattern"` | Standard tool, regex support |
| TypeScript validation | Custom parser | `npx tsc --noEmit` | Official type checker |
| Prisma validation | Custom schema check | `npx prisma validate` | Official schema validator |
| Python syntax | Custom parser | `python -m py_compile` | Official syntax checker |
| JSON/YAML validation | Custom parser | `jq .` / `yq .` | Standard validators |
| Exit code checking | Custom logic | `$?` variable | Unix standard |

**Key insight:** Language ecosystems provide syntax validators. Use them. Don't parse code to check syntax.

## Common Pitfalls

### Pitfall 1: False Positives from Self-Verification

**What goes wrong:** Executor claims VERIFICATION PASSED but artifact is actually incomplete/incorrect.

**Why it happens:** Executor may hallucinate verification passing, or accumulated context affects judgment.

**How to avoid:**
- ALWAYS run independent verifier after executor claims completion
- Verifier spawns in fresh context, doesn't trust executor's claim
- Log "false_completion_promise" events for analysis
- Track false promise rate per task type for improvement

**Warning signs:** High rate of verification failures after VERIFICATION PASSED claims.

### Pitfall 2: Incomplete Verification Coverage

**What goes wrong:** Verification passes but artifact doesn't actually work.

**Why it happens:** Only checking `exists` when `contains` or `runs` is needed.

**How to avoid:**
- Multi-modal verification by default
- Match verification intensity to artifact criticality
- Domain adapters specify minimum verification per type
- Review verification results, not just pass/fail

**Warning signs:** Downstream tasks fail on artifacts that "passed" verification.

### Pitfall 3: Slow Verification Blocking Execution

**What goes wrong:** Verification takes too long, slowing overall execution.

**Why it happens:** Running expensive verifications (full test suites, compilation) on every check.

**How to avoid:**
- Order verifications fast-to-slow (exists first, runs last)
- Fail-fast on first failure
- Cache verification results for identical artifacts
- Consider parallel verification when safe

**Warning signs:** Verification time exceeds execution time.

### Pitfall 4: Syntax Check False Negatives

**What goes wrong:** Syntax check passes but code has runtime errors.

**Why it happens:** Syntax validation only catches parse errors, not logic errors.

**How to avoid:**
- Syntax is necessary but not sufficient
- Combine with `runs` checks that execute code
- Tests are the real verification for logic
- Document verification limitations

**Warning signs:** Code passes syntax but fails at runtime.

### Pitfall 5: Ignoring Verification Failures

**What goes wrong:** Verification fails but execution continues as if successful.

**Why it happens:** Not integrating verification results with state management.

**How to avoid:**
- Verification status determines task status
- Failed verification = failed task (not completed)
- State manager must update based on verification
- Clear decision tree: verify → pass/fail → action

**Warning signs:** Tasks marked complete with verification.status = "failed".

## Code Examples

### Verification Step Implementation (Bash)

```bash
# exists check
verify_exists() {
  local target="$1"
  if [ -f "$target" ]; then
    echo "PASS: exists - $target"
    return 0
  elif [ -d "$target" ]; then
    echo "PASS: exists (directory) - $target"
    return 0
  else
    echo "FAIL: exists - $target not found"
    return 1
  fi
}

# contains check
verify_contains() {
  local target="$1"
  local expected="$2"
  if grep -q "$expected" "$target" 2>/dev/null; then
    echo "PASS: contains - found '$expected' in $target"
    return 0
  else
    echo "FAIL: contains - '$expected' not found in $target"
    return 1
  fi
}

# runs check
verify_runs() {
  local command="$1"
  local expected_exit="${2:-0}"  # Default: expect exit 0

  output=$(eval "$command" 2>&1)
  actual_exit=$?

  if [ "$actual_exit" -eq "$expected_exit" ]; then
    echo "PASS: runs - command exited with $actual_exit"
    return 0
  else
    echo "FAIL: runs - expected exit $expected_exit, got $actual_exit"
    echo "Output: $output"
    return 1
  fi
}

# syntax check (TypeScript example)
verify_syntax_ts() {
  local target="$1"
  if npx tsc --noEmit "$target" 2>&1; then
    echo "PASS: syntax - TypeScript compiles"
    return 0
  else
    echo "FAIL: syntax - TypeScript compilation errors"
    return 1
  fi
}

# syntax check (Python example)
verify_syntax_py() {
  local target="$1"
  if python -m py_compile "$target" 2>&1; then
    echo "PASS: syntax - Python syntax valid"
    return 0
  else
    echo "FAIL: syntax - Python syntax errors"
    return 1
  fi
}

# custom check
verify_custom() {
  local command="$1"
  if eval "$command" 2>&1; then
    echo "PASS: custom - command succeeded"
    return 0
  else
    echo "FAIL: custom - command failed"
    return 1
  fi
}
```

### Verifier Subagent Structure

```markdown
---
name: ptf-verifier
description: Independently verifies task outputs against verification criteria
tools: Read, Bash, Glob, Grep
---

<role>
You are a PTF verification specialist. You independently verify task outputs.

You are spawned by the orchestrator AFTER a task executor claims completion.

You receive:
- Task ID to verify
- Verification criteria from task definition
- Outputs declared by task

Your job: Run all verification checks and report pass/fail with details.

**Critical:** You are independent from the executor. Don't trust claims.
Verify by actually checking artifacts.
</role>

<verification_flow>

<step name="load_criteria">
Load verification criteria from task definition:
- Read task from .orchestrator/decomposition/tasks/{task-id}.yaml
- Extract verify array
- Extract outputs array
</step>

<step name="verify_outputs_exist">
For each output in task.outputs:
- Check file exists at path
- If missing: record failure, continue to other outputs
</step>

<step name="run_verification_steps">
For each step in task.verify:
- Execute verification based on type
- Record result (passed, message, actual vs expected)
- Continue even if some fail
</step>

<step name="compile_results">
Aggregate all verification results:
- Overall status: passed if ALL checks pass
- Individual results with details
- Summary message
</step>

<step name="return_results">
Return structured verification result:
- VERIFICATION PASSED if all checks pass
- VERIFICATION FAILED if any check fails
</step>

</verification_flow>
```

### /ptf:verify Command Structure

```markdown
---
name: ptf:verify
description: Run verification for a specific task
allowed-tools:
  - Read
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Run verification checks for a specific task. Can be used:
- After manual task completion
- To re-verify an existing task
- To debug verification failures

**Usage:**
- `/ptf:verify task-id` - Verify specific task
- `/ptf:verify --all` - Verify all completed tasks

**Requires:**
- Task definition in .orchestrator/decomposition/tasks/
- Task outputs should exist (unless checking for missing)
</objective>

<process>

## Phase 1: Load Task

1. Parse task ID from argument
2. Load task definition from .orchestrator/decomposition/tasks/{task-id}.yaml
3. Extract verification criteria

## Phase 2: Spawn Verifier

Dispatch ptf-verifier subagent:

```
Task(prompt="
Verify task: {task_id}

Verification criteria:
{task.verify}

Expected outputs:
{task.outputs}

Run all verification checks and return detailed results.
", subagent_type="ptf-verifier")
```

## Phase 3: Record Results

Update task state with verification results:
```
Task(prompt="
Record verification results for task {task_id}:
Status: {passed | failed}
Results: {detailed results}
", subagent_type="ptf-state-manager")
```

## Phase 4: Display Results

Show verification summary with details on any failures.

</process>
```

### Verification Result Format (from Founding Document)

```yaml
verification:
  task_id: auth-schema
  timestamp: 2025-01-17T10:35:00Z
  status: passed  # passed | failed

  checks:
    - type: exists
      target: /src/db/migrations/001_auth_schema.sql
      passed: true

    - type: contains
      target: /src/db/migrations/001_auth_schema.sql
      expected: [CREATE TABLE users, CREATE TABLE sessions]
      passed: true
      found: [CREATE TABLE users, CREATE TABLE sessions, CREATE INDEX]

    - type: runs
      target: psql -f /src/db/migrations/001_auth_schema.sql --dry-run
      passed: true
      exit_code: 0
      stdout: |
        DRY RUN: Schema validation successful

  summary: "All 3 verification checks passed"
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Trust agent completion claims | Independent verification | 2025 (Ralph pattern) | Catches false promises |
| Single verification type | Multi-modal verification | Standard practice | Comprehensive validation |
| Manual verification | Automated verification commands | Standard practice | Consistent, repeatable |
| Verification in executor | Separate verifier agent | 2025 (PTF design) | Fresh context for auditing |

**Key insight from founding document:**
> "Without verification, parallel execution is blind. Every task must have: observable completion criteria, automatable verification steps (where possible), clear pass/fail determination, recorded verification results."

## Integration Points

### Inputs from Prior Phases

| Artifact | Location | Used For |
|----------|----------|----------|
| Task definitions | .orchestrator/decomposition/tasks/*.yaml | Verification criteria (verify array) |
| Task state | .orchestrator/state/tasks/*.yaml | Current verification status |
| Domain adapters | adapters/*.yaml | Verification strategies per artifact type |
| Artifact manifest | .orchestrator/artifacts/manifest.yaml | Track verified status |

### Outputs for Later Phases

| Artifact | Location | Consumed By |
|----------|----------|-------------|
| Verification results | task-state.yaml verification field | Phase 7 (Failure Handling) |
| Verification events | events.jsonl | Debugging, audit trail |
| Artifact verified status | manifest.yaml | Resume validation |

### Integration with Executor/Orchestrator

| Component | Integration Point | Behavior |
|-----------|------------------|----------|
| ptf-executor | Internal verify step | Executor runs verification before claiming VERIFICATION PASSED |
| ptf-orchestrator | Post-execution | Orchestrator spawns independent verifier after executor completion |
| ptf-state-manager | Result recording | Updates task state and artifact manifest |
| Failure handling | Verification failure | Triggers retry or escalation per policy (Phase 7) |

## Open Questions

1. **Verification caching**
   - What we know: Same artifact may be verified multiple times (retries, resume)
   - What's unclear: Should verification results cache based on artifact checksum?
   - Recommendation: Cache verification by artifact path + checksum; re-verify if checksum changes

2. **Parallel verification**
   - What we know: Multiple tasks complete simultaneously in a wave
   - What's unclear: Can verification run in parallel across tasks?
   - Recommendation: Yes, verification is independent per task; respect max_parallel_tasks

3. **Custom verification security**
   - What we know: Custom verification runs arbitrary commands
   - What's unclear: Should there be sandboxing or command restrictions?
   - Recommendation: Document risk; custom commands run with user permissions; no additional sandboxing for v1

4. **Verification timeout**
   - What we know: `runs` verification can hang on slow commands
   - What's unclear: Should there be per-check timeouts?
   - Recommendation: Default 60s timeout per check; configurable in task definition

## Sources

### Primary (HIGH confidence)
- PTF Founding Document (PARALLEL-TASK-FRAMEWORK.md) - Section 10.5 Verification Result Format, Section 6.5 Completion Promise
- task.schema.yaml - VerificationStep definition (exists, contains, runs, syntax, custom)
- task-state.schema.yaml - Verification result recording
- software-development.yaml adapter - Verification strategies per artifact type
- ptf-executor.md - Internal verification step implementation

### Secondary (MEDIUM confidence)
- Phase 5 Research (05-RESEARCH.md) - Ralph-style iteration, completion promise pattern
- Phase 4 Plans - State manager operations for verification recording

### Tertiary (LOW confidence - derived)
- Syntax validation commands by language (tsc, prisma validate, py_compile) - common ecosystem patterns

## Metadata

**Confidence breakdown:**
- Verification types: HIGH - Founding document Section 10.5 specifies exactly
- Verifier independence: HIGH - Founding document Section 6.5 specifies
- State recording: HIGH - task-state.schema.yaml defines structure
- Adapter integration: HIGH - software-development.yaml shows pattern
- Command implementations: HIGH - Shell standard commands
- Retry integration: MEDIUM - Requires Phase 7 coordination

**Research date:** 2026-01-18
**Valid until:** 2026-02-17 (30 days - verification patterns are stable once defined)
