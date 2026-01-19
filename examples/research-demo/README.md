# Research Demo

This example demonstrates the `research` domain adapter in action with a literature review project on technical debt.

## What This Example Demonstrates

1. **Research-focused questioning** - How `init_questions` gather methodology and scope bounds
2. **By-question decomposition** - The `by-question` heuristic splitting work by research sub-questions
3. **Research artifact flow** - Source -> Finding -> Summary -> Synthesis pattern
4. **Research verification** - Citation checking, claim-evidence binding

## The Goal

```
Literature review: What are effective strategies for reducing technical debt?
```

This is a typical research goal that the adapter shapes into a structured investigation plan.

## How the Adapter Shaped This Project

### 1. Clarification Questions (from adapter)

The adapter's `init_questions` gathered:

| Category | Question | Response |
|----------|----------|----------|
| research_question | What's the core research question? | What strategies effectively reduce technical debt? |
| existing_knowledge | What do you already know? | Some familiarity |
| methodology | What research methodology? | Literature review |
| scope_bounds | What are the boundaries? | Time period: 2015-2024, Source types: academic papers, industry reports |

### 2. Decomposition Strategy

The adapter's `by-question` heuristic decomposed the goal into subgoals:

```
Subgoal 1: Definitions
  - Task: define-technical-debt (What is technical debt?)
  - Task: define-reduction (What does "reducing" mean?)

Subgoal 2: Sources and Causes
  - Task: identify-causes (What causes technical debt?)

Subgoal 3: Interventions
  - Task: survey-strategies (What strategies exist?)
  - Task: evaluate-evidence (Which are most effective?)

Subgoal 4: Synthesis
  - Task: cross-theme-synthesis (Integrate findings across themes)
  - Task: recommendations (Final conclusions and recommendations)
```

### 3. Resulting Wave Structure

Dependencies form a 4-wave execution plan:

```
Wave 1: [define-technical-debt, define-reduction]  # Foundation questions
Wave 2: [identify-causes, survey-strategies]       # Depend on definitions
Wave 3: [evaluate-evidence]                        # Depends on strategies survey
Wave 4: [cross-theme-synthesis, recommendations]   # Depend on all findings
```

Parallelism factor: 1.75x (7 tasks / 4 waves)

### 4. Atomicity Applied

Each task follows the adapter's atomicity criteria:

| Criterion | How Applied |
|-----------|-------------|
| single-question | Each task addresses one research question |
| bounded-sources | Limited to 3-5 sources per task |
| verifiable-finding | Produces concrete finding with citations |
| fresh-context-completable | Source summaries kept under context limit |
| explicit-sources | All source dependencies declared |

## Exploring the Files

```
research-demo/
  README.md              # This walkthrough
  .orchestrator/
    config.yaml          # Project configuration
    decomposition/
      analysis.yaml      # Goal analysis with clarifications
      constitution.yaml  # Research integrity principles
```

### Key Files to Examine

**config.yaml** - References the research adapter:
```yaml
domain_adapter: research
```

**analysis.yaml** - Shows clarification responses matching adapter questions:
```yaml
clarifications:
  research_question: "What strategies effectively reduce technical debt?"
  methodology: "Literature review"
```

**constitution.yaml** - Generated from adapter template with research principles:
- Source Integrity: Always cite sources
- Claim-Evidence Binding: Every claim references evidence
- Methodology Consistency: Same criteria across sources
- Bias Awareness: Document limitations

## Dependency Graph Visualization

```
define-technical-debt --+
                        |
define-reduction -------+--> identify-causes --+
                        |                      |
                        +--> survey-strategies --+--> evaluate-evidence
                                                 |
                                                 +--> cross-theme-synthesis --> recommendations
```

The `source-to-finding` and `finding-to-synthesis` patterns from the adapter's `common_patterns` section drive this structure.

## Artifact Types in This Project

The research adapter defines these artifact types:

| Type | Purpose | Example |
|------|---------|---------|
| finding | Individual research finding | definitions/technical-debt-finding.md |
| summary | Source summary | sources/fowler-tech-debt-summary.md |
| synthesis | Integrated analysis | synthesis/strategies-synthesis.md |
| data | Extracted data | data/strategy-effectiveness.yaml |
| methodology | Research method docs | methodology/search-criteria.md |
| bibliography | Citations | bibliography.yaml |

## Verification Example

For the `evaluate-evidence` task, verification strategies from the adapter:

```yaml
verify:
  - type: exists
    target: findings/strategy-effectiveness.md
  - type: contains
    target: findings/strategy-effectiveness.md
    expected: "Evidence:"
  - type: contains
    target: findings/strategy-effectiveness.md
    expected: "Source:"
```

This follows the adapter's `finding` verification strategy (claims must reference sources).

## Comparison with Software Demo

| Aspect | Software Demo | Research Demo |
|--------|---------------|---------------|
| Decomposition | by-layer | by-question |
| Primary artifacts | source-code | finding, synthesis |
| Dependencies | schema -> repo -> service | source -> finding -> synthesis |
| Atomicity focus | single-file | bounded-sources |
| Verification | type checking | citation checking |

## Running This Example

This is a demonstration of adapter behavior, not a runnable project. To create a similar project:

1. Run `/ptf:init` with a research goal
2. Answer clarification questions from the adapter
3. Run `/ptf:decompose` to generate subgoals and tasks
4. Run `/ptf:execute-all` to execute the research plan

---

*Generated to demonstrate research.yaml adapter*
