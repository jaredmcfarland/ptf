# Execution Plan: Fields Medal 2026 Research

**Generated:** 2026-01-19T10:00:00Z
**Domain:** prediction-market
**Tasks:** 16 tasks in 5 waves
**Estimated parallel speedup:** 3.2x (16 tasks / 5 waves)

---

## Overview

Research the Kalshi Fields Medal 2026 prediction market to determine if any candidate represents a 15%+ edge opportunity. Requires comprehensive research from minimal starting knowledge of the mathematics community and candidates.

### Success Criteria
- Understand Fields Medal selection criteria and historical patterns
- Research each candidate's mathematical contributions and eligibility
- Produce calibrated probability estimates from multiple perspectives
- Identify any candidates with 15%+ estimated edge vs market
- Deliver actionable recommendation (position or pass)

---

## Wave 1 (1 task)

| Task | Description | Output | Est. Context |
|------|-------------|--------|--------------|
| `fields-medal-context` | Research Fields Medal selection process, eligibility, history | `fields-medal-context.md` | 15% |

**No dependencies - can start immediately**

---

## Wave 2 (10 tasks - parallel)

| Task | Description | Output | Est. Context |
|------|-------------|--------|--------------|
| `candidate-hong-wang` | Research candidate profile - Hong Wang (81%) | `candidates/hong-wang.md` | 20% |
| `candidate-jacob-tsimerman` | Research candidate profile - Jacob Tsimerman (67%) | `candidates/jacob-tsimerman.md` | 20% |
| `candidate-john-pardon` | Research candidate profile - John Pardon (46%) | `candidates/john-pardon.md` | 20% |
| `candidate-yu-deng` | Research candidate profile - Yu Deng (36%) | `candidates/yu-deng.md` | 20% |
| `candidate-julian-sahasrabudhe` | Research candidate profile - Julian Sahasrabudhe (30%) | `candidates/julian-sahasrabudhe.md` | 20% |
| `candidate-sam-raskin` | Research candidate profile - Sam Raskin (24%) | `candidates/sam-raskin.md` | 20% |
| `candidate-jack-thorne` | Research candidate profile - Jack Thorne (19%) | `candidates/jack-thorne.md` | 20% |
| `candidate-will-sawin` | Research candidate profile - Will Sawin (16%) | `candidates/will-sawin.md` | 20% |
| `candidate-aleksandr-logunov` | Research candidate profile - Aleksandr Logunov (8%) | `candidates/aleksandr-logunov.md` | 20% |
| `candidate-alexander-efimov` | Research candidate profile - Alexander Efimov (7%) | `candidates/alexander-efimov.md` | 20% |

**Depends on:** Wave 1 (`fields-medal-context`)

---

## Wave 3 (3 tasks - parallel)

| Task | Description | Output | Est. Context |
|------|-------------|--------|--------------|
| `estimate-base-rate` | Base rate probability estimates using historical precedent | `estimates/base-rate-estimate.md` | 25% |
| `estimate-narrative` | Narrative probability estimates using momentum/stories | `estimates/narrative-estimate.md` | 25% |
| `estimate-contrarian` | Contrarian probability estimates challenging consensus | `estimates/contrarian-estimate.md` | 25% |

**Depends on:** Wave 1 (`fields-medal-context`) + Wave 2 (all 10 candidate profiles)

---

## Wave 4 (1 task)

| Task | Description | Output | Est. Context |
|------|-------------|--------|--------------|
| `synthesis` | Synthesize perspectives into calibrated consensus | `synthesis.md` | 25% |

**Depends on:** Wave 3 (all 3 estimate files)

---

## Wave 5 (1 task)

| Task | Description | Output | Est. Context |
|------|-------------|--------|--------------|
| `recommendation` | Calculate edge vs market, produce final recommendation | `recommendation.md` | 20% |

**Depends on:** Wave 4 (`synthesis`)

---

## Dependency Graph

```
Wave 1: [fields-medal-context]
   │
   └──> Wave 2: [candidate-hong-wang, candidate-jacob-tsimerman, ...]
                 (10 parallel candidate research tasks)
           │
           └──> Wave 3: [estimate-base-rate, estimate-narrative, estimate-contrarian]
                         (3 parallel perspective estimates)
                   │
                   └──> Wave 4: [synthesis]
                           │
                           └──> Wave 5: [recommendation]
```

### Critical Path

```
fields-medal-context -> candidate-* -> estimate-* -> synthesis -> recommendation
        │                   │              │            │             │
     Wave 1              Wave 2        Wave 3       Wave 4        Wave 5
```

---

## Dependency Summary

| Type | Count | Confidence |
|------|-------|------------|
| artifact | 56 | HIGH |
| semantic | 0 | - |
| implicit | 0 | - |
| resource | 0 | - |
| **Total** | **56** | |

---

## Warnings

No warnings - all dependencies are high confidence artifact dependencies.

---

## Market Context

| Candidate | Market Price | Notes |
|-----------|--------------|-------|
| Hong Wang | 81% | Highest probability |
| Jacob Tsimerman | 67% | |
| John Pardon | 46% | |
| Yu Deng | 36% | |
| Julian Sahasrabudhe | 30% | |
| Sam Raskin | 24% | |
| Jack Thorne | 19% | |
| Will Sawin | 16% | |
| Aleksandr Logunov | 8% | |
| Alexander Efimov | 7% | Lowest probability |

**Edge Threshold:** 15% (conservative)

---

## Ready for Execution

All dependencies resolved. No cycles detected.

**Execution Policy:**
- Max parallel tasks: 3 (per config)
- Ralph mode: enabled (fresh context per attempt)
- Max iterations per task: 3
- Checkpoints: at wave boundaries

**Next:** Run `/ptf:execute` to begin execution

<sub>`/clear` first -> fresh context window recommended</sub>
