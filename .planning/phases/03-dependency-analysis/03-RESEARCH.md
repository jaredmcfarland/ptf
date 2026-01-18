# Phase 3: Dependency Analysis - Research

**Researched:** 2026-01-18
**Domain:** Dependency Inference, Graph Algorithms, Wave Computation, Plan Generation
**Confidence:** HIGH

## Summary

Phase 3 transforms decomposed tasks into an executable plan by inferring dependencies, detecting cycles, computing parallel execution waves, and generating human-readable output. Research confirms the algorithms and patterns needed are well-established in build systems (Bazel, Make), workflow orchestrators (Airflow, Dagster, Prefect), and graph theory (Kahn's algorithm, Tarjan's algorithm).

Key findings:
- Multi-pass dependency inference (artifact matching, type matching, semantic analysis, domain heuristics) is the standard approach used by artifact-based build systems
- Kahn's algorithm is the optimal choice for wave computation - it naturally groups tasks with zero in-degree into parallel waves
- Cycle detection should use Tarjan's algorithm for O(V+E) linear time performance with clear cycle reporting
- The dependency analyzer should be a subagent following the pattern established in Phase 2
- graph.yaml output integrates naturally with existing schemas (Dependency, Wave from Phase 1)

**Primary recommendation:** Implement 5-pass dependency inference (artifact, type, semantic, heuristic, resource-conflict), use Kahn's algorithm for wave computation with built-in cycle detection, create a dependency analyzer subagent, and build /ptf:plan command that produces human-readable plan output.

## Standard Stack

### Core Algorithms
| Algorithm | Purpose | Why Standard |
|-----------|---------|--------------|
| Kahn's Algorithm | Topological sort into waves | O(V+E), naturally produces parallel levels, cycle detection built-in |
| Tarjan's Algorithm | Strongly connected components | O(V+E), reports exact cycle members for resolution guidance |
| Hash-based path matching | Artifact dependency inference | Fast O(1) lookup for output-to-input matching |
| Glob/pattern matching | Type-based inference | Standard POSIX glob semantics |

### Existing PTF Components
| Component | Phase | Purpose | Integration Point |
|-----------|-------|---------|-------------------|
| dependency.schema.yaml | 1 | Dependency structure | Multi-pass inference populates this |
| wave.schema.yaml | 1 | Wave structure | Kahn's algorithm produces these |
| plan.schema.yaml | 1 | Complete plan | Final output structure |
| Domain adapters | 2 | common_patterns, inference_hints | Pass 4 heuristic inference |
| Task files | 2 | .orchestrator/decomposition/tasks/*.yaml | Input to analysis |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Kahn's algorithm | DFS topological sort | Kahn's naturally produces waves; DFS requires post-processing |
| Tarjan's algorithm | Kosaraju's algorithm | Tarjan's requires single DFS pass (more efficient) |
| LLM semantic analysis | Regex-only pattern matching | LLM catches references regex misses ("uses the service from task X") |
| Multi-pass inference | Single-pass | Multi-pass provides confidence levels, catches more dependencies |

## Architecture Patterns

### Recommended Project Structure
```
.claude/
+-- commands/ptf/
|   +-- plan.md                   # /ptf:plan command (this phase)
|
+-- agents/
    +-- ptf-dependency-analyzer.md # Dependency analyzer subagent

.orchestrator/decomposition/       # Already exists from Phase 2
+-- tasks/                        # Input: atomic tasks from decomposition
|   +-- {task-id}.yaml
+-- graph.yaml                    # Output: dependencies + waves
```

### Pattern 1: Multi-Pass Dependency Inference

**What:** Sequential passes that build dependency graph with decreasing confidence.

**When to use:** All dependency analysis - this is the core algorithm.

**Implementation:**

```
infer_dependencies(tasks: Task[]) -> Dependency[]:

  dependencies = []

  # Pass 1: Artifact matching (HIGH confidence)
  # Task A.outputs contains path that Task B.inputs contains
  for each task B in tasks:
    for each input in B.inputs:
      for each task A in tasks where A != B:
        if A.outputs contains input.path:
          dependencies.add({
            from: A.id,
            to: B.id,
            type: "artifact",
            confidence: "high",
            reason: "{B.id} needs {input.path} produced by {A.id}"
          })

  # Pass 2: Type/pattern matching (MEDIUM confidence)
  # Input path is glob pattern, matches output type
  for each task B in tasks:
    for each input in B.inputs where input.path contains * or **:
      for each task A in tasks where A != B:
        for each output in A.outputs:
          if glob_match(input.path, output.path):
            add_if_not_exists({
              from: A.id,
              to: B.id,
              type: "artifact",
              confidence: "medium",
              reason: "Pattern {input.path} matches {output.path}"
            })

  # Pass 3: Semantic analysis (MEDIUM confidence)
  # LLM analyzes descriptions for references
  semantic_deps = llm_analyze_descriptions(tasks)
  for each dep in semantic_deps:
    add_if_not_exists(dep)

  # Pass 4: Domain heuristic patterns (LOW confidence)
  # Adapter provides common_patterns (e.g., "tests depend on code")
  for each pattern in adapter.dependencies.common_patterns:
    for each task A where A.outputs.type == pattern.from_type:
      for each task B where B.outputs.type == pattern.to_type:
        if A != B:
          add_if_not_exists({
            from: A.id,
            to: B.id,
            type: "implicit",
            confidence: "low",
            reason: pattern.description
          })

  # Pass 5: Resource conflict detection (HIGH confidence)
  # Tasks modifying same file cannot parallelize
  for each pair (A, B) where A.id < B.id:
    overlap = A.outputs intersect B.outputs
    if overlap is not empty:
      # Order by subgoal hierarchy or alphabetically
      dependencies.add({
        from: earlier(A, B).id,
        to: later(A, B).id,
        type: "resource",
        confidence: "high",
        reason: "Both modify: {overlap.paths}"
      })

  return dependencies
```

### Pattern 2: Kahn's Algorithm for Wave Computation

**What:** Topological sort that groups tasks with zero in-degree into parallel waves.

**When to use:** After dependency inference, to compute execution order.

**Implementation:**

```
compute_waves(tasks: Task[], dependencies: Dependency[]) -> Wave[]:

  # Build in-degree map and adjacency list
  in_degree = {}  # task_id -> count of dependencies
  dependents = {} # task_id -> list of task_ids that depend on it

  for each task in tasks:
    in_degree[task.id] = 0
    dependents[task.id] = []

  for each dep in dependencies:
    in_degree[dep.to] += 1
    dependents[dep.from].append(dep.to)

  # Kahn's algorithm with wave tracking
  waves = []
  remaining = set(all task.ids)

  while remaining is not empty:
    # Find all tasks with no unmet dependencies
    ready = [t for t in remaining if in_degree[t] == 0]

    if ready is empty and remaining is not empty:
      # Cycle detected - this shouldn't happen if validated first
      error("Cycle detected in dependency graph")

    # Create wave
    wave_number = len(waves) + 1
    wave = {
      number: wave_number,
      tasks: ready,
      status: "pending",
      depends_on_waves: [wave_number - 1] if wave_number > 1 else []
    }
    waves.append(wave)

    # Remove tasks from remaining, decrement dependents' in-degrees
    for each task_id in ready:
      remaining.remove(task_id)
      for each dependent_id in dependents[task_id]:
        in_degree[dependent_id] -= 1

  return waves
```

### Pattern 3: Tarjan's Algorithm for Cycle Detection

**What:** Find all strongly connected components to detect and report cycles.

**When to use:** After dependency inference, before wave computation.

**Implementation:**

```
detect_cycles(tasks: Task[], dependencies: Dependency[]) -> CycleReport:

  # Build adjacency list
  adj = {}
  for each task in tasks:
    adj[task.id] = []
  for each dep in dependencies:
    adj[dep.from].append(dep.to)

  # Tarjan's algorithm
  index_counter = 0
  stack = []
  lowlinks = {}
  indices = {}
  on_stack = {}
  sccs = []

  def strongconnect(v):
    nonlocal index_counter
    indices[v] = index_counter
    lowlinks[v] = index_counter
    index_counter += 1
    stack.append(v)
    on_stack[v] = True

    for w in adj[v]:
      if w not in indices:
        strongconnect(w)
        lowlinks[v] = min(lowlinks[v], lowlinks[w])
      elif on_stack.get(w, False):
        lowlinks[v] = min(lowlinks[v], indices[w])

    # If v is a root node, pop the SCC
    if lowlinks[v] == indices[v]:
      scc = []
      while True:
        w = stack.pop()
        on_stack[w] = False
        scc.append(w)
        if w == v:
          break
      sccs.append(scc)

  for v in tasks:
    if v.id not in indices:
      strongconnect(v.id)

  # Report cycles (SCCs with more than one member)
  cycles = [scc for scc in sccs if len(scc) > 1]

  if cycles:
    return {
      has_cycles: True,
      cycles: cycles,
      resolution_hints: generate_resolution_hints(cycles, dependencies)
    }
  else:
    return {has_cycles: False}

def generate_resolution_hints(cycles, dependencies):
  hints = []
  for cycle in cycles:
    # Find which dependency in the cycle has lowest confidence
    cycle_deps = [d for d in dependencies
                  if d.from in cycle and d.to in cycle]
    lowest = min(cycle_deps, key=lambda d: confidence_rank(d.confidence))
    hints.append({
      cycle: cycle,
      suggested_break: lowest,
      reason: "This dependency has lowest confidence ({lowest.confidence})"
    })
  return hints
```

### Pattern 4: Semantic Dependency Analysis with LLM

**What:** Use LLM to analyze task descriptions for implicit references.

**When to use:** Pass 3 of multi-pass inference.

**Implementation:**

```
llm_analyze_descriptions(tasks: Task[]) -> Dependency[]:

  prompt = """
  Analyze these tasks for implicit dependencies not captured by
  input/output declarations. Look for phrases like:
  - "uses the X from..."
  - "after Y is complete..."
  - "builds on the work in..."
  - "references the schema/model/service from..."

  Tasks:
  {for each task: id, name, description, context_notes}

  Return dependencies as:
  - from: task_id (the prerequisite)
  - to: task_id (the dependent)
  - reason: explanation

  Only return HIGH-CONFIDENCE semantic dependencies.
  Do NOT duplicate artifact-based dependencies (input/output matches).
  """

  response = llm_call(prompt)

  dependencies = parse_response(response)
  for each dep in dependencies:
    dep.type = "semantic"
    dep.confidence = "medium"

  return dependencies
```

### Pattern 5: Plan Output Generation

**What:** Transform graph.yaml into human-readable plan.

**When to use:** /ptf:plan command output.

**Example output:**

```markdown
# Execution Plan: auth-system

**Generated:** 2026-01-18T10:30:00Z
**Tasks:** 8 tasks in 4 waves
**Estimated parallel speedup:** 2x (8 tasks / 4 waves)

## Wave 1 (2 tasks - parallel)

| Task | Description | Outputs |
|------|-------------|---------|
| auth-schema | Create Prisma models for User and Session | prisma/schema.prisma |
| config-setup | Initialize authentication configuration | src/config/auth.ts |

**No dependencies - can start immediately**

## Wave 2 (2 tasks - parallel)

| Task | Description | Outputs |
|------|-------------|---------|
| user-repository | Implement User data access layer | src/repositories/user.ts |
| session-repository | Implement Session data access layer | src/repositories/session.ts |

**Depends on:** Wave 1 (auth-schema)

## Wave 3 (1 task)

| Task | Description | Outputs |
|------|-------------|---------|
| auth-service | Implement authentication business logic | src/services/auth.ts |

**Depends on:** Wave 2 (user-repository, session-repository)

## Wave 4 (3 tasks - parallel)

| Task | Description | Outputs |
|------|-------------|---------|
| auth-routes | HTTP endpoints for auth | src/routes/auth.ts |
| auth-middleware | Session validation middleware | src/middleware/auth.ts |
| auth-tests | Integration tests | tests/auth.test.ts |

**Depends on:** Wave 3 (auth-service)

---

## Dependency Graph

```
auth-schema ──┬──> user-repository ──┬──> auth-service ──┬──> auth-routes
              │                      │                   ├──> auth-middleware
              └──> session-repository┘                   └──> auth-tests
config-setup ─────────────────────────────────────────────────> auth-routes
```

## Confidence Summary

| Confidence | Count | % |
|------------|-------|---|
| HIGH | 6 | 75% |
| MEDIUM | 2 | 25% |
| LOW | 0 | 0% |

**Ready for execution.** Run `/ptf:execute` to begin.
```

### Anti-Patterns to Avoid

- **Single-pass inference:** Missing dependencies leads to incorrect parallelization. Always use multi-pass.
- **Ignoring confidence levels:** LOW confidence dependencies should be flagged for human review.
- **Cycle panic:** Report cycles with resolution guidance, don't just fail.
- **Over-serialization:** Don't add dependencies "just to be safe" - it defeats parallelism.
- **Ignoring resource conflicts:** Two tasks writing same file MUST be serialized.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Graph traversal | Custom recursion | Kahn's/Tarjan's algorithms | Battle-tested, O(V+E), handles edge cases |
| Pattern matching | Custom parser | Glob library (minimatch pattern) | Standard semantics, tested |
| Cycle reporting | Simple "cycle found" message | Tarjan's SCC with resolution hints | Users need to know WHICH dependency to break |
| Wave numbering | Manual assignment | Algorithm-derived | Kahn's produces correct levels automatically |
| Dependency deduplication | Post-processing | Check-before-add pattern | Prevents duplicate work in multi-pass |

**Key insight:** The algorithms here are classical computer science. Kahn's algorithm (1962) and Tarjan's algorithm (1972) are optimal for their respective problems. Don't reinvent them.

## Common Pitfalls

### Pitfall 1: Circular Dependency Through Resources

**What goes wrong:** Two tasks modify the same file, creating implicit circular dependency.

**Why it happens:** Neither task declares the other as input, but they conflict.

**How to avoid:**
- Pass 5 (resource conflict detection) catches overlapping outputs
- Force serialization with "resource" dependency type
- Order by subgoal hierarchy or creation order

**Warning signs:** Tasks with overlapping output paths.

### Pitfall 2: False Semantic Dependencies

**What goes wrong:** LLM infers dependency where none exists (e.g., "auth" in two unrelated task names).

**Why it happens:** Semantic analysis over-matches on common terms.

**How to avoid:**
- Mark semantic dependencies as MEDIUM confidence
- Only accept HIGH-CONFIDENCE from LLM analysis
- Cross-reference with actual input/output paths

**Warning signs:** Many semantic dependencies with vague reasons.

### Pitfall 3: Missing Transitive Dependencies

**What goes wrong:** Task C needs output from A, but only B is listed as dependency (A -> B -> C).

**Why it happens:** Inference only looks at direct input/output matches.

**How to avoid:**
- Wave computation handles transitive dependencies implicitly
- C runs in wave 3+, A completed in wave 1
- Document that waves enforce transitive ordering

**Warning signs:** N/A - Kahn's algorithm handles this correctly.

### Pitfall 4: Heuristic Over-Connection

**What goes wrong:** Domain heuristics add too many LOW confidence dependencies, serializing parallel work.

**Why it happens:** "test depends on code" matches every test to every code task.

**How to avoid:**
- Heuristics should check task relationships (same subgoal, related names)
- Only apply heuristics when no artifact dependency exists
- Keep heuristic dependencies flagged as LOW

**Warning signs:** Wave count approaching task count (no parallelism).

### Pitfall 5: Glob Pattern Ambiguity

**What goes wrong:** Overly broad glob patterns match unintended files.

**Why it happens:** Input like `src/**/*.ts` matches too many outputs.

**How to avoid:**
- Be specific in input patterns
- Type-based matching uses artifact types, not just paths
- Flag ambiguous matches for human review

**Warning signs:** Single input matching many outputs.

## Code Examples

### Dependency Analyzer Subagent
```markdown
---
name: ptf-dependency-analyzer
description: Analyzes task dependencies and computes parallel execution waves
tools: Read, Write, Bash, Glob, Grep
---

<role>
You are a PTF dependency analyzer. You transform decomposed tasks into an
executable plan by inferring dependencies and computing parallel waves.

Your job: Read all tasks from decomposition, run multi-pass dependency
inference, detect cycles, compute waves, and produce graph.yaml.
</role>

<execution_flow>

<step name="load_tasks">
Read all task files from .orchestrator/decomposition/tasks/
Load domain adapter for inference hints
</step>

<step name="pass1_artifact">
**Artifact Matching (HIGH confidence)**

For each task's inputs:
- Find tasks whose outputs contain the input path
- Create artifact dependency with HIGH confidence

This is the most reliable inference source.
</step>

<step name="pass2_type">
**Type/Pattern Matching (MEDIUM confidence)**

For inputs with glob patterns:
- Match against output paths using glob semantics
- Create artifact dependency with MEDIUM confidence
</step>

<step name="pass3_semantic">
**Semantic Analysis (MEDIUM confidence)**

Analyze task descriptions for implicit references:
- "uses the X created by..."
- "after Y is complete..."
- "builds on..."

Only add if not already captured by artifact matching.
</step>

<step name="pass4_heuristic">
**Domain Heuristics (LOW confidence)**

Apply adapter's common_patterns:
- Match by artifact type (e.g., test depends on source-code)
- Only add if no existing dependency between tasks
</step>

<step name="pass5_resource">
**Resource Conflict Detection (HIGH confidence)**

Find tasks with overlapping outputs:
- These CANNOT run in parallel
- Create resource dependency, order by task ID

This prevents file conflicts.
</step>

<step name="detect_cycles">
Run Tarjan's algorithm to find strongly connected components.

If cycles found:
- Report exact cycle members
- Suggest which dependency to break (lowest confidence)
- Return ANALYSIS BLOCKED

If no cycles: continue to wave computation.
</step>

<step name="compute_waves">
Run Kahn's algorithm:
1. Initialize in-degree for each task
2. Find all tasks with in-degree 0 (wave 1)
3. Remove from graph, decrement dependents
4. Repeat until all tasks assigned to waves
</step>

<step name="write_output">
Write .orchestrator/decomposition/graph.yaml:
```yaml
step: 5-dependency-graph
created: {timestamp}
status: complete

dependencies:
  - from: {task-id}
    to: {task-id}
    type: {artifact|semantic|resource|implicit}
    confidence: {high|medium|low}
    reason: "{explanation}"

waves:
  - number: 1
    tasks: [task-ids]
    depends_on_waves: []
  - number: 2
    tasks: [task-ids]
    depends_on_waves: [1]

validation:
  has_cycles: false
  total_dependencies: {N}
  high_confidence: {count}
  medium_confidence: {count}
  low_confidence: {count}
```
</step>

</execution_flow>

<structured_returns>

## ANALYSIS COMPLETE

**Dependencies:** {N} total ({high} high, {medium} medium, {low} low confidence)
**Waves:** {M} parallel execution waves
**Parallelism factor:** {task_count / wave_count}x

### Wave Summary
| Wave | Tasks | Depends On |
|------|-------|------------|
| 1 | {count} | - |
| 2 | {count} | Wave 1 |
...

### Files Created
- .orchestrator/decomposition/graph.yaml

### Ready for /ptf:plan
Dependency analysis complete. Run `/ptf:plan` to generate human-readable plan.

---

## ANALYSIS BLOCKED

**Blocked by:** Cycle detected in dependency graph

### Cycle Details
{list of tasks in cycle}

### Resolution Suggestion
The dependency with lowest confidence that could break this cycle:
- From: {task-id}
- To: {task-id}
- Type: {type}
- Confidence: {confidence}
- Reason: {reason}

**Consider:**
1. Removing this dependency if it's incorrect
2. Merging the cyclic tasks
3. Adding an intermediate task to break the cycle

</structured_returns>
```

### /ptf:plan Command
```markdown
---
name: ptf:plan
description: Generate human-readable plan from decomposition and dependencies
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - Task
---

<objective>
Generate a human-readable execution plan from the decomposition and
dependency analysis results. If dependencies haven't been analyzed yet,
run the dependency analyzer first.

**Requires:**
- .orchestrator/decomposition/tasks/*.yaml (from /ptf:decompose)

**Creates/Updates:**
- .orchestrator/decomposition/graph.yaml (if not exists)
- .orchestrator/plan.md (human-readable output)

**After this command:** Run `/ptf:execute` to begin execution.
</objective>

<execution_context>
@.claude/skills/ptf/SKILL.md
</execution_context>

<context>
@.orchestrator/decomposition/analysis.yaml
@.orchestrator/decomposition/constitution.yaml
</context>

<process>

## Phase 1: Prerequisites

Check for decomposition output:
```bash
[ ! -d .orchestrator/decomposition/tasks ] && \
  echo "Run /ptf:decompose first" && exit 1
```

## Phase 2: Run Dependency Analysis (if needed)

```bash
if [ ! -f .orchestrator/decomposition/graph.yaml ]; then
  # Spawn dependency analyzer
  Task(
    prompt="Analyze dependencies for all tasks in
            .orchestrator/decomposition/tasks/",
    subagent_type="ptf-dependency-analyzer"
  )
fi
```

Handle analyzer results:
- If ANALYSIS COMPLETE: continue
- If ANALYSIS BLOCKED: present cycle resolution options, stop

## Phase 3: Generate Plan Output

Read graph.yaml and all task files.

Generate .orchestrator/plan.md with:
1. Header (goal, task count, wave count, timestamp)
2. Wave-by-wave breakdown with task tables
3. Dependency graph visualization (ASCII)
4. Confidence summary
5. Next steps

## Phase 4: Commit

```bash
git add .orchestrator/decomposition/graph.yaml
git add .orchestrator/plan.md
git commit -m "ptf: generate execution plan

{N} tasks in {M} waves
Parallelism factor: {X}x"
```

## Phase 5: Present Plan

Display plan.md content to user.
Indicate ready for execution.

</process>

<success_criteria>
- [ ] graph.yaml exists with dependencies and waves
- [ ] No cycles in dependency graph
- [ ] plan.md generated with human-readable format
- [ ] All files committed
- [ ] User knows to run `/ptf:execute` next
</success_criteria>
```

### graph.yaml Schema Extension
```yaml
# .orchestrator/decomposition/graph.yaml
# Extends the Phase 1 schemas with analysis metadata

step: 5-dependency-graph
created: 2026-01-18T10:30:00Z
status: complete

task_count: 8
subgoal_count: 4

dependencies:
  - from: auth-schema
    to: user-repository
    type: artifact
    confidence: high
    reason: "user-repository inputs prisma/schema.prisma produced by auth-schema"

  - from: auth-service
    to: auth-tests
    type: implicit
    confidence: low
    reason: "Domain heuristic: test depends on source-code"

waves:
  - number: 1
    tasks: [auth-schema, config-setup]
    status: pending
    depends_on_waves: []
    rationale: "No dependencies - can start immediately"

  - number: 2
    tasks: [user-repository, session-repository]
    status: pending
    depends_on_waves: [1]
    rationale: "Depend on auth-schema from wave 1"

  - number: 3
    tasks: [auth-service]
    status: pending
    depends_on_waves: [2]
    rationale: "Depends on repositories from wave 2"

  - number: 4
    tasks: [auth-routes, auth-middleware, auth-tests]
    status: pending
    depends_on_waves: [3]
    rationale: "Depend on auth-service from wave 3"

validation:
  has_cycles: false
  all_tasks_assigned: true
  orphan_tasks: []

  dependency_summary:
    total: 10
    by_type:
      artifact: 6
      semantic: 2
      implicit: 2
      resource: 0
    by_confidence:
      high: 6
      medium: 2
      low: 2

warnings:
  - "auth-tests dependency on auth-service is heuristic-based (low confidence)"
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Manual dependency declaration | Automatic inference from I/O | 2010+ (Bazel) | 10x fewer errors, catches hidden deps |
| Task-based build systems | Artifact-based systems | 2015+ (Bazel OSS) | Deterministic, cacheable builds |
| Single topological sort | Wave-based grouping | Standard in orchestrators | Enables true parallelism |
| "Cycle detected" error | Resolution guidance | Recent practice | Actionable error messages |

**Deprecated/outdated:**
- Manual dependency graphs: Inference from I/O is more reliable
- DFS-only cycle detection: Tarjan's SCC provides better cycle information
- Sequential execution planning: Wave computation enables parallelism

## Integration Points

### Inputs from Phase 2
| Artifact | Location | Used For |
|----------|----------|----------|
| Task definitions | .orchestrator/decomposition/tasks/*.yaml | Dependency inference input |
| Analysis | .orchestrator/decomposition/analysis.yaml | Goal context for plan output |
| Constitution | .orchestrator/decomposition/constitution.yaml | Constraint reference |
| Subgoals | .orchestrator/decomposition/subgoals.yaml | Task grouping context |

### Outputs for Phase 4-5
| Artifact | Location | Consumed By |
|----------|----------|-------------|
| Dependency graph | .orchestrator/decomposition/graph.yaml | State management, execution engine |
| Plan document | .orchestrator/plan.md | Human review, /ptf:status |
| Wave assignments | graph.yaml waves[] | Execution engine wave dispatch |

### Schema Integration
- **Dependency** schema (Phase 1) used directly for graph.yaml dependencies[]
- **Wave** schema (Phase 1) used directly for graph.yaml waves[]
- **Plan** schema can be generated from graph.yaml + tasks for complete plan.yaml if needed

## Open Questions

1. **Confidence threshold for flagging**
   - What we know: HIGH/MEDIUM/LOW confidence levels defined
   - What's unclear: Should LOW confidence deps require human confirmation?
   - Recommendation: Include all, flag LOW in warnings, let user review

2. **Semantic analysis depth**
   - What we know: LLM can analyze descriptions for references
   - What's unclear: How thorough should analysis be vs. speed?
   - Recommendation: Single-pass analysis, only HIGH-confidence matches

3. **Resource conflict ordering**
   - What we know: Conflicting tasks must serialize
   - What's unclear: Which should come first when no other ordering?
   - Recommendation: Use subgoal hierarchy first, then alphabetical by ID

## Sources

### Primary (HIGH confidence)
- [PARALLEL-TASK-FRAMEWORK.md Section 4](file:///Users/jaredmcfarland/Developer/ptf/PARALLEL-TASK-FRAMEWORK.md) - Dependency Analysis specification
- [PARALLEL-TASK-FRAMEWORK.md Section 5](file:///Users/jaredmcfarland/Developer/ptf/PARALLEL-TASK-FRAMEWORK.md) - Wave Computation specification
- [Kahn's Algorithm - GeeksforGeeks](https://www.geeksforgeeks.org/dsa/topological-sorting-indegree-based-solution/) - Topological sort algorithm
- [Tarjan's Algorithm - Wikipedia](https://en.wikipedia.org/wiki/Tarjan's_strongly_connected_components_algorithm) - Cycle detection algorithm
- [Bazel Dependencies](https://bazel.build/concepts/dependencies) - Artifact-based build system patterns

### Secondary (MEDIUM confidence)
- [Workflow Orchestration 2025](https://www.pracdata.io/p/state-of-workflow-orchestration-ecosystem-2025) - Airflow/Dagster/Prefect comparison
- [Dagster vs Prefect vs Airflow](https://www.zenml.io/blog/orchestration-showdown-dagster-vs-prefect-vs-airflow) - Modern orchestration patterns
- [How Bazel Works](https://www.gocodeo.com/post/how-bazel-works-dependency-graphs-caching-and-remote-execution) - Dependency graph construction

### Tertiary (LOW confidence)
- Semantic scheduling research papers - LLM-based dependency analysis patterns

## Metadata

**Confidence breakdown:**
- Multi-pass inference algorithm: HIGH - Matches Bazel/build system patterns exactly
- Kahn's algorithm for waves: HIGH - Standard algorithm, well-documented
- Tarjan's for cycle detection: HIGH - Standard algorithm, optimal complexity
- Semantic analysis approach: MEDIUM - LLM-based, requires tuning
- Domain heuristic patterns: HIGH - Already defined in adapters from Phase 2

**Research date:** 2026-01-18
**Valid until:** 2026-03-18 (60 days - algorithms are stable, implementation patterns established)

---
*Phase: 03-dependency-analysis*
*Research completed: 2026-01-18*
