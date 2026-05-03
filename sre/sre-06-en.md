# LLM Observability vs Enterprise APM: Where Dynatrace and Datadog Stop

*By Samuel Desseaux — Aureonis*

---

## A Market That Oversells Its Capabilities

Enterprise APM vendors quickly added "LLM Observability" to their pitch decks. Dynatrace talks about Davis AI for GenAI workloads. Datadog launched LLM Observability in 2024. New Relic followed. These announcements create confusion: if your APM "supports LLMs", do you still need a dedicated tool like Langfuse?

The short answer is yes. This module explains why, without dogmatism, and proposes a clear hybrid architecture based on real-world constraints.

---

## What Enterprise APMs Do Well on LLM Workloads

Start with an honest assessment.

**Auto-instrumentation.** Dynatrace and Datadog automatically capture outbound calls to OpenAI, Anthropic, and Azure OpenAI APIs via their agent. You get latency, error rates, and call counts — without touching your code.

**Infrastructure correlation.** The core strength of mature APMs is correlation: you see that a slow LLM call is preceded by CPU saturation on the pod, or that the timeout originates from a network issue. This infra-to-application correlation is difficult to reproduce with a standalone tool.

**Unified alerting.** Your LLM alerts can share the same channels, escalation policies, and runbooks as your application alerts. For an SRE team managing dozens of services, reducing the number of tools under monitoring has real operational value.

**Enterprise security and compliance.** SOC2, GDPR, SSO, advanced RBAC — enterprise APMs have solved these problems for years. Self-hosted Langfuse solves them too, but with more configuration effort.

---

## What Enterprise APMs Do Not Do, or Do Poorly

**LLM exchange content.** Dynatrace and Datadog store metrics and spans, not the content of prompts and responses. You know the call took 800ms, not what was asked or what was answered. This is the fundamental gap.

Without access to content, you cannot:
- Debug a specific response that caused a problem
- Compare prompts before and after a change
- Build an evaluation dataset from production traces
- Analyze question patterns to improve your system

**Quality evaluation.** Neither Dynatrace nor Datadog offers an LLM evaluation system (LLM-as-judge, human annotation, automated relevance or consistency scoring). They measure technical performance, not response quality.

**Structured LLM pipelines.** A five-step RAG pipeline appears in an APM as a single HTTP span. The Trace > Span > Generation hierarchy, with the detail of each step, does not exist in these tools.

**Per-generation cost.** Enterprise APMs do not calculate LLM cost at the individual trace level. You have aggregated cost metrics via cloud billing dashboards, not per-request cost tracking.

---

## Capability Matrix

```
                     Dynatrace   Datadog   Langfuse
                                 LLM Obs   (self-hosted)
                     ─────────   ───────   ─────────────
Infra correlation      ████████   ████████   ──
Auto-instrumentation   ████████   ████████   ██████
Prompt/response store  ──         ██████     ████████
Per-trace cost         ──         ████       ████████
LLM span hierarchy     ──         ████       ████████
Quality evaluation     ──         ──         ████████
Human annotation       ──         ──         ████████
Production datasets    ──         ──         ████████
Self-hosted option     ██████     ──         ████████
Open-source            ──         ──         ████████
```

| Capability | Dynatrace | Datadog LLM Obs | Langfuse |
|---|---|---|---|
| LLM call latency | Yes | Yes | Yes |
| HTTP error rate | Yes | Yes | Yes |
| Infrastructure correlation | Strong | Strong | None |
| Prompt / response storage | No | Yes (opt-in) | Yes |
| Per-trace cost | No | Partial | Yes |
| LLM span hierarchy | No | Yes (partial) | Yes |
| Quality evaluation | No | No | Yes |
| Human annotation | No | No | Yes |
| Production datasets | No | No | Yes |
| Prompt injection detection | Limited | Limited | Via scoring |
| Self-hosted | Yes (agent) | No | Yes |
| Open-source | No | No | Yes (MIT) |
| Typical cost | High | High | Low / zero |

---

## Datadog LLM Observability — A Closer Look

Datadog has invested more than other APMs in LLM capabilities. Their LLM Observability product (GA since late 2024) provides:

- Prompt and response capture (with optional redaction)
- Multi-step trace visualization (LangChain, LlamaIndex pipelines)
- Basic evaluations (toxicity, relevance via built-in evaluators)
- Per-trace cost estimation

It is the most serious competitor to Langfuse in environments that already run Datadog. The remaining limitations:

- No self-hosting: your LLM data transits through Datadog Cloud — a concern for regulated industries
- Significant additional cost (LLM Observability is a separately priced add-on)
- Less rich evaluation ecosystem compared to Langfuse
- Strong platform lock-in

---

## Three Recommended Hybrid Architectures

### Configuration A — Existing APM + Langfuse as complement

The most common scenario. You keep your APM investment and add Langfuse for the content layer.

```
Dynatrace / Datadog
    |
    +-- Infrastructure and application monitoring (existing, retained)
    +-- Unified alerting (SLOs, on-call rotation)
    +-- Correlation with LLM spans via shared trace_id
    |
Langfuse self-hosted
    |
    +-- LLM traces with content (prompts, responses)
    +-- Evaluation and scoring
    +-- Datasets for the retraining feedback loop
    +-- Regulatory audit trail (DORA, AI Act)
```

The shared OTEL `trace_id` is the bridge: you open the incident in your APM, navigate to the corresponding Langfuse trace via the shared ID.

```python
# Inject APM trace context into Langfuse metadata
from opentelemetry import trace as otel_trace

current_span = otel_trace.get_current_span()
context = current_span.get_span_context()

langfuse_trace = langfuse.trace(
    name="llm-request",
    metadata={
        "apm_trace_id": format(context.trace_id, "032x"),
        "apm_span_id": format(context.span_id, "016x"),
        # Dynatrace-specific
        "dt_trace_id": os.environ.get("DT_TRACE_ID"),
    }
)
```

### Configuration B — Full open-source stack (no enterprise APM)

For teams without an enterprise APM contract, or those building a 100% open-source stack:

```
OpenTelemetry Collector
    |
    +-- Jaeger / Tempo (distributed traces)
    +-- VictoriaMetrics (metrics + alerting)
    +-- Langfuse (LLM traces + evaluation)
```

Covered in detail in modules SRE-01 and SRE-02.

### Configuration C — Datadog + Langfuse for datasets

Even if you use Datadog LLM Observability for real-time monitoring, retain Langfuse for evaluation dataset management and the continuous improvement loop. Datadog does not provide an annotation workflow anywhere near as complete.

---

## Making the Case to a Technical Decision-Maker

Arguments that work:

**Separation of concerns.** Your APM manages the operational health of the service. Langfuse manages the quality of the outputs. These are distinct concerns that deserve distinct tools.

**Return on investment.** Continuous production evaluation surfaces quality regressions before they become user-facing incidents. An LLM system hallucinating at scale costs more in customer support and trust erosion than a Langfuse license.

**Data sovereignty.** For regulated sectors, self-hosted Langfuse keeps your prompts and responses in your infrastructure. Neither Dynatrace Cloud nor Datadog offers this option.

**Open-source and no lock-in.** MIT-licensed Langfuse can be replaced or forked if necessary. That is not true of your enterprise APM contract.

**Complementary, not competitive.** Framing Langfuse as an alternative to your APM loses the argument. Frame it as a specialist layer for a problem your APM was not designed to solve.

---

*Samuel Desseaux is the founder of Aureonis, an observability stack specialist (OpenTelemetry, VictoriaMetrics, Dynatrace, Elastic) and LLM security practitioner. Speaker at FOSDEM 2026 and KubeCon Europe.*
