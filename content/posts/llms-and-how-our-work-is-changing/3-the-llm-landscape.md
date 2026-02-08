---
title: "The LLM Landscape: Models, Capabilities, and Tradeoffs"
date: 2023-03-29
draft: false
summary: >
  Once you understand how LLMs work, the next real question is model choice.
  This post explains how to reason about LLM tradeoffs — capability, cost,
  controllability, and domain fit — without relying on benchmarks alone.
tags: ["llm", "gen-ai", "software-architecture", "model-selection"]
categories: ["gen-ai", "machine-learning"]
series:
  - LLMs and How Our Work Is Changing
ShowToc: true
TocOpen: true
---

## Why Model Choice Is Harder Than It Looks

After teams get past the initial excitement of “LLMs can do this now,” they quickly hit a practical question:

> *Which model should we actually use?*

This is where many discussions go off the rails.

Model choice is often framed as a popularity contest — which model is bigger, newer, or ranks higher on a benchmark. In practice, choosing an LLM is an **architectural decision**, not a feature comparison. The wrong choice won’t just cost more money; it will surface as reliability issues, latency spikes, and systems that are difficult to reason about or control.

The key mistake is assuming that LLMs differ primarily in intelligence. They don’t. They differ in **tradeoffs**.

---

## A More Useful Way to Think About LLMs

At a high level, most modern LLMs share the same underlying architecture and training paradigm (as covered in Part 2). What changes is not *what they are*, but *what they are optimized for*.

The most important dimensions tend to be:

- **Capability** – how well the model performs across a range of tasks
- **Cost** – both training and inference, usually tied to token usage
- **Latency** – how quickly responses are generated at scale
- **Controllability** – how predictable and steerable outputs are
- **Domain alignment** – how well the model matches your problem space

Every model sits somewhere different along these axes. There is no universally “best” model — only models that are better or worse aligned with your system’s constraints.

---

## General-Purpose Models: Breadth First

General-purpose models are trained on massive, diverse corpus and optimized to perform reasonably well across many domains.

They shine when:
- Tasks are open-ended
- Requirements are still evolving
- You need strong zero-shot or few-shot behavior

This is why they’re often the first models, teams reach for. They feel powerful, flexible, and surprisingly capable.

The tradeoff is **variability**. These models are harder to constrain, more expensive to run, and more likely to produce outputs that look correct while subtly missing context. As systems mature, that variability becomes a liability.

General-purpose models are excellent for exploration. They are not always the best choice for stable production systems.

---

## Domain-Specific Models: Precision Over Reach

Domain-specific models narrow the problem space deliberately. They are trained or fine-tuned on data that reflects a particular domain, task, or style of output.

When aligned well, they offer:
- Higher precision
- Lower inference cost
- More predictable behavior
- Lower latency

This predictability matters. In production systems, consistency is often more valuable than peak capability.

The downside is obvious: these models are less flexible. They don’t generalize as broadly, and they require domain data and maintenance. But for many real systems, that is an acceptable — even desirable — tradeoff.

---

## Encoder vs. Decoder Models in Practice

One source of confusion in model selection is architectural class.

- **Encoder-based models** are optimized for understanding: classification, search, clustering.
- **Decoder-based models** are optimized for generation: text, code, structured output.

Using a generative model for pure understanding tasks is often overkill. It increases cost and complexity without improving outcomes. Conversely, forcing encoder models into generative workflows leads to brittle systems and awkward pipelines.

Good architecture starts with matching the model class to the job.

---

## Concrete LLM families

### BERT \[Encoders\] (2018\)

BERT is a bidirectional encoder representation from transformers. It is a natural language processing model that was introduced in 2018 by Google AI. Its bidirectional nature helps consider the context of a word by looking at the words that come before and after it. This is in contrast to unidirectional models, which can only consider the words that come before a given word. BERT is also a transformer model, which means that it uses attention to learn the relationships between words.

BERT was pre\-trained on a massive dataset of text and code. It was then fine\-tuned on a variety of natural language processing tasks, including question answering, natural language inference, and sentiment analysis. BERT has achieved state\-of\-the\-art results on many of these tasks.

BERT is a powerful language processing model that has been used in a variety of applications including but not limited to Google Search (to improve the accuracy of search results), Google Translate (to improve the accuracy of translations), chatbots, machine translation, and text summarization.

BERT was open sourced in 2018 and led to many flavors including RoBERTa (Facebook), DistilBERT (Hugging Face), TinyBERT, etc.

### GPT\-J \[Decoders\] (2021\)

GPT\-J is an open source LLM developed by EleutherAI. It is a 175 billion parameter model, making it one of the largest language models ever created. Trained massive dataset of text and code, it can be used for a variety of tasks, including text generation, translation, and question answering. In a paper published in 2021, researchers showed that GPT\-J was able to achieve state\-of\-the\-art results on a number of natural language processing tasks, including question answering (on diverse topics and can be used for answering homework questions, customer service, etc), generating texts (poems, code, scripts, musical pieces, email, letters, etc.), summarization, and translation.

### FLAN Unified Language Learning (UL2\) \[Encoder\-Decoder\] (2022\)

**Flan Collection** is a newer and more extensive publicly available collection of tasks, templates, and methods for instruction tuning to advance the community’s ability to analyze and improve instruction\-tuning methods. This collection was [first used](https://arxiv.org/abs/2210.11416) in Flan\-T5 and Flan\-PaLM, for which the latter achieved significant improvements over [PaLM](https://ai.googleblog.com/2022/04/pathways-language-model-palm-scaling-to.html).

**Unified Language Learner** (UL2\) is a framework improves the performance of language models universally across datasets and setups. UL2 frames different objective functions for training language models as [denoising](https://arxiv.org/abs/1910.10683) tasks, where the model has to recover missing sub\-sequences of a given input. During pre\-training it uses a novel *mixture\-of\-denoisers* that samples from a varied set of such objectives, each with different configurations. This was open sourced in Q2 2022\.

​
Combining both methods, Flan UL2 (20B checkpoint \- A snapshot in time where the model converged) makes it more usable for few\-shot in\-context learning and removes the need for mode token switching. It outperforms several LLMs with large parameters (PALM \~540 B). Its based on T5 architecture and was open sourced in Feb 2023\. More details available in [Flan References](https://quip-amazon.com/npWqA97btvqs#temp:C:BGEc8f2c0de33b149da80fd489ce).

### Alpaca \[Decoders\] (2023\)

**Alpaca** is fine\-tuned from Meta’s [LLaMA](https://ai.facebook.com/blog/large-language-model-llama-meta-ai/) 7B model. It was created by the research community and Stanford who were unable to study the closed text\-davinci\-003 model from Open AI. Alpaca shows many behaviors similar to OpenAI’s text\-davinci\-003, but is also surprisingly small and easy/cheap to reproduce.

The model was released with training recipe, data, and model weights. Alpaca is intended **only for academic research** and any **commercial use is prohibited** inheriting its licenses from Llama and Open AI instruction data.


---

## Why Benchmarks Rarely Settle the Question

Benchmarks are useful, but incomplete.

They measure performance on static datasets under controlled conditions. Production systems care about different questions:

- How does the model fail?
- How predictable are those failures?
- How expensive is failure at scale?
- How easily can we observe and correct behavior?

These questions don’t show up on leaderboards, but they dominate real-world outcomes.

> **My recommendation is to build your own benchmark for your production system and evaluate models against these instead of relying on public benchmarks.**

---

## Looking Ahead

Once models meet real users, limitations surface quickly. In the next post, we’ll look at the **failure modes engineers consistently encounter in production**, and why many of them are structural rather than prompt-related.
