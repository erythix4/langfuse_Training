# Langfuse in Production — Observability Guide

**By Samuel Desseaux — [Aureonis](https://aureonis.io)**

A complete, practitioner-written guide to deploying and operating LLM observability with Langfuse in production environments. Two learning paths, 13 modules, all code validated against production stacks.

---

## Why this guide

Your existing observability stack — Prometheus, Grafana, Jaeger — answers the wrong questions for LLM workloads. These tools measure whether your service is healthy. They cannot tell you whether your responses are good.

Langfuse closes that gap. This guide shows you how to integrate it into real infrastructure: existing OTEL pipelines, VictoriaMetrics, Kubernetes, enterprise APM, regulated environments under DORA and the AI Act.

---

## Structure

```
langfuse-guide/
├── 00-introduction.md              ← Start here
│
├── devops/                         ← DevOps Generalist path
│   ├── 01-why-classical-tools-fall-short.md
│   ├── 02-first-trace-in-30-minutes.md
│   ├── 03-end-to-end-rag-tracing.md
│   ├── 04-evaluation-in-production.md
│   ├── 05-langfuse-vs-arize-vs-helicone.md
│   └── 06-langfuse-in-mlops.md
│
└── sre/                            ← SRE / Platform Architect path
    ├── 01-otel-bridge.md
    ├── 02-victoriametrics.md
    ├── 03-kubernetes-self-hosted.md
    ├── 04-dora-ai-act-audit-trail.md
    ├── 05-prompt-injection-detection.md
    ├── 06-apm-vs-langfuse.md
    └── 07-multi-step-agents.md
```

---

## Learning paths

### DevOps Generalist

For engineers familiar with distributed systems observability, new to LLM-specific tooling.

| Module | Topic | Format |
|--------|-------|--------|
| [01](devops/01-why-classical-tools-fall-short.md) | Why Prometheus and Jaeger are not enough | Concept |
| [02](devops/02-first-trace-in-30-minutes.md) | First LLM trace in 30 minutes | Lab |
| [03](devops/03-end-to-end-rag-tracing.md) | End-to-end RAG pipeline tracing | Lab |
| [04](devops/04-evaluation-in-production.md) | Continuous quality evaluation loop | Concept + Lab |
| [05](devops/05-langfuse-vs-arize-vs-helicone.md) | Langfuse vs Arize Phoenix vs Helicone | Comparison |
| [06](devops/06-langfuse-in-mlops.md) | Langfuse in a MLOps stack | Architecture |

**Prerequisites:** Docker, Python 3.10+, basic understanding of distributed tracing (you know what a span is).

**Estimated time:** 5–6 hours of active reading.

---

### SRE / Platform Architect

For engineers who design observability stacks and operate Kubernetes in production.

| Module | Topic | Format |
|--------|-------|--------|
| [01](sre/01-otel-bridge.md) | Langfuse + OpenTelemetry: the integration | Architecture + Lab |
| [02](sre/02-victoriametrics.md) | Langfuse + VictoriaMetrics: LLM metrics in Prometheus | Lab |
| [03](sre/03-kubernetes-self-hosted.md) | Self-hosted on Kubernetes: Helm, HA, retention | Ops |
| [04](sre/04-dora-ai-act-audit-trail.md) | Audit trail: DORA and AI Act requirements | Compliance |
| [05](sre/05-prompt-injection-detection.md) | Detecting prompt injections | Security + Lab |
| [06](sre/06-apm-vs-langfuse.md) | LLM observability vs enterprise APM | Architecture |
| [07](sre/07-multi-step-agents.md) | Multi-step agents in production | Lab |

**Prerequisites:** OpenTelemetry, Kubernetes, Prometheus/VictoriaMetrics, PostgreSQL.

**Estimated time:** 7–9 hours of active reading.

---

## Stack coverage

| Layer | Tools covered |
|-------|--------------|
| Tracing | Langfuse v2, OpenTelemetry SDK, OTel Collector |
| Metrics | Prometheus, VictoriaMetrics, VMAlert, Grafana |
| Serving | OpenAI, Anthropic, vLLM, BentoML |
| Orchestration | LangChain, LlamaIndex, LiteLLM |
| Infrastructure | Kubernetes, Helm, Docker Compose |
| MLOps | MLflow, DVC, W&B |
| Security | DORA, AI Act, GDPR, prompt injection detection |
| APM | Dynatrace, Datadog LLM Observability |

---

## Code conventions

- **Python 3.10+** throughout. No 3.9-incompatible syntax.
- **No hardcoded credentials.** All sensitive values use `os.environ`.
- **Production patterns.** Exception handling, explicit buffer flushes, typed parameters.
- **Validated syntax.** All 36 Python code blocks pass AST parsing.
- **Langfuse SDK v2+.** API calls reference the current stable SDK.
- **VictoriaMetrics.** MetricsQL rules use VMAlert syntax, compatible with Prometheus Alertmanager.

---

## Key fixes and known caveats

| File | Issue | Fix applied |
|------|-------|-------------|
| `sre-07` | `langfuse.langchain` module does not exist | Corrected to `langfuse.callback` |
| `sre-05` | Regex `^pattern` without `re.MULTILINE` only matches string start | Changed to `(?m)^pattern` inline flag |
| `sre-01` | `_incubating` semconv import missing version requirement | Added `pip install "opentelemetry-semantic-conventions>=0.46b0"` |
| `sre-03` | ClickHouse `replicaCount: 2` without noting Keeper dependency | Added note on ClickHouse Keeper (built-in since CH 22.4) |
| `devops-06` | `get_traces()` score filtering note | Added inline comment on SDK v2 behavior |

---

## About

**Samuel Desseaux** is the founder of [Aureonis](https://aureonis.io), a consulting and training firm specializing in observability stack design, platform engineering, SRE, and LLM security. He is an official VictoriaMetrics Training Partner for France, Benelux, and Germany.

Speaker at FOSDEM 2024 and 2026, KubeCon Europe, OSMC, Grafana Meetup Paris, and Devoxx.

Certifications: Prometheus Certified Associate, Zabbix Certified Specialist, CEH, CHFI, CCSP.

Training programs referenced in this guide:

| Code | Topic | Provider |
|------|-------|----------|
| AI-701 | AI Observability in production | Aureonis / Technofutur TIC |
| VM-101/201 | VictoriaMetrics (foundation + advanced) | Aureonis / VictoriaMetrics |
| SEC-301 | Securing LLMs in production | Aureonis / Technofutur TIC |
| SEC-302 | LLM agent security | Aureonis |
| SEC-303 | AI system audit and compliance | Aureonis |

Contact: [aureonis.io](https://aureonis.io)

---

## License

Content: [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)
Code samples: [MIT](https://opensource.org/licenses/MIT)
