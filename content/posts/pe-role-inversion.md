---
title: "The Principal Engineer's Inversion"
date: 2026-06-21
draft: false
author: "Girish Shekhar"
summary: >
  The Principal Engineer is back at the keyboard, but the keyboard now
  operates fleets. Review has become the binding constraint, and the work
  has moved up a layer — from reviewing artifacts to authoring the
  criteria against which artifacts get reviewed. Notes on what that
  looks like in practice, and on the longer discipline that I think
  now defines the seat.
tags: ["principal-engineer", "ai-native-engineering", "code-review", "agent-development", "engineering-leadership", "debottlenecking"]
categories: ["Engineering", "AI Engineering"]
ShowToc: true
TocOpen: true
---

I used to spend most of my time on direction. As a Principal Engineer, the work was talking to other Principals, Sr. Principals and Senior SDEs, shaping how problem got framed, influecing upward on what we'd build next quarter. The keyboard sat further away each year with me only coming back for proof of concepts and prototypes to help unblock ideas. That was the accepted arc - seniority meant moving away from production work.

That arc has inverted. 

I'm back at the keyboard, working on 4-5 streams in parallel with teams of agents, still doing the people work alongside it. Some of this can be attributed to my new job. Most of it isn't. The agent-native way of working has made being in the work cheaper than it used to be, and the leverage of pure direction has dropped. Picking up the keyboard isn't a regression.

But the keyboard isn't what it used to be either. It operates fleets now. And the work it produces - multiplied by every other engineer doing the same - has created a problem none of us has completely figured out.

## They keyboard came back, but it operates fleets

When I last say in the role years ago, deep focus blocks were the unit of work. A few uninterrupted hours to read a system, write a spec, talk to people, leave the office having moved on one thing meaningfully. The new shape is different. My day is 4-5 streams running in parallel - one agent investigating a live customer issue, another iterating on a spec for a new feature or project, next one reviewing a PR for me to look at, another helping me rebase my feature branch against the massive influx of changes from the team - and in between I'm in 1:1s, reading partner team's design docs, preparing for a leadership review. 

The context-switching is constant. The window between switches are short. Productivity suffers when I'm sloppy about which stream I attend to. The tradeoff I'm making is real: they keyboard returned, but the people work didn't leave. Both are mine now, and the day has to absorb both.

I don't think this is just a new-job artifact. The AI-native way of working seems to actively require this shape. When you have a smart team of agents to collaborate with, the friction of building drops far enough that staying out of work means leaving leverage on the table. It also means that old social structures - designated implementers, designated reviewers, designated coordinators - start to look strange. We're moving past the titles whether we acknowledge it or not. The poeple who thrived through the recent rounds of org compression were the ones who already moved across the role boundaries, or who learned to adapt.

## The bottleneck moved

The old story about engineering was that bandwidth was the expensive part. Every process we layered on - waterfall, then agile, then design docs, then RFCs - existed to allocate scarce engineering time. Fiona Fung, who leads engineering for Claude Code, put it crisply in her recent Code w/ Claude talk [^fiona]: writing code, writing tests, and refactoring rarely slow her team down anymore.

What slows them down - and what slows us down on my team - is what came next.

> The bottlenecks didn't go away when agentic coding took away the actual need to type code. Verification, code review, and security took their place.

This mental model reframes most of what's happening in engineering orgs right now. The process we layered were never critical because of typing time. They were critical because the *gap* between intent and output - gap that humans naturally fill with judgement, review cycles, design discussions, second opinions. Closing the typing gap didn't close the judgement gap. It exposed it.

And the closing of one gap creates pressure downstream. Adam Bender's Google I/O 2026 talk [^adam] made this explicit through a systems-thinking lens: take any system, crank the output of one node way up, and the rest struggles to keep pace. The weakest links get exposed. The trade-offs an organization built up over decades get rebalanced under productivity multipliers some teams are now measuring in orders of magnitude. 

His concrete example sticks with me. A 10x increase in code volume doesn't trigger a 10x increase in test runs. Because of how dependency graphs amplify, it can trigger and 1000x increase. Compute budget vanishes before the feature ships. Review pipelines turn into a permanent jam. CI gets overwhelmed. The output-side wins do not stay isolatedl they cascade through veery downstream node that was sized for the old throughput. 

So the bottleneck doesn't move neatly. It creates ripple effect.

## Reviewing one PR at a time is losing

The bottleneck I'm currently working on, in my seat, is review.

Most of my team's PR are agent generated. Some engineers don't review their own agent output before sending it. A team's system has 100+ open PRs on a typical day. I do a lot of review myself. The day I noticed I had become the bottleneck was the day I had to do something different.

There's a temptation to push back on the velocity - slow down, review more carefully. I tried that. It doesn't scale. The throughput multiplier is real, and trying to throttle it back into a shape that a single-PR human review can handle just creates a different bottleneck (engineers waiting on reviewers) without solving the underlying one. The right move isn't to slow production. The right move is to change what review *is*.

## The human is not the reviewer

The naive next step is the one most teams will reach for: delegate the review to agents too. Have one agent produce the PR and another agent review it. That works for tactical correctness but raises a question that took me a while to articulate clearly: if agents produce and agents review, where am I in this?

The honest answer is: not in either of those places. I'm a layer up.

The work I'm doing isn't reviewing PRs anymore. It's specificying what review *means* - what we look for, what we'll never tolerate, which patterns are red flags, which trade-offs we've already settled, which invariants are non-negotiable in this codebase. I started encoding that into a review agent that runs against the PR before I see it. The agent applies my first principles. I look only at what it flags as interesting, novel, or beyond its current understanding.

> Mental model authorship is the differentiated work which we do as humans.

This is different craft from coding. Different from directing. Different even from reviewing. It's authorship - the articulation of judgement with enough precision that the articulation can be applied at scale, by something that isn't me.

The Principal Egnineer's seat hasn't disappeared. It moved up one layer. From person who reviews to person who specifies what review means. From individual judgement exercised on each artifact to judgement encoded once and applied to many.

"Trust but verify" is the concrete phrase which helps me. Trust the agents to handle volumel verify with criteria you've authored. The verification isn't human's job anymore. The criteria are.

## Time spent on intent compounds

There's a second surface, and its the one I spend more time pushing on with the team.

Before the agent runs the implementation, somebody has to say waht the implementation should do. That's the spec. The intent. The goal. Under the old model, specs were often vague because humans naturally filled the gaps with judgement as they implemented. A Sr. SDE reading "build a service that handles retries cleanly" would fill in a dozen reasonable assumptions and produce something workable. Agents don't fill gaps the same way. They either ask, or they invent something that probably isn't what you wanted.

The time engineers used to spend on implementation has to be reallocated. Not to validation downstream, though some of it goes there - but to intent upstream. The dominant industry framing right now is *AI made implementation fast, so make validation fast too*. I'd push against that. The bigger leverage is: AI made implementation fast, so invest the saved time in being precise about what you mean. It intent is captured completely, validation has something concrete to check against. If it isn't, you iterate the implementation, then iterate the validation, and burn the supposed productivity gain on rework. 

Throughput is one metric poeople gravitate towards however the real metreic is whether the thing which you were actually trying to solve got solved. Precisely defining **what does success look like?**. A clean PR that ships the wrong feature is still a failure. Without precise intent, you have no way to tell.

The push toward ideation is the harder cultural move on my team. It's easier to start typing - even when "typing" means prompting an agent - than to stop and articulate what success actually looks like. The discipline I'm trying to build is: do the spec first, treat it as the contract, and resist the pull to start generating code before the contract is clear.

## A validation framework

Once intent is captured, the vlaidation work has a shape. The framework I've been converging on:

1. **Original intent must be captured and verifiable.** - Not a single0line description - the complete goal you settled on: what done looks like, what invariants hold, what edge cases matter. Without completeness here, validation has nothing concrete to check against. Throughput against a vague goal is throughput against the wrong thing.

2. **Validation runs in a different session from implementation.** - The implemeneter agent has all the context that biased the implementation - the path it took, the assumptions it folded in, the dead ends it abandoned. A fresh session reveiwing against intent has none of that bias. This operationalizes "trust but verify" structurally: separation of session is separation of context, which is the precondition for honest verification. 

3. **The knowledge base records what you pay attention to.** - This is where mental model authorship lives as an artifact. Invariants. Things hard to refactor later. Patterns that have bittent he team before. The knowledge base as a skill is versioned, diffable, reviewable. When my judgement changes, the files changes. When the team's collective judgement changes, the shared knowledge base of the team change. The mental model is no longer in my head; its in a file other people and other agents can read.

4. **Validation is a workflow that produces structured findings.** - Each finding has severity, source references, and a proposed mitigation. Free form review output is hard to triage and impossible to track. Structured findings can be sorted, filtered, batched, and learned from. The workflow looks at the intent, inspects the change, and emits a structured result - not a long paragraph.

5. **Each step can escalate to a human when no encoded principle applies.** - This is the rsolution of the recursion problem. The agent doesn't decide everything. It decides what it has been taught to decide. When it hits something its mental model doesn't cover, it surfaces the case. The workflow encodes where **human reviews are essential and matter** rather than asking humans to review everything by default.

6. **The primary steps are review, document findings, run tests, static analysis, and rebase.** - The shape is convering across paractitioners. Kun Chen's `no-mistakes` [^kun] pipeline runs `intent -> rebase -> review -> test -> docs -> lint -> push -> PR -> CI`. The overlap with what I'm stating is not accidental. Engineers building this kind of validation arrive at similar steps because the same pressures shape the design.

7. **Each escalation feeds back into the knowledge base.** - When I weigh in on a flagged case, the resolution updates the knowledge base. The next time something similar appears, the agent already knows. This is where the work compounds. The knowledge base isn't a static prompt - its a living artifact that absorbs every novel case it encounters. Some of the principles I write a personal - taste, idiosyncractic preferences for code I'll maintain. Others are generic to the team - invariants we all share. The classification matters: personal skills stay local; team skills get checked into shared paths so leverage compounds across the org.

The seven points are not a recipe. They're a shape. The substance lives in the knowledge base themselves - the actual principles, in the actual files. The framework is the container; the mental model is the content.

## Deterministic structure, non deterministic judgement

The most useful insight I've had on this so far is about where determinism lives in the loop.

The goal is determinism on outcomes. I want the tests to run. I want the lint to pass. I want certain invariants checked everytime, not when somebody remembers. The intuitive solution si to make the agent more deterministic. That's the wrong move. Agents are useful because their judgement is flexible. Forcing them into a deterministic mode eliminates the very thing that makes them useful.

The right answer is to put determinism in the structure, not the judgement. Git push hooks deterministically run after the push. The orchestration script deterministically invokes each workflow step. The skill.md file deterministically loads context into each agent call. The agent's judgement happens *inside* this container - flexible, situational, capable of nuance - but the container around it is rigid. 

That's how you get determinism on outcomes without sacrificing the value of judgement. The hooks make sure the right tests run. The script makes sure each step happens in the right order. The skills make sure the agent sees what it needs to see. What the agent decides, within those rails, is allowed to be context-dependent.

This is alos the architectural answer to Adam's amplifier warning. AI amplifies whatever your pipeline already does. If your pipelien is held by social convention - engineers will remember to run tests, reviewers will catch the obvious bugs - then a 10x througput increase amplifies the holes in social convention. If the pipeline is held together by deterministic structure - hooks always run, scripts always orchestrate, skills always load - then the 10x amplifies a pipeline that doens't have those holes. 

## Where the human sits

So if the agent produces, the agent reviews, and a script orchestrates the loop, where is the human in this?

At the two ends of the loop, doing the authorship work that brackets everything the agent does in between.

The first end is intent. I articulate the goal, the success criteria, and the invariants the implementation has to respect. Most of the typing time we've saved should go here. What gets defined at the start shapes everything downstream; vagueness at this end becomes iteration further out.

The second end is review. The shape of the work is the framework above. The substance is authorship — writing principles that capture what good looks like across everything we build, not just for the feature in front of me. The invariants. The patterns we never want to see. The trade-offs already settled. These principles live in skills, get applied to every PR, and outlive any single feature. Time spent here compounds harder than time spent on intent: a principle written once runs forever, on every change, without me.

The work at both ends isn't only accumulation. The model underneath this loop keeps getting smarter, and as it does, some of what I encoded becomes redundant. Principles I wrote months ago to correct for behaviors the model used to produce are now noise — the model doesn't make those mistakes anymore. The mental model corpus isn't a one-way ledger. It needs pruning, especially as capability moves underneath it. Part of the seat is going back through the skills and deciding what to delete because the model has caught up. A principle that mattered last quarter may be friction this quarter, and a stale principle adds drag without payoff.

I've also been exploring feeding the review criteria back into the spec stage. If I know what validation will look for at the end, the implementer agent should know that shape going in. The spec stops being an isolated artifact and becomes informed by the principles that will judge the result. What I look for in review is what the spec should make explicit upfront. I'm still working through what this looks like in practice — the direction feels right, the mechanics aren't fully figured out. The more the agent knows about the criteria, the less it produces things that need to be flagged later.

Both bookends, together with the work of curating and connecting them, add up to a different definition of the senior IC seat than the one I trained into. Fiona talks about hiring less for raw throughput and more for creative builders and deep systems experts. The seat she's describing — the one I'm trying to occupy — is the one that prioritizes judgment encoded at scale over judgment applied case-by-case. The keyboard came back, but what the keyboard is *for* changed.

## One bottleneck at a time

I keep coming back to a question Adam Bender posed to his audience: do you know what would break first? Not as a guess. As a map.

The honest answer, for most of us, is no. We have a sense of where the pressure is right now. The next bottleneck — the one that will surface after this one resolves — is a guess, sometimes an informed one, often not.

Fiona named several of the bottlenecks that have already moved on her team: review, security, planning rituals, context-gathering, team composition. Adam mapped the cascade structure that explains why fixing one moves pressure to the next. What I'm doing — what a lot of us are doing right now — is taking the bottleneck in front of us and trying to dissolve it before it breaks something irrecoverable.

I've come to think this *is* what the senior IC seat is in the agent-native era. Not a fixed set of responsibilities. Not "review code" or "design systems" or "mentor engineers." Those are activities. The seat itself is a discipline: paying attention to which constraint is binding now, naming it precisely, and being willing to invent the craft that resolves it. Then doing the same thing for the next one.

The discipline has a name in process engineering: debottlenecking. Continuous removal of binding constraints. It's a mature practice in physical systems — refineries have been thinking this way for decades. We're learning to think this way about software organizations because the productivity multipliers are now large enough that bottlenecks cascade visibly within months, not years.

I don't have the full map. Nobody I've talked to does. What I have is the bottleneck in front of me and a set of moves for working through it: invest in intent, encode judgment into skills, run validation in fresh sessions, structure determinism around the agent's flexibility, sit at the escalation point.

I'm making my way through it one bottleneck at a time, trying to dissolve what I'm noticing. Some of the moves will turn out to be right. Some won't. The map will keep redrawing.

The underlying shape of the seat, though, is consistent across the moves. I'm not the reviewer. I'm not the implementer. I'm the author of the principles that get applied to both, by agents, at scales I couldn't touch one PR at a time. The Principal Engineer's seat moved up a layer. The keyboard came back not because there was more typing to do, but because the keyboard now operates the fleet that operates on my judgment. That's where the leverage lives. That's the work.

---

## References
[^fiona]: Fiona Fung, "Running an AI-native engineering org," Code w/ Claude SF 2026. ([running-an-ai-native-engineering-org](https://claude.com/blog/running-an-ai-native-engineering-org))
[^adam]: Adam Bender, (["Software engineering at the tipping point](https://io.google/2026/explore/workshop-2))," Google I/O 2026.
[^kun]: Kun Chen, `no-mistakes`. ([no-mistakes](https://github.com/kunchenguid/no-mistakes))