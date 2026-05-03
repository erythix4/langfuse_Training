# Langfuse in a MLOps Stack: Where It Fits, What It Does Not Cover

*By Samuel Desseaux — Aureonis*

---

## The Silo Risk

When a team adopts Langfuse without integrating it into its existing MLOps process, a silo forms. Production data stays in Langfuse, disconnected from the retraining pipeline, the feature store, and the model registry. Insights about production quality do not inform decisions about training data or model selection.

This module maps Langfuse's position in a complete MLOps architecture and identifies the integration points with common tools.

---

## MLOps Stack Anatomy for LLM Workloads

```
┌──────────────────────────────────────────────────────────────┐
│  DATA LAYER                                                  │
│  Raw storage (S3 / GCS)  +  Feature store (Feast / Hopsworks)│
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│  EXPERIMENTATION & TRAINING                                  │
│  MLflow / DVC / W&B  ←── experiment tracking                │
│  LoRA / PEFT fine-tuning  or  prompt engineering iteration   │
└──────────────────────────────┬───────────────────────────────┘
                               │ model artifact
┌──────────────────────────────▼───────────────────────────────┐
│  MODEL REGISTRY                                              │
│  MLflow Registry  /  Hugging Face Hub                        │
│  version, stage (staging / production), run_id               │
└──────────────────────────────┬───────────────────────────────┘
                               │ deploy
┌──────────────────────────────▼───────────────────────────────┐
│  SERVING LAYER                                               │
│  vLLM / BentoML / TGI  /  External API (OpenAI, Anthropic)  │
└──────────────────────────────┬───────────────────────────────┘
                               │ requests
┌──────────────────────────────▼───────────────────────────────┐
│  PRODUCTION OBSERVABILITY        ← Langfuse lives here       │
│  Langfuse  traces + evaluation + scores + sessions           │
│  APM       infra spans, latency, saturation                  │
│  Prometheus metrics, alerting                                │
└──────────────────────────────┬───────────────────────────────┘
                               │ low-quality traces
┌──────────────────────────────▼───────────────────────────────┐
│  FEEDBACK LOOP                                               │
│  Langfuse API → export hard cases                            │
│  DVC: version dataset                                        │
│  MLflow: new fine-tuning run                                 │
│  → back to TRAINING layer                                    │
└──────────────────────────────────────────────────────────────┘
```

Langfuse operates primarily in the "Production observability" layer and in the "Feedback loop". It is not a training tool, not a feature store, not a model registry.

---

## Integration with MLflow

MLflow is the most widely used experiment tracking tool. Langfuse and MLflow do not integrate natively, but they coexist well with a clear naming convention.

Separation of responsibilities:
- **MLflow**: training experiment tracking (hyperparameters, training metrics, artifacts), model registry management
- **Langfuse**: production quality monitoring, traces, output evaluation

The link between them: the MLflow `model_version` or `run_id` can be passed in Langfuse metadata to correlate production performance with training parameters.

```python
import mlflow
from langfuse import Langfuse

langfuse = Langfuse()

# Retrieve model version from the MLflow registry
client = mlflow.tracking.MlflowClient()
model_version = client.get_latest_versions("my-llm", stages=["Production"])[0]

# Pass it into Langfuse trace metadata
trace = langfuse.trace(
    name="production-inference",
    metadata={
        "mlflow_model_version": model_version.version,
        "mlflow_run_id": model_version.run_id,
        "model_name": "my-llm",
        "training_dataset": "v2.3.0"
    }
)
```

When production scores degrade in Langfuse, you can trace back to the corresponding MLflow `run_id` and inspect the training conditions of the problematic model version.

---

## Integration with DVC

DVC (Data Version Control) manages data and pipeline versioning. Its integration with Langfuse is indirect but powerful: Langfuse traces can feed new training datasets via the API.

```python
import json
import subprocess
from langfuse import Langfuse
from datetime import datetime, timedelta, timezone

langfuse = Langfuse()

def export_low_quality_traces(
    min_score: float = 0.5,
    score_name: str = "judge-faithfulness",
    output_path: str = "data/hard-cases.jsonl",
    days_back: int = 7
):
    """
    Export traces with low quality scores to a DVC-versioned dataset.
    These become training candidates for fine-tuning or prompt improvement.
    """
    cutoff = datetime.now(timezone.utc) - timedelta(days=days_back)

    traces = langfuse.get_traces(
        limit=1000,
        from_timestamp=cutoff
        # Note: get_traces() returns score objects in trace.scores by default in SDK v2+
        # If scores are not populated, use get_trace(id) per trace or the /api/public/scores endpoint
    )

    low_quality = [
        t for t in traces.data
        if any(
            s.name == score_name and s.value < min_score
            for s in (t.scores or [])
        )
    ]

    with open(output_path, "w") as f:
        for trace in low_quality:
            f.write(json.dumps({
                "input": trace.input,
                "output": trace.output,
                "score": min_score,
                "trace_id": trace.id,
                "timestamp": trace.timestamp.isoformat() if trace.timestamp else None,
                "metadata": trace.metadata
            }) + "\n")

    print(f"Exported {len(low_quality)} low-quality traces to {output_path}")

    # Version the dataset with DVC
    subprocess.run(["dvc", "add", output_path], check=True)
    subprocess.run(["git", "add", f"{output_path}.dvc"], check=True)
    subprocess.run([
        "git", "commit", "-m",
        f"Export hard cases from Langfuse ({score_name} < {min_score}, {days_back}d)"
    ], check=True)
```

This flow — extract hard cases from Langfuse, version them with DVC, integrate them into the fine-tuning pipeline — is one of the most effective patterns for continuously improving an LLM in production. The production data becomes the training signal.

---

## Integration with BentoML

BentoML standardizes model serving deployments. Langfuse instrumentation integrates cleanly via a service-level pattern:

```python
import bentoml
from langfuse import Langfuse

langfuse = Langfuse()

@bentoml.service(
    resources={"cpu": "2"},
    traffic={"timeout": 30}
)
class LLMService:
    def __init__(self):
        self.model = self._load_model()

    def _load_model(self):
        # Load your model here
        pass

    @bentoml.api
    def generate(self, question: str, user_id: str = "anonymous") -> str:
        trace = langfuse.trace(
            name="bentoml-inference",
            input={"question": question},
            user_id=user_id,
            metadata={
                "serving_framework": "bentoml",
                "service_version": bentoml.__version__,
                "bento": "llm-service:latest"
            }
        )

        generation = trace.generation(
            name="model-generation",
            input=question,
            model="custom-finetuned-mistral-7b"
        )

        try:
            response = self.model.generate(question)
            generation.end(output=response)
            trace.update(output={"answer": response})
            return response
        except Exception as e:
            generation.end(output={"error": str(e)}, level="ERROR")
            raise
        finally:
            langfuse.flush()
```

---

## Integration with W&B (Weights & Biases)

For teams using W&B for experiment tracking, the integration pattern mirrors MLflow:

```python
import wandb
from langfuse import Langfuse

langfuse = Langfuse()

# Link W&B run metadata to Langfuse traces
run = wandb.init(project="llm-production", job_type="inference")

trace = langfuse.trace(
    name="production-inference",
    metadata={
        "wandb_run_id": run.id,
        "wandb_project": run.project,
        "wandb_entity": run.entity
    }
)
```

---

## What Langfuse Does Not Cover in a MLOps Stack

This list is important for calibrating expectations:

**Classical ML model drift monitoring.** Langfuse does not calculate feature drift, PSI/KL divergence, or concept drift on classification or regression models. For these needs, Evidently AI or Fiddler are better suited.

**Feature store.** Langfuse does not store training features. Production traces can feed a training dataset, but feature management stays in your feature store.

**Experiment reproducibility.** Training run tracking (hyperparameters, loss curves, offline evaluation metrics) stays in MLflow or W&B.

**Serving and load balancing.** Langfuse is not on the critical request path. It does not do routing, load balancing, or model fallback.

**Continuous training pipelines.** Langfuse does not trigger retraining. It provides the data signals that inform retraining decisions. The actual pipeline orchestration stays in Airflow, Prefect, Kubeflow, or your CI/CD system.

---

## The Responsibility Table

| Need | Tool |
|---|---|
| Training data versioning | DVC |
| Experiment tracking | MLflow / W&B |
| Model registry | MLflow Registry / HF Hub |
| Production serving | vLLM / BentoML / TGI |
| LLM traces and evaluation | Langfuse |
| System metrics | Prometheus / VictoriaMetrics |
| Classical ML drift monitoring | Evidently AI / Fiddler |
| Production --> training feedback loop | Langfuse API + DVC + MLflow |

Langfuse's value in this table is precisely bounded: it is the layer that connects what your users experience in production with the decisions your ML team makes about data and models. Not more, not less.

---

*Samuel Desseaux is the founder of Aureonis, an observability stack specialist and LLM security practitioner. Speaker at FOSDEM 2026 and KubeCon Europe.*
