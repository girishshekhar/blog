---
title: "The Harness Is the Agent"
date: 2026-05-20
draft: false
author: "Girish Shekhar"
summary: >
  For two years we have talked about building agents as if an agent were a thing
  you construct. It isn't. The harness — the loop, the tools, the context
  discipline — is the artifact you actually build, version, and debug. This post
  reframes the agent as emergent behavior, explains why agent frameworks now feel
  wrong-shaped in production, and offers a way to decide when to own the loop and
  when to rent it.
tags: ["ai-agents", "software-architecture", "llm", "distributed-systems", "agent-harness", "tooling"]
categories: ["engineering"]
ShowToc: true
TocOpen: true
---

## Why this post exists

For about two years, the dominant way to talk about agents has been to talk about *building* them. You build an agent. You give it a role. You compose agents into crews, chains, graphs. The verb is always *build*, and the noun is always *agent* — a thing you instantiate, configure, and assemble like any other object in your system.

That language shaped the tooling, and the tooling shaped how a generation of engineers thinks about the problem. My perspective has evolved over time and I sincerely believe this is the wrong abstraction. Not saying that the earlier tools were useless — they were a reasonable response to the conditions of their moment — but wrong in a way that is now actively costing teams time, reliability, and the ability to reason about their own systems.

The reframe is simple to state and surprisingly hard to fully absorb:

> The agent is not the artifact you build. The harness is. The agent is what the harness *does* when you run it.


## The mental model most of us are still carrying

The naive model is *An agent is a thing. The framework helps you build the thing. The model is the brain of the thing.* Under this model, the agent is the unit of work: you reach for an agent the way you'd reach for a class, you compose agents the way you'd compose objects, and when something goes wrong you ask "what's wrong with my agent?"

This framing came from somewhere reasonable. The word *harness* itself comes from the evaluation world — `lm-evaluation-harness` and the SWE-bench harness used it to mean the runtime scaffolding that makes a model executable against a task. The loop shape came from the ReAct pattern [^react] — reason, act, observe, repeat. It interleaved reasoning traces with tool actions so a model could plan, act, and incorporate observations in a single loop. And the frameworks that followed — the ones that gave us `Agent`, `Task`, and `Crew` as first-class abstractions — were a sensible bet given what models could do at the time. When the model is weak, the orchestration around it is the hard part, and abstracting that orchestration adds real value. A framework that handled the loop, the tool wiring, and the conversation state for you was doing the genuinely difficult work so you didn't have to.

So the agent-as-object model was a correct response to a world where the model was the unreliable component and the scaffolding was where the engineering lived. That world has evolved and the abstraction built for it is the wrong-shape.

## The harness is the artifact

Strip an agent down to what is actually running and you find four things: a model, a loop that calls the model repeatedly, a set of tools the model can invoke, and a context manager that decides what the model sees on each turn. Layer a permission model on top — what the thing is allowed to do in the world — and you have described every production coding agent in existence. Claude Code, Cursor's agent mode, Aider, Cline, Codex - all have the same shape.

Three of those four things — the loop, the tools, the context discipline — are *code you write and own*. The fourth, the model, is a component you call. The "agent" is not any one of these. The agent is the **behavior you observe when you run the loop.** It is emergent, not constructed. You don't build an agent any more than you build a process — you build the thing that produces it when executed.

Anthropic's own guidance points the same direction. In Building Effective Agents[^building-effective-agents], they describe the basic building block of an agentic system as an LLM augmented with tools, retrieval, and memory. Whats interesting is that they draw the line between *workflows* (where the paths are orchestrated through predefined code) — and *agents* (where the model dynamically directs its own process). The advice is to reach for that machinery only when the task demands it, rather than starting from a heavy framework. 

> Mental Model : the augmented loop is the artifact, and the agent is what it does at runtime.

Comparing to traditional system, this is the kernel, the process, and the program. You write a program and hand it to the system; the kernel schedules it, allocates its memory, and mediates its syscalls; the CPU executes the instructions; and the *process* — the running thing, with its live state and behavior — is what emerges when all of that turns over. You do not construct the process as a first-class object and compose it. You write the program and build (or rely on) the kernel; the process is the runtime consequence. Map it across: the task you hand the agent is the program, the harness is the kernel, the model is the CPU, and the agent — the running behavior — is the process. 

| LLM System   | Traditional System |
|--------------|--------------------|
| Task         | Program            |
| Harness      | Kernel             |
| Model        | CPU                |
| Agent        | Process            |

The engineering lives in the kernel: the scheduler, the syscall surface, the memory model, the isolation boundaries — not in some `Process` class you instantiate. A "process framework" that hid the kernel and handed you composable process objects would be abstracting exactly the wrong layer.

```
   Agent-as-object model            Harness-as-artifact model
   (what people say)                (what is actually running)

   ┌──────────────────┐             ┌──────────────────────────┐
   │      Agent       │             │        Harness           │
   │  (the artifact)  │             │     (the artifact)       │
   │                  │             │  ┌────────────────────┐  │
   │   ┌──────────┐   │             │  │  loop (your code)  │  │
   │   │  model   │   │             │  │  tools (your code) │  │
   │   │ (brain)  │   │             │  │  context (yours)   │  │
   │   └──────────┘   │             │  │  permissions       │  │
   │                  │             │  └─────────┬──────────┘  │
   └──────────────────┘             │            │ calls       │
                                    |            ▼             |
                                    │      ┌────────────┐      │
                                    │      │   model    │      │
                                    │      │(component) │      │
                                    │      └────────────┘      │
                                    └──────────────────────────┘
                                                │ when run
                                                ▼
                                        "the agent" = behavior
```

The takeaway here is the relocation of the model. In the left model, the model is the core and the agent is the box around it. In the right model, the model is one swappable component *inside* the artifact, and the artifact is the loop. The agent is not in the picture at all, because the agent is what comes out when you press play.

The clearest recent signal that this is the real artifact: harnesses are starting to factor out of the products that built them. When Cline extracted its core agent harness[^cline-core] into a standalone, open-source runtime — packaging the loop, tool orchestration, and session handling as something other products are built on rather than logic buried inside one app — that is the abstraction stabilizing in public. The loop, not the agent, became the thing worth shipping on its own.

## Conclusion 1: What is means to "own" the loop

The first noticeable change is control. While the abstraction provided an illusion that the frameworks "hid" what the agent was doing. Anyone who has printed a LangChain trace or opened LangSmith (LangChain's tracing and observability tool) knows the calls were right there. The frameworks gave us observability. We could watch every model call, every tool invocation, every context mutation.

What they did not give us was **control over the loop at the point where we need it.** The distinction is between watching and owning.

In a framework, the loop runs inside the framework's control flow. We observe it from the outside through callbacks and traces. When we want to *change* what happens :
a. inject a step between the model call and the tool execution
b. rewrite a tool result before the model sees it
c. alter how context is compacted under pressure
d. change the retry logic for one specific failure mode 

We aren't editing the loop and instead we are configuring someone else's loop through whatever extension points its authors anticipated. The control is inverted: the framework calls our code; we do not call the loop.

The structural difference is worth drawing, because it is not a difference of features (both have the same parts), but a difference of who holds the loop:

```
      AGENT FRAMEWORK                          HARNESS
   (inversion of control)                   (we hold the loop)

   ┌─────────────────────────┐              our code:
   │   Framework runtime     │              ┌─────────────────────────┐
   │   owns the loop         │              │  while not done:        │
   │                         │              │      msg = call_model() │
   │   ┌─────────────────┐   │              │      if needs_tool(msg):│
   │   │  internal loop  │   │              │  ┌──> result = run(tool)│
   │   │   ─ call model  │   │              │  │   ┌─ # edit HERE     │
   │   │   ─ route tool  │   │              │  │   │  ctx.append(...) │
   │   │   ─ update ctx  │   │              │  │   │  done = check()  │
   │   └───────┬─────────┘   │              │  └───┴──────────────────┘
   │           │ fires       │              └─────────────────────────┘
   │           ▼             │                  the loop IS our code;
   │   ┌─────────────────┐   │                  every line is an edit point
   │   │  our callback   │   │
   │   │  (on_tool_end,  │   │              the model, tools, and context
   │   │   on_step, ...) │   │              are functions we call —
   │   └─────────────────┘   │              not events we react to
   │   fires only where the  │
   │   framework decided to  │
   └─────────────────────────┘

   We react to the loop.                    We write the loop.
   Control points: where the                Control points: every line.
   framework exposes a hook.
```

In the framework, control flows *outward* from the runtime to our callbacks, and we can only intervene where the framework chose to fire one. In the harness, there is no outward flow — the loop is our code, so every line is a point we can change. While both let us watch, only one lets us reach in anywhere.

In a harness, the loop is a `while` statement we wrote. Changing what happens between the model call and the tool execution means editing the line between them.

Watch the difference play out in the one scenario every agent developer eventually hits — the agent does something dumb at step four of a seven-step task:

```
In a framework:
  1. Open the trace. Confirm the bad behavior is at step 4. (observability works)
  2. Decide we want to intercept the tool result at step 4 and correct it.
  3. Look for a callback that fires at the right point. on_tool_end? on_agent_action?
  4. Read the framework source to learn what state is available when it fires.
  5. Discover the hook fires after the result is already committed to context.
  6. Subclass the executor. Or monkeypatch. Or fork. Or file an issue and wait.

In a harness:
  1. Open the loop. Find the line where the tool result returns to context.
  2. Add a conditional. Edit it. Run it.
```

The leverage was the larger problem and not the visibility. The framework let us stand outside the glass and watch the machine; the harness handed us the machine. And the moment this bites hardest is precisely the moment that matters most — when something fails in a way the framework's authors did not anticipate, which in a fast-moving field is most of the time.

This is why "own the loop" makes debugging tractable in a way that "configure the loop" never quite does. We are no longer reasoning about the framework's internal state machine as a prerequisite to reasoning about our own bug.

## Conclusion 2: the highest-leverage work moves elsewhere

The second conclusion follows directly highlighting where we should be spending most of our time.

Once we accept that the harness is the artifact and the model is a component inside it, a question quietly dies: *which agent should I use?* It is replaced by a better one: *what should my loop's surface area be?* The work moves from selection of agents to designing the two concrete surfaces the model interacts with.

The first surface is the **tool set**. A model in a loop is only as capable as the tools we expose and how legibly we expose them. A tool that returns three thousand lines of unstructured log output is a tool that poisons the next turn's context. A tool that returns a tight, structured summary with the relevant fields named is a tool that compounds. Most of the difference between an agent that finishes a task and one that thrashes is not the model — it is whether the tools were designed for a model to use or merely exposed because they existed in our codebase. This is design work, and it is now our highest-leverage activity.

The second surface is **context discipline**: what the model sees on each turn (which includes tool discovery and outputs from the first surface), what gets compacted, what gets dropped, what gets pulled in fresh. The loop's quality is largely determined here. A harness that lets context grow unboundedly degrades; a harness with deliberate compaction and retrieval stays sharp across long tasks. None of this is the model's job. It is the harness's job, which means it is ours.

There is a third conclusion which emerges. *When the model is a component, swapping it becomes a bounded experiment rather than a rewrite.* A well-built harness treats the model as a parameter. We can run the same loop, the same tools, the same context policy against a different model and get a clean comparison for our specific workload — because everything except the model is held constant. "Which model is better - the chat one or the code based reasoning one" starts being a measurement we can actually take. The agent stays the same while the substrate does.

## So should we still build agents?

If the harness is the real abstraction, the obvious question is whether frameworks are now obsolete and everyone should be hand-writing loops. The answer as always is it depends. Both abstractions persist. The interesting question is how to decide which one a given application is suitable for, and the deciding axis is **where our differentiation lives.**

The cleanest way to see it is an analogy every engineer already trusts. We use a managed framework for the same reason we use a managed database: until the layer it abstracts becomes the layer where your quality is determined, the abstraction is pure leverage. Most applications should never run their own Postgres. The handful where the storage engine *is* the product — where replication, sharding, and consistency control are the differentiation — eventually outgrow the managed layer and take ownership of it. Frameworks are managed agents. Harnesses are self-hosted ones. The decision is the one engineers have always made: own the layer where your differentiation lives, rent everything else.

This is not a fringe view. OpenAI's guide to building agents [^open-ai-guide] lays out a spectrum from 
1. declarative graph frameworks, where we predefine every node and edge upfront, 
2. through a code-first approach where we express the loop in ordinary code, 
3. to building directly from scratch 

It recommends starting with 
1. a single agent that loops through tool calls until an exit condition is met
2. escalating to heavier machinery only when complexity demands it. 

Since the framework vendors themselves describe a spectrum, and that the choice between declarative and code-first is the evidence that there is no single right abstraction. There is a right one for where our quality is determined.

That gives a usable test:

| Reach for a framework when… | Reach for a harness when… |
|---|---|
| The loop is generic; standard reason-act-observe with no exotic control flow | The loop *is* the product; how we sequence tools, manage context, and recover from failure is where our quality lives |
| We are prototyping and want defaults that move we fast | We are in production and need to debug, version, and tune behavior at specific points in the loop |
| Our differentiation is in the data, domain, or surrounding UI — not how the agent thinks | Reliability requires owning every decision in the loop rather than inheriting the framework's |
| The abstraction's cost (less control) is cheaper than building the loop | The abstraction now costs more than it saves — every feature is a fight with the framework's assumptions |

Note: This is not a migration where harnesses "win" and frameworks die. It is that the layer of differentiation moved. *When models were weak, orchestration was the hard part, so abstracting it added value and the framework was the right call for almost everyone. Now that models are strong, the loop, the tools, and the context discipline are where quality is won or lost — so for applications whose quality depends on those things, owning them matters more than the convenience of renting them. If model capability shifts again, the conclusions shifts again as well*. 

> The key takeway here is :  *own the layer where your quality is determined; abstract the rest.*

If you are using a framework, and notice that every nontrivial thing you want to do is now a workaround — a subclass, a monkeypatch, a careful study of the framework's internals before you can change your own behavior. That friction is the abstraction telling you it no longer fits. The loop has become your product, and you are still renting it.

## What we carry forward

The reframe is the whole point, so I'm restating it again - ***we do not build agents***. We build harnesses, and the agent is what the harness does when we run it. The model is a component we call, not the core we wrap. The loop, the tools, and the context discipline are the artifact — the thing we version, the thing we debug, the thing where our engineering actually lives. Frameworks abstracted that artifact when abstracting it was the right trade. For a growing class of applications, it no longer is, and the tell is the moment we find ourself fighting the framework to change behavior we should simply own.

There is a question this opens : A harness is one kernel running one agent. Real enterprises do not want one agent — they want many, delegating to and falling back on each other, bounded so they cannot run away. That is a network of kernels, and a network of kernels needs something a single harness was never meant to provide: discovery, a protocol between agents, and a policy layer that enforces what each is allowed to do regardless of what it decides. The harness solved the single agent cleanly. The layer above it — how bounded ecosystems of agents interoperate without going off the rails — is where the next hard problem is, and it is not solved by making the harness bigger. That seems like a topic for a different post :) .

---

## References
[^react]: ReAct - Yao et al. in 2022 ([arXiv:2210.03629](https://arxiv.org/abs/2210.03629))
[^building-effective-agents]: [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)
[^cline-core]: [Cline-SDK](https://github.com/cline/cline), [Rebuilt Harness](https://x.com/cline/status/2054580767779700775)
[^open-ai-guide]: [A Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) 