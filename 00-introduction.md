# Langfuse in Production: A Complete Observability Guide

*By Samuel Desseaux - Erythix*

---

## Why This Guide Exists

LLM observability is a discipline that does not yet have established standards. Teams moving LLM workloads to production quickly discover that their existing stack - Prometheus, Grafana, Jaeger, Datadog - answers the wrong questions. Those tools were designed for deterministic systems. LLMs are not deterministic. A request that returns HTTP 200 in 600ms may still be a complete failure from the user's perspective.

Langfuse is one of the most mature open-source tools addressing this gap. It stores prompts and responses, structures execution traces, enables human and automated evaluation, and integrates natively with the OpenTelemetry ecosystem. It is self-hostable, MIT-licensed, and backed by an active community.

This guide was written by practitioners for practitioners. Every code sample has been validated in production environments. Every architectural recommendation reflects real constraints - regulated industries, multi-tenant Kubernetes clusters, existing OTEL stacks, compliance requirements under DORA and the AI Act.

---

## How to Use This Guide

The guide is structured around two learning paths, designed for different starting points.

**The DevOps Generalist path** is for engineers who understand distributed systems observability but are new to LLM-specific tooling. It starts with the conceptual gap between infrastructure monitoring and LLM semantic observability, moves through hands-on instrumentation, and ends with MLOps integration patterns.

**The SRE / Platform Architect path** assumes fluency with OpenTelemetry, Kubernetes, and production observability stacks. It focuses on integrating Langfuse into existing infrastructure - OTEL pipelines, VictoriaMetrics, self-hosted Kubernetes deployments, and enterprise APM environments.

The two paths share no prerequisite between them. You can read modules from both paths in any order, or follow one path linearly. Each module is designed to be standalone enough to be useful in isolation.

---

## What This Guide Covers

### DevOps Generalist Path

**Module 01 - Why classical tools fall short.** The conceptual foundation. What Prometheus and Jaeger measure, what they cannot measure, and why the gap matters operationally. Three production incidents that existing tools would not detect.

**Module 02 - First steps with Langfuse.** A hands-on lab. Deploy Langfuse locally via Docker Compose, instrument a Python application, and read your first LLM trace in under 30 minutes.

**Module 03 - End-to-end RAG pipeline tracing.** Instrument every step of a RAG pipeline - embedding, vector search, context assembly, generation, post-processing - and use traces to diagnose quality degradation.

**Module 04 - LLM evaluation in production.** Build a continuous quality feedback loop using rule-based scoring, LLM-as-judge evaluation, and human annotation. Monitor score drift in Grafana.

**Module 05 - Langfuse vs Arize Phoenix vs Helicone.** An honest feature comparison across deployment models, OTEL support, evaluation capabilities, cost, and data sovereignty requirements.

**Module 06 - Langfuse in a MLOps stack.** Where Langfuse fits alongside MLflow, DVC, and BentoML. What it does, what it does not do, and how to wire the production feedback loop back into the training pipeline.

### SRE / Platform Architect Path

**Module 01 - Langfuse + OpenTelemetry.** Bridge LLM observability with your existing OTEL stack. Dual SDK export, Collector routing, GenAI Semantic Conventions, and trace ID correlation between infrastructure and LLM spans.

**Module 02 - Langfuse + VictoriaMetrics.** Build a custom Prometheus exporter for the Langfuse API, configure VMScrapeConfig, write VMAlert rules for latency, cost spikes, and error rates, and extend your existing Grafana dashboards.

**Module 03 - Self-hosted on Kubernetes.** Production Helm deployment with external PostgreSQL, ClickHouse for high volume, HPA autoscaling, pod anti-affinity, network policies, and data retention strategy.

**Module 04 - Audit trail for DORA and the AI Act.** Structure Langfuse traces to satisfy auditability requirements under DORA Article 12 and AI Act Article 12. PII redaction patterns, retention policy, and SIEM integration.

**Module 05 - Detecting prompt injections.** Instrument a detection layer for direct injection, indirect injection via RAG, jailbreaks, and system prompt exfiltration. VMAlert rules correlated with application spans.

**Module 06 - LLM observability vs enterprise APM.** An honest capability matrix comparing Langfuse against Dynatrace, Datadog LLM Observability, and New Relic. Three recommended hybrid architectures.

**Module 07 - Multi-step agents in production.** Instrument ReAct and Function Calling agents with LangChain callbacks. Detect loops, tool hallucinations, and cost escalation. Define agentic SLOs.

---

## Prerequisites by Path

### DevOps Generalist
- Familiarity with Docker and Docker Compose
- Basic Python (pip, virtual environments, environment variables)
- Conceptual understanding of distributed tracing (you know what a span is)
- Access to an OpenAI, Anthropic, or Mistral API key for the lab modules

### SRE / Platform Architect
- Working knowledge of OpenTelemetry (TracerProvider, exporters, collectors)
- Kubernetes operations (Helm, kubectl, RBAC, network policies)
- Prometheus / VictoriaMetrics (PromQL, alerting rules, scrape configuration)
- Familiarity with production database operations (PostgreSQL at minimum)

---

## Conventions Used in This Guide

Code samples use Python 3.10+ and are production-grade: environment variables for credentials, exception handling, and explicit flushing of async buffers. No sample hardcodes API keys.

Architecture diagrams use plain ASCII. They are intentionally simple - the goal is to convey structure, not to be visually exhaustive.

Where a module builds on another, the prerequisite is stated explicitly at the top of the module.

All Langfuse API calls reference the v2.x SDK unless otherwise noted. The self-hosted deployment examples target Langfuse 2.x with the official Helm chart.

---

## A Note on Scope

This guide focuses on **Langfuse as an observability and evaluation tool**. It does not cover prompt engineering methodology, LLM fine-tuning, or model selection. It does not cover Langfuse's prompt management features (prompt versioning, prompt playground), which are useful but outside the operational scope of this guide.

The security module (SRE-05) covers detection patterns as part of the observability architecture. It is not a substitute for a dedicated AI security assessment.

---

## About the Author

Samuel Desseaux is the founder of Erythix, a consulting and training firm specialized in observability stack design (OpenTelemetry, VictoriaMetrics, Grafana, Prometheus, Elastic, Dynatrace, Zabbix), platform engineering, SRE, and LLM security. He is an official VictoriaMetrics Training Partner for France, Benelux, and Germany. He has spoken at FOSDEM 2024 and 2026, KubeCon Europe, OSMC, Grafana Meetup Paris, and Devoxx. He holds the Prometheus Certified Associate and Zabbix Certified Specialist certifications, and is preparing for the CKA.

Erythix delivers training and consulting across France and Benelux, with an anchor mission at Banque centrale du Luxembourg. Training programs referenced in this guide (SEC-301/302/303 for LLM security, VM-101/201 for VictoriaMetrics, AI-701 for AI observability) are available through Erythix and its training partners Technofutur TIC, SUSE, and Digicomp.

Contact: [aureonis.io](https://aureonis.io)

---

*This guide is updated continuously. Modules marked "Published" have been reviewed and tested in production. Modules in active development are indicated in the guide index.*
