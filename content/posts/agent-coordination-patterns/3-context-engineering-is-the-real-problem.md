---
title: "Context Is the Real Engineering Problem"
date: 2026-05-10
draft: false
author: "Girish Shekhar"
summary: >
  The coordination patterns are well understood. The protocol layer is converging. The
  unsolved problem is context: how do you decide what to pass between agents, when to
  summarize versus delegate in full, and how to do this without the routing decision
  itself becoming the expensive part. This post lands the thesis — and examines what
  happens when agents coordinate asynchronously across disconnected environments.
tags: ["agents", "distributed-systems", "llm", "context-engineering", "async", "decentralized"]
categories: ["Engineering", "AI Systems"]
series:
  - Agent Communication Patterns
ShowToc: true
TocOpen: true
---

## The Argument So Far

[Part 1](https://blog.girishshekhar.com/posts/agent-coordination-patterns/1-synchornous-patterns/) 
covered how work gets decomposed and coordinated: direct delegation, handoff,
cascaded delegation, scatter-gather, and chaining. 
[Part 2] 
covered how agents self-correct, find each other, and handle failure: iterative refinement, progressive discovery, fallback,
and speculative execution.

Every pattern in both posts is, at root, an answer to the same question: who receives
what, and what do they know when they act on it?

The answer to that question is context — the shared state that an agent
carries into every decision, every tool call, every handoff. And context in an agent
system has properties that have no clean analog in classical distributed systems: it is
expensive to replicate, lossy when compressed, and actively harmful when it grows too
large.

This post is about context. It is the part of the engineering problem that the
patterns from Parts [1](https://blog.girishshekhar.com/posts/agent-coordination-patterns/1-synchornous-patterns/) and [2](https://blog.girishshekhar.com/posts/agent-coordination-patterns/2-feedback-discovery-fault-tolerance/) do not fully solve — and the part that becomes the dominant
constraint when you try to coordinate agents asynchronously across disconnected
environments.

---

## The Naive Mental Model

The naive mental model of multi-agent context sharing is additive: more context passed
to a subagent means better results, because the subagent has more information to work
with. Larger context windows enable this, and the frontier models have made those windows
very large. The limit appears to be mainly cost.

This is wrong in two ways that matter.

The first is positional. The "lost in the middle" finding (Liu et al., published in
Transactions of the Association for Computational Linguistics, 2024) showed that LLM
performance follows a U-shaped curve over context position. Information at the beginning
and end of a prompt receives disproportionate attention. Information in the middle —
regardless of its relevance or importance — is systematically underweighted. This holds
even for models explicitly designed for long-context operation.

> A model handed a 180k-token context with the critical instruction at position 90k
> is not operating on full information. It is operating on a degraded version of that
> information, biased toward what appeared first and last. Passing more context does not
> always produce better results. Sometimes it produces worse ones.

The second is economic. Multi-agent architectures using frontier models at every layer are
expensive. The practical response the industry has converged on is model heterogeneity:
use a large frontier model for reasoning and synthesis as the orchestrator, and
smaller, faster, cheaper models for scoped subtasks as the specialist. OpenClaw's
routing architecture does this explicitly, routing across Claude, GPT, and Gemini based
on task type. Google's Gemini CLI does it via intelligent routing between Flash-Lite,
Flash, and Pro. OpenAI's GPT-5.1 Auto does it between Instant and Thinking variants.

The consequence of model heterogeneity for context is significant. When you spawn a
specialist subagent on a smaller model, the question of what context to pass is not just
a cost question — it is a capability question. A smaller model handed a 180k-token context
may not be able to reason over it as effectively as the orchestrator that produced it.
The right context for a large model and the right context for a small model may be
different in kind, not just in size.

---

## The Real Problem

The real problem is not how to share context. It is how to decide what to share, when to
summarize versus pass in full, and how to make that decision without the decision itself
becoming the expensive part.

This maps cleanly to a classical distributed systems tradeoff. In synchronous RPC, you
pass the full request payload and block for the response — full fidelity, high latency,
high coupling. In a message queue, you pass a structured payload and move on — lower
fidelity by design, lower coupling, async. In shared state (a database, a distributed
cache), you don't pass anything (beyond referential identifiers) — all parties read from 
the same source of truth, with the coordination costs pushed into the consistency model.

Agents face the same three options:
1. **Pass in full.** The subagent gets the full conversation history or the full document.
High fidelity, but the lost-in-the-middle degradation applies, and the token cost is
linear in context size.
2. **Summarize before passing.** The orchestrator compresses the relevant context into a
scoped prompt before delegation. Lower cost, but the summarization is lossy. The specifics
that the subagent needed most are the ones most likely to be compressed away.
3. **Pull on demand.** The subagent receives a minimal prompt and is given tools to retrieve
additional context when it needs it — reading from a file, querying a database, calling
back to the orchestrator. Higher round-trip cost, but the subagent pulls exactly what it
needs rather than receiving everything the orchestrator thought it might need. e.g. 

The right answer depends on the task. 
1. ***Verification tasks*** — where the
subagent is checking work against a known standard — sometimes need full context,
positioned carefully to avoid the middle-of-context degradation.
2. ***Implementation tasks*** — where the subagent needs precise specifications — benefit from
summarized context that front-loads the constraints. 
3. ***Exploration tasks*** — where the subagent's job is to
survey a space — benefit from minimal initial context and pull-on-demand retrieval.

The wrong strategy applied to the wrong task is not benign. A verification subagent given
a summarized prompt when it needed full context will make confident calls based on the
summary. The edge case that was compressed away — "preserve existing session token
behavior" from the chaining example in [Post 1](https://blog.girishshekhar.com/posts/agent-coordination-patterns/1-synchornous-patterns/)  — 
is exactly the kind of specific constraint
that summarization discards. The subagent finishes, reports success, and the failure
surfaces only when the result is tested or used. No exception was raised. No retry was
triggered. The context strategy was wrong and the system had no way to know it.

Anthropic's Claude Code makes this concrete. The default for Explore subagents is minimal
context — fresh start, scoped task, summarized results returned to the orchestrator. The
exception is fork mode, where the subagent inherits the full parent context because the
task genuinely requires awareness of everything that came before. Fork mode is not the
default because the cost and the positional degradation are both real. It is the explicit
override for tasks where full context is genuinely necessary.

```
Context Decision Framework

Task Type          Context Strategy          Reasoning
──────────────     ──────────────────        ─────────────────────────────────
Exploration        Minimal + pull-on-demand  Subagent should discover, not confirm
Implementation     Summarized + front-loaded Constraints must be precise and early
                   constraints               in context to avoid positional loss
Verification       Full context, carefully   Needs ground truth; position matters —
                   structured                put the standard at the top
Long-horizon       Compress across context   Context windows fill; compression must
coordination       windows                   preserve decisions, not conversation
```

The compounding problem in long-horizon tasks is context window exhaustion. An agent
working on a complex task across many turns will eventually fill its context window. The
options are to compress (losing detail), to truncate (losing history), or to save state
externally and start a new context window with a summary. Claude Code's architecture
handles this with a three-layer compression strategy: recent turns in full, older turns
summarized, key decisions and constraints extracted to a persistent file that anchors each
new context window. The file is the memory. The context window is the working set.

---

## 10. Asynchronous Collaboration

The patterns in Parts [1](https://blog.girishshekhar.com/posts/agent-coordination-patterns/1-synchornous-patterns/) and [2](https://blog.girishshekhar.com/posts/agent-coordination-patterns/2-feedback-discovery-fault-tolerance/) 
are predominantly synchronous: the caller blocks until the
callee responds, and the interaction happens within a single session or context window.
Asynchronous collaboration is the pattern where agents submit tasks and retrieve results
without blocking — and where agents may coordinate across sessions, across time, and
across organizational boundaries.

In classical distributed systems, async collaboration is the message queue: post a task,
receive an acknowledgment and an ID, poll or receive a callback when results are ready.
This decouples producers from consumers, enables work to span time boundaries, and
allows the system to absorb load spikes without cascading them to all participants.

MCP's Agents Working Group is shipping async support as SEP-1391. The current MCP
protocol is synchronous — when a tool is called, the caller blocks until it returns. The
async extension lets a server kick off long-running work, return immediately with a task
handle, and let the client retrieve results when they are ready. The framing is
"call-now, fetch-later" — submit a task, continue doing other work, retrieve when ready.

A2A includes async support at the protocol level by design: task management with defined
lifecycle states (pending, working, completed), real-time status notifications via server-
sent events, and streaming support for long-running tasks. This is designed explicitly
for the cross-vendor, cross-cloud scenario — an orchestrator in one environment delegating
to a specialist in a completely different one, without holding a connection open for
minutes or hours.

The harder version of async collaboration is coordination across fully disconnected
environments. An orchestrator running on Google infrastructure needs to delegate to a
specialist running on AWS, operated by a different organization, possibly on a different
model. A2A provides the protocol for the delegation. The unsolved part is context: how
does the orchestrator pass enough context for the specialist to do useful work, across
an organizational boundary, to a system it has no visibility into?

The naive answer is to pass everything. Consider what that means in practice: the
orchestrator serializes its full context — 80k tokens of conversation history, tool call
results, intermediate reasoning — and posts it to the specialist's A2A endpoint. The
specialist is running on a smaller model with a 32k context window. The payload exceeds
the window. The specialist truncates, or errors, or processes a degraded version of the
context and returns a confident but wrong result. The orchestrator receives it, synthesizes
it into a final response, and the user sees the output. The cross-organizational delegation
"succeeded." The result was wrong from the moment the context was truncated.

The emerging answer is reference-based context: instead of embedding context inline, the
orchestrator passes a structured task description and references to the resources the
specialist can retrieve on demand. The specialist pulls what it needs, when it needs it,
from sources it can access. This is the pull-on-demand model applied to cross-
organizational delegation. It requires that the resources be accessible to the specialist
— which requires trust, access control, and a shared understanding of what the references
point to.

---

## The Decentralized Future

Every pattern in this series assumes some form of coordination authority — an
orchestrator/a supervisor/a root agent that owns the trace and synthesizes the result.
The decentralized version removes that authority: agents discover each other dynamically,
coordinate peer-to-peer, and reach outcomes through emergent collaboration rather than
top-down direction.

This is where the research is, not yet where production is.

The Google and MIT scaling paper (Kim et al., December 2025) is the most relevant
quantitative evidence. In their evaluation of decentralized architectures — peer-to-peer
coordination with no central orchestrator — independent agents amplified errors up to 17
times across a task, because mistakes propagated unchecked from one agent to the next
with no centralized validation step. Centralized orchestration limited error propagation
to approximately four times, because the orchestrator validated outputs before passing
them along.

> More agents, without coordination, does not mean better outcomes. The Google/MIT paper's
> finding is precise: independent agents amplified errors 17x. Centralized orchestration
> held error propagation to 4.4x. The coordination layer is not overhead — it is the
> error-correction mechanism.

This is not an argument against decentralization. It is an argument that decentralization
requires explicit error-checking at every hop, which is expensive, and that the cost of
getting this wrong is not a degraded result — it is a compounding failure that reaches
the user having been validated by no one.

The gossip protocol research (Khan et al., December 2025) frames this as an open
engineering problem: how do you enable emergent coordination across large numbers of
agents without a central planner, while maintaining coherence and resisting error
cascade? Gossip mechanisms — the same epidemic protocols distributed systems use for
state dissemination across large clusters — are proposed as a complement to structured
protocols. Instead of direct agent-to-agent messaging for every coordination need, agents
propagate state updates probabilistically through the network, allowing the system to
converge on shared context without requiring every agent to communicate with every other
agent directly.

The practical state today: the ecosystem is fragmented by vendor. Companies building on
Google ADK, Microsoft's Agent Framework, and OpenAI's Agents SDK are building in
distinct environments with different state models, different context representations, and
different coordination primitives. A2A is the bet that these environments will need to
interoperate. That bet is reasonable. The interoperability problem is real. But A2A
solves the protocol problem, not the context problem. Two agents can now speak the same
language. They still need to agree on what to say.

---

## Design Principles

Three posts of patterns lead to a small number of principles. They are not new. They are
the distributed systems principles applied to a new medium, with the adjustments the
medium requires.

- **Partition along independence boundaries.** Before reaching for a multi-agent
architecture, establish that the subtasks are genuinely independent. If step B requires
the output of step A, parallelization adds overhead without adding throughput. The
Google/MIT paper is precise: multi-agent coordination improves performance on
parallelizable tasks by up to 81% and degrades it by up to 70% on sequential ones. The
question is not "can I use multiple agents?" — it is "are the subtasks actually
independent?"
- **Minimize what crosses the boundary.** Every handoff is a compression opportunity and
a compression risk. Pass what the receiver needs to do its job, not everything you have.
The Claude Code architecture demonstrates this working at scale: Explore subagents start
fresh, return summaries, and the noise from their traversal never contaminates the main
context. The fork is the explicit override, not the default.
- **Position context intentionally.** When you do pass context, where it lands in the
prompt matters. Constraints, task specifications, and ground truth belong at the top —
never in the middle. The lost-in-the-middle degradation is not a bug to be patched; it
is a property of the architecture to be designed around.
- **Make failures explicit and specific.** The iterative refinement patterns work because
the failure signal is deterministic — a failed test, a linter error, an exception. Vague
failure signals produce vague reflections produce vague corrections. If you are building
a loop that catches and corrects errors, the first design decision is the error signal,
not the retry logic.
- **Use centralized coordination until you have a reason to decentralize.** The error
amplification numbers from the scaling paper are not theoretical. Four times error
propagation with a supervisor is a tractable engineering problem. Seventeen times without
one is not. Start with a supervisor. Move toward decentralization only when throughput
requires it and only with explicit error-checking at every hop.

---

## What Remains Unsolved

The coordination patterns are converging. The protocol layer is mostly settled. The
context problem is not.

The specific open question is how to make the context routing decision — full pass,
summary, or pull-on-demand — reliably, cheaply, and at scale. If you need a model to
evaluate what context is relevant before you can pass it, you have added a step. If that
step is cheap and accurate, it pays for itself. If it is expensive or unreliable, you
have made the problem worse.

The MCP roadmap's progression toward reference-based results — where tool calls return
pointers to large payloads rather than embedding them inline — is the right direction.
A2A's design for stateless cross-organizational delegation, where specialists pull context
on demand rather than receiving it upfront, is the right direction. Anthropic's Claude
Code's three-layer compression strategy for long-horizon tasks — full recent context,
summarized older context, key decisions in a persistent file — is the right direction.

None of these is a complete solution. They are engineering toward a cleaner separation
between the reasoning layer (what the agent does) and the memory layer (what the agent
knows and how it retrieves it). When that separation is clean, the patterns in this series
compose well. When it is not, the patterns work against each other — scatter-gather
produces results too large to synthesize, chaining compresses specifics into noise,
delegation hands off context that the receiver cannot use effectively.

The mental model from [Part 1](https://blog.girishshekhar.com/posts/agent-coordination-patterns/1-synchornous-patterns/) holds all the way through: agent communication is message
passing where the message is a lossy, positionally biased, token-expensive representation
of shared state. Every design decision is a decision about how much of that shared state
to pass, to whom, and in what form. Getting that decision right — systematically, at
scale, across heterogeneous models and organizational boundaries — is the engineering
problem the next generation of agent infrastructure is being built to solve.

---

## References

1. Liu, N.F. et al., "Lost in the Middle: How Language Models Use Long Contexts,"
   TACL 2024 — https://arxiv.org/abs/2307.03172
2. Kim, Y. et al., "Towards a Science of Scaling Agent Systems," arXiv December 2025 —
   https://arxiv.org/abs/2512.08296
3. Google Research blog on the scaling paper —
   https://research.google/blog/towards-a-science-of-scaling-agent-systems-when-and-why-agent-systems-work/
4. Khan, N.I. et al., "A Gossip-Enhanced Communication Substrate for Agentic AI,"
   December 2025 — https://arxiv.org/pdf/2512.03285
5. MCP Official Roadmap (2026) — https://modelcontextprotocol.io/development/roadmap
6. MCP async support (SEP-1391) — https://modelcontextprotocol.info/blog/mcp-next-version-update/
7. A2A Protocol documentation — https://a2a-protocol.org/latest/
8. LMCache, "Context Engineering & Reuse Pattern Under the Hood of Claude Code,"
   December 2025 —
   https://blog.lmcache.ai/en/2025/12/23/context-engineering-reuse-pattern-under-the-hood-of-claude-code/
9. MCP tool description quality paper, February 2026 — https://arxiv.org/html/2602.14878v1