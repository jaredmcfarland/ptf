# Feature Research: LLM Agent Orchestration Frameworks

**Domain:** LLM Agent Task Decomposition and Parallel Orchestration
**Researched:** 2025-01-18
**Confidence:** HIGH (verified across multiple current sources)

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = framework feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **Task Definition Schema** | Every orchestration framework has a way to define tasks with inputs, outputs, and instructions | LOW | YAML/JSON schema, straightforward |
| **Sequential Execution** | Basic chained task execution is the minimum viable capability | LOW | Simplest orchestration pattern |
| **State Persistence** | Users expect to resume after interruption; LangGraph, CrewAI, Microsoft Agent Framework all provide this | MEDIUM | File-based for v1 is fine; checkpointing at wave boundaries |
| **Basic Error Handling** | Retry with backoff is standard; users expect failures not to crash everything | MEDIUM | Exponential backoff, max retries, circuit breakers |
| **Task Status Tracking** | Users need to see what's running, completed, failed | LOW | State file with task statuses |
| **Logging/Observability** | Every production framework provides execution traces | MEDIUM | JSONL event log, structured for debugging |
| **Tool/Function Integration** | Agents need to call external tools (file system, APIs, etc.) | MEDIUM | Claude Code provides this via bash/tools |
| **Goal-to-Task Decomposition** | The core value proposition; all frameworks do some form of this | HIGH | PTF's 5-step process is differentiated in depth |
| **Dependency Declaration** | Tasks must declare what they depend on | LOW | Explicit in task schema |
| **Verification/Completion Criteria** | Users need to know when tasks are "done" | MEDIUM | exists/contains/runs/custom verification types |

### Differentiators (Competitive Advantage)

Features that set PTF apart. Not required by all frameworks, but valuable.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Wave-Based Parallel Execution** | True parallel independence within waves; most frameworks do sequential or ad-hoc parallel | MEDIUM | PTF's core thesis: tasks in a wave have zero interdependence |
| **Fresh Context Per Task** | Every task gets a fresh agent context; addresses context degradation problem directly | MEDIUM | Sub-agent dispatch with minimal context loading |
| **Automatic Dependency Inference** | Multi-pass inference (artifact, type, semantic, heuristic, resource) vs. manual declaration | HIGH | Most frameworks require manual dependency specification |
| **Ralph-Style Execution (Repeat Until Verified)** | Temporal iteration within tasks; most frameworks are single-shot | MEDIUM | Combines spatial decomposition with temporal iteration |
| **Domain Adapters** | Pluggable domain knowledge (software, research) shapes decomposition | MEDIUM | Most frameworks are domain-agnostic but don't adapt decomposition |
| **Context Budget Awareness** | Explicit tracking of context consumption; tasks sized to fit fresh context | MEDIUM | Unique to PTF; treats context as scarce resource |
| **Constitution/Principles Generation** | Domain-shaped immutable principles (from SpecKit concept) | MEDIUM | Constrains downstream decisions; rare in other frameworks |
| **Cycle Detection with Resolution Guidance** | Not just detecting cycles, but suggesting how to break them | LOW | Most frameworks detect but don't guide resolution |
| **Hierarchical Recursive Decomposition** | 5-step process with validation at each level; deeper than most | HIGH | ROMA, AgentOrchestra have similar depth; most are shallow |
| **Artifact-Centric Dependency Model** | Dependencies are data flow (what B needs that A produces), not just ordering | MEDIUM | Enables smarter parallelization than "do A before B" |
| **Failure Cascade Handling** | When task fails, intelligently handle all dependents (skip, replan, escalate) | MEDIUM | Most frameworks retry the failed task only |
| **Decomposition Validation** | 100% rule, no overlap, atomicity checks before execution | MEDIUM | Quality gate that most frameworks lack |

### Anti-Features (Deliberately NOT Building for v1)

Features that seem good but create problems.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| **Real-Time Agent Communication** | "Agents should coordinate dynamically" | Introduces distributed state problems, race conditions, complexity; violates wave independence | Wave boundaries are sync points; within waves, tasks are isolated |
| **Autonomous Replanning During Execution** | "Agent should adapt on the fly" | Unpredictable behavior, hard to debug, context pollution from plan changes | Checkpoint at wave boundaries; human-triggered replan if needed |
| **Shared Memory Between Parallel Tasks** | "Tasks in same wave should share context" | Defeats fresh context thesis; introduces coordination overhead | File-based artifacts are the communication mechanism |
| **Complex Branching/Conditional Flows** | "Support if/else/loops in task graph" | Adds graph complexity; most goals decompose to DAG naturally | Simple DAG with wave structure; complex logic lives in task execution |
| **Multi-LLM Provider Orchestration** | "Use GPT-4 for planning, Claude for coding" | Coordination complexity, API inconsistencies, cost tracking nightmare | Single provider (Claude via Claude Code) for v1 |
| **Visual DAG Editor** | "I want to drag-and-drop tasks" | Development overhead, doesn't match CLI workflow, premature optimization | Human-readable YAML/Markdown; visual output for inspection only |
| **Persistent Conversational Memory** | "Agent should remember across projects" | Scope creep; project-specific state is enough for v1 | Project-scoped .orchestrator/ directory; no cross-project memory |
| **MCP Server Implementation** | "Framework should be a standalone server" | Separate runtime, deployment complexity, not needed when Claude Code is the executor | Claude Code plugin-first; file-based IPC |
| **Learning From Past Executions** | "Framework should get smarter over time" | Requires ML infrastructure, unpredictable changes to behavior | Log everything; humans analyze; explicit updates to adapters |
| **Agent-to-Agent Protocol (A2A)** | "Agents should communicate via standard protocol" | Overkill for single-framework; adds coordination layer | Sub-agent dispatch via Task tool; file-based state |

## Feature Dependencies

```
[Task Definition Schema]
    └──requires──> (nothing - foundational)

[Sequential Execution]
    └──requires──> [Task Definition Schema]

[Dependency Declaration]
    └──requires──> [Task Definition Schema]

[Dependency Inference (automatic)]
    └──requires──> [Dependency Declaration]
    └──requires──> [Task Definition Schema]
    └──enhances──> [Wave Computation]

[Wave Computation]
    └──requires──> [Dependency Declaration]
    └──requires──> [Cycle Detection]

[Parallel Execution]
    └──requires──> [Wave Computation]
    └──requires──> [Fresh Context Dispatch]

[Fresh Context Dispatch]
    └──requires──> [Task Definition Schema]
    └──requires──> [State Persistence]

[State Persistence]
    └──requires──> [Task Definition Schema]
    └──enables──> [Resume Capability]

[Resume Capability]
    └──requires──> [State Persistence]
    └──requires──> [Verification]

[Verification]
    └──requires──> [Task Definition Schema]
    └──enables──> [Ralph-Style Execution]

[Ralph-Style Execution]
    └──requires──> [Verification]
    └──requires──> [Fresh Context Dispatch]

[Goal Analysis]
    └──requires──> (nothing - can run standalone)

[Subgoal Identification]
    └──requires──> [Goal Analysis]
    └──requires──> [Domain Adapter]

[Recursive Decomposition]
    └──requires──> [Subgoal Identification]
    └──produces──> [Task Definition]

[Decomposition Validation]
    └──requires──> [Recursive Decomposition]
    └──gates──> [Dependency Graph Construction]

[Domain Adapter]
    └──enhances──> [Subgoal Identification]
    └──enhances──> [Recursive Decomposition]
    └──enhances──> [Dependency Inference]
    └──enhances──> [Verification]

[Error Handling]
    └──requires──> [State Persistence]
    └──enhances──> [Parallel Execution]

[Failure Cascade Handling]
    └──requires──> [Error Handling]
    └──requires──> [Dependency Declaration]

[Human-in-the-Loop] ──conflicts──> [Fully Autonomous Execution]
    └──enhances──> [Error Handling]
    └──optional for v1
```

### Dependency Notes

- **Wave Computation requires Cycle Detection:** Must detect cycles before computing waves, otherwise topological sort fails
- **Parallel Execution requires Fresh Context Dispatch:** The parallelism only delivers value if each parallel task gets fresh context
- **Ralph-Style Execution requires Verification:** Can't repeat until verified without verification capability
- **Domain Adapter enhances multiple features:** Pluggable adapters shape decomposition, inference, and verification
- **Decomposition Validation gates Dependency Graph:** Quality check before building the execution graph
- **Human-in-the-Loop conflicts with Fully Autonomous:** Design choice; PTF is batch-execution focused, not conversational

## MVP Definition

### Launch With (v1)

Minimum viable product - what's needed to validate the core thesis that fresh context per task produces better outcomes.

- [ ] **Task, Artifact, Dependency, Wave YAML schemas** - Foundation for everything else
- [ ] **Goal Analysis (Step 1)** - Transform raw goal into structured specification
- [ ] **Subgoal Identification (Step 2)** - First-level decomposition with domain adapter
- [ ] **Recursive Decomposition (Step 3)** - Break subgoals into atomic tasks
- [ ] **Decomposition Validation (Step 4)** - Quality gate before execution
- [ ] **Dependency Graph Construction (Step 5)** - Build DAG, detect cycles, compute waves
- [ ] **Wave-based Execution** - Execute one wave at a time, parallel within wave
- [ ] **Fresh Context Dispatch** - Each task runs in fresh sub-agent context
- [ ] **Basic Verification** - exists, contains, runs verification types
- [ ] **State Persistence** - File-based in .orchestrator/, survives interruption
- [ ] **Resume Capability** - Pick up from last checkpoint
- [ ] **Software Development Adapter** - Prove decomposition works for one domain
- [ ] **Basic Error Handling** - Retry with backoff, max attempts, skip policy

### Add After Validation (v1.x)

Features to add once core wave execution is working and validated.

- [ ] **Research Adapter** - Second domain proves generalization
- [ ] **Ralph-Style Execution** - Repeat task until verified; add after basic verification works
- [ ] **Automatic Dependency Inference** - Multi-pass inference (currently just explicit); add after manual dependencies validated
- [ ] **Constitution Generation** - Domain-shaped principles; add after adapters stabilize
- [ ] **Failure Cascade Handling** - Smart handling of dependent failures; add after basic error handling proven
- [ ] **Context Budget Tracking** - Explicit context consumption monitoring; add after fresh context thesis validated
- [ ] **Human Escalation Points** - Optional approval gates; add based on user feedback

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] **Additional Domain Adapters** - Creative writing, music, etc.; defer until adapter API is stable
- [ ] **Visual Execution Dashboard** - Web UI for monitoring; defer until CLI workflow proven
- [ ] **Partial Wave Execution** - Execute subset of wave if some tasks ready; adds complexity
- [ ] **Multi-Project Orchestration** - Coordinate across multiple goal hierarchies; scope creep
- [ ] **Learning/Optimization** - Improve decomposition based on past runs; requires ML infrastructure
- [ ] **MCP Server Mode** - Standalone service for non-Claude Code environments; major scope expansion
- [ ] **A2A Protocol Support** - Cross-framework agent communication; wait for standards to mature

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Task Schema | HIGH | LOW | P1 |
| Goal Analysis | HIGH | MEDIUM | P1 |
| Recursive Decomposition | HIGH | HIGH | P1 |
| Wave Computation | HIGH | MEDIUM | P1 |
| Parallel Execution | HIGH | MEDIUM | P1 |
| Fresh Context Dispatch | HIGH | LOW | P1 |
| State Persistence | HIGH | MEDIUM | P1 |
| Basic Verification | HIGH | MEDIUM | P1 |
| Resume Capability | HIGH | LOW | P1 |
| Software Adapter | HIGH | MEDIUM | P1 |
| Error Handling (basic) | HIGH | MEDIUM | P1 |
| Decomposition Validation | MEDIUM | MEDIUM | P1 |
| Dependency Inference (auto) | MEDIUM | HIGH | P2 |
| Research Adapter | MEDIUM | MEDIUM | P2 |
| Ralph-Style Execution | MEDIUM | MEDIUM | P2 |
| Constitution Generation | LOW | MEDIUM | P2 |
| Failure Cascade | MEDIUM | MEDIUM | P2 |
| Context Budget Tracking | MEDIUM | LOW | P2 |
| Human Escalation | LOW | MEDIUM | P3 |
| Visual Dashboard | LOW | HIGH | P3 |
| Additional Adapters | LOW | MEDIUM | P3 |

**Priority key:**
- P1: Must have for launch - validates core thesis
- P2: Should have, add after core works
- P3: Nice to have, future consideration

## Competitor Feature Analysis

| Feature | LangChain/LangGraph | CrewAI | AutoGen/MS Agent Framework | PTF Approach |
|---------|---------------------|--------|---------------------------|--------------|
| **Task Decomposition** | Manual chain definition; LangGraph adds graph structure | Role-based task assignment; visual task builder | Conversation-based; Microsoft Agent Framework has Magentic orchestration | 5-step recursive decomposition with validation gates |
| **Parallel Execution** | LangGraph supports concurrent nodes | Supports parallel within crews | Concurrent orchestration pattern; async message-based | Wave-based: all tasks in wave execute in parallel, zero interdependence |
| **State Management** | LangGraph has reducer-driven state, checkpointing | Crews vs Flows for different state needs | Event-driven, durable execution, long-term memory preview | File-based .orchestrator/ directory; wave boundaries are checkpoints |
| **Context Handling** | Uses full context window; some compression strategies | Context per agent in crew | Async messages reduce blocking; conversation-based | Fresh context per task; context is scarce resource to budget |
| **Dependency Graph** | LangGraph: DAG with conditional edges | Task delegation based on agent roles | Hand-off patterns, sequential/concurrent orchestration | Auto-inferred DAG with artifact-centric dependencies |
| **Verification** | External validation; LangSmith for observability | No built-in verification | Human-in-the-loop for validation | Built-in verification types (exists, contains, runs, custom) |
| **Error Recovery** | Basic retry; circuit breakers via integrations | Hierarchical delegation handles failures | Durable execution, automatic resume | Retry strategies (retry, skip, escalate, replan) per task |
| **Human-in-the-Loop** | LangGraph interrupt() function | Not emphasized | Strong HITL support, approval workflows | Optional escalation points; batch-execution focus |
| **Domain Adaptation** | Generic; user writes domain logic | Role specialization per agent | Tool/API integration; OpenAPI support | Explicit domain adapters shape decomposition and verification |
| **Observability** | LangSmith tracing, Studio v2 debugging | Real-time tracing, agent training | Built-in observability, Azure integration | JSONL event log, human-readable state files |

### Key Differentiators vs. Competitors

1. **Fresh Context Thesis:** PTF explicitly treats context as scarce resource. Other frameworks accumulate context or rely on compression. PTF ensures each task executes with fresh context.

2. **Wave Independence:** Tasks within a wave have zero interdependence - no shared state, no coordination. Other frameworks allow communication between parallel agents, introducing complexity.

3. **Decomposition Depth:** 5-step process with validation at each stage. Most frameworks do shallow decomposition or leave it to the user.

4. **Artifact-Centric Dependencies:** "B needs what A produces" vs. "do A before B". Enables smarter parallelization.

5. **Domain Adapters:** Pluggable domain knowledge shapes decomposition, not just tool selection.

## Sources

### Framework Comparisons
- [Turing: Top 6 AI Agent Frameworks 2025](https://www.turing.com/resources/ai-agent-frameworks) - Framework overview
- [Langflow: Complete Guide to AI Agent Frameworks 2025](https://www.langflow.org/blog/the-complete-guide-to-choosing-an-ai-agent-framework-in-2025) - Selection guidance
- [Langfuse: Comparing Open-Source AI Agent Frameworks](https://langfuse.com/blog/2025-03-19-ai-agent-comparison) - Technical comparison
- [AIMultiple: Top 5 Open-Source Agentic Frameworks 2026](https://research.aimultiple.com/agentic-frameworks/) - Current landscape

### Task Decomposition
- [MGX.dev: Task Decomposition for Coding Agents](https://mgx.dev/insights/task-decomposition-for-coding-agents-architectures-advancements-and-future-directions/a95f933f2c6541fc9e1fb352b429da15) - Architecture patterns
- [SparkCo: Deep Dive into Agent Task Decomposition](https://sparkco.ai/blog/deep-dive-into-agent-task-decomposition-techniques) - Decomposition techniques
- [ArXiv: AgentOrchestra Hierarchical Framework](https://arxiv.org/html/2506.12508v1) - Academic research
- [MarkTechPost: ROMA Framework](https://www.marktechpost.com/2025/10/11/sentient-ai-releases-roma-an-open-source-and-agi-focused-meta-agent-framework-for-building-ai-agents-with-hierarchical-task-execution/) - Hierarchical task execution

### State & Memory Management
- [SparkCo: Mastering LangGraph State Management 2025](https://sparkco.ai/blog/mastering-langgraph-state-management-in-2025) - State patterns
- [AWS: Amazon Bedrock AgentCore Memory](https://aws.amazon.com/blogs/machine-learning/amazon-bedrock-agentcore-memory-building-context-aware-agents/) - Memory architecture
- [InfoQ: Microsoft Foundry Long-Term Memory](https://www.infoq.com/news/2025/12/foundry-agent-memory-preview/) - Enterprise memory

### Context Engineering
- [Weaviate: Context Engineering for AI Agents](https://weaviate.io/blog/context-engineering) - Context strategies
- [GetMaxim: Context Window Management Strategies](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/) - Window management
- [JetBrains: Efficient Context Management](https://blog.jetbrains.com/research/2025/12/efficient-context-management/) - Research findings
- [ttoss: Mastering Context Window in Agentic Development](https://ttoss.dev/blog/2025/12/06/mastering-the-context-window-in-agentic-development) - Practical guidance

### Error Handling & Recovery
- [Galileo: Multi-Agent AI Failure Recovery](https://galileo.ai/blog/multi-agent-ai-system-failure-recovery) - Recovery patterns
- [SparkCo: Mastering Retry Logic Agents 2025](https://sparkco.ai/blog/mastering-retry-logic-agents-a-deep-dive-into-2025-best-practices) - Retry best practices
- [Portkey: Retries, Fallbacks, Circuit Breakers](https://portkey.ai/blog/retries-fallbacks-and-circuit-breakers-in-llm-apps/) - Production patterns

### Verification & Validation
- [Shakudo: 5 Agentic AI Design Patterns 2025](https://www.shakudo.io/blog/5-agentic-ai-design-patterns-transforming-enterprise-operations-in-2025) - Design patterns
- [PromptEngineering.org: 2026 Playbook for Reliable Agentic Workflows](https://promptengineering.org/agents-at-work-the-2026-playbook-for-building-reliable-agentic-workflows/) - Production patterns
- [agentic-patterns.com: Awesome Agentic Patterns](https://agentic-patterns.com/) - Pattern catalog

### Human-in-the-Loop
- [Permit.io: Human-in-the-Loop for AI Agents](https://www.permit.io/blog/human-in-the-loop-for-ai-agents-best-practices-frameworks-use-cases-and-demo) - Best practices
- [Microsoft Learn: Human-in-the-Loop with AG-UI](https://learn.microsoft.com/en-us/agent-framework/integrations/ag-ui/human-in-the-loop) - Framework support
- [GitHub: Agentic Patterns HITL Framework](https://github.com/nibzard/awesome-agentic-patterns/blob/main/patterns/human-in-loop-approval-framework.md) - Implementation patterns

### Framework Documentation
- [CrewAI: Tasks Documentation](https://docs.crewai.com/en/concepts/tasks) - Task concepts
- [LangChain: LangGraph](https://www.langchain.com/langgraph) - Graph-based orchestration
- [Microsoft: Agent Framework Overview](https://learn.microsoft.com/en-us/agent-framework/overview/agent-framework-overview) - Enterprise framework
- [GitHub: AutoGen](https://github.com/microsoft/autogen) - Microsoft Research framework

---
*Feature research for: LLM Agent Orchestration Frameworks*
*Researched: 2025-01-18*
