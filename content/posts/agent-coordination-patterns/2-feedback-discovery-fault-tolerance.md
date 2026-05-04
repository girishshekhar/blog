---
title: "Feedback Loops, Discovery, and Fault Tolerance"
date: 2026-05-03
draft: false
author: "Girish Shekhar"
summary: >
  The synchronous coordination patterns tell you how to decompose work. This post covers
  what happens when that work goes wrong — or when agents don't yet know who can help.
  Iterative refinement, progressive discovery, fallback, and speculative execution are the
  patterns that govern self-correction, capability negotiation, and graceful degradation
  in production agent systems.
tags: ["agents", "distributed-systems", "llm", "architecture", "fault-tolerance", "reflection"]
categories: ["Engineering", "AI Systems"]
series:
  - Agent Communication Patterns
ShowToc: true
TocOpen: true
---

## Picking Up Where Part 1 Left Off

[Part 1](https://blog.girishshekhar.com/posts/agent-coordination-patterns/1-synchornous-patterns/) covered the synchronous coordination patterns: direct delegation, handoff, cascaded delegation, scatter-gather, and chaining. Those patterns answer one question — how do you decompose work across agents and coordinate the results?

This post covers a different set of concerns. 
- What happens when an agent produces a wrong answer and needs to correct it? 
- What happens when an agent doesn't know which specialist exists for a given task? 
- What happens when the model or service it depends on is unavailable? 
- And how do you reduce the latency cost of the classification and routing decisions that sit in front of every pattern from Part 1?

These are the reliability and discovery problems. In classical distributed systems, they have well-established names: control loops, service discovery, circuit breakers, and speculative execution. In the agent world, each pattern takes a specific form shaped by the medium — natural language context rather than structured messages — but the underlying engineering logic is the same.

---

## 6. Iterative Refinement

An agent produces an output, evaluates it against a success criterion, generates a reflection on what went wrong, and uses that reflection as additional context for the next attempt. The loop continues until the criterion is met or an iteration cap is reached.

This is not multi-agent verification. There is no second agent reviewing the first agent's work. The same agent is acting as its own evaluator and critic — producing, assessing, and correcting in a closed loop.

The naive implementation of this is a retry loop: if the agent fails, run it again with the same prompt. This is not self-correction. It is repetition. The same model, given the same input, will produce the same output distribution. What makes iterative refinement work is that the failure is converted into a specific, informative signal — and that signal becomes part of the next attempt's input. The loop is not "try again." It is "try again with this specific information about why the last attempt was wrong."

In classical distributed systems, this is the control loop: observe current state, compute error against desired state, apply correction, repeat. The PID controller is the canonical example. The agent version replaces the error signal with a verbal reflection and the correction with a new attempt conditioned on that reflection.

The formal version is the Reflexion framework (Shinn et al., NeurIPS 2023). The agent maintains two memory components: short-term memory holding the most recent trajectory, and long-term memory accumulating reflections across trials. On each failure, the agent converts the failure signal into verbal feedback — a specific, actionable description of what went wrong and how to approach it differently. The next attempt is conditioned on the accumulated reflection history. Reflexion showed an 11% improvement on HumanEval coding benchmarks over the GPT-4 baseline, and a 22% improvement on sequential decision-making tasks within 12 trials.

```
Control Loop (Distributed Systems)       Iterative Refinement (Agent World)
──────────────────────────────────       ───────────────────────────────────
  Desired State                            Task + Success Criterion   <─────      
       |                                          |                         |
       v                                          v                         |
  Controller ──► Actuator ──► Plant          Agent (Actor)                  |
       ^                        |                 |                         |
       |                        v                 v                         |
  Error Signal ◄── Sensor ◄── Output       Evaluator checks output          |
                                                  |                         |
                                                  v                         |
                                          Fail: Self-Reflection Model       |
                                          generates verbal feedback         |
                                                  |                         |
                                                  v                         | 
                                          Feedback stored in memory         |
                                                  |                         |
                                                  v                         |
                                          Next attempt conditioned  ────────
                                          on accumulated reflections
```

The production instantiation is the Ralph Wiggum technique — named after the persistently optimistic Simpsons character. In its purest form, it is a bash while loop: run Claude Code on a task, intercept the exit signal, feed the same prompt back, repeat until a completion promise is emitted. The sophistication is entirely in the prompt: the success criterion must be deterministic (all tests green, linter clean), and the failure output must be fed back as context for the next iteration. Failures from tests and linters become the signal. The loop handles retry logic. The operator's job is to make success detectable and failure informative.

The known failure mode of single-agent iterative refinement, documented in the Multi-Agent Reflexion paper (December 2025), is degeneration of thought. When one model acts as its own generator and critic, it reproduces the same flawed reasoning structure across iterations — the same structural mistake reappears with different surface wording. The fix is to introduce a second agent with a deliberately different perspective as the critic. At that point, iterative refinement becomes multi-agent verification, and the pattern changes character: it is no longer a control loop but a debate.

> Tip: use single-agent iterative refinement when the success criterion is deterministic and the failure signal is specific. Use multi-agent verification when the evaluation requires judgment that the actor is unlikely to apply accurately to its own work.

---

## 7. Progressive Discovery

An agent does not know at startup what other agents or capabilities are available to it. It discovers them at runtime, lazily, as it needs them — rather than loading a complete registry into context upfront.

In classical distributed systems, this is service discovery: instead of hardcoding the address of every service you might call, you query a registry (Consul, Eureka, DNS-SD) at runtime for services matching your needs. The registry maintains the catalog; callers query on demand.

Progressive discovery applies this pattern to capabilities rather than endpoints.

The MCP version of this problem starts with tool catalog bloat. When an agent connects to an MCP server, it receives a list of available tools. If the server exposes many tools, all tool descriptions land in the agent's context on every turn. A February 2026 paper studying MCP deployments in production found that tool metadata repeatedly injected into a model's context during a typical interaction is inflating token usage and increasing execution cost. Two mechanisms are emerging to address this. Agent Skills exposes only tool names, descriptions, and paths at startup, loading full instructions only when a skill is selected. Tool Search lets the model query for a tool on demand rather than loading the full catalog upfront.

The 2026 MCP roadmap adds MCP Server Cards: a `.well-known/agent.json` endpoint that any registry or crawler can read without establishing a live connection. This makes capability discovery offline and non-blocking — the agent card is available before a session begins, the same way a DNS record is available before a TCP connection is opened.

A2A takes the same approach at the agent level. Every A2A-compliant agent exposes an Agent Card advertising its capabilities in a standardized format. A client agent can read this card without establishing a live session. Registries can index Agent Cards and present them to orchestrators as a searchable catalog.

```
Eager Loading (simple & naive)         Progressive Discovery (correct)
───────────────────────────────        ───────────────────────────────
Agent starts                           Agent starts
    |                                      |
    v                                      v
Load ALL tool descriptions             Load tool names + short descriptions
into context (200 tools = large        into context (lightweight)
context hit on every turn)                 |
    |                                      v
    v                                  Task arrives
Task arrives                               |
    |                                      v
Agent reasons over bloated context     Agent queries: "which tool handles X?"
    |                                      |
    v                                      v
Relevant tool buried in noise          Retrieve full tool definition on demand
                                           |
                                           v
                                       Context stays clean
```

The gotcha is that the quality of discovery depends entirely on the quality of the metadata. A poorly described Agent Card or tool description produces the same result as a misconfigured service registry: the agent routes to the wrong specialist, or fails to route at all, because the capability description didn't match the query. In distributed systems you validate service registry entries with health checks and schema contracts. In agent systems, the equivalent discipline — writing precise, unambiguous capability descriptions — is mostly informal today.

---

## 8. Fallback and Circuit Breaking

When a primary agent or model fails — due to unavailability, capability mismatch, or quality threshold not met — the system routes to an alternative without surfacing the failure upstream.

In classical distributed systems, this is the circuit breaker pattern (Nygard, "Release It!"): when a downstream service starts failing, the circuit opens and subsequent calls fail fast rather than waiting for timeouts. When the downstream recovers, the circuit closes. The purpose is to prevent one failing service from cascading its failure through the entire dependency graph.

In agent systems, fallback operates at two levels.

1. At the provider level, it is model redundancy: if the primary model provider degrades, route to an alternative. OpenClaw, the open-source autonomous agent runtime, implements this explicitly — routing across Claude, GPT, and Gemini with configurable fallback chains and exponential backoff. The agent continues operating across a provider outage by transparently routing to the next available option.
Lets consider what happens without it. An orchestrator spawns three parallel Explore subagents. Halfway through the task, the primary provider degrades. All three subagent calls begin timing out. The orchestrator waits. The timeouts cascade. The task fails silently — not with an exception, but with an empty or partial result that the orchestrator has no way to distinguish from a completed one. The user sees a wrong answer. The system saw nothing unusual.

2. At the capability level, it is quality-based escalation: if a lightweight model produces output below a quality threshold, escalate to a more capable one. Amazon Bedrock's supervisor with routing mode implements this: when a simple query is routed to a specialist and that specialist cannot handle it — ambiguous intent, out-of-scope request — the system escalates to the full supervisor rather than returning an error.
The gotcha is determining what constitutes a failure worth routing around. A model that produces a low-quality response is not obviously distinguishable from one that produces a high-quality response to a subtly wrong question. Deterministic failure signals — a failed test, an exception, an empty result set — make circuit breaking tractable. Soft failures — "this answer seems off" — require either a quality evaluator in the loop, which adds cost, or a human in the loop, which adds latency. The Reflection loop and the fallback pattern are often combined: the agent tries, the loop detects failure via a deterministic signal, the fallback routes to a more capable model for the next attempt.

---

## 9. Speculative Execution

A system begins processing the most likely version of a request before it has confirmed which version is correct, then commits to that execution or discards it once the actual request is confirmed.

In classical distributed systems, speculative execution appears in CPU branch prediction (execute both paths of a conditional, discard the wrong one when the branch resolves), read-repair in Cassandra (speculatively repair stale replicas in the background while serving the read), and optimistic concurrency control (execute a transaction assuming no conflict, validate on commit, retry on conflict).

At the inference layer, speculative decoding is production-confirmed. A small draft model proposes multiple tokens ahead; a large verify model validates them in a single parallel forward pass. Tokens where the models agree are accepted; tokens where they diverge are rejected and replaced. The output is identical to what the verify model would have produced alone — the speedup comes from exploiting GPU capacity that single-token autoregressive generation leaves idle. Production deployments achieve two to three times throughput improvement with no quality loss.

The architectural extension to the agent coordination layer is the obvious next step: pre-execute the most likely branch of a request while classification is still in progress, then commit if the branch was correct and discard if not. If the classification accuracy is high enough and the pre-execution is cheap enough relative to the cost of waiting, the expected latency savings outweigh the waste on incorrect branches. This is the same tradeoff CPU architects have been making for decades.

---

## The Reliability Stack

The four patterns in this post form a reliability stack. Progressive discovery determines whether the right agent or capability can be found at all. Iterative refinement catches quality failures within a single agent's work. Fallback and circuit breaking handle infrastructure and model failures. Speculative execution reduces the latency cost of the routing and classification decisions that sit in front of everything else.

In a well-designed system, these are not independent choices — they compose. A scatter-gather orchestrator uses progressive discovery to find its specialists, iterative refinement if a specialist's output fails validation, fallback if a specialist is unavailable, and speculative execution to reduce the latency of the routing decision that started the whole thing.

Every pattern in this post works better when it knows precisely what "wrong" looks like. Deterministic signals — failed tests, exceptions, empty responses, quality scores below a threshold — make these patterns tractable. Vague signals — "this seems off" — make them expensive. Designing for failure detection is not an afterthought in agent systems. It is the prerequisite for everything in this post.

There is a deeper assumption embedded in all of this: that the context being passed between agents is well-formed — correctly scoped, correctly positioned, correctly sized for the receiving model. The reliability patterns here assume that assumption holds. Post 3 is about what happens when it doesn't — and why getting the context decision right is harder than any individual coordination or reliability pattern makes it appear.

---

## References

1. Shinn, N. et al., "Reflexion: Language Agents with Verbal Reinforcement Learning," NeurIPS 2023 — https://arxiv.org/abs/2303.11366
2. Multi-Agent Reflexion (MAR), December 2025 — https://arxiv.org/html/2512.20845
3. Ralph Wiggum loop technique — https://awesomeclaude.ai/ralph-wiggum
4. MCP Official Roadmap (2026) — https://modelcontextprotocol.io/development/roadmap
5. MCP tool description quality paper, February 2026 — https://arxiv.org/html/2602.14878v1
6. A2A Protocol documentation — https://a2a-protocol.org/latest/
7. Amazon Bedrock multi-agent collaboration GA, March 2025 — https://aws.amazon.com/blogs/aws/introducing-multi-agent-collaboration-capability-for-amazon-bedrock/
8. Speculative decoding production guide — https://introl.com/blog/speculative-decoding-llm-inference-speedup-guide-2025