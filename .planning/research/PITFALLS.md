# Pitfalls Research

**Domain:** LLM Agent Orchestration & Task Decomposition
**Researched:** 2026-01-18
**Confidence:** HIGH (corroborated by multiple academic papers, production post-mortems, and framework documentation)

## Executive Summary

Research reveals multi-agent LLM systems fail at rates between **41-86.7% in production**. The root causes are not primarily technical infrastructure issues (~16% of failures) but rather specification problems (41.77%) and coordination failures (36.94%). For PTF specifically, the core thesis of "fresh context execution" directly addresses the #1 documented failure mode: context degradation. However, several domain-specific pitfalls require deliberate architectural decisions to avoid.

---

## Critical Pitfalls

### Pitfall 1: The Infinite Loop Trap

**What goes wrong:**
Agents get stuck in recursive loops, attempting the same failed task repeatedly without progress. AutoGPT's original architecture became infamous for this—agents would try, fail, and retry the exact same approach for hundreds of iterations until hitting preset limits or human intervention.

**Why it happens:**
- LLMs lack true memory of what they've already attempted
- Naive semantic search over previous actions increases loop probability (keywords in goals match keywords in failed actions)
- Finite context windows cause the agent to "forget" previous failures
- Limited reasoning ability combined with restricted tool sets creates situations where no valid path exists

**How to avoid:**
- Implement explicit loop detection with configurable thresholds (e.g., same action signature >3 times = escalate)
- Maintain a "tried actions" log outside the LLM's context that's checked before execution
- Add "escape hatch" mechanisms: if stuck, decompose differently or escalate to human
- Use structured action signatures for comparison, not just semantic similarity

**Warning signs:**
- Same tool calls appearing in logs repeatedly
- Task duration exceeding expected bounds by >2x
- Agent producing nearly identical reasoning chains
- Token consumption spiking without state progress

**Phase to address:** Core Execution Engine (early phase)—loop detection must be a primitive, not an afterthought

---

### Pitfall 2: Context Degradation (The "Context Rot" Problem)

**What goes wrong:**
LLM performance degrades predictably as context fills. Research shows at 32k tokens, 11 out of 12 tested models dropped below 50% of their short-context performance. Even with perfect retrieval, reasoning capability still degrades substantially as input length increases.

**Why it happens:**
- Attention mechanisms scale quadratically (O(n^2)) with sequence length
- "Working memory" bottleneck in transformers is exceeded long before context window fills
- More complex tasks show more severe degradation—exactly the tasks agents need to do
- Information presented early gets "lost" as context grows (primacy/recency effects)

**How to avoid:**
- **PTF's core thesis addresses this directly**: fresh context for every task
- Design tasks to complete within 0-30% of context window (peak performance zone)
- Wave-based execution ensures each task starts fresh, not inheriting accumulated context
- Never pass accumulated conversation history between independent tasks

**Warning signs:**
- Quality metrics declining in later waves despite similar task complexity
- Agents "forgetting" constraints established earlier in execution
- Increasing hallucination rates as execution progresses
- Verification failures clustering in later execution phases

**Phase to address:** Foundation (task schemas must include context budget estimates)

---

### Pitfall 3: Task Granularity Mismatch

**What goes wrong:**
Tasks decomposed too coarsely fail because they exceed LLM capability. Tasks decomposed too finely create coordination overhead that exceeds the value of parallelization. Both failure modes are common and often coexist in the same system.

**Why it happens:**
- No principled measure of "task complexity" exists—decomposition is largely heuristic
- Over-engineering leads to diminishing returns or negative value from excessive decomposition
- Under-decomposition creates tasks where "inability to execute any sub-task may lead to task failure"
- LLMs are not great strategic planners—initial decomposition is often deeply flawed

**How to avoid:**
- Use adaptive decomposition (ADaPT pattern): decompose further only when execution fails
- Define "atomic task" criteria: completable within context budget, single verifiable output
- Monitor decomposition quality metrics: tasks_failed_as_too_complex vs coordination_overhead
- Allow re-decomposition at execution time, not just planning time

**Warning signs:**
- High percentage of tasks failing on first attempt
- Coordination/orchestration time exceeding task execution time
- Tasks producing partial outputs that require continuation
- Wave sizes becoming extremely large (>10 parallel tasks may indicate over-decomposition)

**Phase to address:** Decomposition Commands (core decomposition logic) + Execution Engine (adaptive re-decomposition)

---

### Pitfall 4: Cascading Failure Propagation

**What goes wrong:**
A single early mistake propagates through subsequent decisions, leading to complete task failure. One agent's erroneous output becomes another agent's corrupted input, and errors compound rather than correct.

**Why it happens:**
- Agents lack mechanisms for verifying their own outputs
- Multi-agent workflows create chains where garbage-in produces garbage-out at each step
- No "circuit breaker" pattern—errors flow through the entire system
- Context drift: each agent slightly misinterprets the goal, and these misinterpretations compound

**How to avoid:**
- Implement verification at wave boundaries, not just at final output
- Use independent judge agents for validation (not integrated into production workflow)
- Design for "fail-fast": stop execution early when quality degrades
- Require each task to produce structured outputs with explicit success/failure signals

**Warning signs:**
- Final outputs dramatically different from initial intent
- Intermediate artifacts diverging from specification
- Later tasks "correcting" earlier work in ways that indicate misunderstanding
- Verification failures only catching problems at the end

**Phase to address:** State & Verification (verifier subagent must be independent, not embedded)

---

### Pitfall 5: Inter-Agent Misalignment (Role Confusion)

**What goes wrong:**
Carefully designed specialist agents start behaving like generalists. Agents drift from responsibilities, duplicate each other's work, or fail to maintain boundaries that make specialization valuable. Research shows inter-agent misalignment is "the single most common failure mode in production systems."

**Why it happens:**
- Agent role definitions are underspecified or contain ambiguous boundaries
- Shared context causes agents to "see" tasks that aren't theirs
- LLMs naturally want to be helpful—they'll attempt tasks outside their scope
- No enforcement mechanism for role boundaries

**How to avoid:**
- Define explicit "constitution" for each agent role (what it MUST do, what it MUST NOT do)
- Limit what context each agent receives—only what's needed for its specific task
- Implement output validation that rejects out-of-scope work
- Use structured handoffs with explicit role transitions

**Warning signs:**
- Multiple agents producing overlapping outputs
- Agents "helping" with tasks assigned to other agents
- Role descriptions being ignored in favor of general helpfulness
- Coordination overhead increasing as agents step on each other

**Phase to address:** Execution Engine (task executor isolation) + Domain Adapters (role templates)

---

### Pitfall 6: Dependency Inference Failures (False Positives/Negatives)

**What goes wrong:**
Dependency analysis produces false positives (tasks marked dependent when they're not, preventing parallelization) or false negatives (tasks marked independent when they're not, causing race conditions or inconsistent state).

**Why it happens:**
- Implicit dependencies hidden in natural language task descriptions
- Artifact-based dependency detection misses semantic dependencies
- Over-eager inference creates spurious dependencies from keyword matching
- Under-inference misses non-obvious data flow dependencies

**How to avoid:**
- Multi-pass dependency inference: artifact-based, type-based, semantic, heuristic
- Explicit dependency declaration in task schemas (don't rely solely on inference)
- Validate dependency graph before execution (check for cycles, verify parallelizability)
- Conservative default: when uncertain, assume dependency exists (serialize)

**Warning signs:**
- Waves containing only 1-2 tasks when parallelization was expected
- Tasks failing due to missing inputs from "parallel" tasks
- Cycle detection triggering unexpectedly
- Execution order varying between runs for supposedly independent tasks

**Phase to address:** Dependency Analysis (multi-pass algorithm) + Wave Computation

---

### Pitfall 7: State Persistence Failures Across Sessions

**What goes wrong:**
Workflows crash or timeout, and upon restart, all progress is lost. Even worse, the restarted agent makes different decisions inconsistent with prior execution, creating orphaned artifacts or contradictory state.

**Why it happens:**
- LLMs are fundamentally stateless—every interaction starts fresh
- Most frameworks treat "memory" as an add-on bandaid rather than core primitive
- Checkpoint granularity is wrong: too frequent = overhead, too sparse = lost work
- Restored state is inconsistent with what the agent "remembers"

**How to avoid:**
- Design for resumption from day one—checkpoints at wave boundaries (natural breakpoints)
- Store both artifacts AND decision context (why this approach was chosen)
- Validate state consistency before resuming (catch corruption early)
- Support rollback to known-good checkpoints when resumption fails

**Warning signs:**
- Users reporting "had to start over" after interruptions
- Inconsistent artifacts from pre/post-crash execution
- Resume operations taking longer than restart
- Decision logs showing contradictory choices across sessions

**Phase to address:** State & Verification (wave boundary checkpointing) + Failure Handling (resume logic)

---

### Pitfall 8: Hallucinated Plans and Phantom Dependencies

**What goes wrong:**
The LLM generates plans with non-existent tools, fabricated capabilities, or dependencies on things that don't exist. The orchestrator dutifully tries to execute these hallucinated plans, wasting resources and failing mysteriously.

**Why it happens:**
- LLMs hallucinate—producing unfaithful content is a known limitation
- Tool documentation deficiencies cause agents to imagine capabilities
- Planning happens without grounding in actual available resources
- Enhanced reasoning capabilities paradoxically increase tool hallucination rates

**How to avoid:**
- Validate plans against explicit capability inventory before execution
- Ground planning in structured tool/resource schemas, not natural language descriptions
- Require plan verification step before execution begins
- Implement "does this tool/capability actually exist?" checks

**Warning signs:**
- Task execution failing with "tool not found" or "capability unavailable" errors
- Plans referencing resources not in project scope
- Unrealistic time/resource estimates in generated plans
- Dependencies on outputs that no task is configured to produce

**Phase to address:** Decomposition Commands (plan validation) + Execution Engine (pre-execution verification)

---

### Pitfall 9: Verification Theater (Looks Done But Isn't)

**What goes wrong:**
The verification system reports success, but the actual output doesn't meet requirements. "Garbage in, garbage out, but with more steps and higher costs." Systems orchestrate elaborate workflows but never verify if work meets requirements.

**Why it happens:**
- Verification checks existence, not quality
- LLM-as-judge reliability isn't guaranteed—adversarial triggers can inflate scores
- Verification is integrated into production workflow (not independent)
- Success criteria are vague ("it works") rather than specific and testable

**How to avoid:**
- Multi-modal verification: exists, contains expected content, runs/compiles, passes tests
- Independent verifier agent that doesn't share context with executor
- Explicit success criteria in task specification (not just "complete the task")
- Custom verification predicates for domain-specific requirements

**Warning signs:**
- High verification pass rate but low downstream satisfaction
- Verification completing much faster than expected
- All tasks passing verification (no failures = verification probably broken)
- Users discovering issues that verification should have caught

**Phase to address:** State & Verification (verifier independence and multi-modal checks)

---

### Pitfall 10: Coordination Overhead Exceeds Value

**What goes wrong:**
The overhead of coordinating multiple agents exceeds the benefit of parallelization. Multi-agent systems fragment per-agent token budgets, leaving insufficient capacity for complex tool orchestration. The coordination itself becomes the bottleneck.

**Why it happens:**
- Network architectures create communication overhead
- Centralized orchestrators become bottlenecks
- More agents = more failure points = more recovery overhead
- "As base LLMs gain extended context windows and improved self-reflection, the unique value proposition of multi-agent collaboration becomes unclear"

**How to avoid:**
- Start simple: single agent → multi-agent only when proven necessary
- Minimize inter-agent communication (wave-based isolation helps)
- Measure coordination overhead explicitly—if >30% of execution time, simplify
- Right-size agent count to task complexity (more isn't always better)

**Warning signs:**
- Orchestration logs longer than task execution logs
- Agents waiting on other agents more than executing
- Total token usage dominated by coordination messages
- Simpler approaches achieving comparable results

**Phase to address:** Execution Engine (parallel dispatch design) + Foundation (architecture decisions)

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Skip wave boundary checkpoints | Faster execution | Full restart on failure | Never in production |
| Inference-only dependencies | No explicit declaration needed | False positive/negative failures | Early prototyping only |
| Single verification mode | Simpler implementation | Missing quality issues | Demo/MVP only |
| Embedded verifier | Fewer components | Biased verification | Never |
| Global context sharing | Easy data passing | Context pollution, role confusion | Never |
| String-based task specs | Quick to write | Parsing errors, ambiguity | Never |
| No loop detection | Faster per-iteration | Runaway execution costs | Never |
| Synchronous-only execution | Simpler control flow | Can't scale parallelization | Prototyping only |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| LLM API calls | No retry logic for rate limits | Exponential backoff with jitter, circuit breaker for sustained failures |
| File system state | Race conditions on concurrent writes | Wave-based isolation (no concurrent writes to same file) or explicit locking |
| Tool execution | Assuming tools succeed | Capture exit codes, stdout, stderr; verify artifacts exist |
| External services | No timeout handling | Configurable timeouts, fallback behavior on timeout |
| Git operations | Conflicts from parallel writes | Serialize git operations or use branch-per-wave pattern |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Sequential wave execution | Total time = sum of all wave times | Maximize within-wave parallelism | >5 waves with single-task waves |
| Context accumulation | Later tasks slower, lower quality | Fresh context per task (PTF core) | >50% context window used |
| Over-decomposition | Many small tasks, high coordination | Adaptive decomposition | >20 tasks for simple goals |
| Verification at end only | Late failure detection, wasted work | Wave boundary verification | Complex multi-wave workflows |
| Single orchestrator | Bottleneck on dispatch | Parallel dispatch within waves | >10 parallel tasks |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Passing secrets in task context | LLM may leak or log secrets | Environment variable references, never literal secrets |
| Unvalidated tool execution | Code injection, arbitrary command execution | Whitelist allowed tools, sandbox execution |
| Trusting LLM-generated file paths | Path traversal attacks | Validate all paths against allowed directories |
| No execution sandboxing | Runaway resource consumption | Resource limits, timeout enforcement |
| Agent impersonation | Agents claiming other agents' capabilities | Cryptographic agent identity or strict role enforcement |

---

## "Looks Done But Isn't" Checklist

- [ ] **Task Completion:** Often missing final verification — verify artifact exists AND meets criteria
- [ ] **Dependency Graph:** Often missing cycle detection — verify graph is a DAG before execution
- [ ] **Wave Computation:** Often missing dependency validation — verify all inputs available before wave starts
- [ ] **State Persistence:** Often missing decision context — verify not just artifacts but reasoning is persisted
- [ ] **Verification:** Often missing multi-modal checks — verify exists AND contains AND runs/compiles
- [ ] **Resume Logic:** Often missing consistency validation — verify restored state matches checkpoint
- [ ] **Loop Detection:** Often missing action signature comparison — verify semantic similarity isn't fooled by rephrasing
- [ ] **Error Handling:** Often missing cascade prevention — verify failure doesn't corrupt downstream tasks
- [ ] **Parallel Execution:** Often missing race condition prevention — verify shared state access is coordinated
- [ ] **Plan Validation:** Often missing capability inventory check — verify all referenced tools/resources exist

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Infinite loop | LOW | Stop execution, analyze loop signature, adjust decomposition, retry |
| Context degradation | MEDIUM | Restart task with fresh context, reduce task scope if needed |
| Cascading failures | HIGH | Identify root cause task, revert to pre-failure checkpoint, re-execute from there |
| State corruption | HIGH | Validate all artifacts, identify corruption point, rollback to last known-good state |
| Role confusion | MEDIUM | Stop execution, re-assert agent boundaries, resume with clarified roles |
| Dependency graph cycles | LOW | Identify cycle, break by reordering or splitting tasks, recompute waves |
| Hallucinated plans | MEDIUM | Validate plan against capabilities, regenerate with grounding, re-execute |
| Verification failures | LOW-MEDIUM | Identify specific failure, retry task with clearer success criteria |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Infinite loop | Core Execution Engine | Loop detection triggers before iteration limit |
| Context degradation | Foundation + Task Schemas | Tasks complete within context budget |
| Task granularity mismatch | Decomposition Commands | Adaptive re-decomposition available |
| Cascading failures | State & Verification | Wave boundary verification catches issues early |
| Inter-agent misalignment | Execution Engine + Domain Adapters | Agents stay within defined roles |
| Dependency inference failures | Dependency Analysis | Multi-pass inference, cycle detection works |
| State persistence failures | State & Verification + Failure Handling | Resume from checkpoint succeeds |
| Hallucinated plans | Decomposition Commands | Plan validation catches non-existent resources |
| Verification theater | State & Verification | Independent verifier with multi-modal checks |
| Coordination overhead | Execution Engine + Foundation | Overhead <30% of total execution time |

---

## Sources

**Academic Research:**
- [Why Do Multi-Agent LLM Systems Fail?](https://arxiv.org/html/2503.13657v1) - arXiv (MAST taxonomy, 41-86.7% failure rates)
- [LLM-based Agents Suffer from Hallucinations: A Survey](https://arxiv.org/html/2509.18970v1) - Agent hallucination taxonomy
- [ADaPT: As-Needed Decomposition and Planning](https://aclanthology.org/2024.findings-naacl.264/) - Adaptive decomposition approach
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://research.trychroma.com/context-rot) - Chroma research on context degradation
- [Where LLM Agents Fail and How They Can Learn From Failures](https://arxiv.org/abs/2509.25370) - AgentErrorTaxonomy

**Framework Documentation & Production Lessons:**
- [Top 5 LangGraph Agents in Production 2024](https://www.blog.langchain.com/top-5-langgraph-agents-in-production-2024/) - Real-world deployment patterns
- [Stateful Agents: The Missing Link in LLM Intelligence](https://www.letta.com/blog/stateful-agents) - State persistence challenges
- [Building Reliable AI Agents with Durable Workflows](https://www.decodingai.com/p/building-reliable-ai-agents-with) - Checkpoint/recovery patterns
- [Multi-Agent AI Failure Recovery That Actually Works](https://galileo.ai/blog/multi-agent-ai-system-failure-recovery) - Recovery strategies

**Post-Mortems & Issue Discussions:**
- [AutoGPT Infinite Loop Issues](https://github.com/Significant-Gravitas/AutoGPT/issues/1994) - Original loop problem documentation
- [Auto-GPT Unmasked: Hype and Hard Truths](https://jina.ai/news/auto-gpt-unmasked-hype-hard-truths-production-pitfalls/) - Production pitfalls analysis
- [LangGraph vs CrewAI Production Comparison](https://xcelore.com/blog/langgraph-vs-crewai/) - Framework evolution lessons

**Industry Analysis:**
- [Why Multi-Agent LLM Systems Fail (and How to Fix Them)](https://www.augmentcode.com/guides/why-multi-agent-llm-systems-fail-and-how-to-fix-them) - Augment Code guide
- [LLM Agents in Production: Architectures, Challenges, and Best Practices](https://www.zenml.io/blog/llm-agents-in-production-architectures-challenges-and-best-practices) - ZenML production guide
- [The AI Agent Framework Landscape in 2025](https://medium.com/@hieutrantrung.it/the-ai-agent-framework-landscape-in-2025-what-changed-and-what-matters-3cd9b07ef2c3) - Framework evolution

---

## PTF-Specific Implications

PTF's core thesis ("fresh context execution for every task") directly addresses the **#1 documented pitfall** (context degradation). This is a strong architectural foundation. However, research reveals additional critical areas:

**Strengths of PTF's approach:**
- Wave-based execution naturally creates checkpoint boundaries
- Task isolation prevents context pollution
- Artifact-centric verification aligns with multi-modal verification patterns

**Areas requiring deliberate design:**
1. **Loop detection** must be built into execution primitives, not added later
2. **Dependency inference** needs multi-pass algorithm with conservative defaults
3. **Verification** must be independent (separate verifier agent, not integrated)
4. **State persistence** needs decision context, not just artifacts
5. **Adaptive decomposition** should be possible at execution time, not just planning

**Key insight from research:** Specification problems (41.77%) and coordination failures (36.94%) cause ~79% of breakdowns. This means:
- Task specification quality is paramount (schemas, validation, clarity)
- Minimizing coordination is a feature, not a limitation
- "More agents" is not better—bounded agency within clear roles is better

---
*Pitfalls research for: LLM Agent Orchestration & Task Decomposition*
*Researched: 2026-01-18*
