# Langfuse + VictoriaMetrics: LLM Metrics in Your Native Prometheus Stack

*By Samuel Desseaux — Aureonis*

---

## The Operational Gap

Most articles on LLM observability treat Langfuse as a standalone tool: you browse traces in its UI, consult its built-in dashboards, export CSVs for quarterly reviews. That workflow is sufficient for a proof of concept. It is not sufficient for production.

In production, your SLOs live in Prometheus. Your alerts fire through Alertmanager. Your operational dashboards run in Grafana, fed by VictoriaMetrics. Your on-call engineer is not going to open a third tool at 3am to understand why the LLM pipeline is degraded.

The goal of this module is to show you how to pull Langfuse's LLM metrics into VictoriaMetrics, build Prometheus-compatible alerts, and extend your existing Grafana dashboards — without proprietary middleware, without vendor lock-in, and without touching your application code.

---

## What Langfuse Exposes

Langfuse does not push metrics to Prometheus natively. It exposes two exploitable surfaces:

**A REST API** with aggregation endpoints: metrics by project, by model, by user, windowed in time. Average latency, tokens consumed, estimated cost, generation count, evaluation scores. Full API reference at `https://api.reference.langfuse.com`.

**An OTLP endpoint** for receiving traces. LLM spans arrive in Langfuse via OTEL — but the reverse export (Langfuse to your metrics stack) is not native.

The solution is a **custom exporter**: a lightweight process that polls the Langfuse API at a regular interval and exposes metrics in Prometheus format. VictoriaMetrics scrapes it like any other target.

---

## Target Architecture

```
┌───────────────────────────────────────────────────────────────┐
│  Langfuse (self-hosted or Cloud)                              │
│  REST API: /api/public/metrics/daily                          │
│            /api/public/scores                                 │
└──────────────────────┬────────────────────────────────────────┘
                       │  HTTP polling — every 60s
                       ▼
┌───────────────────────────────────────────────────────────────┐
│  langfuse-exporter  (Python, Deployment in monitoring ns)     │
│  :9090/metrics  →  Prometheus text format                     │
│                                                               │
│  langfuse_generations_total{model, status}                    │
│  langfuse_generation_latency_p99_ms{model}                    │
│  langfuse_tokens_input_total{model}                           │
│  langfuse_cost_usd_window{model}                              │
│  langfuse_score_average{score_name}                           │
└──────────────────────┬────────────────────────────────────────┘
                       │  VMScrapeConfig / scrape
                       ▼
┌───────────────────────────────────────────────────────────────┐
│  VictoriaMetrics                                              │
├────────────────────────┬──────────────────────────────────────┤
│  VMAlert               │  Grafana                             │
│  · LLMHighLatencyP99   │  · LLM Operations Dashboard          │
│  · LLMCostSpike        │  · LLM Quality Dashboard             │
│  · LLMQualityDrift     │  · Cost Dashboard                    │
│       │                │  Variable: $model                    │
│       ▼                └──────────────────────────────────────┘
│  Alertmanager                                                 │
│  · PagerDuty / Slack / OpsGenie                               │
└───────────────────────────────────────────────────────────────┘
```

This design is deliberately conservative: no new component on the critical request path, the exporter is stateless and restartable without data loss, and VictoriaMetrics handles cardinality without special configuration.

---

## The Exporter

A complete, production-ready implementation. Stateless, under 50MB RAM, deployable as a sidecar or standalone Kubernetes deployment.

```python
# langfuse_exporter.py
import os
import time
import logging
import requests
from datetime import datetime, timedelta, timezone
from prometheus_client import (
    start_http_server, Gauge, Counter, Histogram, Info
)

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger(__name__)

LANGFUSE_BASE_URL = os.environ["LANGFUSE_BASE_URL"]
LANGFUSE_PUBLIC_KEY = os.environ["LANGFUSE_PUBLIC_KEY"]
LANGFUSE_SECRET_KEY = os.environ["LANGFUSE_SECRET_KEY"]
LANGFUSE_PROJECT_ID = os.environ["LANGFUSE_PROJECT_ID"]
SCRAPE_INTERVAL = int(os.environ.get("SCRAPE_INTERVAL", "60"))
METRICS_PORT = int(os.environ.get("METRICS_PORT", "9090"))

session = requests.Session()
session.auth = (LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY)
session.headers.update({"User-Agent": "langfuse-prometheus-exporter/1.0"})

# ---- Metric definitions ----

exporter_info = Info("langfuse_exporter", "Langfuse Prometheus exporter metadata")
exporter_info.info({"version": "1.0.0", "project_id": LANGFUSE_PROJECT_ID})

generation_total = Counter(
    "langfuse_generations_total",
    "Total number of LLM generations",
    ["project", "model", "status"]
)
generation_latency_p50 = Gauge(
    "langfuse_generation_latency_p50_ms",
    "Median generation latency (ms)",
    ["project", "model"]
)
generation_latency_p95 = Gauge(
    "langfuse_generation_latency_p95_ms",
    "P95 generation latency (ms)",
    ["project", "model"]
)
generation_latency_p99 = Gauge(
    "langfuse_generation_latency_p99_ms",
    "P99 generation latency (ms)",
    ["project", "model"]
)
tokens_input_total = Counter(
    "langfuse_tokens_input_total",
    "Total input tokens consumed",
    ["project", "model"]
)
tokens_output_total = Counter(
    "langfuse_tokens_output_total",
    "Total output tokens produced",
    ["project", "model"]
)
cost_usd = Gauge(
    "langfuse_cost_usd_window",
    "Estimated cost in USD for the current scrape window",
    ["project", "model"]
)
score_average = Gauge(
    "langfuse_score_average",
    "Average evaluation score",
    ["project", "score_name"]
)
score_below_threshold_total = Counter(
    "langfuse_score_below_threshold_total",
    "Number of evaluations below quality threshold",
    ["project", "score_name"]
)
scrape_errors_total = Counter(
    "langfuse_exporter_scrape_errors_total",
    "Number of failed scrape attempts",
    ["error_type"]
)
scrape_duration_seconds = Gauge(
    "langfuse_exporter_scrape_duration_seconds",
    "Duration of the last successful scrape"
)


def fetch_daily_metrics(window_minutes: int = 5) -> dict:
    now = datetime.now(timezone.utc)
    from_ts = (now - timedelta(minutes=window_minutes)).isoformat()
    to_ts = now.isoformat()

    resp = session.get(
        f"{LANGFUSE_BASE_URL}/api/public/metrics/daily",
        params={
            "projectId": LANGFUSE_PROJECT_ID,
            "fromTimestamp": from_ts,
            "toTimestamp": to_ts,
        },
        timeout=15
    )
    resp.raise_for_status()
    return resp.json()


def fetch_scores(score_name: str = None) -> list:
    params = {"projectId": LANGFUSE_PROJECT_ID, "limit": 500}
    if score_name:
        params["name"] = score_name

    resp = session.get(
        f"{LANGFUSE_BASE_URL}/api/public/scores",
        params=params,
        timeout=15
    )
    resp.raise_for_status()
    return resp.json().get("data", [])


def collect(quality_threshold: float = 0.7):
    start = time.monotonic()
    try:
        data = fetch_daily_metrics()

        for entry in data.get("data", []):
            model = entry.get("model", "unknown")
            project = LANGFUSE_PROJECT_ID
            labels = {"project": project, "model": model}

            # Generation counts
            generation_total.labels(
                project=project, model=model, status="success"
            ).inc(entry.get("countSuccessfulObservations", 0))
            generation_total.labels(
                project=project, model=model, status="error"
            ).inc(entry.get("countErroredObservations", 0))

            # Token consumption
            tokens_input_total.labels(**labels).inc(entry.get("inputUsage", 0))
            tokens_output_total.labels(**labels).inc(entry.get("outputUsage", 0))

            # Cost
            cost_usd.labels(**labels).set(entry.get("totalCost", 0.0))

            # Latency percentiles (if available in your Langfuse version)
            if "latencyP50" in entry:
                generation_latency_p50.labels(**labels).set(entry["latencyP50"])
            if "latencyP95" in entry:
                generation_latency_p95.labels(**labels).set(entry["latencyP95"])
            if "latencyP99" in entry:
                generation_latency_p99.labels(**labels).set(entry["latencyP99"])

        # Evaluation scores
        scores = fetch_scores()
        score_buckets: dict[tuple, list] = {}
        for s in scores:
            name = s.get("name", "unknown")
            value = s.get("value")
            if value is not None:
                key = (LANGFUSE_PROJECT_ID, name)
                score_buckets.setdefault(key, []).append(float(value))

        for (project, name), values in score_buckets.items():
            avg = sum(values) / len(values)
            score_average.labels(project=project, score_name=name).set(avg)
            below = sum(1 for v in values if v < quality_threshold)
            score_below_threshold_total.labels(
                project=project, score_name=name
            ).inc(below)

        scrape_duration_seconds.set(time.monotonic() - start)
        log.info(f"Scrape completed in {time.monotonic() - start:.2f}s")

    except requests.exceptions.Timeout:
        scrape_errors_total.labels(error_type="timeout").inc()
        log.error("Scrape timed out")
    except requests.exceptions.HTTPError as e:
        scrape_errors_total.labels(error_type="http_error").inc()
        log.error(f"HTTP error: {e}")
    except Exception as e:
        scrape_errors_total.labels(error_type="unexpected").inc()
        log.exception(f"Unexpected error: {e}")


if __name__ == "__main__":
    start_http_server(METRICS_PORT)
    log.info(f"Exporter running on :{METRICS_PORT}/metrics")

    while True:
        collect()
        time.sleep(SCRAPE_INTERVAL)
```

### Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY langfuse_exporter.py .
EXPOSE 9090
CMD ["python", "langfuse_exporter.py"]
```

```txt
# requirements.txt
prometheus-client==0.20.0
requests==2.32.3
```

---

## Kubernetes Deployment

```yaml
# langfuse-exporter-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: langfuse-exporter
  namespace: monitoring
  labels:
    app: langfuse-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: langfuse-exporter
  template:
    metadata:
      labels:
        app: langfuse-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      containers:
        - name: exporter
          image: your-registry/langfuse-exporter:1.0.0
          ports:
            - containerPort: 9090
          env:
            - name: LANGFUSE_BASE_URL
              value: "https://langfuse.internal.yourcompany.com"
            - name: LANGFUSE_PROJECT_ID
              value: "your-project-id"
            - name: LANGFUSE_PUBLIC_KEY
              valueFrom:
                secretKeyRef:
                  name: langfuse-exporter-creds
                  key: public-key
            - name: LANGFUSE_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: langfuse-exporter-creds
                  key: secret-key
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          livenessProbe:
            httpGet:
              path: /metrics
              port: 9090
            initialDelaySeconds: 30
            periodSeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: langfuse-exporter
  namespace: monitoring
spec:
  selector:
    app: langfuse-exporter
  ports:
    - port: 9090
      targetPort: 9090
```

---

## VictoriaMetrics Scrape Configuration

If you use the VictoriaMetrics Operator:

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMScrapeConfig
metadata:
  name: langfuse-exporter
  namespace: monitoring
spec:
  staticConfigs:
    - labels:
        job: langfuse-exporter
        env: production
        team: platform
      targets:
        - langfuse-exporter.monitoring.svc.cluster.local:9090
  scrapeInterval: 60s
  metricsPath: /metrics
  relabelings:
    - sourceLabels: [__address__]
      targetLabel: instance
      replacement: langfuse-exporter
```

If you use `vmagent` with a static config file:

```yaml
# vmagent-config.yml (scrape_configs section)
scrape_configs:
  - job_name: langfuse-exporter
    scrape_interval: 60s
    static_configs:
      - targets:
          - langfuse-exporter.monitoring.svc.cluster.local:9090
        labels:
          env: production
```

---

## VMAlert Rules

VMAlert uses standard Prometheus alerting syntax, extended with MetricsQL when needed. The following rules are production-tested.

```yaml
groups:
  - name: llm.production
    interval: 1m
    rules:

      # P99 latency exceeds SLO
      - alert: LLMHighLatencyP99
        expr: |
          langfuse_generation_latency_p99_ms{env="production"} > 3000
        for: 5m
        labels:
          severity: warning
          team: platform
          runbook: "https://wiki.internal/runbooks/llm-latency"
        annotations:
          summary: "LLM P99 latency > 3s ({{ $labels.model }})"
          description: >
            Model {{ $labels.model }} in project {{ $labels.project }}
            has P99 latency of {{ $value }}ms. SLO threshold: 3000ms.

      # P99 latency critical
      - alert: LLMCriticalLatencyP99
        expr: |
          langfuse_generation_latency_p99_ms{env="production"} > 8000
        for: 2m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "LLM P99 latency critical: {{ $value }}ms ({{ $labels.model }})"

      # Hourly cost spike
      - alert: LLMCostSpike
        expr: |
          increase(langfuse_cost_usd_window{env="production"}[1h]) > 50
        for: 0m
        labels:
          severity: critical
          team: finance-engineering
        annotations:
          summary: "LLM cost spike: +${{ $value | humanize }} in 1h ({{ $labels.project }})"
          description: >
            Project {{ $labels.project }} has accrued ${{ $value }} in the last hour.
            Investigate for prompt loops, unusually long contexts, or unexpected traffic.

      # High error rate
      - alert: LLMHighErrorRate
        expr: |
          (
            rate(langfuse_generations_total{status="error"}[5m])
            /
            rate(langfuse_generations_total[5m])
          ) > 0.05
        for: 3m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "LLM error rate > 5% ({{ $labels.model }})"
          description: >
            {{ $labels.model }} has a generation error rate of
            {{ $value | humanizePercentage }} over the last 5 minutes.

      # Quality score drift
      - alert: LLMQualityScoreDrift
        expr: |
          langfuse_score_average{score_name="judge-faithfulness"} < 0.65
        for: 15m
        labels:
          severity: warning
          team: ml
        annotations:
          summary: "LLM faithfulness score below threshold ({{ $value | humanize }})"
          description: >
            The judge-faithfulness score has been below 0.65 for 15 minutes.
            Review recent trace content for hallucination patterns.

      # Exporter health
      - alert: LangfuseExporterDown
        expr: |
          up{job="langfuse-exporter"} == 0
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "Langfuse Prometheus exporter is unreachable"
```

---

## Grafana Dashboard

With VictoriaMetrics as the datasource, MetricsQL gives you aggregation capabilities beyond standard PromQL. Recommended panel structure:

### Executive row (top of dashboard)

- **Total generations / 24h**: `increase(langfuse_generations_total{env="production"}[24h])`
- **Estimated cost today**: `sum(langfuse_cost_usd_window) by (project)`
- **Current error rate**: `rate(langfuse_generations_total{status="error"}[5m]) / rate(langfuse_generations_total[5m])`
- **Active models**: `count(langfuse_generation_latency_p99_ms) by (model)`

### Performance row

- **P50 / P99 latency timeseries by model** (multi-line, variable `$model`)
- **Token consumption rate**: `rate(langfuse_tokens_input_total[5m])` and `rate(langfuse_tokens_output_total[5m])`
- **Cost per hour**: `increase(langfuse_cost_usd_window[1h])`

### Quality row (requires evaluation scoring)

- **Average score by scorer**: `langfuse_score_average`
- **Score below threshold count**: `langfuse_score_below_threshold_total`
- **Quality trend over 7 days**: use `avg_over_time(langfuse_score_average[7d])`

### Template variable

Add a `$model` variable linked to the `model` label of `langfuse_generation_latency_p99_ms`. A single dashboard covers all your models with dynamic filtering.

---

## What This Approach Cannot Do

Be explicit about the limits.

**Aggregated metrics do not replace individual traces.** If you need to debug a specific generation — see the exact prompt, the tokens, the raw response — you stay in the Langfuse UI or its traces API. VictoriaMetrics does not store that level of detail, and it should not: that is not its purpose.

**Evaluation scores require an active scoring system.** The exporter exposes scores that already exist in Langfuse. It does not generate them. LLM-as-judge, human annotation, or automated scoring must be configured separately (see module DevOps-04).

**The polling interval introduces a lag.** With a 60s scrape interval, your Grafana dashboard is at most 60s behind reality. For real-time anomaly detection, pair the metrics dashboard with direct Langfuse trace alerts on individual score values via the API.

---

## Summary Table

| Need | Tool |
|---|---|
| P99 latency alerts, error rate, cost spike | VMAlert + VictoriaMetrics |
| On-call operational dashboard | Grafana + VM datasource |
| Debug a specific generation | Langfuse UI / Traces API |
| Offline quality evaluation | Langfuse datasets + scoring |
| Correlation with infrastructure spans | Jaeger / Tempo via OTEL |

The two stacks are complementary. Routing LLM metrics into VictoriaMetrics does not replace Langfuse — it integrates it into your existing operational culture. Your on-call engineer finds the anomaly in Grafana, opens Langfuse for the content details, closes the ticket. That is what mature LLM observability looks like.

---

*Samuel Desseaux is the founder of Aureonis and an official VictoriaMetrics Training Partner (France, Benelux, Germany). Speaker at FOSDEM 2026 and KubeCon Europe.*
