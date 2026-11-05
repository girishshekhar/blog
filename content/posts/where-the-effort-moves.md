---
title: "Where the Effort Moves When Agents Write the Code"
date: 2026-11-02
draft: false
author: "Girish Shekhar"
summary: >
  Lessons from delivering AI-native projects with a team learning to build this way.
  Agents made generating code cheap, and the effort moved to steering, ownership,
  delivery, evaluation, and operations. The post covers what changed in each and
  what stayed the same.
tags: ["agentic-ai", "software-engineering", "ai-native-development", "evaluation", "operability", "engineering-practice"]
categories: ["engineering"]
ShowToc: true
TocOpen: true
---

## Context

I've been lucky to deliver a couple of AI-native projects at my past jobs, some on my own and some with a team that was adapting to building this way. When I left my last company, I wrote the team a set of parting notes. This post is the broader version of those notes: what I learned, what I had wrong at the start, and what I would carry forward.

These are lessons, not rules. Several of them began as a belief I held and later revised.

## The mental model most of us carried in

The model most of us carried was that code generation was the expensive part of software, and agents had made it cheap. The conclusion followed naturally: the job gets smaller and faster. Describe what you want, accept what comes back, ship it. I held that model too end of 2025 and beginning 2026.

Coding agents have improved quickly at understanding intent and generating code, and a human still sits in the driver's seat. Earlier this year I started a project in a builder ecosystem I did not fully know. The agent harness worked around my knowledge gap, but what it produced was suboptimal at best and needed continued supervision. Some of that behavior comes from how these agents are tuned and how their harnesses are built. Most of it came down to my steering, the context I was managing, and how clearly I had written down what the agent needed in order to work without me. All of it needed constant rework.

This is not a faster typist. Generation was one stage in a pipeline that also includes plan, design, review, testing, delivery, and operation. Making one stage nearly free does not make the pipeline cheap. It moves the load onto the stages around it.

> **Mental model:** Agents don't remove effort from software delivery. They move it, upstream into steering and design, and downstream into evaluation, delivery, and operations.

Distributed systems engineers know this as a shifting bottleneck. Scale one tier and the queue forms at the next; throughput is set by the slowest stage, which is now a different stage. The effects cascade in both directions: a weak plan gets executed completely and quickly, and a higher volume of changes lands on a delivery system built for a lower rate.

```
Before:   intent ──► generate ──► merge

With agents:

  steering ──► plan ──► generate ──► local eval ──► golden set ──► shared env ──► operate
     ▲                                                                              │
     └───────────────── customer bugs become regression cases ◄─────────────────────┘
```

Generation is the same box in both rows. Everything else in the second row is work that someone has to author, and the feedback loop at the bottom is the part teams skip.

## Steering files are the upstream work

You are no longer chatting with an assistant. You are writing the instructions that carry your team's mental model into the agent: how software should be designed, built, operated, and maintained. I call these steering files, and zero-shotting skips them. The first output often looks plausible and then fails to fit how the team works.

On that project, the gaps showed up once the design was simple enough to trust and I let the agent run overnight for several nights. I had not invested enough in evaluation. The agent was good at checking the correctness of independent pieces, and it often skipped the integration tests, which meant I had to step in and correct. When my cloud credentials expired, it looked for suboptimal ways to test the integration and recorded those workarounds as learnings, which I then had to undo. It sometimes forgot that it had access to MCP servers that would have let it test end to end from the front end. And it sometimes stopped midway, deciding that it had reached a good checkpoint and wanted human input to proceed.

Each of these was something the steering could address. Steering also ages. When a new agent version rolls out, some instructions become obsolete, and keeping the planning and evaluation instructions effective takes ongoing effort.

Refining them took weeks and months, not an afternoon. The closest classical analog is infrastructure as code. The declarative definition is the artifact, and any individual environment is a disposable output. The same holds here: steering files persist across model versions and teammates, while any one generated implementation is output. They are also worth keeping portable, since the same instructions should survive a move between Claude Code, Codex and Gemini, or a model upgrade. Maintaining them is as much pruning as authoring: the core beliefs stay, and the instructions around them can become narrower and less specific as agents improve, a theme from [PE Role Inversion](https://blog.girishshekhar.com/posts/pe-role-inversion/).

Because implementation cycles got short, planning and design deserved more of the time, not less. A flawed design used to surface slowly, over a long build. With agents it gets built in full before lunch.

```
Zero-shot
  prompt:   "Collect the logs from the dev environment and find out why the
             integration test failed."
  result:   The agent works out from scratch which bastion tunnels it needs and
            tries adhoc routes. When the cloud credentials expire mid-run, it
            falls back to a weaker test and records that workaround as a learning.

Steered
  steering: Environment access is declared in terraform state files 
            (infrastructure as code). Read the declared state to find which bastion tunnels are required before collecting logs. If credentials expire, stop and report. Do not substitute a weaker test or record a workaround.
  result:   The agent opens the declared tunnels and collects the logs. On expired
            credentials it stops with a clear message instead of improvising.
```

The prompt is identical in both runs. The difference is that the second starts from the team's declared knowledge of the environment, the way infrastructure as code lets a new engineer skip the folklore, and it fails loudly where the first one improvised quietly.

## Complexity is the first thing to challenge

Agents lean toward complexity at every stage, from design through implementation. Extra layers, extra abstractions, extra dependencies. On that project the agent gravitated toward a complex design even while I used spec-driven development and expressed my intent clearly. I spent a lot of time simplifying the design, and from then on I added a layer only when I saw the need for it. When I let the agent continue without reducing complexity, the software got harder to evolve, and each incremental change cost more than the last. More than once I abandoned an implementation and restarted.

You are in the driver's seat, and the habit that paid off was asking three questions: why is this abstraction needed, what does this dependency buy, and would something simpler work?

The cost of complexity is highest in 0→1 work, because the product will probably change shape. A lean implementation leaves room to pivot and add features. Complexity introduced early becomes something you work around. Perfect software was always a myth, and more so now that some of what we build gets replaced within months or years. Make sound decisions on what you know today and leave room to change them.

> **The ownership test:** If you cannot trace a system end to end before it launches, you cannot operate it.

That inability is also a forecast. Evolving, maintaining, troubleshooting, and paying for a system you don't understand will be hard, and you will be the one on call for it. The pager does not ask who typed the code, so accountability for AI-generated code is the same as accountability for any other code.

I saw this at team scale on a project where I guided the high-level architecture and decisions and let the team build the system. The team was still adopting AI-native development and had split the work into many workstreams, each staffed by people using coding agents. When I asked the team to trace a request end to end to troubleshoot a bug while introducing a new feature, only a small number of engineers could attempt it. Others leaned on AI to do the trace and failed, and some could not go beyond the components they owned.

Failed assumptions and overcomplicated contracts between components made it worse, and we ended up adding a large amount of logging to compensate. Looking at the system as a whole, I saw that the agents had introduced several layers of indirection. Each defensive check made sense in isolation, but together there were so many of them that the system became very hard to change.

This refines an argument from [my last post](https://blog.girishshekhar.com/posts/pe-role-inversion/#a-validation-framework), where review principles let an agent escalate only novel cases to a human. I still believe that for routine volume. The boundary I would add is that principles decide what reaches you, but they don't transfer ownership. For code you own, you need to understand it well enough to explain it and challenge it.

## Delivery becomes the next bottleneck

The rate of change per engineer rose visibly once agents were in the loop, and the bottleneck moved to delivery. On that project it became impossible to test changes in a dev environment on their own. The team had not invested enough in a local harness, and against an aggressive timeline, integration testing only happened once all the systems came together in a shared development stage. That stage stayed broken most of the time. Many people waited for it to stabilize, or for someone else's changes to move forward, before they could test their own. The queueing view applies directly: a shared staging environment is a contended resource, and one broken deploy blocks everyone waiting behind it.

Two investments would have moved that queue. The first is continuous integration and delivery (CI/CD) and environment stability, built ahead of the curve and before the bottleneck arrives. The second is moving as much testing as practical onto your own machine i.e. Shift Left. That protects shared environments from your mistakes, and it gives your agents a faster feedback loop, which matters as much as the first benefit.

The same logic covers being blocked. An agent can build context from unfamiliar code, investigate a dependency, and find a change you can own instead of waiting for someone else's. When the full system has too many moving parts to exercise end to end, test and develop components in isolation, the way contract tests stand in for a neighbor service, then validate the integration last. Isolation uncovers gaps early and keeps you moving.

## Evaluation for customer-facing behavior

Natural-language behavior is hard to pin down, and it shifts when models or instructions change. The answer is to treat evaluation as an ongoing investment instead of a launch task: end-to-end suites, refined plugin instructions, and a pipeline that turns each customer bug into a reproducible regression case. A golden set of critical customer scenarios becomes a release gate. If it fails, the release doesn't go.

The classical analog is the regression suite with contract tests at the boundary, and the golden set plays the role of the contract. When we took one implementation to pilot and let customers use it, two things surfaced. Customers described the job to be done very differently from how we, as engineers, had written it into the golden set. And because the job ran for a long time, they had no way to poll its status, an observability gap we had not accounted for. That second finding became a regression case.

Both findings have a familiar shape. A contract written by the producer shares the producer's blind spots, which is the argument for consumer-driven contract tests. A long-running operation needs a status endpoint, the same reason an asynchronous API returns a handle the caller can poll. This is the downstream half of the argument in [this post](https://blog.girishshekhar.com/posts/pe-role-inversion/#a-validation-framework): the human's value concentrates in authoring the validation principles, and the golden set is where those principles become executable.

## Operations and the customer segment

The next place the effort lands is operating what we built. One generic piece of that was a machine-readable health surface: an endpoint that exposes profiling data and standard metrics such as p90 latency, callable by an agent or a control plane using its own credentials. Interactive single sign-on is built for humans, and a scheduled agent cannot click through it. Health and metrics endpoints already work this way for monitoring systems, and the new part is designing them for an agent that recommends tuning, not only one that alerts.

Who runs that agent depends on who the customer is. The default way to deliver an AI-native experience is as a plugin into an assistant the customer's enterprise already licenses, such as Claude Code or Codex. A small business often has no such subscription. Serving that segment means running the harness in the cloud on their behalf, and that choice should shape the design early.

We learned this late. As we closed in on delivery timelines, sitting in live customer calls, it became clear that a separate cohort of customers needed a product that did not depend on their own licenses. It was not a surprise. We had watched other teams move in that direction on a different product, and we had deferred it to later in the roadmap on the assumption that most customers would hold the right license or subscription. That assumption was wrong, and it was an assumption about who the customer is, not about the technology.

It also points to a design constraint: build the harness so it is agnostic to where it runs. The loop, the tools, and the permission model stay the same, and only the host changes. This follows from the argument in [The Agent Harness](https://blog.girishshekhar.com/posts/harness-is-the-agents/). If the harness is the real engineering artifact, it can move.

## The fundamentals

Most of the fundamentals stayed the same: ownership, clear design, testing, observability, and operational discipline. What changed is how much of each stage agents can automate across the lifecycle, and how much judgment is needed to direct that automation.

## Judgment stays with the engineer

The next concrete problem is agents operating the systems they helped build, which raises the question of who is accountable when the one on call is an agent. I don't have a settled answer yet.

The model I would carry forward is the one at the top. Agents move the effort instead of removing it. It lands in steering files upstream and in evaluation, delivery, and operations downstream, and the engineer's judgment is what connects them. As the tools get more capable, that judgment gets exercised in different places, and it still has to be exercised: stay close to the code, to the customer experience, and to the systems you will have to operate.