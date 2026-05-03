# End-to-End RAG Pipeline Tracing with Langfuse

*By Samuel Desseaux - Erythix*

---

## The Opaque Pipeline Problem

A RAG (Retrieval Augmented Generation) pipeline chains multiple steps: document retrieval from a vector store, optional reranking, context assembly, generation, and post-processing. Without instrumentation, you have a black box.

When quality degrades, you have no idea where the failure is. Is the retriever surfacing irrelevant documents? Is the assembled prompt too long, saturating the context window? Is the model ignoring the context entirely? Each hypothesis requires manual debugging if your pipeline is not traced.

This module shows you how to instrument every step of a RAG pipeline with Langfuse, so you have complete visibility into what happened for every request.

---

## Architecture of the Instrumented Pipeline

```
User request: "What is VictoriaMetrics?"
      │
      ▼
┌─────────────────────────────────────────────────────────────────┐
│  Langfuse Trace: rag-pipeline                                   │
│  user_id, session_id, pipeline_version                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Span: embed-query                             [42ms]   │   │
│  │  Input:  text="What is VictoriaMetrics?"               │   │
│  │  Output: dimensions=1536                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Span: vector-search                          [156ms]   │   │
│  │  Input:  query, k=3                                     │   │
│  │  Output: 3 docs, distances=[0.18, 0.31, 0.45]          │   │
│  │  ← key diagnostic: are distances low enough?            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Span: assemble-prompt                           [2ms]  │   │
│  │  Input:  question + 3 docs                              │   │
│  │  Output: messages[] with system+context+user            │   │
│  │  ← key diagnostic: is context complete / not too long?  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Generation: llm-answer                       [1580ms]  │   │
│  │  Model: gpt-4o-mini  |  Temp: 0.1                      │   │
│  │  Tokens: 312 in / 67 out  →  $0.000042                 │   │
│  │  Output: "VictoriaMetrics is a time-series database..." │   │
│  │  ← key diagnostic: did the model follow the context?    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Span: post-process                              [0ms]  │   │
│  │  Output: is_refusal=false, length=284                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Scores: has-answer=1.0  |  retrieval-quality=0.82              │
└─────────────────────────────────────────────────────────────────┘
      │
      ▼
  Answer returned to user
```

This structure lets you see, for every request, the detail of every step, inputs/outputs, and durations.

---

## Complete Implementation

The example uses Chroma as the vector store, OpenAI embeddings, and GPT-4o-mini for generation. Adapt the components to your stack.

```python
import os
from langfuse import Langfuse
from langfuse.openai import openai
import chromadb
from chromadb.utils.embedding_functions import OpenAIEmbeddingFunction

langfuse = Langfuse()

embedding_fn = OpenAIEmbeddingFunction(
    api_key=os.environ["OPENAI_API_KEY"],
    model_name="text-embedding-3-small"
)
chroma_client = chromadb.Client()
collection = chroma_client.get_or_create_collection(
    name="docs",
    embedding_function=embedding_fn
)

# Index documents once
collection.add(
    documents=[
        "An OpenTelemetry span represents a single unit of work in a distributed system.",
        "VictoriaMetrics is a Prometheus-compatible time-series database.",
        "RAG combines an external knowledge base with a language model.",
    ],
    ids=["doc1", "doc2", "doc3"]
)


def rag_pipeline(question: str, user_id: str, session_id: str) -> str:
    trace = langfuse.trace(
        name="rag-pipeline",
        input={"question": question},
        user_id=user_id,
        session_id=session_id,
        metadata={"pipeline_version": "2.1.0"}
    )

    try:
        # Step 1: embed the question
        embed_span = trace.span(
            name="embed-query",
            input={"text": question, "model": "text-embedding-3-small"}
        )
        query_embedding = embedding_fn([question])[0]
        embed_span.end(output={"dimensions": len(query_embedding)})

        # Step 2: vector search
        search_span = trace.span(
            name="vector-search",
            input={"query": question, "k": 3}
        )
        results = collection.query(query_texts=[question], n_results=3)
        retrieved_docs = results["documents"][0]
        distances = results["distances"][0]
        search_span.end(output={
            "documents": retrieved_docs,
            "scores": distances,
            "count": len(retrieved_docs)
        })

        # Step 3: assemble prompt
        assemble_span = trace.span(
            name="assemble-prompt",
            input={"question": question, "doc_count": len(retrieved_docs)}
        )
        context = "\n\n".join(
            f"[Doc {i+1}] {doc}" for i, doc in enumerate(retrieved_docs)
        )
        system_prompt = (
            "You are a technical assistant. Answer only based on the provided context. "
            "If the context does not allow you to answer, say so explicitly.\n\n"
            f"Context:\n{context}"
        )
        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": question}
        ]
        assemble_span.end(output={"messages": messages})

        # Step 4: generation
        generation = trace.generation(
            name="llm-answer",
            model="gpt-4o-mini",
            input=messages,
            metadata={"temperature": 0.1}
        )
        response = openai.chat.completions.create(
            model="gpt-4o-mini", messages=messages, temperature=0.1
        )
        answer = response.choices[0].message.content
        generation.end(
            output=answer,
            usage={
                "input": response.usage.prompt_tokens,
                "output": response.usage.completion_tokens
            }
        )

        # Step 5: post-process
        post_span = trace.span(name="post-process", input={"answer": answer})
        is_refusal = any(p in answer.lower() for p in [
            "cannot answer", "i don't know", "not in the context"
        ])
        post_span.end(output={"is_refusal": is_refusal, "answer_length": len(answer)})

        # Automatic scoring
        trace.score(name="has-answer", value=0.0 if is_refusal else 1.0)

        # Check retrieval quality - penalize empty or low-relevance results
        avg_distance = sum(distances) / len(distances) if distances else 1.0
        trace.score(name="retrieval-quality", value=max(0.0, 1.0 - avg_distance))

        trace.update(output={"answer": answer})
        return answer

    except Exception as e:
        trace.update(output={"error": str(e)}, metadata={"status": "error"})
        raise
    finally:
        langfuse.flush()
```

---

## What You See in the Interface

After execution, the trace in Langfuse looks like this:

```
Trace: rag-pipeline | user-demo | session-demo-001
  Duration: 1847ms
  ├── embed-query         [42ms]   dimensions: 1536
  ├── vector-search       [156ms]  3 docs, distances: [0.18, 0.31, 0.45]
  ├── assemble-prompt     [2ms]    2 messages assembled
  ├── llm-answer          [1580ms] 312 input tokens, 67 output tokens, $0.000042
  └── post-process        [0ms]    is_refusal: false, length: 284
  Score: has-answer = 1.0
  Score: retrieval-quality = 0.82
```

At a glance: 85% of the latency is in generation, retrieval found 3 documents with reasonable scores, and the response is not a refusal.

---

## Diagnosing a Quality Degradation

The canonical scenario: a user complains the chatbot answered off-topic. With this instrumentation, you open the corresponding trace and check in order:

**1. Did retrieval find the right documents?**
Look at the `vector-search` span. If distances are high (>0.6 on cosine), retrieval found no relevant documents. Root cause: insufficient indexing, or the question phrasing is too far from the indexed content.

**2. Was the assembled prompt correct?**
Look at the `assemble-prompt` span. Is the injected context relevant? Is it too long (risk of context window saturation - the model starts ignoring early content)?

**3. Did the model follow the context?**
Compare the `llm-answer` output with the context provided. If the response contains information absent from the context, the model hallucinated - despite the RAG context.

**4. Is the quality score drifting over time?**
In Langfuse, filter traces by `score:has-answer = 0.0`. If the refusal rate is increasing, your document base no longer covers your users' questions.

---

## Comparing Pipeline Versions

One of the most powerful uses of Langfuse is version comparison. You modify your system prompt, deploy in A/B, and compare the traces of both versions side by side.

Use the `metadata` field to tag traces with the pipeline version:

```python
trace = langfuse.trace(
    name="rag-pipeline",
    metadata={
        "pipeline_version": "2.2.0",
        "prompt_template": "v3-strict",
        "retrieval_k": 5,
        "rerank_enabled": True
    }
)
```

You can then filter in Langfuse by `metadata.pipeline_version` and compare average scores, latencies, and refusal rates between versions. This is what makes Langfuse essential for iterative prompt engineering in production.

---

## Production Considerations

**Input/output storage cost.** Langfuse stores raw prompts and responses. For high-volume applications, define a retention policy in project settings. In self-hosted mode, monitor PostgreSQL table growth directly.

**PII in prompts.** If your users send personal data in their questions, it ends up in stored inputs. Implement a redaction step before instrumentation, or configure Langfuse to mask sensitive fields. See module SRE-04 for the full redaction architecture.

**Sampling in production.** For high-volume applications, 10-20% sampling is reasonable for complete traces. Keep 100% sampling for traces that trigger a low quality score - these are the cases you most need to debug.

---

*Samuel Desseaux is the founder of Erythix, an observability stack specialist and LLM security practitioner. Speaker at FOSDEM 2026 and KubeCon Europe.*
