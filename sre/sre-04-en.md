# LLM Audit Trail with Langfuse: Meeting DORA and AI Act Requirements

*By Samuel Desseaux -Erythix*

---

## The Regulatory Context Is Changing Fast

```
┌───────────────────────────────────────────────────────────────────┐
│  Regulation timeline for LLM systems in the EU                    │
│                                                                   │
│  2025 ─── DORA applicable (Jan 2025)                             │
│           Financial entities + critical ICT providers             │
│           Art. 9: ICT risk documentation                         │
│           Art. 12: logging requirements                           │
│           Art. 17: incident management                           │
│                                                                   │
│  2025 ─── AI Act: prohibited practices ban (Feb 2025)            │
│                                                                   │
│  2026 ─── AI Act: GPAI model obligations (Aug 2025)              │
│                                                                   │
│  2027 ─── AI Act: high-risk systems full compliance              │
│           Art. 11: technical documentation                        │
│           Art. 12: automated logging                              │
│           Art. 14: human oversight requirements                   │
└───────────────────────────────────────────────────────────────────┘

What Langfuse traces directly address:
  DORA Art. 12  ←─── who queried, what input, what output, when
  AI Act Art. 12 ←── automated event logs during operational lifecycle
  AI Act Art. 14 ←── interpretable outputs + anomaly detection via scores
``` The AI Act enters into progressive application through 2027. Both texts have direct implications for LLM systems deployed in production.

This module does not replace legal counsel. It identifies the concrete technical requirements these regulations impose on LLM system operators, and how Langfuse can address them in your architecture.

---

## What DORA Requires for LLM Systems

DORA applies to financial entities and their critical ICT service providers. If you operate an LLM in a financial context — credit analysis assistance, report generation, advisor support tooling — you are potentially within its scope.

**Article 9 — ICT risk management.** You must maintain documentation on your information system components, including third-party services (OpenAI, Anthropic APIs). Langfuse traces constitute part of this operational documentation.

**Article 12 — Logging.** ICT systems must produce activity logs sufficient to detect incidents and support forensic investigation. For an LLM, this includes: who queried the system, with what input, what output was produced, at what timestamp.

**Article 17 — Incident management.** Incidents must be detectable, classifiable, and reportable within defined timeframes. An LLM system producing erroneous responses at scale constitutes an ICT incident under DORA.

---

## What the AI Act Requires

The AI Act classifies LLM systems by risk level. High-risk systems (Article 6) — which include decision-support systems in financial services, education, employment, and essential services — are subject to the strictest requirements.

**Article 12 — Record keeping.** Systems must automatically log relevant events during their operational lifecycle, with sufficient granularity to identify situations that may generate risks.

**Article 13 — Transparency.** Documentation must enable competent authorities to assess system conformity.

**Article 14 — Human oversight.** Operators must be able to interpret system outputs and detect anomalies.

---

## Structuring Langfuse Traces for Auditability

A standard Langfuse trace captures inputs, outputs, and technical metadata. To meet audit requirements, you must enrich these traces with information that enables forensic investigation.

```python
from langfuse import Langfuse
import hashlib
import time
from typing import Optional

langfuse = Langfuse()


def create_auditable_trace(
    user_id: str,
    user_role: str,
    session_id: str,
    request_context: dict,
    ip_address: Optional[str] = None
):
    """
    Creates a Langfuse trace enriched for regulatory auditability.
    IP address is hashed (GDPR pseudonymization) before storage.
    """
    # Hash the IP address (GDPR-compliant pseudonymization)
    ip_hash = (
        hashlib.sha256(ip_address.encode()).hexdigest()[:16]
        if ip_address else None
    )

    trace = langfuse.trace(
        name="llm-request",
        user_id=user_id,
        session_id=session_id,
        metadata={
            # Identity and authorization
            "user_role": user_role,
            "ip_hash": ip_hash,
            "request_timestamp_utc": time.time(),

            # Application context
            "application": request_context.get("application"),
            "feature": request_context.get("feature"),
            "environment": "production",
            "deployment_version": request_context.get("version"),

            # Regulatory references
            "data_classification": request_context.get("data_classification", "internal"),
            "processing_purpose": request_context.get("purpose"),
            "legal_basis": request_context.get("legal_basis"),

            # System traceability
            "request_id": request_context.get("request_id"),
            "upstream_system": request_context.get("upstream"),
            "model_provider": request_context.get("model_provider"),
        }
    )

    return trace
```

### Critical Fields for Audit

| Field | Why | Regulation |
|---|---|---|
| `user_id` | Identifies who used the system | DORA Art. 12, AI Act Art. 12 |
| `user_role` | User's authorization level | AI Act Art. 14 |
| `request_timestamp_utc` | Immutable UTC timestamp | DORA Art. 12 |
| `data_classification` | Sensitivity level of processed data | GDPR |
| `processing_purpose` | Legal basis for processing | GDPR Art. 13 |
| `deployment_version` | Which version was in production | AI Act Art. 12 |
| `ip_hash` | Network traceability without storing raw IP | GDPR (pseudonymization) |
| `model_provider` | Third-party ICT service identification | DORA Art. 9 |

---

## PII Redaction Policy

LLM inputs and outputs may contain personal data. Storing them raw in Langfuse without redaction violates GDPR's data minimization principle.

**Redaction at the instrumentation level** (recommended):

```python
import re
from typing import Optional

def redact_pii(text: str) -> str:
    """
    Basic redaction — extend based on your domain's PII patterns.
    """
    # Email addresses
    text = re.sub(
        r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
        '[EMAIL]', text
    )
    # EU/FR phone numbers
    text = re.sub(
        r'\b(?:\+33|0033|0)[1-9](?:[0-9]{8})\b',
        '[PHONE_FR]', text
    )
    # Belgian phone numbers
    text = re.sub(
        r'\b(?:\+32|0032|0)[1-9](?:[0-9]{7,8})\b',
        '[PHONE_BE]', text
    )
    # IBAN (EU format)
    text = re.sub(
        r'\b[A-Z]{2}\d{2}[A-Z0-9]{4}\d{7}(?:[A-Z0-9]?){0,16}\b',
        '[IBAN]', text
    )
    # Credit card numbers
    text = re.sub(
        r'\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b',
        '[CARD]', text
    )
    # French social security numbers (NIR)
    text = re.sub(
        r'\b[12]\s?\d{2}\s?\d{2}\s?\d{2}\s?\d{3}\s?\d{3}\s?\d{2}\b',
        '[NIR]', text
    )
    return text


# Usage in instrumentation
def traced_generate(user_question: str, user_id: str) -> str:
    redacted_question = redact_pii(user_question)

    trace = langfuse.trace(
        name="llm-request",
        input={"question": redacted_question},  # Redacted input
        user_id=user_id
    )

    # Generate with original input (not redacted — redaction is only for logs)
    response = generate(user_question)

    trace.update(output={"answer": redact_pii(response)})  # Redact output too
    langfuse.flush()
    return response
```

**Redaction at the OTel Collector level** (for centralized redaction without code changes):

```yaml
# otel-collector-config.yaml — processor section
processors:
  redaction:
    allow_all_keys: true
    blocked_values:
      - "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}"  # Emails
      - "[A-Z]{2}\\d{2}[A-Z0-9]{4}\\d{7}"                    # IBAN
    summary: info
```

---

## Retention and Export for Auditors

DORA requires a minimum 5-year retention for major incident logs, and 3 years for routine operational logs. Your Langfuse retention settings must align with these durations.

The Langfuse API enables exporting traces filtered by date range and user:

```python
import json
from datetime import datetime
from langfuse import Langfuse

langfuse = Langfuse()


def export_user_activity(
    user_id: str,
    from_date: str,
    to_date: str,
    output_path: str
):
    """
    Export all traces for a user over a time period.
    Output: JSONL format signable for auditors.
    """
    traces = langfuse.get_traces(
        user_id=user_id,
        from_timestamp=datetime.fromisoformat(from_date),
        to_timestamp=datetime.fromisoformat(to_date),
        limit=1000
    )

    with open(output_path, "w") as f:
        for trace in traces.data:
            record = {
                "trace_id": trace.id,
                "user_id": trace.user_id,
                "timestamp": trace.timestamp.isoformat() if trace.timestamp else None,
                "input": trace.input,
                "output": trace.output,
                "metadata": trace.metadata,
                "latency_ms": trace.latency,
                "scores": [
                    {"name": s.name, "value": s.value}
                    for s in (trace.scores or [])
                ]
            }
            f.write(json.dumps(record) + "\n")

    print(f"Exported {len(traces.data)} traces for user {user_id}")
```

This export integrates into your GDPR data access request procedure, as well as regulatory authority requests (DORA national competent authorities).

---

## SIEM Integration

For environments with a SIEM (Splunk, Elastic SIEM, Microsoft Sentinel, IBM QRadar), Langfuse traces can be forwarded in real-time:

```python
import json
import logging
import logging.handlers


def configure_siem_logger(siem_host: str, siem_port: int) -> logging.Logger:
    siem_logger = logging.getLogger("siem.llm")
    siem_logger.setLevel(logging.INFO)
    handler = logging.handlers.SysLogHandler(address=(siem_host, siem_port))
    handler.setFormatter(logging.Formatter("%(message)s"))
    siem_logger.addHandler(handler)
    return siem_logger


siem = configure_siem_logger("siem.internal", 514)


def log_generation_to_siem(trace, generation):
    siem.info(json.dumps({
        "event_type": "llm_generation",
        "trace_id": trace.id,
        "user_id": trace.user_id,
        "model": generation.model if generation else None,
        "input_tokens": generation.usage.input if (generation and generation.usage) else None,
        "output_tokens": generation.usage.output if (generation and generation.usage) else None,
        "latency_ms": generation.latency if generation else None,
        "timestamp": generation.start_time.isoformat() if (generation and generation.start_time) else None,
        "data_classification": trace.metadata.get("data_classification"),
        "processing_purpose": trace.metadata.get("processing_purpose"),
        "deployment_version": trace.metadata.get("deployment_version"),
    }))
```

This integration enables correlating LLM activity with security events detected by your SIEM (suspicious connection patterns, abnormal volumes, potential data exfiltration).

---

## What Langfuse Does Not Replace

Langfuse is one piece of the compliance architecture, not the entirety. It does not replace:

- A GDPR Article 30 data processing register
- A Data Protection Impact Assessment (DPIA, Article 35)
- AI Act Article 11 technical documentation - the technical documentation required for high-risk systems extends far beyond the scope of an observability tool
- A DORA incident response plan (business continuity plans, resilience testing)

But it constitutes the raw data foundation that these frameworks rely on. Without reliable and auditable traces, these frameworks remain theoretical.

---

