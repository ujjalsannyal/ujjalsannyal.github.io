---
title: "What is RAG? Why LLMs Need It (RAG vs Fine-Tuning vs Long Context)"
date: 2026-11-24
draft: false
description: "LLMs are frozen, blind to your data, and confidently wrong. Here is how RAG fixes that, and when to use fine-tuning or long context instead."
tags: ["rag", "llm", "hallucination", "fine-tuning", "long-context", "chromadb"]
categories: ["RAG From Scratch"]
series: ["RAG From Scratch"]
toc: true
---

> **Series:** RAG From Scratch · Module 1: Foundations · Episode 1
> **TL;DR:** RAG separates *knowledge* from *reasoning*. The model reasons; a retrieval layer supplies facts at question time. Use RAG for facts, fine-tuning for behavior, long context for small static corpora.

## The problem: three gaps in every LLM

An LLM is a compressed snapshot of its training data. That snapshot has three gaps.

| Gap | What it means | Consequence |
|---|---|---|
| **Frozen in time** | Knowledge cutoff | Nothing after the cutoff exists for the model |
| **Blind to your data** | Internal docs, tickets, DBs were never in training | Cannot answer questions about your domain |
| **No calibrated uncertainty** | Trained to produce plausible text, not to abstain | Fluent, confident, wrong answers (hallucination) |

The first two are *missing knowledge*. The third is what happens as a result. The real engineering question: **how do we get the right knowledge in front of the model at the moment it answers?**

## Hallucination demo

To make the failure provable, use a document no model has seen. Take a one-page fake handbook for an imaginary company, **Zentra Labs**:

```text
Remote work stipend: ₹18,500/month
PTO carry-over: maximum 7 days
Security incident hotline: ext. 4417
```

Ask a model **with no retrieval** three questions:

| # | Question | Ground truth | Typical no-RAG result |
|---|---|---|---|
| 1 | Monthly remote work stipend at Zentra Labs? | ₹18,500 | Plausible invented figure |
| 2 | How many PTO days carry over? | 7 | Invented number or generic policy |
| 3 | Security incident hotline extension? | 4417 | Invented number |

The answers come back specific, fluent, and unhedged. Nothing in the output signals an error. That is what makes hallucination dangerous: **a wrong answer looks identical to a right one.**

Now ask the same questions, same model, but first search the handbook and paste the relevant passages into the prompt. All three answers come back correct, and each can be traced to a source chunk. The model did not get smarter. It got the right information at the right moment. That is RAG.

## What RAG is

Three steps:

1. **Retrieve**: search a knowledge base for the passages most relevant to the question. Usually embedding-based semantic search; the best systems combine it with keyword search (hybrid).
2. **Augment**: insert those passages into the prompt next to the question.
3. **Generate**: the LLM answers from that context, ideally with citations.

```text
User question → Embed → Vector search → Top-k chunks
                                            ↓
        Answer ← LLM ← Prompt (question + chunks)
```

The key idea: **facts live in a store you can update, permission, and audit; the model supplies language and reasoning.** No retraining required when facts change.

### Minimal implementation (ChromaDB)

```python
import chromadb

client = chromadb.Client()
col = client.create_collection("handbook")

col.add(
    ids=["c1", "c2", "c3"],
    documents=[
        "Remote work stipend: ₹18,500 per month.",
        "PTO carry-over is capped at 7 days.",
        "Security incident hotline: extension 4417.",
    ],
)

def answer(question: str, llm) -> str:
    hits = col.query(query_texts=[question], n_results=2)
    context = "\n".join(hits["documents"][0])
    prompt = (
        "Answer using ONLY the context below. "
        "If the answer is not in the context, say you don't know.\n\n"
        f"Context:\n{context}\n\nQuestion: {question}"
    )
    return llm(prompt)  # your LLM client of choice
```

Two details matter even in a toy: the **"use only the context / say you don't know"** instruction, and keeping the retrieved chunks around so you can show sources.

## RAG vs fine-tuning vs long context

The common objections: *"Why not fine-tune on our data?"* and *"Context windows hold a million tokens, why not paste everything?"*

- **Fine-tuning** changes weights. Excellent for **behavior**: tone, output format, task patterns. Poor for **facts**: knowledge baked into weights is hard to update, cannot be cited, and the model can still hallucinate around it.
- **Long context** puts documents directly in the prompt. Genuinely the simplest option for a **small, static corpus**. It does not scale: you pay for those tokens on every query, latency grows with input size, details buried mid-prompt can be missed, and per-user permissions are hard to enforce on a pasted blob.
- **RAG** retrieves only what is relevant per query. Scales to millions of documents, updates the moment you reindex, supports citations and access control. The cost: you now own a retrieval system, and **answer quality is capped by retrieval quality**.

### Decision table

| Dimension | RAG | Fine-tuning | Long context |
|---|---|---|---|
| **Best at** | Injecting facts / private data | Changing behavior, style, format | Small corpus, fast prototype |
| **Freshness** | Real-time (update the index) | Stale until retrained | Fresh per request |
| **Corpus size** | Millions of docs | N/A (not for facts) | Limited by window |
| **Citations** | Native | None | Possible, manual |
| **Per-user access control** | Yes (filter at retrieval) | No | Hard |
| **Factual hallucination** | Reduced; grounded in retrieved text | Not reliably reduced | Reduced if info is found |
| **Cost per query** | Low to medium | Low after training | High at scale |
| **Upfront cost** | Pipeline + index | Data prep + training | Almost none |
| **Latency** | Retrieval + generation | Lowest | Grows with context size |
| **Main failure mode** | Bad retrieval | Stale, uncited facts | Lost-in-the-middle, cost |

### Rules of thumb

| If you need... | Use |
|---|---|
| Answers from private or changing data, with sources | **RAG** |
| A specific tone, format, or task behavior | **Fine-tuning** |
| A quick answer over one short document | **Long context** |
| A production assistant | **Combination**: RAG for facts + fine-tuning for behavior |

These are not rivals. Many production systems combine them. Pick by the problem, not the hype.

## Where RAG breaks

RAG is not magic. Most RAG failures are **retrieval failures, not model failures**.

| Failure | Symptom | Fixed by (later in series) |
|---|---|---|
| Bad chunking | Answer split across chunks | Chunking strategies |
| Weak embeddings | Retrieves text that sounds related but isn't | Embedding model selection |
| Poor ranking | Right doc ranks 11th, you pass 5 | Hybrid search, reranking |
| Context ignored | Model has the facts, answers anyway from priors | Prompting, evaluation |
| No measurement | You can't tell if changes help | Retrieval and answer evals |

## Recap

- LLMs are **frozen**, **blind to your data**, and **confidently wrong** when they lack facts.
- **RAG** retrieves relevant knowledge at question time and hands it to the model, separating knowledge from reasoning.
- **Fine-tuning** shapes behavior, not facts. **Long context** works for small corpora but gets expensive and unreliable at scale.
- Retrieval quality is the ceiling on RAG quality.

**Next up:** Episode 2, chunking and embeddings. How you split documents decides whether retrieval works at all.
