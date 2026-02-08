---
title: "How LLMs Work (Only What You Need to Know)"
date: 2023-03-22
draft: false
summary: >
  A deeper look at how Large Language Models actually work — focusing on tokens,
  transformers, and training stages — explained in a way that helps engineers,
  product managers, and technical leaders reason about behavior and tradeoffs.
tags: ["writing", "engineering", "science", "llm", "gen-ai", "machine-learning", "transformers", "ai-fundamentals"]
categories: ["gen-ai", "machine-learning"]
series:
  - LLMs and How Our Work Is Changing
ShowToc: true
TocOpen: true  
---

## Purpose

In Part 1, we focused on *what changed* and why Large Language Models (LLMs) feel fundamentally different from earlier machine learning systems.

This post goes one level deeper.

The goal here is **not** to turn you into an ML researcher. Instead, this post explains *just enough* of how LLMs work so you can reason about:
- Why models behave the way they do
- Why prompts sometimes fail unexpectedly
- Why cost, latency, and accuracy trade off against each other
- Why certain architectural patterns keep showing up

> **Reader note:**  
> This is a depth article. If you’re a product manager or engineering leader, feel free to skim the deeper sections and focus on the summaries and callouts. The structure is intentionally designed to support that.

---

## Tokens, Revisited (With Practical Implications)

In Part 1, we introduced tokens as the basic unit models operate on. Let’s make that more concrete.

### What Is a Tokenizer?

A **tokenizer** converts raw text into tokens — numeric IDs that the model can work with.

```
INPUT TEXT: "The stratosphere"
      |
      V
+-----------+       +-------+-------+----------+
| TOKENIZER | ----> |  The  | stra  | tosphere |  (Tokens)
+-----------+       +-------+-------+----------+
      |                 |       |         |
      V                 V       V         V
 OUTPUT IDs:        [  464,   8134,     15234  ]
```

Unlike humans, models don’t see words. They see:
- Common words as single tokens
- Rare words split into multiple tokens
- Punctuation and symbols as tokens

This process is learned from data and optimized for compression and coverage, not readability.

---

### Why Tokenization Matters in Practice

Tokenization directly affects:
- **Cost**: pricing is usually per token
- **Latency**: more tokens = more computation
- **Context limits**: long inputs can silently overflow
- **Prompt reliability**: small phrasing changes can alter token boundaries

> **Example:**  
> Two prompts that look similar to a human may tokenize very differently — leading to different model behavior.

This is one reason prompt engineering often feels unintuitive at first.

---

### How This Differs from Classic Text Models

Older text-based ML systems typically relied on:
- Bag-of-words
- N-grams
- Manually engineered features

Those approaches required humans to decide what mattered. Tokenization plus large-scale learning shifts that burden to the model — at the cost of opacity.

---

## Transformers: The Architecture That Made LLMs Practical

Most modern LLMs are built on **transformer architectures**. You don’t need to understand the math to understand why they matter.

### The Core Idea: Attention

At a high level, transformers use **attention** to decide:
- Which parts of the input matter most
- How strongly different tokens relate to each other

Instead of processing text one word at a time, transformers look at **entire sequences at once** and assign importance dynamically.

> **Mental model:**  
> Attention lets the model ask, *“Which earlier parts of this text should I focus on right now?”*

---

### Why Transformers Scaled When Others Didn’t

Before transformers, language models often used:
- **Recurrent Neural Networks (RNNs)** — processed text sequentially
```
RNN  
  Step 1: [The] -> (Memory A)
                      |
  Step 2: [cat] +  (Memory A) -> (Memory B)
                                    |
  Step 3: [sat] + (Memory B) -> (Memory C)
```

- **Convolutional Neural Networks (CNNs)** — While they are primarily used for image tasks, they use a "sliding window" (kernel) to look at 2, 3, or 5 words at a time to find local patterns (like phrases).
```
CNN
  Sentence:  [ The ] [ cat ] [ sat ] [ silently ] [ in ]
             \_______|_______/          |           |
                 Filter 1 (sees "The cat sat")      |
                         \___________|____________/
                             Filter 2 (sees "cat sat silently")
```

These approaches worked but struggled with:
- Long-range context
- Parallelization
- Training efficiency at scale (required supervision)

Transformers removed these bottlenecks, which is why LLMs became feasible only in the last few years.

```
Transformers
---------------------------------------------------------
|  The [cat] sat silently in the corner of the room.  |
|      ^                                                |
|      |                                                |
|      | (Strong Attention Weight)                      |
|      |                                                |
|      |          [Many sentences later...]             |
|      |                                                |
|      |                                                |
|      -------------------------------                  |
|                                    |                  |
|  Even though the lights were off, [it] still saw.     |
---------------------------------------------------------
                                     |
           [Self-Attention Mechanism] +---> "It" = "Cat"
```


| Feature  | RNN (The Conveyor)     | CNN (The Scanner)           | Transformers (The Crowd)       |
|----------|------------------------|-----------------------------|-------------------------------|
| Speed    | Slow (One by one)      | Fast (Parallel chunks)      | Fastest (Entire text at once) |
| Memory   | Forgets long distance  | Only sees local ""windows"" | Infinite (Relates every word) |
| Best For | Short, fluid sequences | Detecting specific phrases  | Everything (LLMs)             |


---

## Encoder, Decoder, and Encoder–Decoder Models (With Use Cases)

Transformers show up in different configurations depending on the task.

---

### Encoder Models: Understanding

Encoder models read text and produce representations that capture meaning.

Common uses:
- Search and retrieval
- Semantic similarity
- Classification
- Embeddings for analytics, clustering, and RAG systems

If your goal is *understanding* text rather than generating new text, encoders are often cheaper and more predictable.

---

### Decoder Models: Generation

Decoder models generate text one token at a time, using previous tokens as context.

This pattern powers:
- Chat interfaces
- Code generation
- Autocomplete
- Content creation

Most general-purpose LLMs in production today are decoder-only models.

---

### Encoder–Decoder Models: Transformation

Encoder–decoder models first encode an input, then generate an output conditioned on it.

They are well-suited for:
- Translation
- Summarization
- Structured text transformation

> **Why this matters:**  
> Understanding these patterns helps avoid overusing large, general-purpose models when simpler architectures are sufficient.

---

## How LLMs Are Trained (Narrative View)

Training LLMs typically happens in stages. The details vary by model, but the high-level flow is consistent.

---

### 1. Pre-Training: Learning Language

In pre-training, models learn general language structure by predicting missing or next tokens across massive datasets.

This phase:
- Uses self-supervised learning
- Requires no manual labeling
- Produces a *foundation model*

Most general capability comes from this stage.

---

### 2. Fine-Tuning: Shaping Behavior

Fine-tuning adapts a foundation model for specific tasks or domains.

This may involve:
- Task-specific examples
- Domain-relevant data
- Partial or full parameter updates

Fine-tuning improves usefulness but increases cost and specialization.

---

### 3. Reinforcement Learning from Human Feedback (RLHF)

RLHF aligns models with human preferences.

At a high level:
- The model generates multiple responses
- Humans rank or select preferred outputs
- A reward model is trained
- Reinforcement learning optimizes the model against that reward

This stage is a major reason modern LLMs feel usable in real products.

---

## Why These Details Matter (Even If You Skipped Them)

You don’t need to remember every concept here. But understanding *that they exist* helps explain:

- Why prompts behave probabilistically
- Why long context windows are expensive
- Why models sometimes sound confident but are wrong
- Why architectural patterns like RAG keep emerging

These mechanics directly shape system design decisions.

---

## What’s Next

Now that we have a working mental model, the next post will zoom out and look at the **model landscape**:
- General-purpose vs domain-specific models
- Capability vs cost tradeoffs
- Why model choice is an architectural decision, not just a vendor decision

That’s where theory starts turning into concrete choices.

---

