# Execution Plan: Purchase Spike Root Cause Investigation

**Generated:** 2026-01-19T00:00:00Z
**Domain:** mixpanel-analytics
**Tasks:** 7 tasks in 3 waves
**Estimated parallel speedup:** 2.3x (7 tasks / 3 waves)

---

## Overview

Identify the root cause of the massive purchase spike that occurred on November 6th and 7th, 2021. Determine what triggered the anomalous behavior by analyzing Mixpanel event data around these dates.

### Success Criteria

- Identify the primary driver(s) of the purchase spike
- Quantify the spike magnitude vs baseline
- Provide evidence trail linking cause to effect
- Document methodology for reproducibility

---

## Wave 1 (5 tasks - parallel)

| Task | Description | Outputs |
|------|-------------|---------|
| baseline-purchases | Query baseline purchase events (Nov 1-5, 8-14) | baseline-purchases.json |
| spike-purchases | Query spike period purchase events (Nov 6-7) | spike-purchases.json |
| acquisition-segments | Segment spike purchases by acquisition source | acquisition-segments.json |
| campaign-correlation | Correlate spike with marketing campaigns | campaign-correlation.json |
| user-property-segments | Segment spike purchases by user properties | user-property-segments.json |

**No dependencies - can start immediately**

---

## Wave 2 (1 task)

| Task | Description | Outputs |
|------|-------------|---------|
| spike-magnitude | Calculate spike magnitude vs baseline | spike-magnitude.md |

**Depends on:** baseline-purchases, spike-purchases (Wave 1)

---

## Wave 3 (1 task)

| Task | Description | Outputs |
|------|-------------|---------|
| root-cause-analysis | Synthesize root cause analysis report | root-cause-analysis.md |

**Depends on:** All Wave 1 tasks + spike-magnitude (Wave 2)

---

## Dependency Graph

```
Wave 1: [baseline-purchases, spike-purchases, acquisition-segments, campaign-correlation, user-property-segments]
   │
   ├── baseline-purchases ──┬──> Wave 2: [spike-magnitude]
   │                        │           │
   └── spike-purchases ─────┘           │
                                        │
   ├── acquisition-segments ────────────┤
   │                                    │
   ├── campaign-correlation ────────────┼──> Wave 3: [root-cause-analysis]
   │                                    │
   └── user-property-segments ──────────┘
```

---

## Dependency Summary

| Type | Count | Confidence |
|------|-------|------------|
| artifact | 8 | HIGH |
| **Total** | **8** | |

---

## Warnings

No warnings - all dependencies are high confidence (artifact-based).

---

## Ready for Execution

All dependencies resolved. No cycles detected.

**Next:** Run `/ptf:execute` to begin execution

<sub>`/clear` first -> fresh context window recommended</sub>
