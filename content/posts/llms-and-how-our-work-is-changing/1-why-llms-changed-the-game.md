---
title: "Why Large Language Models Changed the Game"
date: 2023-03-07
draft: false
author: "Girish Shekhar"
summary: >
  Large Language Models feel different from earlier machine learning systems.
  This post explains why — focusing on tokens, training data, self-supervision,
  and the architectural patterns that changed how we build software.
tags: ["writing", "engineering", "science", "llm", "gen-ai", "machine-learning", "ai-fundamentals"]
categories: ["gen-ai", "machine-learning"]
series:
  - LLMs and How Our Work Is Changing
ShowToc: true
TocOpen: true  
---

## Purpose

Over the last few years, generative AI — and Large Language Models (LLMs) in particular — have started to show up everywhere: in developer tools, customer support, analytics workflows, and product experiences.

This isn’t just another machine learning trend.

LLMs change **how we think about software design**. Instead of hard-coding rules or building narrowly scoped models, we now interact with systems that can reason over language, code, and data in a much more general way.

This post lays the foundation. We’ll keep the concepts intentionally high-level and accessible, so engineers, product managers, and data practitioners all start from the same mental model.

---

## What Is a Language Model?

At its simplest, a **language model predicts what comes next**.

Given some text, the model estimates the probability of the *next* piece of text. There is no rules engine or symbolic reasoning system underneath. Everything the model produces comes from repeatedly answering one question:

> *“Given everything I’ve seen so far, what is most likely to come next?”*

When trained at sufficient scale, this surprisingly simple objective leads to behavior that feels intelligent.

---

## Tokens: How Models Actually See Text

One detail that’s easy to miss: **models don’t operate on words**.

Before text reaches a model, it is broken into **tokens**. A token might be:
- A whole word  
- Part of a word  
- Punctuation or symbols  

For example, the word “unbelievable” may be split into multiple tokens depending on the tokenizer.

```
Sentence:  "Subterranean        depths       are      cool       !"

WORDS:      [Subterranean]       [depths]    [are]    [cool]    [!]
               |                    |          |        |        |
TOKENS:     [Sub] [terr] [anean] [depths]    [are]    [cool]    [!]
```

**Why this matters:**
- Tokenization affects cost, latency, and accuracy
- It explains why long or oddly formatted inputs behave differently
- It’s a key difference from older text models that relied on hand-crafted features

> **Mental model:**  
> If words are how *humans* read text, tokens are how *machines* read text.

We’ll revisit tokenization in more detail later in the series.

---

## How LLMs Differ from Classic Machine Learning Models

If you’ve worked with machine learning before, LLMs may feel unfamiliar. That’s because they differ in a few fundamental ways.

---

### 1. The Data Is Broad, Not Narrow

Traditional ML models are usually trained on **task-specific datasets**:
- Fraud detection
- Recommendations
- Classification
- Forecasting

LLMs are trained on **large, diverse collections of text**, including:
- Documentation
- Articles and books
- Code
- Forums and Q&A
- Structured and semi-structured text

This breadth allows LLMs to generalize across tasks instead of being good at just one thing.

| Feature     | Classic Machine Learning                 | Large Language Models (LLMs)          |
|-------------|------------------------------------------|---------------------------------------|
| Data Source | Specialized (Medical logs; Stock prices) | General (Common Crawl; Books; GitHub) |
| Size        | Megabytes to Gigabytes                   | Terabytes to Petabytes                |
| Variety     | Single domain / Structured               | Multi-domain / Unstructured           |
| Labeling    | Human-labeled (Supervised)               | Self-supervised (Predict next word)   |

```
Classic ML (Laser Focus)          LLM (The Library of Babel)
      [ Narrow ]                        [  Broad  ]
      [ Curated]                        [  Mixed  ]
      [ Data   ]                        [  Corpus ]
         |                                 |
         v                                 v
   Specific Tasks                   General Reasoning
(e.g. Fraud Detection)            (e.g. Writing Poetry)
```

---

### 2. Learning Is Self-Supervised

A key enabler of LLMs is **self-supervised learning**.

Instead of relying on large, manually labeled datasets, LLMs learn by using the data itself as the training signal. The model is shown text and trained to predict missing or next tokens — effectively generating its own labels.

**Why this matters:**
- It scales to massive datasets
- It removes the bottleneck of human annotation
- It allows models to learn general language structure before being adapted to specific tasks

We’ll dive deeper into how self-supervision works in a later post, but it’s one of the main reasons LLMs became feasible at this scale.

---

### 3. Features Are Learned, Not Engineered

Classic models often depend on humans to decide:
- What features matter
- How text should be represented

LLMs learn internal representations directly from data. As models grow larger, these representations become reusable across many tasks.

> **Why this matters:**  
> This is why a single LLM can summarize text, explain code, and generate SQL — without being explicitly trained for each task.

---

### 4. Scale Unlocks New Behavior

As models grow in size and see more data, new capabilities appear:
- Following instructions from examples
- Adapting behavior based on prompts
- Handling tasks they were never directly trained on

These capabilities are often described as **emergent behavior** and are one of the defining characteristics of modern LLMs.

---

## A Simple Tour of Model Architectures

Most modern LLMs are built using a family of models called **transformers**. You don’t need to understand the math to understand the roles these models play.

At a high level, there are three common patterns.

---

### Encoder Models: Understanding Text

Encoder models focus on **understanding input**.

They take text in and produce representations that capture meaning. These models are commonly used for:
- Search and retrieval
- Semantic similarity
- Classification
- Embeddings for analytics or clustering

---

### Decoder Models: Generating Text

Decoder models focus on **generation**.

They produce text one token at a time, using previously generated tokens as context. This is the pattern behind:
- Chatbots
- Code generation
- Autocomplete systems

Most general-purpose LLMs used today fall into this category.

| Architecture    | How it works                                          | Primary Use Cases                                                   |
|-----------------|-------------------------------------------------------|---------------------------------------------------------------------|
| Encoder-Only    | Reads the whole input at once to understand context.  | Sentiment analysis, Named Entity Recognition (NER), Classification. |
| Decoder-Only    | Predicts the next token one by one (auto-regressive). | Text generation (ChatGPT, Claude), Creative writing, Coding help.   |
| Encoder-Decoder | Maps an input sequence to a specific output sequence. | Translation (English -> French), Summarization, Paraphrasing.       |


```
ENCODER             DECODER             ENCODER-DECODER
[Input]             [Prompt]            [Input] -> [Hidden State]
   |                   |                            |
[Understanding]     [Generation]                [Translation]
   |                   |                            |
[Label/Class]       [Response]                  [Output]
```

---

### Encoder–Decoder Models: Transforming Text

Encoder–decoder models combine both ideas and is used with the input representation is different than what we want to produce on the output.

They:
1. Encode an input
2. Generate an output conditioned on that input

These models are commonly used for:
- Translation
- Summarization
- Structured text transformation

> **Why this distinction matters:**  
> Understanding these patterns helps you choose the right tool — and avoid using a general-purpose LLM when a simpler, cheaper model would suffice.

---

## What About Older Approaches Like RNNs and CNNs?

Before transformers, language models were often built using:
- **Recurrent Neural Networks (RNNs)**, which process text sequentially
- **Convolutional Neural Networks (CNNs)**, adapted from image processing

These approaches worked but struggled with:
- Long-range context
- Parallelization
- Scaling to very large datasets

Transformers addressed these limitations by allowing models to look at **entire sequences at once**, which is a major reason LLMs scale as well as they do today.

---

## Why This Changes How We Build Software

LLMs blur boundaries that used to be clear:
- Logic vs data
- Code vs configuration
- Input vs interface

Instead of encoding every rule, we increasingly design **prompts, context, and feedback loops**. Instead of rigid APIs, we design systems that can adapt based on examples.

That shift has real architectural consequences — and that’s what the rest of this series will explore.

---

## What’s Next

In the next post, we’ll look more closely at how text is represented inside models:
- Tokenization
- Text encoding
- Why these choices matter in real systems