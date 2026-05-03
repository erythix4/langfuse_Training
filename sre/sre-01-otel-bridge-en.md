# Langfuse + OpenTelemetry: Bridging LLM Observability with Your Existing Stack

*By Samuel Desseaux -Erythix*

---

## The Problem Nobody Wants to Admit

You have invested in your observability stack. OpenTelemetry for instrumentation, Prometheus for metrics, Grafana for dashboards, maybe VictoriaMetrics as a long-term backend. Your SRE team knows how to read a flamegraph, configure an exemplar, and diagnose an orphaned span.

Then comes the first LLM workload in production.

Your tools don't know what to do with it. The span `POST /v1/chat/completions` lands in Jaeger. P99 latency is in Prometheus. But you have zero visibility into what happened inside: which prompt caused that quality regression? Which retriever call silently failed? Why is this RAG pipeline returning 400ms more than yesterday?

This is exactly the space Langfuse occupies. But the question every platform architect faces is: do you replace your OTEL stack with Langfuse, or make them coexist?

The answer is: coexist — and this is by design.

---

## Scope Clarification: What Each Tool Does

Before talking integration, clarify the perimeters.

**OpenTelemetry** is a general-purpose instrumentation protocol and SDK. It captures distributed spans, system metrics and structured logs. It has no concept of a "prompt", a "token count", or a "hallucination score". To OTEL, an LLM call is an HTTP call with a latency and a status code.

**Langfuse** is a *semantic* observability system for LLM workloads. It understands the structure of an LLM trace: Trace > Generation > Span. It exposes tokens consumed, associated costs, human and automated evaluation scores, user sessions, and input/output content for debugging and offline evaluation.

Both are necessary. OTEL tells you your service is slow. Langfuse tells you *why* your LLM is returning degraded results.

---

## How the Integration Works

Langfuse exposes a native OTLP endpoint since version 2.x. The integration does not require replacing your existing pipeline : it extends it.

```
┌─────────────────────────────────────────────────────────────────┐
│  Application Layer                                              │
│  (Python / Node / Java / Go)                                    │
│                                                                 │
│  openai.chat.completions.create(...)                            │
│  └── gen_ai.* attributes set by auto-instrumentation            │
└────────────────────────┬────────────────────────────────────────┘
                         │ OTLP (gRPC / HTTP)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  OpenTelemetry Collector                                        │
│                                                                 │
│  receiver: otlp                                                 │
│       │                                                         │
│       ├── pipeline: traces/infra ──────────────────────────────┼──▶ Jaeger / Tempo
│       │   processor: filter (exclude gen_ai.* spans)           │
│       │                                                         │
│       └── pipeline: traces/llm ───────────────────────────────┼──▶ Langfuse OTLP
│           processor: none (all LLM spans)                       │    /api/public/otel/v1/traces
└─────────────────────────────────────────────────────────────────┘

                    Shared trace_id propagated to both destinations
                    ──────────────────────────────────────────────
                    Jaeger: infrastructure correlation
                    Langfuse: prompt content, tokens, cost, scores
```

The key design decision is **where to bifurcate**: at the SDK level (dual export) or at the Collector level (pipeline routing). Each has tradeoffs.

---

## Option 1 — Dual Export from the SDK

The simplest configuration. You add Langfuse as a second exporter in your TracerProvider. Both destinations receive the same spans.

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
import base64

# Langfuse OTLP auth uses HTTP Basic (base64-encoded key:secret)
langfuse_auth = base64.b64encode(
    f"{LANGFUSE_PUBLIC_KEY}:{LANGFUSE_SECRET_KEY}".encode()
).decode()

langfuse_exporter = OTLPSpanExporter(
    endpoint="https://cloud.langfuse.com/api/public/otel/v1/traces",
    headers={"Authorization": f"Basic {langfuse_auth}"}
)

# Your existing exporter — replace with your actual destination
infra_exporter = OTLPSpanExporter(
    endpoint="http://jaeger-collector:4318/v1/traces"
)

provider = TracerProvider()
provider.add_span_processor(BatchSpanProcessor(langfuse_exporter))
provider.add_span_processor(BatchSpanProcessor(infra_exporter))

from opentelemetry import trace
trace.set_tracer_provider(provider)
```

**When to use this:** small teams, simple architectures, development environments or when you cannot modify the Collector configuration.

**Drawbacks:** changing the routing requires a code change and redeployment. Both exporters are tightly coupled to the application.

---

## Option 2 : Routing via OpenTelemetry Collector

The production-recommended approach. The Collector receives all spans and routes them based on filtering rules. Applications export to a single destination (the Collector); routing is infrastructure configuration, not application code.

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"

processors:
  batch:
    timeout: 5s
    send_batch_size: 512

  # Filter processor to exclude non-LLM spans from Langfuse
  # (send everything to infra backend, only LLM spans to Langfuse)
  filter/llm_only:
    error_mode: ignore
    traces:
      span:
        - 'attributes["gen_ai.system"] == nil'

exporters:
  otlp/jaeger:
    endpoint: "jaeger-collector:4317"
    tls:
      insecure: true

  otlp/langfuse:
    endpoint: "https://cloud.langfuse.com/api/public/otel/v1/traces"
    headers:
      Authorization: "Basic <base64(public_key:secret_key)>"
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s

service:
  pipelines:
    # All spans go to infra backend
    traces/infra:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger]

    # Only LLM spans go to Langfuse
    traces/llm:
      receivers: [otlp]
      processors: [filter/llm_only, batch]
      exporters: [otlp/langfuse]
```

This architecture is preferred for several reasons:
- Routing changes require no application redeployment
- The Collector handles retries and buffering if Langfuse is temporarily unavailable
- You can add additional destinations (S3 archiving, SIEM) without touching the application
- Sampling rules can be applied at the Collector level, different for infra and LLM spans

---

## Option 3 : Using Langfuse as the Primary OTEL Endpoint

For teams starting fresh without an existing OTEL backend, Langfuse can serve as the primary trace sink. You can always add a secondary export to Jaeger or Tempo later.

```python
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(
            endpoint="http://localhost:3000/api/public/otel/v1/traces",
            headers={"Authorization": f"Basic {langfuse_auth}"}
        )
    )
)
```

---

## GenAI Semantic Conventions

OpenTelemetry published the GenAI Semantic Conventions in 2024, now stable for LLM spans. These are the attributes Langfuse understands natively when receiving OTEL spans.

| OTEL Attribute | Meaning | Example |
|---|---|---|
| `gen_ai.system` | LLM provider | `openai`, `anthropic`, `mistral` |
| `gen_ai.request.model` | Requested model | `gpt-4o` |
| `gen_ai.response.model` | Actual model used | `gpt-4o-2024-11-20` |
| `gen_ai.usage.input_tokens` | Prompt token count | `512` |
| `gen_ai.usage.output_tokens` | Completion token count | `128` |
| `gen_ai.operation.name` | Operation type | `chat`, `embeddings`, `completions` |
| `gen_ai.request.temperature` | Sampling temperature | `0.1` |
| `gen_ai.request.max_tokens` | Max tokens requested | `1024` |

If you use an auto-instrumented library (`opentelemetry-instrumentation-openai`, LangChain's OTEL callback, LlamaIndex instrumentation), these attributes are set automatically. Langfuse maps them to its internal data model on ingestion.

If you are instrumenting manually, set these attributes on your spans:

```python
from opentelemetry import trace
# Requires: pip install "opentelemetry-semantic-conventions>=0.46b0"
from opentelemetry.semconv._incubating.attributes import gen_ai_attributes

tracer = trace.get_tracer("my-llm-service")

with tracer.start_as_current_span("llm-generation") as span:
    span.set_attribute(gen_ai_attributes.GEN_AI_SYSTEM, "openai")
    span.set_attribute(gen_ai_attributes.GEN_AI_REQUEST_MODEL, "gpt-4o-mini")

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages
    )

    span.set_attribute(
        gen_ai_attributes.GEN_AI_USAGE_INPUT_TOKENS,
        response.usage.prompt_tokens
    )
    span.set_attribute(
        gen_ai_attributes.GEN_AI_USAGE_OUTPUT_TOKENS,
        response.usage.completion_tokens
    )
    span.set_attribute(
        gen_ai_attributes.GEN_AI_RESPONSE_MODEL,
        response.model
    )
```

---

## Concrete Benefits of the Integration

### Distributed Trace Correlation

The OTEL `trace_id` is propagated into Langfuse. If a user request produces a slow span in Jaeger and a degraded LLM generation in Langfuse, you can navigate between the two using the shared identifier. This is the missing link between infrastructure observability and LLM semantic observability.

```python
# Propagate trace context into Langfuse metadata
from opentelemetry import trace

current_span = trace.get_current_span()
context = current_span.get_span_context()

langfuse_trace = langfuse.trace(
    name="rag-pipeline",
    metadata={
        "otel_trace_id": format(context.trace_id, "032x"),
        "otel_span_id": format(context.span_id, "016x"),
    }
)
```

With this correlation, your Jaeger link inside a Grafana annotation can point directly to the corresponding Langfuse trace. Your on-call engineer sees the infrastructure anomaly in one tool and the LLM content details in the other, via a single click.

### Unified Alerting

Your existing Alertmanager cannot alert on "hallucination rate exceeds 15%". But it can alert on metrics exposed from the Langfuse API via a custom exporter (see module SRE-02). The combination gives you:

- Prometheus/VMAlert: latency, error rates, resource saturation on adjacent services
- Langfuse API + custom alerting: quality drift, cost per session, negative evaluation rate
- Both routing through the same Alertmanager, PagerDuty, or Slack integration

### RAG Pipeline Debugging

A RAG pipeline produces several nested spans: retrieval, reranking, prompt assembly, generation. With OTEL alone, you see total duration. With Langfuse, you see each step, the retrieved documents, the final prompt, and the raw response. When quality drops, you can compare traces before and after the regression and pinpoint the step that changed.

---

## Production Considerations

### Sampling Strategy

Your OTEL infrastructure likely uses probabilistic sampling at the head. LLM traces often need to be captured at 100% for offline evaluation — you cannot evaluate a generation you did not store. Configure differentiated sampling rules:

```yaml
# Collector-level tail sampling
processors:
  tail_sampling:
    decision_wait: 10s
    policies:
      # Always sample LLM spans
      - name: llm-spans-always
        type: string_attribute
        string_attribute:
          key: gen_ai.system
          values: [openai, anthropic, mistral, cohere]
          enabled_regex_matching: false

      # Sample infra spans at 10%
      - name: infra-probabilistic
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
```

### PII in Span Attributes

LLM inputs and outputs may contain personal data. Langfuse stores raw span attributes. Implement a redaction strategy either at the instrumentation level (scrub inputs before creating the span) or at the Collector level using the `redaction` processor. For regulated industries, document your redaction policy explicitly.

### Storage Cost at Scale

An LLM span with full input/output can be several kilobytes. At 1M generations/month, this is significant storage. In Langfuse Cloud, plan your retention settings. In self-hosted Langfuse with PostgreSQL, monitor table growth. If you exceed ~1M traces/month, consider enabling ClickHouse as the analytics backend (covered in module SRE-03).

### Latency Impact

The `BatchSpanProcessor` is asynchronous and non-blocking. On latency-sensitive workloads, validate that the buffer size and flush interval are tuned to your throughput:

```python
from opentelemetry.sdk.trace.export import BatchSpanProcessor

processor = BatchSpanProcessor(
    exporter,
    max_queue_size=2048,
    max_export_batch_size=512,
    schedule_delay_millis=5000,    # Flush every 5 seconds
    export_timeout_millis=30000    # 30s timeout per export batch
)
```

---

## Verification

After configuring the integration, verify it is working end-to-end:

```python
# test_otel_langfuse.py
import os
import time
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter

provider = TracerProvider()
provider.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(
            endpoint="http://localhost:3000/api/public/otel/v1/traces",
            headers={"Authorization": f"Basic {os.environ['LANGFUSE_AUTH']}"}
        )
    )
)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("integration-test")

with tracer.start_as_current_span("test-llm-span") as span:
    span.set_attribute("gen_ai.system", "openai")
    span.set_attribute("gen_ai.request.model", "gpt-4o-mini")
    span.set_attribute("gen_ai.usage.input_tokens", 42)
    span.set_attribute("gen_ai.usage.output_tokens", 17)
    time.sleep(0.1)

# Force flush before script exit
provider.force_flush()
print("Span exported — check Langfuse UI > Traces")
```

The trace should appear in Langfuse within a few seconds. If it does not, check the Collector logs or the Langfuse application logs for export errors.

---

## What Comes Next

The OTEL/Langfuse integration is the plumbing. Once it is in place, the questions that become addressable include:

- How do you expose Langfuse's LLM metrics into VictoriaMetrics for unified alerting? (Module SRE-02)
- How do you automatically evaluate generation quality and feed the results back into the training pipeline? (Module DevOps-04)
- How do you deploy Langfuse self-hosted on Kubernetes for data sovereignty? (Module SRE-03)

---


