# Langfuse vs Arize Phoenix vs Helicone: An Honest Comparison

*By Samuel Desseaux - Erythix*

---

## Why This Comparison Matters Now

The LLM observability market has densified rapidly since 2023. Langfuse, Arize Phoenix, Helicone, Traceloop, Weave (W&B), LangSmith - each positions differently. Choosing the wrong tool is expensive: migration, re-instrumentation, loss of historical data.

This comparison focuses on the three most widely deployed tools outside hyperscaler ecosystems, evaluated against objective and tested criteria: deployment model, technical integration, evaluation capabilities, cost, and data governance.

---

## The Three Candidates in One Sentence

**Langfuse** - open-source LLM observability, self-hostable, trace- and evaluation-focused, with native OTLP support since v2.

**Arize Phoenix** - open-source LLM analysis and evaluation tool, data-science-oriented with strong drift detection, best for offline analysis.

**Helicone** - SaaS LLM proxy with automatic logging, cost and usage-oriented, one-line deployment.

---

## Deployment and Data Sovereignty

| Criterion | Langfuse | Arize Phoenix | Helicone |
|---|---|---|---|
| Self-hosted | Yes (Docker, Helm) | Yes (pip install) | No (cloud only) |
| Open-source | Yes (MIT) | Yes (Apache 2.0) | No |
| Data stays on-premise | Yes | Yes | Depends on plan |
| Deployment model | Full app (Postgres + worker) | Local process or server | Cloud proxy |
| Production-grade HA | Yes (Kubernetes Helm) | Limited | Managed by vendor |

**Langfuse** is the most mature option for production self-hosted deployments. The Helm chart is actively maintained, PostgreSQL is a standard choice, and ClickHouse is available for high-volume workloads.

**Arize Phoenix** deploys with `pip install arize-phoenix` followed by `px.launch_app()`. This is an asset in experimentation, but it is not designed for high-availability production deployments with persistent storage.

**Helicone** offers no self-hosting. If your primary constraint is data sovereignty - financial sector, healthcare, defense, public sector - Helicone is ruled out immediately.

---

## Technical Integration and OTEL Support

| Criterion | Langfuse | Arize Phoenix | Helicone |
|---|---|---|---|
| Python SDK | Yes | Yes | Yes |
| TypeScript/Node SDK | Yes | Partial | Yes |
| Native OTLP endpoint | Yes (v2+) | Yes | No |
| GenAI Semantic Conventions | Yes | Yes | No |
| Proxy mode (no SDK) | No | No | Yes (native) |
| LangChain integration | Yes (callbacks) | Yes (callbacks) | Yes (proxy) |
| LlamaIndex integration | Yes | Yes | Partial |
| Instrumentation-free option | Via OTEL auto-instrumentation | Via OTEL auto-instrumentation | Via proxy headers |

Helicone's proxy mode is its key differentiator: redirect your OpenAI calls to `https://oai.helicone.ai/v1` and add two headers. No code changes. Appealing for a PoC, but it places Helicone on the critical path of every LLM call - a single point of failure.

Both Langfuse and Phoenix support the GenAI Semantic Conventions, which means OTEL auto-instrumented libraries work without proprietary SDK changes.

---

## Evaluation Capabilities

| Criterion | Langfuse | Arize Phoenix | Helicone |
|---|---|---|---|
| Human annotation | Yes (annotation queues) | Yes | Limited |
| Rule-based scoring | Yes | Yes | No |
| LLM-as-judge | Yes (Evals SDK) | Yes (evals library) | No |
| Production datasets | Yes | Yes | No |
| Drift detection | Limited | Strong | No |
| A/B prompt comparison | Yes | Yes | Limited |
| Pre-built evaluators | Basic | Extensive | No |
| Real-time scoring | Yes | Partial | No |

Arize Phoenix's evaluation library is its main strength. It provides pre-built evaluators (hallucination, relevance, toxicity, security) that work on entire datasets, not only on individual traces. It is the preferred tool for deep offline analysis work - especially for data science teams running experiments in notebooks.

Langfuse is stronger on continuous production evaluation: annotation queues, real-time scoring, Grafana integration via its API. The workflow for turning production traces into a regression dataset is more mature.

Helicone is not positioned for evaluation. It is a cost and usage monitoring tool.

---

## Cost and Commercial Model

| Criterion | Langfuse | Arize Phoenix | Helicone |
|---|---|---|---|
| Self-hosted (free) | Yes (unlimited) | Yes (unlimited) | N/A |
| Free cloud tier | Yes (limited events) | Yes (limited) | Yes (10k req/month) |
| Paid cloud | From $59/month | Contact sales | From $20/month |
| Open-core model | Yes | Yes | No |
| Predictable TCO | High (self-hosted) | High (self-hosted) | Low predictability at scale |

At scale on cloud plans, Helicone costs can grow non-linearly with request volume. Self-hosted Langfuse and Phoenix have a predictable cost model: your infrastructure bill.

---

## Decision Guide

**Choose Langfuse if:**
- You must self-host for data sovereignty
- You have an existing OTEL stack to integrate
- You want a continuous production evaluation pipeline
- You operate in a regulated sector (finance, healthcare, defense)
- You need a Kubernetes-native deployment with Helm

**Choose Arize Phoenix if:**
- Your priority is offline analysis and drift detection
- You have an ML team working primarily in notebooks
- You want pre-built evaluators without writing custom scorers
- You are in heavy experimentation mode with limited immediate production constraints

**Choose Helicone if:**
- You want cost and usage monitoring in five minutes, zero infrastructure
- You have no data sovereignty constraints
- Your use case is limited (prototype, side project, internal tool)
- You do not need quality evaluation

---

## What None of Them Do

It is worth being honest about shared limitations:

None of them replaces an enterprise APM (Dynatrace, Datadog) for general application observability - infrastructure correlation, distributed tracing across non-LLM services, APM dashboards.

None of them handles monitoring of classical ML model quality - feature drift, PSI/KL divergence, classification performance degradation. For those needs, Evidently AI or Fiddler are better suited.

None of them provides native SIEM integration for correlating LLM activity with security events. This requires a custom export layer (see module SRE-04 for the Langfuse approach).

For workloads with requirements in these domains, a hybrid architecture is necessary: Langfuse for the LLM semantic layer, your existing APM for the infrastructure layer, and a custom export for security correlation.

---


