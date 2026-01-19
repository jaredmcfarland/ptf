# Phase 8: Domain Adapters - Research

**Researched:** 2026-01-18
**Domain:** Domain adapter verification and completion
**Confidence:** HIGH

## Summary

Phase 8 focuses on verifying and completing the domain adapter system. The core adapter files were already created in Phase 2, but their integration with all framework phases needs verification and gaps addressed. Research reveals that the existing adapters are substantially complete, with most integration points already functional in the decomposer, dependency analyzer, and verifier agents.

The primary gaps are:
1. No formal adapter schema (ADAPT-01)
2. No end-to-end example projects demonstrating adapter usage
3. Limited SKILL.md documentation on adapter customization
4. Verification strategies defined but not fully exercised in tests

**Primary recommendation:** Focus on validation testing, formal schema creation, and documentation rather than major implementation work.

## Current State Analysis

### Adapter Files Present

| File | Lines | Sections Present |
|------|-------|------------------|
| `adapters/software-development.yaml` | 261 | questioning, decomposition, constitution, artifacts, dependencies |
| `adapters/research.yaml` | 269 | questioning, decomposition, constitution, artifacts, dependencies |
| `adapters/template.yaml` | 239 | All sections with [CUSTOMIZE] markers |

### Requirements Gap Analysis

| Requirement | Status | Gap |
|-------------|--------|-----|
| **ADAPT-01**: Domain adapter interface | PARTIAL | No formal schema in `schemas/adapter.schema.yaml` |
| **ADAPT-02**: Adapter loading/integration | COMPLETE | Decomposer, verifier, dependency-analyzer all load adapters |
| **ADAPT-03**: Software adapter complete | NEEDS VERIFICATION | All sections present, needs validation testing |
| **ADAPT-04**: Software decomposition heuristics | COMPLETE | 4 heuristics: by-layer, by-feature, by-file-boundary, by-interface |
| **ADAPT-05**: Software atomicity criteria | COMPLETE | 5 criteria: single-file, fresh-context-completable, verifiable, focused, explicit-inputs |
| **ADAPT-06**: Software artifact types | COMPLETE | 5 types: source-code, migration, config, test, documentation |
| **ADAPT-07**: Software verification strategies | COMPLETE | Strategies defined per artifact type |
| **ADAPT-08**: Software dependency patterns | COMPLETE | 4 patterns + 4 inference hints |
| **ADAPT-09**: Research adapter complete | NEEDS VERIFICATION | All sections present, needs validation testing |
| **ADAPT-10**: Research decomposition heuristics | COMPLETE | 4 heuristics: by-question, by-source, by-stage, by-synthesis |
| **ADAPT-11**: Research atomicity criteria | COMPLETE | 5 criteria: single-question, bounded-sources, verifiable-finding, fresh-context-completable, explicit-sources |
| **ADAPT-12**: Research artifact types | COMPLETE | 6 types: finding, summary, synthesis, data, methodology, bibliography |
| **ADAPT-13**: Research verification strategies | COMPLETE | Strategies defined per artifact type |
| **ADAPT-14**: Research dependency patterns | COMPLETE | 4 patterns + 4 inference hints |
| **ADAPT-15**: Template adapter | COMPLETE | Full template with [CUSTOMIZE] markers and extensive comments |
| **ADAPT-16**: Constitution templates | COMPLETE | Both adapters have domain-specific constitution templates |

## Integration Points

### Current Integration Status

| Agent/Command | Integration Status | How It Uses Adapters |
|---------------|-------------------|----------------------|
| `ptf:init` | COMPLETE | Detects domain, loads adapter, uses questioning section |
| `ptf:decompose` | COMPLETE | Uses decomposition heuristics, atomicity criteria |
| `ptf-decomposer` | COMPLETE | Loads adapter, applies subgoal_heuristics, atomicity_criteria |
| `ptf-verifier` | PARTIAL | Has adapter verification_strategies section but not fully exercised |
| `ptf-dependency-analyzer` | COMPLETE | Uses dependencies.common_patterns and inference_hints |
| `ptf:plan` | COMPLETE | References adapter for dependency analysis |

### Integration Code Patterns

**Adapter Loading (from ptf-decomposer):**
```bash
DOMAIN=$(grep "^domain:" .orchestrator/decomposition/analysis.yaml | awk '{print $2}')
cat adapters/${DOMAIN}.yaml
```

**Verification Strategy Loading (from ptf-verifier):**
```bash
load_adapter() {
  local config_path=".orchestrator/config.yaml"
  local adapter_name=$(yq '.domain_adapter' "$config_path")
  local adapter_path="adapters/${adapter_name}.yaml"
}
```

## Don't Hand-Roll

Problems that have existing solutions in the adapters:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Domain-specific task breakdown | Custom decomposition logic | `decomposition.subgoal_heuristics` | Adapters already define domain patterns |
| Atomicity checking | Custom validation | `decomposition.atomicity_criteria` | Checklist pattern already defined |
| Artifact verification | Per-project commands | `artifacts.verification_strategies` | Verification methods per type exist |
| Dependency inference | Manual dependency declaration | `dependencies.common_patterns` | Adapter patterns enable automatic inference |
| Constitution principles | Per-project principles | `constitution.template` | Domain templates capture best practices |

## Gaps to Address

### Gap 1: Formal Adapter Schema (ADAPT-01)

**Current state:** No `schemas/adapter.schema.yaml` exists
**Required:** Formal YAML schema documenting adapter interface
**Impact:** Without formal schema, adapter validation and IDE support unavailable

**Recommended structure:**
```yaml
# schemas/adapter.schema.yaml
$schema: "http://json-schema.org/draft-07/schema#"
title: Domain Adapter Schema
type: object
required:
  - name
  - description
  - version
  - questioning
  - decomposition
  - constitution
  - artifacts
  - dependencies
properties:
  name: {type: string}
  description: {type: string}
  version: {type: string}
  questioning:
    type: object
    required: [init_questions]
    properties:
      init_questions:
        type: array
        items:
          type: object
          required: [category, question, why, options_template]
  # ... (full schema definition needed)
```

### Gap 2: End-to-End Example Projects

**Current state:** Example plans exist but no complete "run with adapter" demonstration
**Required:**
- Software development example: Simple API with schema -> repo -> service -> API flow
- Research example: Literature review with source -> finding -> synthesis flow

**Example structure:**
```
examples/
  software-demo/
    README.md           # Instructions to run end-to-end
    .orchestrator/      # Pre-populated for demonstration
      config.yaml
      decomposition/
        analysis.yaml
        constitution.yaml
  research-demo/
    README.md
    .orchestrator/
      config.yaml
      decomposition/
        analysis.yaml
        constitution.yaml
```

### Gap 3: SKILL.md Documentation

**Current state:** Brief mention of adapters (3 lines in SKILL.md)
**Required:** Comprehensive adapter documentation

**Missing documentation:**
- How to create a custom adapter
- Adapter section reference (questioning, decomposition, etc.)
- How each section integrates with framework phases
- Validation/testing guidance for custom adapters

### Gap 4: Verification Strategy Exercising

**Current state:** Verification strategies defined in adapters, verifier agent references them
**Required:** Verification that adapter strategies are actually applied during verification

**Test scenarios needed:**
- Source-code artifact: exists + syntax + runs (tsc --noEmit)
- Migration artifact: exists + syntax + runs (prisma validate)
- Test artifact: exists + runs (npm test)
- Config artifact: exists + syntax

## Verification Patterns

### Software Development Verification

| Artifact Type | Verification Chain |
|--------------|-------------------|
| source-code | exists -> syntax -> runs (type-check) |
| migration | exists -> syntax -> runs (validate) |
| test | exists -> runs (test execution) |
| config | exists -> syntax (YAML/JSON parse) |
| documentation | exists -> contains (required sections) |

### Research Verification

| Artifact Type | Verification Chain |
|--------------|-------------------|
| finding | exists -> contains (claim, evidence, source) |
| summary | exists -> contains (key points, source reference) |
| synthesis | exists -> integrates (references findings) -> contains (conclusions) |
| data | exists -> syntax (CSV/JSON/YAML) |
| methodology | exists -> contains (criteria, process) |
| bibliography | exists -> contains (all cited sources) |

## Common Pitfalls

### Pitfall 1: Adapter Loading Assumes File Exists

**What goes wrong:** Commands fail silently if adapter file missing
**Why it happens:** `cat adapters/${DOMAIN}.yaml` without existence check
**How to avoid:** All integration points already have existence checks, but should return clear error
**Warning signs:** "Unknown domain" errors during decomposition

### Pitfall 2: Verification Strategy Not Applied

**What goes wrong:** Verifier uses task-declared verification only, ignores adapter strategies
**Why it happens:** Adapter strategy lookup is optional/fallback in current verifier
**How to avoid:** Ensure get_verification_steps() merges task + adapter strategies
**Warning signs:** Source files verified without type-checking

### Pitfall 3: Template Markers Left in Custom Adapter

**What goes wrong:** [CUSTOMIZE] markers appear in generated constitutions
**Why it happens:** User copies template without replacing all markers
**How to avoid:** Add validation that rejects adapters with [CUSTOMIZE] markers
**Warning signs:** Constitution contains "[CUSTOMIZE]" text

### Pitfall 4: Heuristic Mismatch to Domain

**What goes wrong:** Software heuristics applied to research tasks (or vice versa)
**Why it happens:** Domain detection incorrect or adapter manually overridden wrong
**How to avoid:** Validate domain in analysis.yaml matches adapter being loaded
**Warning signs:** "by-layer" heuristics appearing in research decomposition

## Code Examples

### Adapter Section Usage

**Questioning (ptf:init):**
```yaml
# From adapter
questioning:
  init_questions:
    - category: core_value
      question: "What's the ONE thing that must work perfectly?"
      why: "Identifies critical path"

# Used in init command
for question in adapter.questioning.init_questions:
    response = AskUserQuestion(question.question)
    clarification[question.category] = response
```

**Decomposition (ptf-decomposer):**
```yaml
# From adapter
decomposition:
  subgoal_heuristics:
    - name: by-layer
      when_to_use: "Feature spans multiple layers"

# Used in decomposer
if goal_spans_layers(goal):
    apply_heuristic("by-layer", goal)
```

**Atomicity Evaluation (ptf-decomposer):**
```yaml
# From adapter
atomicity_criteria:
  - criterion: single-file
    check: "Task produces at most one file"
    fail_signal: "outputs array has >3 files"

# Used in decomposer
for criterion in adapter.atomicity_criteria:
    if not passes_criterion(task, criterion):
        issues.append(criterion.fail_signal)
        needs_decomposition = True
```

**Verification (ptf-verifier):**
```yaml
# From adapter
artifacts:
  verification_strategies:
    source-code:
      - method: exists
      - method: syntax
      - method: runs
        command_template: "npx tsc --noEmit {path}"

# Used in verifier
for output in task.outputs:
    strategies = adapter.artifacts.verification_strategies[output.type]
    for strategy in strategies:
        run_verification(strategy, output.path)
```

**Dependency Patterns (ptf-dependency-analyzer):**
```yaml
# From adapter
dependencies:
  common_patterns:
    - name: schema-to-repository
      from_type: migration
      to_type: source-code
      confidence: high

# Used in dependency analyzer
for pattern in adapter.dependencies.common_patterns:
    producers = find_tasks_producing(pattern.from_type)
    consumers = find_tasks_producing(pattern.to_type)
    for p in producers:
        for c in consumers:
            add_dependency(p, c, "implicit", pattern.confidence)
```

## State of the Art

| Aspect | Current State | Notes |
|--------|--------------|-------|
| Adapter structure | Stable | 5 sections: questioning, decomposition, constitution, artifacts, dependencies |
| Software adapter | Complete | All sections populated, ready for validation |
| Research adapter | Complete | All sections populated, ready for validation |
| Template adapter | Complete | Self-documenting with [CUSTOMIZE] markers |
| Integration | Mostly complete | Decomposer, analyzer, verifier all use adapters |
| Documentation | Partial | Brief mention in SKILL.md, no detailed guide |
| Validation | Missing | No adapter schema, no validation tests |
| Examples | Partial | Plan examples exist, no end-to-end project demos |

## Open Questions

### Question 1: Adapter Versioning

- What we know: Adapters have a `version: "1.0"` field
- What's unclear: How version compatibility is checked, if at all
- Recommendation: For v1, ignore versioning; document for future

### Question 2: Custom Verification Commands

- What we know: `command_template` uses `{path}` placeholder
- What's unclear: Are other placeholders supported? (`{project_root}`, `{artifact_type}`)
- Recommendation: Document current `{path}` only, defer extensions to v2

### Question 3: Adapter Extension/Inheritance

- What we know: Template adapter is a copy-and-modify pattern
- What's unclear: Could adapters extend/override others? (e.g., "extends: software-development")
- Recommendation: Out of scope for v1, document copy pattern as standard approach

## Recommended Phase Structure

Based on research, Phase 8 should focus on:

1. **Validation Testing** - Verify existing adapters work end-to-end
2. **Formal Schema** - Create `schemas/adapter.schema.yaml`
3. **Example Projects** - Create software-demo and research-demo examples
4. **Documentation** - Expand SKILL.md adapter section
5. **Gap Fixes** - Address any integration issues found in testing

## Sources

### Primary (HIGH confidence)
- Existing adapter files (`adapters/*.yaml`) - read and analyzed
- Agent files (`ptf-decomposer.md`, `ptf-verifier.md`, `ptf-dependency-analyzer.md`) - integration patterns
- Command files (`ptf/init.md`, `ptf/decompose.md`, `ptf/plan.md`) - usage patterns
- REQUIREMENTS.md - requirement definitions ADAPT-01 through ADAPT-16

### Secondary (MEDIUM confidence)
- SKILL.md - framework documentation
- Example files (`examples/plans/*.yaml`) - usage patterns

## Metadata

**Confidence breakdown:**
- Requirements analysis: HIGH - direct from REQUIREMENTS.md and code
- Integration status: HIGH - verified in agent/command files
- Gap identification: HIGH - verified by file existence checks
- Recommendations: MEDIUM - based on analysis, not tested

**Research date:** 2026-01-18
**Valid until:** 2026-02-18 (stable domain, 30-day validity)
