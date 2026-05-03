# Multi-Step Agents in Production: Visibility and Diagnosis with Langfuse

*By Samuel Desseaux — Aureonis*

---

## The Specific Challenge of Agents

An LLM agent is not a pipeline. A pipeline has a predefined structure: step 1, step 2, step 3. An agent decides which actions to take, in what order, and how many times. This characteristic — autonomy — is what makes agents powerful, and what makes their observability difficult.

In production, an agent may:
- Call a tool, observe the result, decide to call a different tool
- Loop several times on the same action before converging
- Abandon one branch and try another
- Produce a non-deterministic decision tree, different on every execution for the same input

Your classical observability tools see the total execution duration and the final result. Langfuse can show you every decision, every tool call, every iteration — if you instrument correctly.

---

## Structure of an Agentic Trace

The key difference between a pipeline and an agent in Langfuse is the branching, variable-depth trace tree:

```
Trace: agent-run
├── Metadata: user_id, session_id, agent_version, tools[]
│
├── [Step 1 — Thinking]
│     Span: agent-step-1/think
│     Input:  "What is the refund policy for order #4421?"
│     Output: "I need to search documentation first."
│
├── [Step 1 — Tool call]
│     Span: agent-step-1/tool: search_documentation
│     Input:  {"query": "refund policy"}
│     Output: [3 documents returned]        ← visible
│     Duration: 340ms
│
├── [Step 1 — Reasoning]
│     Generation: gpt-4o-mini               ← tokens + cost
│     Input:  question + search results
│     Output: "Docs incomplete. Need pricing for #4421."
│     412 in / 87 out  →  $0.000062
│
├── [Step 2 — Tool call]
│     Span: agent-step-2/tool: get_pricing
│     Input:  {"product_id": "XYZ-42"}
│     Output: {"price": 299, "currency": "EUR"}
│     Duration: 180ms
│
└── [Step 2 — Final answer]
      Generation: gpt-4o-mini
      Input:  question + docs + pricing
      Output: "You can return within 30 days. Price: €299."
      687 in / 234 out  →  $0.000148

Scores:
  agent-steps-count: 2
  tool-calls-count:  2
  total-tokens:      1420
  loop-detected:     0.0
```

Recommended span hierarchy for a ReAct (Reason + Act) agent:

```
Trace: agent-run | session-id | user-id
  Total duration: 12.4s
  |
  +-- Span: agent-step-1 [thinking]
  |     Input: initial question
  |     Output: "I need to search the documentation"
  |
  +-- Span: agent-step-1 [tool-call: search_docs]
  |     Input: {"query": "refund procedure"}
  |     Output: [3 documents]
  |     Duration: 340ms
  |
  +-- Generation: agent-step-1 [reasoning]
  |     Input: question + search results
  |     Output: "Documents don't fully answer. I also need to check pricing."
  |     Tokens: 412 in / 87 out
  |
  +-- Span: agent-step-2 [tool-call: get_pricing]
  |     Input: {"product_id": "XYZ-42"}
  |     Output: {"price": 299, "currency": "EUR"}
  |     Duration: 180ms
  |
  +-- Generation: agent-step-2 [final-answer]
  |     Input: question + docs + pricing
  |     Output: "Here is the complete procedure with pricing..."
  |     Tokens: 687 in / 234 out
  |
  Score: agent-steps-count = 2
  Score: tool-calls-count = 2
  Score: total-tokens = 1420
```

This structure shows the complete reasoning path, tools used, and cost of each step.

---

## Implementation with LangChain

```python
from langchain.agents import AgentExecutor, create_react_agent
from langchain.tools import tool
from langchain_openai import ChatOpenAI
from langchain.callbacks.base import BaseCallbackHandler
from langchain.prompts import PromptTemplate
from langfuse import Langfuse
from langfuse.callback import CallbackHandler as LangfuseCallback
import time

langfuse = Langfuse()


class AgentObservabilityCallback(BaseCallbackHandler):
    """LangChain callback that enriches Langfuse traces with agentic metrics."""

    def __init__(self, trace):
        self.trace = trace
        self.step_count = 0
        self.tool_calls = []
        self.tool_errors = 0
        self.start_time = time.time()

    def on_agent_action(self, action, **kwargs):
        self.step_count += 1
        self.tool_calls.append({
            "step": self.step_count,
            "tool": action.tool,
            "input": str(action.tool_input)[:500]  # Truncate long inputs
        })

    def on_tool_error(self, error, **kwargs):
        self.tool_errors += 1
        self.trace.score(
            name="tool-error",
            value=1.0,
            comment=str(error)[:200]
        )

    def on_agent_finish(self, finish, **kwargs):
        duration = time.time() - self.start_time

        # Detect loops: same tool called 3+ consecutive times
        loop_detected = self._detect_loop()

        # Record agentic metrics as Langfuse scores
        self.trace.score(name="agent-steps-count", value=float(self.step_count))
        self.trace.score(name="agent-tool-calls-count", value=float(len(self.tool_calls)))
        self.trace.score(name="tool-error-count", value=float(self.tool_errors))
        self.trace.score(name="loop-detected", value=1.0 if loop_detected else 0.0)

        self.trace.update(metadata={
            "tool_calls_sequence": self.tool_calls,
            "agent_duration_s": round(duration, 2),
            "loop_detected": loop_detected
        })

    def _detect_loop(self) -> bool:
        """Returns True if the same tool is called 3+ consecutive times."""
        tools = [c["tool"] for c in self.tool_calls]
        for i in range(len(tools) - 2):
            if tools[i] == tools[i+1] == tools[i+2]:
                return True
        return False


@tool
def search_documentation(query: str) -> str:
    """Search the product documentation."""
    # Your implementation here
    return f"Documentation results for '{query}': [simulated content]"


@tool
def get_pricing(product_id: str) -> dict:
    """Retrieve pricing for a product."""
    return {"product_id": product_id, "price": 299, "currency": "EUR"}


@tool
def create_ticket(title: str, description: str, priority: str = "medium") -> dict:
    """Create a support ticket."""
    return {
        "ticket_id": "TKT-4892",
        "title": title,
        "status": "open",
        "priority": priority
    }


def run_agent(
    user_question: str,
    user_id: str,
    session_id: str,
    max_iterations: int = 10
) -> str:
    trace = langfuse.trace(
        name="agent-run",
        input={"question": user_question},
        user_id=user_id,
        session_id=session_id,
        metadata={
            "agent_type": "react",
            "agent_version": "1.0.0",
            "max_iterations": max_iterations,
            "tools": ["search_documentation", "get_pricing", "create_ticket"]
        }
    )

    # In Langfuse SDK v2+, pass root trace via stateful client or use trace_id
    langfuse_cb = LangfuseCallback(
        trace_id=trace.id,
        root=trace  # Preferred in v2: pass the trace object directly as root
    )
    observability_cb = AgentObservabilityCallback(trace)

    llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.0)
    tools = [search_documentation, get_pricing, create_ticket]

    # Create a ReAct prompt
    react_prompt = PromptTemplate.from_template("""
Answer the following question using the available tools.

Tools available:
{tools}

Tool names: {tool_names}

Question: {input}
{agent_scratchpad}
""")

    agent = create_react_agent(llm, tools, react_prompt)
    executor = AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=False,
        max_iterations=max_iterations,
        handle_parsing_errors=True,
    )

    try:
        result = executor.invoke(
            {"input": user_question},
            config={"callbacks": [langfuse_cb, observability_cb]}
        )
        answer = result["output"]
        trace.update(output={"answer": answer, "completed": True})
        return answer

    except Exception as e:
        trace.update(
            output={"error": str(e), "completed": False},
            metadata={"status": "error", "error_type": type(e).__name__}
        )
        raise
    finally:
        langfuse.flush()
```

---

## Agentic Anomaly Patterns

### Infinite loops

An agent that fails to converge can loop indefinitely. Two protection mechanisms:

The `max_iterations` parameter in `AgentExecutor` (shown above) provides a hard stop. The `_detect_loop()` callback method (shown above) detects and logs the loop as a Langfuse score, enabling alerting.

### Cost escalation

An agent that calls many tools consumes tokens at every intermediate generation. The cost of an agent run can be 10-50x higher than a simple generation.

```python
def on_llm_end(self, response, **kwargs):
    usage = response.llm_output.get("token_usage", {})
    total_tokens = usage.get("total_tokens", 0)

    # Track cumulative token consumption across the agent run
    self.total_tokens = getattr(self, "total_tokens", 0) + total_tokens

    if self.total_tokens > 5000:   # Threshold — calibrate for your use case
        self.trace.score(
            name="high-token-cost",
            value=1.0,
            comment=f"Cumulative tokens: {self.total_tokens}"
        )
```

### Tool hallucination

An agent may call a tool with invented or invalid parameters. Log tool errors in traces (shown in the callback above) and track their frequency.

---

## Agentic SLOs

Classical SLOs (P99 latency, availability) apply, but agents have additional SLOs:

| SLO | Definition | Typical Target |
|---|---|---|
| Completion rate | % of runs that finish without error | > 98% |
| Median steps | Steps needed to resolve a task | < 4 steps |
| Median cost per run | Tokens consumed per successful run | Baseline + 20% max |
| Loop rate | % of runs with detected loop | < 1% |
| Tool error rate | % of tool calls that error | < 5% |
| P99 run duration | End-to-end agent execution time | < 30s (context-dependent) |

These metrics build on Langfuse scores and are exposed via the VictoriaMetrics exporter.

---

## VMAlert Rules for Agents

```yaml
groups:
  - name: llm.agents
    interval: 30s
    rules:

      # Agent loop detected
      - alert: LLMAgentLoopDetected
        expr: |
          langfuse_score_average{score_name="loop-detected"} > 0.05
        for: 5m
        labels:
          severity: warning
          team: platform
        annotations:
          summary: "LLM agent loops detected ({{ $labels.project }})"
          description: >
            More than 5% of agent runs are entering loops.
            Investigate tool behavior and agent prompt.

      # High agent run cost
      - alert: LLMAgentHighCost
        expr: |
          langfuse_score_average{score_name="agent-steps-count"} > 6
        for: 10m
        labels:
          severity: warning
          team: ml
        annotations:
          summary: "LLM agents averaging more than 6 steps — investigate convergence"

      # Agent completion rate dropping
      - alert: LLMAgentLowCompletionRate
        expr: |
          rate(langfuse_generations_total{status="error"}[5m])
          / rate(langfuse_generations_total[5m]) > 0.10
        for: 5m
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "LLM agent error rate > 10%"
```

---

## Why Agentic Observability Is a Control Problem, Not Just a Performance Problem

Agents deployed in sensitive contexts — external API access, data modification, email dispatch, financial calculations — can cause real damage if their behavior drifts. Observability for agents is not only a performance tool: it is a control mechanism.

The complete Langfuse trace of an agent run constitutes an audit trail of every action the agent took and every intermediate decision. In a regulated context (DORA, AI Act), this traceability is a requirement, not a bonus. An agent that cannot be traced cannot be audited, and an agent that cannot be audited cannot be deployed in a regulated environment.

The combination of per-step span logging, score-based anomaly detection, and VictoriaMetrics alerting gives you operational confidence that your agents are behaving within expected bounds — and rapid diagnosis when they are not.

---

*Samuel Desseaux is the founder of Aureonis, an observability stack specialist, agentic AI practitioner, and LLM security engineer. Speaker at FOSDEM 2026 and KubeCon Europe.*
