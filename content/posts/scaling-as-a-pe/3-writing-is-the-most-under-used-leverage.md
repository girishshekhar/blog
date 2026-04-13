---
title: "Writing Is the Most Underused Leverage a Principal Engineer Has"
date: 2026-04-12
draft: false
author: "Girish Shekhar"
summary: >
  Most Principal Engineers treat writing as documentation—a secondary activity
  that happens after the real work. But writing is the highest-leverage mechanism
  available at staff-plus scope. It scales mentorship, compounds over time, creates
  artifacts that outlive individual conversations, and builds the connective tissue
  for relationships you can't maintain through meetings alone. This post covers why
  PEs underinvest in writing, what changes when they stop, and how imperfect
  artifacts often create more value than perfect silence.
tags: ["principal-engineering", "staff-plus", "writing", "leverage", "mentorship", "knowledge-sharing", "artifacts"]
categories: ["engineering-leadership"]
series:
  - Scaling as a Principal Engineer
ShowToc: true
TocOpen: true
---


## Why This Post Exists

The [first post in this series](https://blog.girishshekhar.com/posts/scaling-as-a-pe/1-step-function-change/) established that scaling as a Principal Engineer is a step-function change, not a gradient, and that your voice carries weight you need to actively manage. The [second post](https://blog.girishshekhar.com/posts/scaling-as-a-pe/2-multiply-impact-through-mechanisms/) argued that your primary lever at scale isn't technical authority exercised directly—it's mechanisms that carry your judgment into rooms you'll never enter.

This post is about the most accessible, most underused, and most compounding of those mechanisms: writing.

Not writing in the sense of documentation. Not writing in the sense of filling out templates or updating wiki pages. Writing as *infrastructure*—the deliberate practice of encoding your technical judgment, your pattern recognition, and your hard-won lessons into artifacts that operate independently of your presence, accumulate value over time, and reach people you'll never meet.

Most Principal Engineers including myself and the Senior PE I talked to arrived at it through burnout. And the path from burnout to one of the most effective scaling practices I've encountered is worth tracing carefully, because the obstacles we hit are the same ones that keep most PEs from ever starting.

## The Misconception: Writing Is a Secondary Activity

If you ask Principal Engineers how they create impact, you'll hear about design reviews, architecture decisions, mentoring, technical strategy, cross-team coordination. Writing rarely makes the list. And when it does, it's framed as a byproduct—"I also write up my decisions" or "I document things when I have time"—not as a primary lever.

This framing reveals the misconception: most PEs think of writing as *recording*. You do the real work—the thinking, the designing, the deciding—and then, if time permits, you write some of it down. The writing is downstream of the impact. It's a nice-to-have. It's what diligent people do after the important work is finished.

This is backwards. And understanding why it's backwards requires looking at what happens when you don't write.

When your mentorship is purely conversational, it has the same scaling ceiling as every other form of direct leverage we discussed in the previous post. You say something insightful in a one-on-one, and one person benefits. You explain a mental model in a design review, and the five people in that room hear it. You share a hard-won lesson during a team meeting, and maybe a dozen people absorb it. Then the next person asks you the same question, and you explain it again. And again. And again.

The Senior PE hit this wall in a way that's worth describing precisely, because it illustrates the failure mode that writing solves.

Before joining his current organization, he had accumulated over forty active mentoring relationships. Not forty people he'd mentored at some point in the past—forty concurrent 1:1s with people who were actively seeking his guidance. He never said no. He didn't have the skills to turn people away when they asked for help. And the demand kept growing because he was good at it. People benefited, told others, and those others showed up asking for the same thing.

Forty concurrent mentoring relationships is not sustainable for anyone. But here's the part that matters: a huge fraction of those conversations covered the same ground. The same questions about navigating ambiguity at the staff plus level. The same confusion about how to influence without authority. The same patterns of getting stuck on technical depth when the real problem was organizational. He was delivering essentially the same insights, customized slightly for each person's context, forty times over.

That's not mentorship at scale. That's a single-threaded server handling the same request repeatedly, with no caching layer.

**Mental model:** Writing is the caching layer for your judgment. A conversation is a synchronous, single-use transfer of knowledge. An artifact—a blog post, a technical guide, an architecture document, a lessons-learned writeup—is an asynchronous, multi-use transfer that compounds over time. Every person who reads it gets the benefit without consuming your time. Every new reader adds to the impact without adding to your load. And unlike a conversation, the artifact doesn't degrade as it passes through intermediaries. The tenth person to read your post gets the same fidelity as the first.

## The Pivot: When Writing Becomes Infrastructure

When the Senior PE transitioned to his current organization, he recognized the pattern forming again. New colleagues who had never worked with a Senior Principal Engineer started asking for mentoring. "Will you mentor me?" became a regular request. And he thought, "I've seen this before. I'm going to get myself into the same unsustainable situation. How do I avoid it?"

His answer was to start blogging internally. Every time he found himself giving the same advice in multiple 1:1s, he'd write it down as a post. Not polished thought leadership — practical, specific guidance addressing the questions people were actually asking. When the next person came to him with a familiar question, he could point them to the post and use their one-on-one time for the parts that were genuinely unique to their situation.

This is the shift from writing-as-recording to writing-as-infrastructure. He wasn't documenting what he'd already done. He was building a system that would handle a class of requests without his direct involvement, freeing him to focus on the problems that actually required his presence.

The math here is worth making explicit:

```
MENTORSHIP WITHOUT WRITING:
┌──────────────┐
│ Same insight │──▶ Person 1  (1 hour)
│ delivered    │──▶ Person 2  (1 hour)
│ 40 times     │──▶ Person 3  (1 hour)
│              │──▶ ...
│              │──▶ Person 40 (1 hour)
└──────────────┘
Total time: 40 hours for 40 people
Ongoing cost: every new person = another hour
Fidelity: degrades with fatigue and context-switching

MENTORSHIP WITH WRITING:
┌──────────────┐
│ Same insight │──▶ Written once (3-4 hours)
│ written once │      │
│              │      ├──▶ Person 1 reads it (0 hours from you)
│              │      ├──▶ Person 2 reads it (0 hours from you)
│              │      ├──▶ ...
│              │      ├──▶ Person 400 reads it (0 hours from you)
│              │      └──▶ Person 401 reads it next year (still 0 hours)
└──────────────┘
Total time: 3-4 hours for unlimited people
Ongoing cost: zero marginal cost per reader
Fidelity: consistent, reviewable, improvable over time
```

This isn't a subtle difference. It's a structural change in the economics of knowledge transfer. And it's available to every PE who's willing to invest the upfront cost of writing things down.

## The Power of Imperfect Artifacts

If the scaling argument for writing is so clear, why don't more PEs do it? The Senior PE's experience points to two obstacles, and the first one is the more insidious: the belief that an artifact needs to be comprehensive and correct before it's worth sharing.

He told a story that illustrates why this belief is exactly wrong.

When a new senior leader joined the organization and asked to be walked through the architecture of their core platform, the Senior PE realized there was no current, accurate architecture document. The only thing he could find was a 2 year old diagram created by a Distinguished Engineer — a good starting point, but so much had changed in the actual implementation that it no longer reflected reality.

Then he found something unexpected. A senior engineering manager in a different part of the organization had created a simplified version of the architecture on an internal website, intended to help application developers understand the builder experience platform they were building. It wasn't fully accurate—there were gaps and some outright errors. But it existed. It was tangible. It was something you could point at and react to.

The Senior PE made a deliberate decision: instead of starting from scratch and trying to create the definitive architecture document (which would have taken weeks and might never have been finished), he took the imperfect artifact, improved what he could, and shared it broadly. His reasoning drew on Cunningham's Law — the observation that the best way to get the right answer on the internet is not to ask a question but to post a wrong answer. People are far more motivated to correct errors than to respond to open-ended requests for input.

And that's exactly what happened. Engineers responsible for different parts of the system came out of the woodwork to correct inaccuracies, fill in gaps, and add context. The artifact became a living document—not because anyone was assigned to maintain it, but because the imperfect first version created a gravitational pull that drew in contributions from people who cared about accuracy.

The before-and-after here is stark:

```
BEFORE:
  - No current architecture artifact exists
  - Knowledge lives in individuals' heads
  - New leaders/engineers spend weeks piecing together
    the system through conversations
  - Each person reconstructs a partial, possibly
    incorrect mental model independently

AFTER:
  - Imperfect artifact published and shared
  - Domain experts correct and extend it organically
  - New leaders/engineers have a starting point that's
    80% right on day one
  - Corrections improve the artifact for everyone,
    not just the person who asked
```

The Senior PE admitted he didn't have the foresight to see how valuable that artifact would become. He wasn't executing a strategy to position himself as the architecture authority. He was solving an immediate problem—his new boss needed to understand the system—in a way that happened to create lasting value. But the lesson generalizes: an imperfect artifact that exists is worth more than a perfect artifact that doesn't. The gap between "something" and "nothing" is always larger than the gap between "something" and "something better."

This matters because perfectionism is the primary killer of PE writing output. The engineer's instinct—make it correct, make it complete, make it thorough—actively works against the goal of getting artifacts into the world where they can create value. A blog post that's 80% right and published today will help people. A blog post that's 100% right and published never will help no one.

## The Unexpected Consequence: Writing Builds Connective Tissue

Here's where the argument goes beyond the obvious "writing scales your knowledge" point. Writing creates a second-order effect we don't anticipate: For the Senior PE - it fundamentally changed the quality and reach of his professional relationships.

The trajectory started internally. He kept the blog inside the company because he was—his words—"terrified to post on the internet and potentially get negative feedback from strangers." That's worth pausing on, because it's the second obstacle that prevents PEs from writing. The first is perfectionism. The second is exposure. Writing publicly means making your thinking visible to people who might disagree, might criticize, might find errors. For people who built their careers on being technically rigorous and correct, that vulnerability is genuinely uncomfortable.

What changed for him was circumstantial. Someone connected him with a communications coach who pushed him to post externally as part of a structured program. He did—reluctantly—and the response was immediate. Former colleagues who had left the company engaged with his posts. People he'd never met reached out. His first public post got hundreds of thousands of impressions, which was terrifying in its own way. "Oh shoot," he thought, "now I'm committed. Now I better do this regularly."

But the interesting part isn't the audience size. It's what happened to his relationships.

Through writing publicly, he connected with other senior technical people who also write about their work. These are people across different organizations who share the practice of thinking through problems in writing and publishing their conclusions. The Senior PE described this as finding a "tribe"—people who speak the same "mother tongue" of thinking in public.

And that shared practice created a connective tissue that made cold outreach warm. He gave a specific example: "It's easier for me to reach out to someone and say, 'Hey, you're an expert in this area—in the world of AI, how does testing change?' Normally a cold Slack message like that would get ignored. But because of that extra connection, because we speak the same mother tongue, there's an openness to engage."

This is not networking in the traditional, transactional sense. It's the natural consequence of making your thinking visible. When you write about what you're learning, what you're struggling with, and what you've figured out, you become findable by people who care about the same problems. Those people are often exactly the ones you need relationships with to be effective at scale—the senior engineers in adjacent organizations, the technical leaders working on parallel challenges, the people whose expertise complements yours.

The relationship-building isn't the goal of the writing. But it's a compounding side effect that makes every other aspect of the PE role easier. Cross-organizational influence, which the first post identified as essential at Senior PE scope, depends on relationships with people you don't work with daily. Writing creates and maintains those relationships at a scale that no meeting cadence or offsite schedule can match.

## From Artifacts to Authority (Whether You Wanted It or Not)

There's a final dimension to this that the Senior PE was visibly uncomfortable discussing: writing creates visibility, and visibility creates a form of authority that compounds whether you seek it or not.

After publishing the architecture artifact and sharing lessons learned from his organization's work on agentic AI, the Senior PE found himself invited to present at the company's major technical conference. That visibility led to people asking him to represent his organization in cross-company technical discussions. He started noticing that teams in other parts of the company were months behind where his organization was in the agentic AI journey—making the same mistakes his team had already made and learned from.

So he did something that felt natural but turned out to be strategically important: he started writing and presenting specifically about lessons learned. Not "here's how great our system is" but "here's what we tried that didn't work," "here's what we learned the hard way," and "here's what you should avoid." The framing was deliberately generous—not claiming credit for successes but offering the failures and hard-won insights so that others could benefit.

This is a form of leadership that requires no positional authority. You're not telling people what to do. You're not making decisions for them. You're sharing knowledge so openly that people can't help but benefit from it. And when you do this consistently, something happens that you can't manufacture through any other means: your name becomes associated with the domain. Not because you campaigned for it, but because you were the person who made your knowledge available when others kept theirs locked in meetings and Slack threads.

The Senior PE was candid about his discomfort with this: "There are so many people who deserve more credit than me. The people building the next version—they're doing the hard work. But if I can take one for the team, I'll do it. I feel bad running toward the spotlight. I'm the opposite of that. I run away from it."

That discomfort is worth naming because it's common among PEs. Most Principal Engineers built their careers by going deep, not by being visible. The spotlight feels unearned, or self-promotional, or like it takes credit away from the team. These feelings are understandable, and to some degree they reflect good instincts—you should be generous with credit and cautious about self-promotion.

But there's a reframe that the Senior PE arrived at: the visibility isn't *for you*. It's for the knowledge. When you write publicly about lessons learned, the visibility that accrues to you personally is a side effect of making important knowledge accessible. If you hold that knowledge back because the visibility feels uncomfortable, you're prioritizing your own comfort over the benefit that others would get from your experience. The discomfort is real. But the calculus favors sharing.

## What This Looks Like in Practice

If you're convinced that writing is infrastructure rather than documentation, the practical question is: where do you start?

**Start with what you're already repeating.** If you're giving the same advice in multiple 1:1s, that's your first post. If you're explaining the same architectural concept in every design review, that's your second. The signal for what to write is the repetition in your conversations—anywhere you're acting as a single-threaded server handling the same request, there's an artifact waiting to be created.

**Start internal, start imperfect.** The Senior PE began with internal blog posts, not public ones. The audience was smaller, the stakes were lower, and the feedback was from people he knew. More importantly, he didn't wait until his posts were perfect. He wrote them well enough to be useful and published them. Corrections came. Refinements followed. The artifacts improved over time, which is exactly how infrastructure works—you iterate on it, you don't wait until it's perfect to deploy it.

**Let Cunningham's Law work for you.** If you're creating an artifact that maps complex territory—an architecture document, a technical decision history, a guide to how your organization's systems interconnect—don't let the fear of inaccuracy stop you. Publish the 80% version. The people who know the remaining 20% will find you, and they'll be more motivated to contribute corrections than they ever would have been to create the document from scratch.

**Pay attention to what compounds.** Not all writing has the same leverage. A post about a specific production incident is valuable but time-bound—it loses relevance as the system changes. A post about the *pattern* of failures that led to that incident, and the mental model for preventing that class of failure, is durable. It will still be useful a year from now, to engineers working on systems you haven't built yet. Prioritize the writing that encodes transferable judgment over writing that documents specific events.

**Accept that visibility is part of the mechanism.** If you write but nobody reads it, the artifact has no leverage. Distribution matters. Sharing your writing in relevant forums, referencing it in reviews and discussions, and yes, posting it where a broader audience can find it—these aren't self-promotion. They're the equivalent of deploying your code to production. An artifact that sits unread in a wiki is like a service that's never deployed. It technically exists, but it creates no value.

## The Real Shift

Writing is the most accessible mechanism a Principal Engineer has. It requires no organizational permission. It has no marginal cost per reader. It compounds over time. It builds relationships you can't build through meetings. And it creates a form of authority that's grounded in generosity rather than hierarchy.

But calling it "accessible" doesn't mean it's easy. Writing well takes time. Writing publicly takes courage. Writing imperfectly—which you must, or you'll never publish—takes a willingness to be wrong in front of people whose respect you value. Every obstacle the Senior PE described is real: the perfectionism, the fear of exposure, the discomfort with visibility, the feeling that writing is somehow less legitimate than "real" technical work.

The reframe that resolves all of these obstacles is the one this post opened with: writing is not documentation. It is not a secondary activity that happens after the important work. Writing *is* the work, at the scale you're now operating at. When you write down the insight you've been giving in 1:1s, you've just multiplied it from reaching one person to reaching hundreds. When you publish an imperfect architecture artifact, you've just created a nucleus for collaborative knowledge that no meeting could have produced. When you share lessons learned from your team's failures, you've just prevented other teams from spending months rediscovering what you already know.

If there's one takeaway from this series, it's this: the Principal Engineers who scale their impact aren't the ones who become better individual contributors in a larger room. They're the ones who build mechanisms—processes, frameworks, forums, and above all, written artifacts—that carry their judgment into conversations they'll never be part of, for engineers they'll never meet, on timelines that extend far beyond their own tenure. Writing is the simplest, most durable, and most underused of those mechanisms. The only thing standing between you and that leverage is the decision to start.
