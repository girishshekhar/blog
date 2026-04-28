---
title: "Agent Communication Patterns, Part 1: The Distributed Systems Playbook in a New Medium"
date: 2026-04-26
draft: false
author: "Girish Shekhar"
summary: >
  Multi-agent systems are not a new class of engineering problem. They are distributed
  systems with a different payload. This post maps the synchronous coordination patterns
  emerging in agent ecosystems — delegation, handoff, scatter-gather, chaining — to the
  primitives distributed systems engineers have used for decades, and explains why the
  medium changes the failure modes in ways that matter.
tags: ["agents", "distributed-systems", "llm", "architecture", "multi-agent", "orchestration"]
categories: ["Engineering", "AI Systems"]
series:
  - Agent Communication Patterns
ShowToc: true
TocOpen: true
---

## Why This Post Exists

In fall 2023, I was in a room with a Distinguished Engineer at Amazon, working through what would become Alexa+'s agent architecture. The question on the table was simple: when a generalist agent needs help from a specialist, what does that communication look like? Do you delegate and wait? Hand the customer off entirely? Broadcast and see who responds?

I wrote a one-pager based on this conversation, that gave structure to the problem before it was fully understood. It defined terms: operations, APIs, tools, agents, orchestrators. It sketched two coordination patterns — delegate (the generalist stays in the loop) and handoff (the specialist takes over). It asked three questions it couldn't yet answer: when do you create a new agent versus a tool, how many agents will there be, and how does the ecosystem discover who can do what.

That was late 2023. In the two and a half years since, the industry produced a wave of frameworks, protocols, and papers trying to answer those questions. MCP standardized agent-to-tool communication. A2A standardized agent-to-agent communication. Amazon Bedrock shipped multi-agent collaboration as a managed service. Google and MIT published the first quantitative scaling principles for agent systems. Anthropic's Claude Code became one of the most studied real-world agent implementations. Every hyperscaler converged their agent frameworks toward production.

What emerged is not a new science. It is distributed systems with a different payload.

Every pattern in the agent communication landscape — delegation, handoff, scatter-gather, chaining, fallback, speculative execution — has a direct analog in how distributed systems engineers have coordinated processes for decades. The orchestration problems are the same. The failure modes are recognizable. The tradeoffs are familiar.

What differentiates them is that you are passing natural language and semantic context instead of structured payloads over a wire. And that has cascading effects on how you reason about correctness, cost, and failure.

This is the first of three posts mapping those patterns. This post covers the synchronous coordination patterns — the ones most teams encounter first. Post 2 covers feedback loops, discovery, and fault tolerance. Post 3 lands the thesis: why context is the real engineering problem across all of them, and where the decentralized future creates challenges the current patterns don't fully solve.

---

## The Mental Model

The instinct when approaching multi-agent systems is to treat natural language context like a well-typed API payload (not following the strict semantics, but fundamentally the same kind of thing); a message with a sender, a receiver, and a contract. It is not.

In a classical distributed system, processes communicate by passing messages with schemas — structured contracts that define what the sender asserts and what the receiver does with it. When the contract is violated, something deterministic fails: a type error, a serialization exception, a protocol rejection.

In an agent system, the messages are natural language. The contracts are implicit. When they are violated, you do not get an exception. You get a subtly wrong response, a misunderstood task, a subagent that made a reasonable inference from bad context and propagated the error forward. The failure mode is qualitative degradation, not a stack trace.

Everything else — the coordination patterns, the discovery mechanisms, the context-sharing strategies — is engineering around this one fact.

> **Mental model:** Agent communication is message passing where the message is a lossy, positionally biased, token-expensive representation of shared state. Every design decision in a multi-agent system is, at root, a decision about how much of that shared state to pass, to whom, and in what form.

Distributed systems engineers reason about this constantly under different vocabulary — partitioning, coupling, shared mutable state, message ordering, idempotency. The underlying question is identical: how do you coordinate independent processes without creating the coupling that defeats the purpose of independence?

The patterns that follow are all answers to that question. None of them are new. What is new is that getting the answer wrong is harder to detect, because the failure looks like a slightly worse result rather than a crashed process.

---

## The Protocol Layer

Before the patterns, it helps to understand what infrastructure the industry has converged on. Two protocols define the current stack.

**MCP** (Model Context Protocol, Anthropic, November 2024) standardizes how agents connect to tools, data sources, and external services. Think of it as the plugin system: a single interface through which an agent discovers and invokes capabilities without
bespoke integration code. Every major platform supports it. OpenAI abandoned their proprietary Assistants API in favor of MCP. The 2026 roadmap focuses on two problems: scaling the transport layer for production deployments, and progressive discovery — letting agents learn what a server can do without loading the full tool catalog into
context upfront.

**A2A** (Agent2Agent Protocol, Google, April 2025), albeit very nascent, standardizes how agents communicate with other agents. Where MCP is the plugin system, A2A is the networking layer. Agents advertise capabilities via an Agent Card, accept tasks from client agents, and return results — without exposing their internal state, memory, or tooling to the caller. Microsoft's Agent Framework, Google's ADK, and CrewAI all support it natively.

The framing Google uses is worth keeping: MCP connects agents to tools; A2A connects agents to agents. You use MCP to give an agent access to a database. You use A2A to let that agent delegate to a specialist in another organization's infrastructure.

These two protocols are the substrate. The patterns below operate on top of them.

---

## The Patterns

### 1. Direct Delegation

The orchestrator receives a request, determines it requires a specialist, and delegates a scoped subtask. The specialist completes it and returns a result. The orchestrator synthesizes the final response. Control never transfers — the orchestrator stays in the loop throughout.

This is the pattern we settled on in 2023 and it remains the most common in production today. The reason is observability. When the orchestrator retains control, you always have the trace: what was sent, what came back, where the chain stands. In a world where LLM behavior is probabilistic and debugging is hard, that trace is the difference between diagnosable failures and mysterious ones.

In classical distributed systems, this maps to a synchronous RPC call: service A invokes service B with a well-defined request, blocks until it gets a response, and integrates that response into its own output. The contract is explicit. The caller owns the result.

The implementation detail that matters most in the agent version is what the orchestrator passes to the specialist. Anthropic's Claude Code, whose internals have been studied in unusual detail by the LMCache team, reveals the right default. When the main agent spawns an Explore subagent, it does not pass its full conversation history. The subagent starts fresh, with only the delegated task in its context. When the subagent finishes, only its summary returns to the main context — not the intermediate file reads, search results, or exploratory tool calls. The noise stays inside the subagent's context window and never touches the main conversation.

> Claude Code's Explore→Plan→Implement subagent chain achieves a stunning 92% prompt prefix reuse rate across phases. This is an intentional architecture choice, not an optimization — each phase inherits a lean, scoped prompt rather than the accumulated noise of everything that came before.

The alternative — passing full context to every subagent — is expensive, noisy, and runs directly into the lost-in-the-middle problem (more on this in Post 3). A subagent handed a 180k-token conversation history with the relevant task buried in the middle will not perform as well as one handed a clean, scoped prompt.

```
Naive (full context)                   Direct Delegation (scoped)
────────────────────                   ──────────────────────────

Orchestrator                           Orchestrator
│ [180k tokens: full history]          │ [Scoped task prompt only]
│                                      │
▼                                      ▼
Specialist                             Specialist
│ reasons over bloated context         │ reasons over clean, scoped task
│ relevant task buried in middle       │
▼                                      ▼
Result (degraded)                      Summary only ──► Orchestrator
                                                        synthesizes
                                                        final response
```

The key asymmetry: in the naive version, noise travels in both directions. In direct delegation, the orchestrator sends a scoped task and receives only a summary — the specialist's intermediate work never contaminates the main context.

The gotcha in direct delegation is when the summary that crosses the boundary is lossy in a way that matters. The compressed handoff captures the shape of prior context; it loses the specifics. For tasks where specifics are everything — a particular middleware
behavior, an edge case surfaced three hours ago — the subagent makes a confident call on incomplete information. The failure looks correct until it isn't.

---

### 2. Handoff

The orchestrator transfers control entirely to a specialist. The specialist responds directly to the user or downstream system. The originating agent exits the loop.

It maps to the call center transfer in the real world: the generalist agent determines it cannot or should not handle the request, identifies the right specialist, and passes the conversation along. The specialist now owns the interaction.

In classical distributed systems, handoff appears in load balancer sticky sessions, TCP connection migration, and process handoff in operating system scheduling. The key property is the same: once the transfer completes, the originating party is no longer in the
communication path.

The model routing systems that every major AI product shipped in 2025 are handoff patterns. When a ChatGPT query is classified as requiring image generation, the request is handed to the image generation model, which responds directly. The classification layer
made a routing decision and exited. When a query is classified as requiring deep reasoning, it is handed to the Thinking variant. The router does not synthesize — it transfers.

The critical distinction from direct delegation: in handoff, the originating agent has no visibility into what happens after the transfer. There is no trace that connects the original request to the specialist's handling. In TCP connection migration — the cleanest
distributed systems analog — the originating party truly exits the path. It has no way to observe what the new endpoint does with the connection. The agent handoff is identical: once the transfer completes, the router is out of the conversation.

This matters because when the specialist handles the request poorly, you see the outcome from the user's side but cannot diagnose it from the system's side. The failure is observable; the cause is not. In delegation, the trace connects the symptom to the source. In handoff, you have only the symptom.

```
Direct Delegation                      Handoff
─────────────────                      ───────

User ──► Orchestrator                  User ──► Router
              │                                    │
              │ [scoped task]                      │ [transfers control]
              ▼                                    ▼
         Specialist                           Specialist
              │                                    │
              │ [result]                           │ [responds directly]
              ▼                                    ▼
         Orchestrator ──► User                   User
              │
         [full trace: request,
          result, synthesis]           [no trace after transfer]

Failure: orchestrator sees it,         Failure: user sees it,
         can diagnose it.                       system cannot.
```

This is why most production agent frameworks have converged on delegation over handoff as the default. Handoff is appropriate when the specialist is authoritative and the originating agent genuinely has nothing to add to the synthesis. It is not appropriate when you need the trace.

---

### 3. Cascaded Delegation

Direct delegation applied recursively. Each agent in the chain makes a routing or decomposition decision before passing work along. The result travels back through each layer, with each agent synthesizing before returning upstream.

In classical distributed systems, this is the API gateway pattern: a request arrives at the gateway, which routes to a service, which calls a downstream dependency, which queries a database. Each layer adds context, transforms the response, and returns it to the layer above. The caller at the top owns the final composition.

The production agent example that best illustrates this is Amazon Bedrock's supervisor mode. The supervisor agent receives a complex request, decomposes it, and delegates subtasks to specialist subagents. Each specialist executes and returns its result to the supervisor. The supervisor synthesizes across all results and returns the final response. This is two levels of delegation: user → supervisor → specialist, with results traveling back up the chain.

The depth of the cascade in Bedrock's routing mode is determined dynamically. Simple queries skip the supervisor and route directly to the appropriate specialist — one level of delegation. Ambiguous or complex queries escalate to the supervisor, which coordinates across multiple specialists — two levels. The system adjusts the cascade depth based on query complexity.

```
Simple query (1 level)                 Complex query (2 levels)
──────────────────────                 ────────────────────────

User                                      User
 │                                         │
 ▼                                         ▼
Router ──────────────────►               Router
 │         [direct]                        │
 ▼                                         │ [escalate]
Specialist A                               ▼
 │                                       Supervisor
 │ [result]                           │             │
 ▼                                    ▼             ▼
Router ──────────────────►        Specialist A  Specialist B
 │         [result]                   │             │
 ▼                                    └──── ▼ ──────┘
User                                   Supervisor
                                            │ [synthesized result]
                                            ▼
                                           User

Results travel back up through every layer they descended.
```

The gotcha specific to cascaded delegation is error compounding. A misclassification or a bad decomposition decision at the first level propagates to every downstream agent. The classifier or supervisor, being a chokepoint by design, is also the place where subtle errors have the largest blast radius. A wrong routing decision at level one means every specialist below it is working on the wrong subproblem.

The distributed systems mitigation is validation at each layer boundary — the equivalent of schema validation or a contract test between services. In agent systems, this means the supervisor should verify that the specialist's output is coherent with the original task before synthesizing. This adds latency. The tradeoff is explicit.

---

### 4. Scatter-Gather

The orchestrator fans out a task to multiple specialists simultaneously, waits for all to complete, and synthesizes their results into a single response. In distributed systems this is the fan-out/fan-in pattern — familiar from search engines, distributed query processing, and MapReduce.

Anthropic's Claude Code uses scatter-gather as its default exploration strategy. When a task arrives, the main agent spawns three parallel Explore subagents with different exploration goals: one traces a specific implementation, one investigates the relevant API surface, one checks related utilities. All three run simultaneously, return summaries to the main agent, and the main agent synthesizes before spawning the Plan subagent with only those summaries as input.

The Google and MIT paper "Towards a Science of Scaling Agent Systems" (Kim et al., December 2025) quantifies when scatter-gather earns its cost. Multi-agent coordination improved performance on parallelizable tasks by up to 81% over a single-agent baseline — but degraded it by 39 to 70% on sequential tasks. The key variable is whether subtasks are genuinely independent. If step B requires the output of step A, parallelization does not help. The coordination overhead displaces reasoning budget that would have been better spent on the task itself.

```
Scatter-Gather (Agent World)          Fan-Out / Fan-In (Distributed Systems)
─────────────────────────────         ──────────────────────────────────────
       Orchestrator                            Query Coordinator
      /      |      \                         /        |        \
  Explore  Explore  Explore              Shard 0   Shard 1   Shard 2
  Agent 1  Agent 2  Agent 3             (search)  (search)  (search)
      \      |      /                         \        |        /
        Plan Agent                               Result Merger
           |                                           |
       Final Output                             Ranked Results
```

The paper introduces a "45% rule": if a single agent already solves a task with greater than 45% accuracy, adding more agents yields diminishing or negative returns. Token costs are also non-linear — hybrid scatter-gather architectures consume approximately five to six times more tokens than equivalent single-agent runs.

The distributed systems engineer's instinct here is the right one: partition along independence boundaries, and only parallelize when subtasks share no world state. The agent version adds one wrinkle — the "state" being shared is often context, which does not have a well-defined ownership boundary the way a database row does. Two subagents exploring the same codebase are not mutually exclusive, but they may reach contradictory conclusions that the orchestrator then has to reconcile.

---

### 5. Chaining

Each agent in the chain receives the output of the previous agent as its input, transforms or extends it, and passes the result forward. The chain is a pipeline: input flows through a series of specialized stages, each adding value, each depending on what came before.

In classical distributed systems, chaining is the Unix pipe, the Kafka topic-to-topic processor, the ETL pipeline. Each stage reads from a well-defined input, produces a well-defined output, and has no knowledge of what comes before or after it in the chain.

LangGraph's state machine model is the canonical production implementation for agents. Each node in the graph is an agent or function; edges define the flow; state is a typed schema that nodes read from and write to. This forces precision about what each stage consumes and produces. CrewAI's sequential process is the social-metaphor version of the same thing — agents with roles pass task outputs to the next agent in line — with less upfront schema work and correspondingly less control.

The failure mode of chaining is context pollution, also called information degradation. In a long chain, the original task specification gets pushed progressively further from the active attention window by the accumulated output of intermediate stages. Agent C receives a summary of Agent B's work, which was itself a summary of Agent A's. The original constraints are three compressions deep. No individual agent made an error; the pipeline architecture degraded the signal.

```
Original Task Specification: "Refactor auth module to support OAuth2,
                              preserving existing session token behavior"
        |
        v
   Agent A (Analysis)
   Output: "Auth module uses JWT. Three token types: access, refresh, session."
        |
        v
   Agent B (Design)
   Input: Agent A's output (original constraint about session tokens: attenuated)
   Output: "Proposed OAuth2 flow with access and refresh tokens."
        |
        v
   Agent C (Implementation)
   Input: Agent B's output (session token constraint: absent)
   Output: Code that breaks existing session token behavior.
```

The mitigation in distributed systems is explicit contracts at each stage boundary — schema validation, checksums, assertions. In agent chains, the equivalent is either explicit state schemas (LangGraph's approach) or structured handoff prompts that restate the original task constraints at each stage. The cost of the former is upfront schema work. The cost of the latter is prompt size. Both are preferable to discovering the degradation at the end of the chain.

---

## What the Patterns Have in Common

Each of the five patterns above is a different answer to the same question: who controls the result, and what do they know when they synthesize it?

In direct delegation, the orchestrator controls the result and knows the full task context. In handoff, the specialist controls the result and knows only what the router passed. In cascaded delegation, each layer synthesizes with progressively less context than the layer above it. In scatter-gather, the orchestrator synthesizes across parallel results and must reconcile contradictions without having been in any of the subagent conversations. In chaining, each stage synthesizes with only the output of its immediate predecessor.

The distributed systems lens makes the tradeoffs legible. More control = more observability = more overhead. Less control = faster execution = harder to debug. This tradeoff is not new. What is new is that the payload is natural language, so the "schema violation" that signals something going wrong is not a type error — it is a subtly wrong answer that a user has to notice, in a system where no exception was raised and no trace flagged the failure.

That is the fundamental difference. And it is why the patterns in this post converge on the same default: keep the orchestrator in the loop, minimize what crosses the boundary, and make the trace explicit. The distributed systems instincts are right. The cost of ignoring them is just harder to see.

Post 2 covers what happens when the coordination is right but the execution still goes wrong: the feedback loops, discovery mechanisms, and fault tolerance patterns that govern how agent systems self-correct and recover.

---

## References

1. LMCache, "Context Engineering & Reuse Pattern Under the Hood of Claude Code," December
   2025 — https://blog.lmcache.ai/en/2025/12/23/context-engineering-reuse-pattern-under-the-hood-of-claude-code/
2. Kim, Y. et al., "Towards a Science of Scaling Agent Systems," arXiv December 2025 —
   https://arxiv.org/abs/2512.08296
3. Google Research blog on the scaling paper —
   https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/
4. MCP Official Roadmap (2026) — https://modelcontextprotocol.io/development/roadmap
5. Google A2A launch post, April 2025 —
   https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/
6. Amazon Bedrock multi-agent collaboration GA, March 2025 —
   https://aws.amazon.com/blogs/aws/introducing-multi-agent-collaboration-capability-for-amazon-bedrock/
7. Claude Code subagent documentation — https://code.claude.com/docs/en/sub-agents