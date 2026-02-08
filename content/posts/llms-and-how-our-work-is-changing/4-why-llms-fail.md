---
title: "Why LLMs Fail in Surprising Ways (And Why That’s Expected)"
date: 2023-04-05
draft: false
author: "Girish Shekhar"
summary: >
  Large Language Models often fail in ways that feel inconsistent or irrational.
  This post explains why those failures are not bugs, but natural consequences
  of probabilistic systems trained on language — and what that means for real
  product and system design.
tags: ["engineering", "llm", "gen-ai", "software-architecture", "ai-fundamentals"]
categories: ["gen-ai", "machine-learning"]
series:
  - LLMs and How Our Work Is Changing
ShowToc: true
TocOpen: true
---

## Purpose

By this point in the series, we’ve covered:

- Why LLMs changed the game  
- How they work internally (at a usable level of detail)  
- How different model families trade off capability, cost, and control  

And yet, for many teams, the lived experience still looks like this:

> *“It worked perfectly yesterday.”*  
> *“Why did it answer confidently but completely wrong?”*  
> *“Why does changing one sentence break the whole thing?”*

This post is about that gap.

The goal here is not to catalog edge cases or failure anecdotes.  
It’s to explain **why these failures are expected**, and why treating LLMs like deterministic software almost always leads to frustration.

---

## These Systems Are Not Executing Logic

Traditional software executes instructions.

Even most classical machine learning systems — classifiers, recommenders, regressors — produce outputs that feel mechanical. Given the same input, you reliably get the same output.

LLMs do not work that way.

An LLM is sampling from a probability distribution over tokens. Even when temperature is set low, the system is still choosing *plausible* continuations, not *correct* ones.

> **Mental model:**  
> An LLM is not retrieving facts or running rules.  
> It is producing the most statistically likely continuation of text given the context provided.

Once you internalize that, many “weird” behaviors stop being surprising.

---

## Confident Nonsense Is a Feature, Not a Bug

One of the most jarring behaviors for new users is how confidently wrong LLMs can be.

They:
- Use authoritative tone  
- Cite plausible-sounding details  
- Follow formatting perfectly  
- Remain calm even when fabricating  

This is not deception. It’s alignment between *style* and *training objective*.

| Feature         | The Goal: Plausibility                             | The Reality: Accuracy                              |
|-----------------|----------------------------------------------------|----------------------------------------------------|
| Training Reward | ""Does this look like a smart person wrote it?""   | ""Is this statement true in the real world?""      |
| Mechanism       | Pattern matching and style replication.            | Fact-checking against a live database.             |
| Outcome         | High Fluency: Perfect grammar, authoritative tone. | Variable Truth: Correct only if the pattern holds. |

During training, models are rewarded for producing text that looks like high-quality human writing — not for checking facts against an external source.

```mermaid
graph TD
    subgraph "Training Objective"
    A[Text Quality] --> B[Fluent Syntax]
    A --> C[Authoritative Tone]
    A --> D[Logical Formatting]
    end

    subgraph "The 'Hallucination' Loop"
    E[No Fact Found in Weights] --> F[Predict 'Most Likely' Next Token]
    C -->|Conditioning| F
    F --> G[Confident Nonsense]
    end

    style G fill:#ff4d4d,stroke:#333,color:#fff
    style A fill:#bbf,stroke:#333
```

If the training data contains:
- Confident explanations  
- Structured answers  
- Well-formed arguments  

Then confidence becomes part of the learned pattern.

### Why this matters architecturally

You cannot rely on tone or fluency as a proxy for correctness.  
Systems must verify outputs **outside the model**.

This is why retrieval, validation, and constraints show up again and again in production designs.

---

## Prompt Sensitivity Is Structural

Small prompt changes can cause disproportionately large output changes.

This often gets dismissed as “prompt engineering immaturity,” but the deeper reason is simpler:

**Prompts are not instructions. They are conditioning signals.**

A prompt does not tell the model what to do.  
It shifts the probability distribution over what comes next.

Change the wording, and you change:
- Token boundaries  
- Attention patterns  
- Which prior examples the model implicitly activates  

> **Analogy:**  
> Think of prompts less like function calls and more like nudging a crowd’s attention.

```mermaid
graph TD
    A[Raw Model State] -->|Neutral| B(Equal Probability for All Tokens)
    
    subgraph Conditioning
    C[Prompt: 'Explain like a scientist'] 
    D[Prompt: 'Explain like a pirate']
    end
    
    B --> C
    B --> D
    
    C -->|High Prob| E[Technical Jargon / Passive Voice]
    D -->|High Prob| F[Slang / Nautical Metaphors]
    
    E --> G((Result A))
    F --> H((Result B))

    style C fill:#d1e7ff,stroke:#004a99
    style D fill:#ffe5d1,stroke:#994f00
    style G fill:#00ff00,stroke:#000
    style H fill:#00ff00,stroke:#000
```

This is why:
- Long prompts degrade unpredictably  
- Over-specified instructions sometimes perform worse  
- Examples (“few-shot”) often outperform explicit rules  

---

## Long Context Feels Like Memory — But Isn’t

Large context windows give the *illusion* of memory.

The model can reference things you mentioned earlier, sometimes dozens of pages back. But that doesn’t mean it remembers them in the human sense.

Context is:
- A sliding window  
- Reprocessed on every token  
- Not stored or retrieved later  

As context grows:
- Costs increase  
- Attention becomes diffuse  
- Important details compete with irrelevant ones  

```mermaid
graph TD
    subgraph "The 'Sliding' Window"
    A[Earliest Input] --- B[Middle Context]
    B --- C[Recent Instructions]
    C --- D[Current Query]
    end

    D --> E{Attention Mechanism}
    
    subgraph "Dilution Effect"
    E -.->|Weak Focus| A
    E -.->|Moderate Focus| B
    E ==>|High Focus| C
    E ==>|Priority| D
    end

    style A fill:#f9f,stroke:#333,stroke-dasharray: 5 5
    style D fill:#00ff00,stroke:#333
    style E fill:#fff,stroke:#333,stroke-width:4px
```

> **Key misconception:**  
> More context does not always mean better answers.

This is why summarization, chunking, and retrieval are architectural necessities — not optimizations.

---

## Generality Hides Unspoken Assumptions

LLMs feel flexible because they’ve seen a little bit of everything.

But that breadth also means they carry:
- Implicit cultural assumptions  
- Averaged conventions  
- Conflicting norms across domains  

When you ask a vague question, the model fills in the blanks using statistical priors — not your intent.

This shows up most clearly when:
- Domain-specific language overlaps with general terms  
- Organizational context is missing  
- Edge cases contradict common patterns  

---

## Why These Failures Keep Reappearing

None of the failure modes above are solved by:
- Bigger models alone  
- Better prompts alone  
- More training data alone  

They are **system-level problems**.

LLMs are powerful components, but fragile foundations.

Treating them like:
- APIs  
- Rule engines  
- Deterministic services  

almost always leads to surprises.

---

## The Architectural Takeaway

If there’s one thing to internalize from this post, it’s this:

> **LLMs must be surrounded by structure.**

That structure typically includes:
- External data sources  
- Validation layers  
- Feedback loops  
- Human-in-the-loop checkpoints  

Not because models are bad — but because they are probabilistic by design.

---

## What’s Next

In the next post, we’ll look at **Retrieval-Augmented Generation (RAG)**.

Not as a buzzword, but as an architectural pattern that:
- Compensates for model weaknesses  
- Shifts reliability boundaries  
- Changes where complexity actually lives  

That’s where theory turns into systems engineering.
