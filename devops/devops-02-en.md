# Getting Started with Langfuse: Your First LLM Trace in 30 Minutes

*By Samuel Desseaux — Aureonis*

---

## Objective

By the end of this module, you have Langfuse running locally, a Python application instrumented, and an LLM trace readable in the interface. No additional theory — we install, instrument, and read.

---

## Prerequisites

- Docker and Docker Compose installed
- Python 3.10+
- An OpenAI, Anthropic, or Mistral API key (examples use OpenAI, adaptable)
- 30 minutes

---

## Step 1 — Deploy Langfuse Locally

Langfuse self-hosted runs three components: the Next.js application, a PostgreSQL database, and a worker container for asynchronous trace processing.

```bash
git clone https://github.com/langfuse/langfuse.git
cd langfuse
cp .env.example .env
```

Edit `.env` to define the minimum required variables:

```env
NEXTAUTH_SECRET=changeme-local-secret-32chars-min
SALT=changeme-local-salt
ENCRYPTION_KEY=0000000000000000000000000000000000000000000000000000000000000000
DATABASE_URL=postgresql://postgres:postgres@db:5432/langfuse
NEXTAUTH_URL=http://localhost:3000
```

Start the stack:

```bash
docker compose up -d
```

The interface is available at `http://localhost:3000` after approximately 30 seconds. Create an administrator account on first access.

---

## Step 2 — Create a Project and Retrieve API Keys

In the Langfuse interface:

1. **Create a project** — "my-first-llm-app" or any name
2. **Go to Settings > API Keys**
3. Copy the `Public Key` and `Secret Key`

These two keys identify your project. They are distinct from your OpenAI API key — they authenticate against Langfuse only.

---

## Step 3 — Instrument a Python Application

Install the SDK:

```bash
pip install langfuse openai
```

A minimal instrumented application:

```python
import os
from langfuse.openai import openai  # instrumented wrapper

os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-lf-..."
os.environ["LANGFUSE_SECRET_KEY"] = "sk-lf-..."
os.environ["LANGFUSE_HOST"] = "http://localhost:3000"
os.environ["OPENAI_API_KEY"] = "sk-..."

# This wrapper replaces `from openai import OpenAI`
# All generations are automatically traced
response = openai.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "You are a concise technical assistant."},
        {"role": "user", "content": "What is an OpenTelemetry span? Two sentences."}
    ]
)

print(response.choices[0].message.content)

# Force flush before script exit
from langfuse import Langfuse
Langfuse().flush()
```

Run the script. Return to the Langfuse interface. You should see your first trace appear under **Traces**.

---

## Step 4 — Reading a Trace

Click on the trace. You see the hierarchy:

```
Trace: chatcmpl-abc123
  └── Generation: gpt-4o-mini
        Input: [system message] + [user message]
        Output: "An OpenTelemetry span represents..."
        Model: gpt-4o-mini
        Tokens: 45 input / 38 output
        Cost: $0.000012
        Duration: 612ms
```

This is exactly what your Prometheus/Jaeger stack was not showing you: the prompt content, the raw response, token counts, and cost — in a single view.

---

## Step 5 — Manual Trace Structuring

The automatic wrapper is convenient, but you will quickly want to structure traces manually — especially if you have multiple steps (document retrieval, validation, post-processing).

```python
from langfuse import Langfuse
from langfuse.openai import openai

langfuse = Langfuse()

# Create a parent trace grouping multiple operations
trace = langfuse.trace(
    name="question-answer-pipeline",
    user_id="user-42",
    session_id="session-abc",
    metadata={"source": "api", "version": "1.2.0"}
)

# Span for context retrieval
retrieval_span = trace.span(
    name="retrieve-context",
    input={"query": "What is an OTEL span?"}
)

# Simulate document retrieval
docs = ["An OTEL span represents a single unit of work..."]
retrieval_span.end(output={"documents": docs, "count": len(docs)})

# Generation with retrieved context
generation = trace.generation(
    name="answer-generation",
    model="gpt-4o-mini",
    input=[
        {"role": "system", "content": f"Context: {docs[0]}"},
        {"role": "user", "content": "Explain what an OTEL span is."}
    ]
)

response = openai.chat.completions.create(
    model="gpt-4o-mini",
    messages=generation.input
)

answer = response.choices[0].message.content
generation.end(output=answer)

# Add a manual score (useful for evaluations)
trace.score(
    name="manual-review",
    value=1.0,
    comment="Correct and concise"
)

langfuse.flush()
print(answer)
```

Return to the interface. The trace now displays two nested spans: `retrieve-context` and `answer-generation`. You can see the duration of each step, inputs/outputs, and the score you assigned.

---

## Step 6 — Exploring the Interface

Take five minutes to navigate the Langfuse UI:

**Traces view.** All traces for your project, sorted by time. Click any trace to open the detail view with the span tree, inputs, outputs, token counts, and scores.

**Generations view.** A flattened view of all LLM generations across all traces. Useful for filtering by model, token count range, or latency.

**Dashboard.** Aggregated metrics: total generations, average latency, token consumption over time, cost trends. These are pre-built; the custom metrics pipeline is in module SRE-02.

**Users view.** All traces grouped by `user_id`. If you instrument user identifiers, you can see the full history of any user's interactions.

---

## What You Have Built

In 30 minutes, you have:

- Langfuse self-hosted running (Docker Compose)
- A project with authentication keys
- A Python application that automatically traces LLM calls
- A structured trace with nested spans, tokens, cost, and a score

These are the foundations. The next step is instrumenting a more realistic pipeline — with retrieval, reranking, and generation — to see how Langfuse handles operation hierarchies and enables quality drift diagnosis.

---

## Going Further

**Environment variables in production.** Outside of local development, use a secrets manager (Vault, Kubernetes Secrets, AWS Secrets Manager) rather than hardcoding values in code.

**Sampling.** For high-volume applications, you can configure a sampling rate in the Langfuse SDK to send only a fraction of traces. In debugging mode, keep 100%.

**Tags and metadata.** The `metadata` property accepts any JSON dictionary. Use it to store contextual information (application version, environment, feature flag ID) that will make filtering in the interface much more useful.

**Multi-language support.** Langfuse provides SDKs for Python, TypeScript/JavaScript, and integrations for LangChain, LlamaIndex, LiteLLM, and others. The instrumentation patterns in this module apply across all SDKs.

---

*Samuel Desseaux is the founder of Aureonis, an observability stack specialist and LLM security practitioner. Speaker at FOSDEM 2026 and KubeCon Europe.*
