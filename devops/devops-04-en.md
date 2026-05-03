# LLM Evaluation in Production: Building a Continuous Quality Feedback Loop

*By Samuel Desseaux — Aureonis*

---

## Quality Is Not Latency

An LLM that responds in 400ms with HTTP 200 may be actively degrading your user experience. An inaccurate, off-topic, or hallucinated response is an application incident that your traditional SLOs will never catch.

LLM evaluation in production is the discipline that answers: "are our responses actually good?" — not "is our service available?". This module covers the three evaluation approaches available in Langfuse, how to combine them into a continuous feedback loop, and how to expose quality metrics to your existing monitoring stack.

---

## The Three Levels of Evaluation

### Level 1 — Human Annotation

The gold standard. A human reads the response and evaluates it according to defined criteria. Most reliable, most time-intensive.

In Langfuse, you define score templates with named dimensions and numeric or boolean scales. The annotation queue interface presents traces one by one to annotators, who rate each dimension. Scores are stored on the trace and become queryable via the API.

```python
# Define score configs via SDK or the Langfuse UI
langfuse.create_score_config(name="response-quality", data_type="NUMERIC", min_value=0, max_value=5)
langfuse.create_score_config(name="factual-accuracy", data_type="BOOLEAN")
langfuse.create_score_config(name="tone-appropriate", data_type="BOOLEAN")
```

Use human annotation for: calibrating your automated scorers, building a reference dataset, validating major prompt or model changes, and resolving ambiguous cases flagged by automated scoring.

### Level 2 — Rule-Based Automatic Scoring

Deterministic rules you can trigger on every generation. Fast, no additional cost, but limited to what is measurable without an LLM.

```python
def score_automatically(trace, answer: str, question: str, expected_language: str = "en"):
    import langdetect

    scores = {}

    # Response length check
    word_count = len(answer.split())
    scores["length-ok"] = 1.0 if word_count >= 10 else 0.0

    # Explicit refusal detection
    refusal_patterns = [
        "i don't know", "cannot answer", "i'm unable to",
        "not in the context", "i have no information"
    ]
    scores["no-refusal"] = 0.0 if any(p in answer.lower() for p in refusal_patterns) else 1.0

    # Language detection
    try:
        detected_lang = langdetect.detect(answer)
        scores["correct-language"] = 1.0 if detected_lang == expected_language else 0.0
    except Exception:
        scores["correct-language"] = -1.0  # Indeterminate

    # Keyword presence check (domain-specific)
    required_keywords = []  # Define based on your domain
    if required_keywords:
        found = sum(1 for kw in required_keywords if kw.lower() in answer.lower())
        scores["keyword-coverage"] = found / len(required_keywords)

    for name, value in scores.items():
        if value >= 0:
            trace.score(name=name, value=value)

    return scores
```

### Level 3 — LLM-as-Judge

You use an LLM (typically more capable than the one in production) to evaluate the responses from your pipeline. This approach enables evaluating subjective dimensions — relevance, consistency, faithfulness to context — without human annotation overhead.

```python
from langfuse.openai import openai
import json

def llm_judge(trace, question: str, context: str, answer: str) -> dict:
    judge_prompt = f"""You are a quality evaluator for a Q&A system.

Question asked: {question}

Context provided to the system: {context}

Generated response: {answer}

Evaluate the response on these three criteria. Respond only with valid JSON.

{{
  "relevance": <0.0 to 1.0 — does the response answer the question?>,
  "faithfulness": <0.0 to 1.0 — is the response faithful to the context?>,
  "completeness": <0.0 to 1.0 — is the response complete?>,
  "rationale": "<one or two sentences explaining your scores>"
}}"""

    response = openai.chat.completions.create(
        model="gpt-4o",  # Judge model — more capable than the evaluated model
        messages=[{"role": "user", "content": judge_prompt}],
        temperature=0.0,
        response_format={"type": "json_object"}
    )

    scores = json.loads(response.choices[0].message.content)

    for metric in ["relevance", "faithfulness", "completeness"]:
        if metric in scores:
            trace.score(
                name=f"judge-{metric}",
                value=float(scores[metric]),
                comment=scores.get("rationale", "")
            )

    return scores
```

**Important caveats on LLM-as-judge.** The judge has its own biases — preference for longer responses, for certain writing styles, for confident-sounding text. Calibrate it on a human-annotated dataset before deploying it in production. A miscalibrated judge creates false confidence.

---

## Building a Continuous Evaluation Loop

The goal is not to evaluate each trace individually, but to monitor score evolution over time. The three evaluation levels form a feedback loop at different time resolutions:

```
                        PRODUCTION
                            │
             ┌──────────────▼──────────────┐
             │   Every generation (100%)    │
             │   Rule-based scoring         │
             │   · length-ok               │
             │   · no-refusal              │
             │   · correct-language        │
             └──────────────┬──────────────┘
                            │
              ┌─────────────▼──────────────┐
              │         LANGFUSE            │
              └──┬──────────┬──────────────┘
                 │          │
    ┌────────────▼──┐  ┌────▼──────────────┐
    │  Score < 0.5  │  │   Daily batch      │
    │  annotation   │  │   LLM-as-judge     │
    │  queue        │  │   on sample (10%)  │
    └────────┬──────┘  └────┬──────────────┘
             │              │
    ┌────────▼──────────────▼──────────────┐
    │          METRICS STORE               │
    │  VictoriaMetrics (via exporter)      │
    │  · avg score / 1h                    │
    │  · score drift vs baseline           │
    └───────────────┬──────────────────────┘
                    │
         ┌──────────▼─────────────┐
         │  GRAFANA + VMALERT     │
         │  · Alert if drift      │
         │  · On-call notification│
         └────────────────────────┘
```

Recommended flow:

```
Production
    |
    | Rule-based scoring (100% of traces, every generation)
    |
Langfuse
    |
    +---> Average score / 1h --> Grafana (via exporter or API)
    |
    +---> Low score (below threshold) --> Human annotation queue
    |
    +---> Production traces --> Evaluation dataset
    |
    +---> LLM-as-judge batch (daily, on dataset sample)
    |
    +---> Results vs baseline --> Alert if drift detected
```

Rule-based scoring on 100% of traces gives you a real-time trend. LLM-as-judge in daily batch gives you a more reliable measure on a representative sample. Human annotation handles ambiguous cases and calibrates the other levels.

---

## Building an Evaluation Dataset from Production

Langfuse lets you create datasets from production traces. This is one of the most valuable uses: your real production cases become your regression benchmark.

```python
# Create a dataset
langfuse.create_dataset(name="rag-eval-v1")

# Add items from production traces (via API or UI)
langfuse.create_dataset_item(
    dataset_name="rag-eval-v1",
    input={"question": "What is VictoriaMetrics?"},
    expected_output={"answer": "VictoriaMetrics is a time-series database..."},
    metadata={"source": "production", "trace_id": "trace-abc123"}
)

# Run the evaluation against a new pipeline version
def evaluate_dataset(dataset_name: str, pipeline_fn):
    dataset = langfuse.get_dataset(dataset_name)

    results = []
    for item in dataset.items:
        # Run the new pipeline on this input
        output = pipeline_fn(item.input["question"])

        # Score against expected output
        similarity = compute_similarity(output, item.expected_output["answer"])

        item.link(
            run_name="pipeline-v2.3.0",
            run_description="Evaluation with strict prompt template"
        )

        results.append({"item_id": item.id, "similarity": similarity})

    avg_similarity = sum(r["similarity"] for r in results) / len(results)
    print(f"Dataset {dataset_name}: avg similarity = {avg_similarity:.3f}")
    return results
```

Before every significant prompt or model change, run this evaluation against your production dataset and compare scores to the previous baseline. This turns evaluation from an ad-hoc activity into a deployment gate.

---

## Exposing Quality Metrics to Grafana

Once scores are in place, expose them to your existing monitoring stack (via the Langfuse API or the exporter from module SRE-02):

Recommended metrics for a quality dashboard:

- **Average score by dimension** over 24h / 7d rolling window
- **Refusal rate** (score `no-refusal = 0`)
- **Score distribution by model** (compare gpt-4o-mini vs gpt-4o)
- **Score vs latency correlation** — are fast responses also the lowest quality?
- **Score change after a deployment** — did the prompt update improve or regress quality?

---

## Practical Recommendations

**Start with two automatic scorers, not ten.** Too many scores drown the signal in noise. Identify the two dimensions that matter most for your use case. For a support chatbot: "answered the question" and "no hallucination". Instrument those two first.

**Deploy LLM-as-judge only after calibrating on a human-annotated dataset.** Uncalibrated judge = false confidence.

**Treat low scores as incidents.** If `judge-faithfulness` stays below 0.7 for more than 30 minutes, that is an incident with the same operational weight as a rising HTTP error rate. It needs a runbook and an alert.

**Keep a baseline per model and prompt version.** Every time you change the system prompt or switch models, record the baseline scores before the change. This makes regressions immediately visible.

---

*Samuel Desseaux is the founder of Aureonis, an observability stack specialist and LLM security practitioner. Speaker at FOSDEM 2026 and KubeCon Europe.*
