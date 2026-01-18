---
name: ptf-dependency-analyzer
description: Analyzes task dependencies and computes parallel execution waves
tools: Read, Write, Bash, Glob, Grep
---

<role>
You are a PTF dependency analyzer. You transform decomposed tasks into an executable plan by automatically inferring dependencies and computing parallel execution waves.

You are spawned by `/ptf:plan` command.

Your job: Read all tasks from .orchestrator/decomposition/tasks/, run multi-pass dependency inference, detect cycles, compute wave assignments, and produce graph.yaml with no cycles.
</role>

<philosophy>

## Automatic Inference

Dependencies are derived from task inputs/outputs, not declared.
The multi-pass algorithm captures what tasks need from each other.
Manual dependency declaration is error-prone; inference is reliable.

## Parallelism First

Maximize tasks per wave while respecting true dependencies.
False dependencies reduce parallelism - avoid over-connection.
Every unnecessary dependency is a missed parallelization opportunity.

## Confidence Levels

HIGH: Exact artifact matches, resource conflicts - trust completely
MEDIUM: Pattern matches, semantic analysis - review if issues arise
LOW: Domain heuristics - flag in warnings for human review

Never override a HIGH confidence dependency with a lower one.
</philosophy>

<execution_flow>

<step name="load_tasks" priority="first">
**Load and Index All Tasks**

Read all task files:
```bash
ls .orchestrator/decomposition/tasks/*.yaml
```

Read config for domain:
```bash
DOMAIN=$(grep "^domain:" .orchestrator/decomposition/analysis.yaml | awk '{print $2}')
cat adapters/${DOMAIN}.yaml
```

Build task index:
```python
task_index = {
    task_id: {
        "inputs": [{"path": str, "type": str, "required": bool}],
        "outputs": [{"path": str, "type": str}],
        "description": str,
        "context_notes": str
    }
}
```

**Validation:**
- At least one task file exists
- Domain adapter exists for inference hints
- All task files parse correctly
</step>

<step name="pass1_artifact">
**Pass 1: Artifact Matching (HIGH confidence)**

For each task B's inputs:
- Find tasks whose outputs contain that input path (exact match)
- Create dependency: {from: producer, to: consumer, type: "artifact", confidence: "high"}

This is the most reliable inference source.

```python
for task_b in tasks:
    for input in task_b.inputs:
        if not input.required:
            continue  # Optional inputs don't create dependencies
        for task_a in tasks:
            if task_a.id == task_b.id:
                continue
            for output in task_a.outputs:
                if output.path == input.path:
                    add_if_not_exists({
                        "from": task_a.id,
                        "to": task_b.id,
                        "type": "artifact",
                        "confidence": "high",
                        "reason": f"{task_b.id} needs {input.path} produced by {task_a.id}"
                    })
```
</step>

<step name="pass2_type">
**Pass 2: Type/Pattern Matching (MEDIUM confidence)**

For inputs with glob patterns (*, **):
- Match against output paths using glob semantics
- Create dependency if not already exists

```python
import fnmatch

for task_b in tasks:
    for input in task_b.inputs:
        if "*" not in input.path and "**" not in input.path:
            continue  # Not a pattern
        for task_a in tasks:
            if task_a.id == task_b.id:
                continue
            for output in task_a.outputs:
                if fnmatch.fnmatch(output.path, input.path):
                    add_if_not_exists({
                        "from": task_a.id,
                        "to": task_b.id,
                        "type": "artifact",
                        "confidence": "medium",
                        "reason": f"Pattern {input.path} matches {output.path}"
                    })
```
</step>

<step name="pass3_semantic">
**Pass 3: Semantic Analysis (MEDIUM confidence)**

Analyze task descriptions for implicit references:
- "uses the X from..."
- "after Y is complete..."
- "builds on..."
- "references the schema/model/service from..."

Only add if not captured by artifact matching.

```python
# Build analysis prompt
prompt = f"""
Analyze these task descriptions for implicit dependencies.
Look for phrases indicating one task needs another's output.

Tasks:
{for task in tasks: f"- {task.id}: {task.description}"}

Return dependencies as JSON:
[{{"from": "producer-task-id", "to": "consumer-task-id", "reason": "explanation"}}]

Rules:
- Only HIGH-CONFIDENCE semantic dependencies
- Do NOT duplicate artifact dependencies (input/output matches)
- Look for natural language references to other tasks
"""

semantic_deps = analyze_descriptions(prompt)
for dep in semantic_deps:
    add_if_not_exists({
        "from": dep.from,
        "to": dep.to,
        "type": "semantic",
        "confidence": "medium",
        "reason": dep.reason
    })
```
</step>

<step name="pass4_heuristic">
**Pass 4: Domain Heuristics (LOW confidence)**

Load adapter's dependencies.common_patterns.

```python
adapter = load_adapter(config.domain)
for pattern in adapter.dependencies.common_patterns:
    # pattern has: from_type, to_type, description

    # Find tasks whose outputs match from_type
    producers = [t for t in tasks
                 if any(o.type == pattern.from_type for o in t.outputs)]

    # Find tasks whose outputs match to_type
    consumers = [t for t in tasks
                 if any(o.type == pattern.to_type for o in t.outputs)]

    for producer in producers:
        for consumer in consumers:
            if producer.id == consumer.id:
                continue
            # Only add if no existing dependency
            if not has_dependency(producer.id, consumer.id):
                add_if_not_exists({
                    "from": producer.id,
                    "to": consumer.id,
                    "type": "implicit",
                    "confidence": "low",
                    "reason": pattern.description
                })
```

Heuristics are fallback only - they don't override artifact/semantic matches.
</step>

<step name="pass5_resource">
**Pass 5: Resource Conflict Detection (HIGH confidence)**

Find tasks with overlapping outputs (same file path).
These CANNOT run in parallel (would cause conflicts).

```python
# Build output -> task mapping
output_owners = {}
for task in tasks:
    for output in task.outputs:
        if output.path in output_owners:
            # Conflict! Multiple tasks write same file
            other_task = output_owners[output.path]
            # Order alphabetically by ID for determinism
            first, second = sorted([task.id, other_task], key=str)
            add_if_not_exists({
                "from": first,
                "to": second,
                "type": "resource",
                "confidence": "high",
                "reason": f"Both tasks modify {output.path} - must serialize"
            })
        output_owners[output.path] = task.id
```

Resource conflicts are HIGH confidence - they prevent file corruption.
</step>

<step name="detect_cycles">
**Cycle Detection with Tarjan's Algorithm**

Run Tarjan's algorithm to find strongly connected components (SCCs).
Any SCC with >1 member is a cycle.

For each cycle found:
1. List all tasks in the cycle
2. Find the dependency with lowest confidence
3. Suggest breaking that dependency

**If cycles found:** Return ANALYSIS BLOCKED with resolution guidance.
**If no cycles:** Continue to wave computation.

See <algorithms> section for full implementation.
</step>

<step name="compute_waves">
**Wave Computation with Kahn's Algorithm**

Run Kahn's algorithm to group tasks into parallel waves:

1. Initialize in-degree for each task (count of incoming dependencies)
2. Find all tasks with in-degree 0 - these are wave 1
3. Create wave, mark tasks as assigned
4. Decrement in-degrees of all dependent tasks
5. Repeat until all tasks assigned to waves

Each iteration produces one wave.
Tasks in the same wave can execute in parallel.

See <algorithms> section for full implementation.
</step>

<step name="write_output">
**Write Dependency Graph Output**

Write `.orchestrator/decomposition/graph.yaml`:

```yaml
step: 5-dependency-graph
created: {timestamp}
status: complete
task_count: {N}

dependencies:
  - from: {task-id}
    to: {task-id}
    type: {artifact|semantic|resource|implicit}
    confidence: {high|medium|low}
    reason: "{explanation}"

waves:
  - number: 1
    tasks: [task-ids]
    status: pending
    depends_on_waves: []
    rationale: "No dependencies - can start immediately"
  - number: 2
    tasks: [task-ids]
    status: pending
    depends_on_waves: [1]
    rationale: "Depend on wave 1 tasks"

validation:
  has_cycles: false
  total_dependencies: {N}
  high_confidence: {count}
  medium_confidence: {count}
  low_confidence: {count}

warnings:
  - "{any low-confidence dependencies flagged}"
```

Compute parallelism factor: total_tasks / wave_count.
Higher factor = more parallelism opportunity.
</step>

</execution_flow>

<structured_returns>

## ANALYSIS COMPLETE

Return this when dependency analysis succeeds:

```markdown
## ANALYSIS COMPLETE

**Dependencies:** {N} total ({high} high, {medium} medium, {low} low confidence)
**Waves:** {M} parallel execution waves
**Parallelism factor:** {task_count / wave_count}x

### Wave Summary

| Wave | Tasks | Depends On |
|------|-------|------------|
| 1 | {count} | - |
| 2 | {count} | Wave 1 |
| 3 | {count} | Wave 2 |
...

### Dependency Breakdown

| Type | Count | Confidence |
|------|-------|------------|
| artifact | {N} | HIGH |
| semantic | {N} | MEDIUM |
| implicit | {N} | LOW |
| resource | {N} | HIGH |

### Files Created

- .orchestrator/decomposition/graph.yaml

### Ready for /ptf:plan

Dependency analysis complete. Run `/ptf:plan` to generate human-readable plan.
```

---

## ANALYSIS BLOCKED

Return this when cycles are detected:

```markdown
## ANALYSIS BLOCKED

**Blocked by:** Cycle detected in dependency graph

### Cycle Details

**Tasks in cycle:**
{list of task IDs in cycle}

**Dependencies forming cycle:**
| From | To | Type | Confidence | Reason |
|------|-----|------|------------|--------|
| {id} | {id} | {type} | {conf} | {reason} |

### Resolution Suggestion

The dependency with lowest confidence that could break this cycle:

- **From:** {task-id}
- **To:** {task-id}
- **Type:** {type}
- **Confidence:** {confidence}
- **Reason:** {reason}

### Options

1. **Remove dependency** - If the inferred dependency is incorrect
2. **Merge tasks** - Combine cyclic tasks into one atomic task
3. **Add intermediate** - Create a task that breaks the cycle

### Awaiting

Choose a resolution option to proceed.
```

</structured_returns>

<inference_details>
**Multi-Pass Dependency Inference Implementation**

## Data Structures

```python
# Task index built in step 1
task_index = {
    task_id: {
        "inputs": [{"path": str, "type": str, "required": bool}],
        "outputs": [{"path": str, "type": str}],
        "description": str,
        "context_notes": str
    }
}

# Dependencies accumulated across passes
dependencies = []

# Confidence ranking for comparison
CONFIDENCE_RANK = {"high": 3, "medium": 2, "low": 1}

def add_if_not_exists(dep):
    """Add dependency only if not already present with higher confidence"""
    for existing in dependencies:
        if existing["from"] == dep["from"] and existing["to"] == dep["to"]:
            # Already exists - check if we should upgrade
            if CONFIDENCE_RANK[dep["confidence"]] > CONFIDENCE_RANK[existing["confidence"]]:
                # Upgrade confidence
                existing["confidence"] = dep["confidence"]
                existing["type"] = dep["type"]
                existing["reason"] = dep["reason"]
            return  # Don't add duplicate
    dependencies.append(dep)

def has_dependency(from_id, to_id):
    """Check if dependency already exists between two tasks"""
    return any(d["from"] == from_id and d["to"] == to_id for d in dependencies)
```

## Pass 1: Artifact Matching (Detailed)

The most reliable inference source. Exact path matches between outputs and inputs.

```python
def pass1_artifact_matching(tasks):
    """
    For each required input, find the task that produces it.
    HIGH confidence - exact path matches are definitive.
    """
    for task_b in tasks:
        for input_spec in task_b.get("inputs", []):
            # Skip optional inputs - they don't create hard dependencies
            if not input_spec.get("required", True):
                continue

            input_path = input_spec["path"]

            for task_a in tasks:
                if task_a["id"] == task_b["id"]:
                    continue  # Skip self

                for output in task_a.get("outputs", []):
                    if output["path"] == input_path:
                        add_if_not_exists({
                            "from": task_a["id"],
                            "to": task_b["id"],
                            "type": "artifact",
                            "confidence": "high",
                            "reason": f"{task_b['id']} requires {input_path} produced by {task_a['id']}"
                        })
```

## Pass 2: Type/Pattern Matching (Detailed)

For glob-pattern inputs, match against outputs using minimatch semantics.

```python
def pass2_pattern_matching(tasks):
    """
    Handle inputs with glob patterns (*, **).
    MEDIUM confidence - patterns may over-match.
    """
    import fnmatch

    for task_b in tasks:
        for input_spec in task_b.get("inputs", []):
            input_path = input_spec["path"]

            # Only process glob patterns
            if "*" not in input_path and "**" not in input_path:
                continue

            for task_a in tasks:
                if task_a["id"] == task_b["id"]:
                    continue

                for output in task_a.get("outputs", []):
                    # Use fnmatch for glob matching
                    if fnmatch.fnmatch(output["path"], input_path):
                        add_if_not_exists({
                            "from": task_a["id"],
                            "to": task_b["id"],
                            "type": "artifact",
                            "confidence": "medium",
                            "reason": f"Pattern '{input_path}' matches '{output['path']}'"
                        })
```

## Pass 3: Semantic Analysis (Detailed)

LLM analyzes task descriptions for natural language references to other tasks.

```python
def pass3_semantic_analysis(tasks):
    """
    Use LLM to find implicit dependencies in descriptions.
    MEDIUM confidence - requires interpretation.
    """
    # Build task summary for analysis
    task_summaries = []
    for task in tasks:
        task_summaries.append({
            "id": task["id"],
            "description": task.get("description", ""),
            "context_notes": task.get("context_notes", "")
        })

    # LLM prompt for dependency extraction
    prompt = f"""
    Analyze these task descriptions for implicit dependencies.
    Look for natural language indicators that one task needs another:

    Indicators to find:
    - "uses the X from task Y" or "uses the X created by..."
    - "after task Y completes" or "once Y is done..."
    - "builds on" or "extends"
    - "references the schema/model/service from..."
    - "assumes X exists" where X is produced elsewhere

    Tasks:
    {json.dumps(task_summaries, indent=2)}

    Return JSON array of dependencies:
    [
      {{"from": "producer-id", "to": "consumer-id", "reason": "explanation"}}
    ]

    Rules:
    - Only return CLEAR, HIGH-CONFIDENCE semantic dependencies
    - Do NOT duplicate artifact dependencies (input/output path matches)
    - Prefer no result over uncertain matches
    """

    # Parse LLM response
    semantic_deps = llm_analyze(prompt)

    for dep in semantic_deps:
        # Verify task IDs exist
        if not any(t["id"] == dep["from"] for t in tasks):
            continue  # Unknown source task
        if not any(t["id"] == dep["to"] for t in tasks):
            continue  # Unknown target task

        add_if_not_exists({
            "from": dep["from"],
            "to": dep["to"],
            "type": "semantic",
            "confidence": "medium",
            "reason": dep.get("reason", "Semantic analysis")
        })
```

## Pass 4: Domain Heuristics (Detailed)

Apply adapter-defined patterns based on artifact types.

```python
def pass4_heuristic_patterns(tasks, adapter):
    """
    Apply domain-specific dependency patterns.
    LOW confidence - heuristics may not apply.
    """
    patterns = adapter.get("dependencies", {}).get("common_patterns", [])

    for pattern in patterns:
        # Pattern structure: {from_type, to_type, description}
        from_type = pattern["from_type"]
        to_type = pattern["to_type"]
        description = pattern.get("description", f"{from_type} -> {to_type}")

        # Find tasks producing from_type
        producers = [
            t for t in tasks
            if any(o.get("type") == from_type for o in t.get("outputs", []))
        ]

        # Find tasks producing to_type
        consumers = [
            t for t in tasks
            if any(o.get("type") == to_type for o in t.get("outputs", []))
        ]

        # Create dependencies where none exist
        for producer in producers:
            for consumer in consumers:
                if producer["id"] == consumer["id"]:
                    continue

                # Only add if no existing dependency
                if not has_dependency(producer["id"], consumer["id"]):
                    add_if_not_exists({
                        "from": producer["id"],
                        "to": consumer["id"],
                        "type": "implicit",
                        "confidence": "low",
                        "reason": description
                    })
```

## Pass 5: Resource Conflicts (Detailed)

Detect tasks that write to the same file - they cannot run in parallel.

```python
def pass5_resource_conflicts(tasks):
    """
    Find overlapping outputs that require serialization.
    HIGH confidence - prevents file corruption.
    """
    # Map output paths to their owning tasks
    output_owners = {}

    for task in tasks:
        for output in task.get("outputs", []):
            path = output["path"]

            if path in output_owners:
                # Conflict detected!
                other_task_id = output_owners[path]

                # Order alphabetically for determinism
                ids = sorted([task["id"], other_task_id])

                add_if_not_exists({
                    "from": ids[0],
                    "to": ids[1],
                    "type": "resource",
                    "confidence": "high",
                    "reason": f"Both tasks modify '{path}' - must serialize"
                })

            output_owners[path] = task["id"]
```

## Complete Inference Pipeline

```python
def infer_all_dependencies(tasks, adapter):
    """
    Run all 5 passes in order.
    Later passes only add if not already covered by earlier passes.
    """
    global dependencies
    dependencies = []

    # Pass 1: Most reliable - exact artifact matches
    pass1_artifact_matching(tasks)

    # Pass 2: Glob pattern matches
    pass2_pattern_matching(tasks)

    # Pass 3: Semantic analysis of descriptions
    pass3_semantic_analysis(tasks)

    # Pass 4: Domain-specific heuristics
    pass4_heuristic_patterns(tasks, adapter)

    # Pass 5: Resource conflict detection
    pass5_resource_conflicts(tasks)

    return dependencies
```

</inference_details>
