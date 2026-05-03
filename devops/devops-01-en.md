# Why Prometheus and Jaeger Are Not Enough to Observe LLMs in Production

*By Samuel Desseaux - Erythix*

---

## Your observability stack works. And yet.

Prometheus scrapes your metrics every 15 seconds. Jaeger captures your distributed spans. Grafana displays dashboards your team consults at every incident. You have SLOs, alerts, and runbooks.

Then you deploy your first LLM service in production - a chatbot, a documentation Q&A system, an automated summarization pipeline. Users start complaining: the answers are off-topic, sometimes incoherent, sometimes invented.

You open Grafana. P99 latency is stable. The HTTP error rate is at zero. Prometheus flags nothing. Jaeger shows a `POST /v1/chat/completions` span completing in 800ms with a 200 status.

The problem is there, but your tools cannot see it. Why?

---

## What your tools measure - and what they ignore

OpenTelemetry, Prometheus, and Jaeger were designed to observe deterministic systems. A request enters a service, processing follows predictable logic, a response exits. The metrics that matter are latency, error rate, and resource saturation.

An LLM is not deterministic. For the same input, it can produce radically different outputs depending on temperature, conversation context, and system instructions. A "correct" response in the HTTP sense (status 200, normal latency) can be completely wrong in the application sense - fabricated information, off-topic answer, inappropriate tone.

Your current tools measure the **health of the pipe**. They do not measure the **quality of what flows through it**.

### The semantic gap

The diagram below illustrates the gap between what your existing stack captures and what you actually need for LLM diagnosis:

```
┌──────────────────────────────────────────────────────────────────────┐
│  What Prometheus + Jaeger see                                        │
│                                                                      │
│  span: POST /v1/chat/completions                                     │
│    ├── duration_ms: 823                                              │
│    ├── http.status_code: 200                                         │
│    └── http.url: api.openai.com                                      │
│                                                                      │
│  ✓ Service is up    ✓ Latency within SLO    ✗ Quality: unknown       │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  What Langfuse sees                                                  │
│                                                                      │
│  Trace: rag-pipeline                                                 │
│    ├── span: embed-query           42ms                              │
│    ├── span: vector-search        156ms   3 docs, sim: 0.72         │
│    ├── span: assemble-prompt        2ms                              │
│    └── generation: llm-answer     623ms                              │
│          ├── model: gpt-4o-mini-2024-07-18                           │
│          ├── prompt_tokens: 512   completion_tokens: 128             │
│          ├── cost: $0.000052                                         │
│          ├── input: "What is the refund policy for order #4421?"     │
│          ├── output: "You can return items within 30 days..."        │
│          └── scores: faithfulness=0.92  relevance=0.88              │
└──────────────────────────────────────────────────────────────────────┘
```

Here is what you actually need to diagnose a quality regression:

- What was the exact prompt sent?
- Which model responded, and which exact version?
- How many tokens were consumed on input and output?
- Was the response coherent with the question?
- Was this a step in a longer pipeline - which one?
- Did the user express dissatisfaction after this response?

None of this information exists in a standard HTTP span. And that is by design: OpenTelemetry was not built to understand the content of LLM calls.

---

## Three production incidents your current tools will not detect

### 1. Silent quality drift

Your Prometheus metrics are stable. But for three days, the model has been responding in English while your users write in French. No alert fires, because no metric measures output language.

With an LLM observability tool, you would have: output history, an automatic language scorer, and an alert on the out-of-language response rate.

### 2. Retrieval silently returning nothing useful

Your RAG pipeline runs in three steps: retrieve relevant documents, assemble context, generate. Retrieval sometimes returns zero relevant results - the LLM then generates a confident, fluent, and completely fabricated answer.

Your current stack sees a 200ms generation span. It does not see that the context provided to the model was empty. Result: users trusting hallucinated information.

### 3. Undetected cost explosion

A prompt bug doubles the number of tokens sent to the model. Latency increases slightly but stays within your SLOs. Your API bill explodes. You discover it at month end.

An LLM observability tool would have alerted in real-time on the estimated cost per request at the first deviations.

---

## What an LLM observability tool actually does

Tools like Langfuse were designed specifically to close this gap. They operate at a semantic level that general-purpose tools cannot reach.

Concretely, they capture and expose:

**The structure of LLM execution.** An LLM pipeline is not an atomic call - it is a hierarchy of operations: retrieval > reranking > generation > post-processing. Langfuse represents this hierarchy as Trace > nested Spans, with the detail of each step.

**The content of exchanges.** Inputs (prompts, messages, retrieved documents) and outputs (generated responses) are stored and queryable. This is what enables real debugging: you see exactly what was sent to the model and what it returned.

**Consumption metrics.** Input tokens, output tokens, model used, estimated cost per generation. These data points enable real-time monitoring of cost and consumption drift.

**Evaluation scores.** Human annotations, automated scores (relevance, consistency, language), LLM-as-judge evaluations. These scores measure the quality of responses over time, not just latency.

**User sessions.** A user who sends several messages in a conversation forms a session. The LLM observability tool lets you follow that session end-to-end and understand where it degrades.

---

## This is not a replacement - it is a complementary layer

It would be wrong to conclude that your current tools are useless for LLM workloads. They remain indispensable for what they do well:

- Prometheus measures CPU and memory saturation on your inference service
- Jaeger traces distributed calls between your microservices, including calls to the LLM API
- Grafana aggregates your operational dashboards

What you need to add is a layer that understands LLM semantics. The two stacks coexist and complement each other: your infra stack alerts on system problems, your LLM stack alerts on quality problems.

A useful analogy: you already have sensors on your electrical network (voltage, current, frequency). You add a system that measures whether the devices plugged into it are working correctly. Both levels of measurement are necessary.

---

## The three pillars of LLM observability

Once you accept the need for a dedicated tool, the discipline has three core pillars:

```
┌──────────────┐   ┌──────────────┐   ┌──────────────────────┐
│  TRACING     │   │  EVALUATION  │   │  DATASETS            │
│              │   │              │   │                      │
│  Capture     │   │  Score every │   │  Extract production  │
│  every       │   │  generation  │   │  traces into labeled │
│  interaction │   │  · rules     │   │  benchmarks          │
│  · prompts   │   │  · LLM judge │   │                      │
│  · responses │   │  · human     │   │  Enables:            │
│  · steps     │   │              │   │  · regression tests  │
│  · tokens    │   │  Monitors:   │   │  · fine-tuning data  │
│  · cost      │   │  quality     │   │  · A/B comparison    │
│              │   │  over time   │   │                      │
│  Enables:    │   │              │   │  Closes the loop     │
│  debugging   │   │  Enables:    │   │  between production  │
│              │   │  alerting    │   │  and training        │
└──────────────┘   └──────────────┘   └──────────────────────┘
      │                   │                     │
      └───────────────────┴─────────────────────┘
                          │
                      Langfuse
```

**Tracing.** Capturing the complete execution of every LLM interaction - inputs, outputs, intermediate steps, duration, token consumption - in a queryable store. This is what enables debugging.

**Evaluation.** Assigning quality scores to generations - automatically (rule-based, LLM-as-judge) or manually (human annotation). This is what enables systematic quality monitoring.

**Datasets.** Extracting production traces into labeled datasets for regression testing, offline evaluation, and fine-tuning data. This is what closes the feedback loop between production and model improvement.

Langfuse covers all three. This guide covers each in depth.

---

## Where to go next

Module DevOps-02 covers the installation of Langfuse in self-hosted mode and the instrumentation of your first LLM call. In under 30 minutes, you will have a readable, navigable LLM trace.

What you will discover: the difference between seeing an HTTP span at 800ms and seeing 512 prompt tokens, 128 output tokens, the exact model version, and the raw response - all in a single view.

---

*Samuel Desseaux is the founder of Erythix, an observability stack specialist and LLM security practitioner. Speaker at FOSDEM 2026 and KubeCon Europe.*
