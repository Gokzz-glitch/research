# Temporal NLP for Legally Enforceable Debt Verification

A **multi-agent NLP system** that ingests unstructured business communications
(WhatsApp chat exports, email threads, ledger PDFs) and autonomously determines
whether a cheque was issued for a *pre-existing, legally enforceable debt* —
the central evidentiary requirement of **Section 138, Negotiable Instruments
Act, 1881**.

---

## Background

Indian courts (including the Supreme Court) frequently dismiss Section 138
cheque-dishonour complaints because the cheque was given as a **"security"** or
an **"advance payment"** rather than for the discharge of an existing liability.
This system replaces hours of manual evidence discovery by using temporal and
causal reasoning to construct a complete timeline of the business relationship
between the parties.

---

## Architecture

```
┌─────────────────────────────────────────────┐
│               Orchestrator                  │
│  (coordinates ingestion, Agent A, Agent B)  │
└───────────┬────────────────┬────────────────┘
            │                │
┌───────────▼──────┐  ┌──────▼──────────────┐
│  Ingestion Layer │  │    Agent A           │
│                  │  │  (Classifier)        │
│  • WhatsApp      │  │                      │
│  • Email / mbox  │  │  Assigns Intent to   │
│  • PDF (ledger)  │  │  each Communication  │
└──────────────────┘  └──────┬───────────────┘
                             │
                      ┌──────▼───────────────┐
                      │    Agent B            │
                      │  (Chronologist)       │
                      │                       │
                      │  Builds timeline,     │
                      │  applies legal rules, │
                      │  produces conclusion  │
                      └───────────────────────┘
```

### Agent A — The Classifier

Scans each parsed message and assigns one of the following **Intent** labels:

| Intent | Legal significance |
|---|---|
| `PRE_EXISTING_DEBT` | Evidence of a prior liability — *supports* Section 138 |
| `REPAYMENT` | Repayment of an earlier amount — *supports* Section 138 |
| `ACKNOWLEDGEMENT` | Acknowledgement of a balance — *supports* Section 138 |
| `ADVANCE_PAYMENT` | Payment before delivery — may *defeat* Section 138 |
| `SECURITY_CHEQUE` | Cheque given as collateral — strongly *defeats* Section 138 |
| `NEUTRAL` | No clear financial signal |

Classification is **rule-based** (compiled regex patterns with confidence
weights), making every decision fully explainable and auditable.

### Agent B — The Chronologist

Anchors each classified communication on the timeline relative to the cheque
date, then applies the legal decision rules in priority order:

1. **Security signals** → `ADVANCE_OR_SECURITY` (defeats Section 138)
2. **Advance signals before cheque, no debt signals** → `ADVANCE_OR_SECURITY`
3. **Debt/repayment signals before cheque, no advance signals** → `LEGALLY_ENFORCEABLE`
4. **Debt signals only after cheque** → `AMBIGUOUS`
5. **Conflicting signals** → `AMBIGUOUS`
6. **Too few high-confidence signals** → `INSUFFICIENT_DATA`

---

## Quick Start

### Installation

```bash
pip install -r requirements.txt
```

### Python API

```python
from datetime import datetime
from src import Orchestrator

orchestrator = Orchestrator()

result = orchestrator.analyse(
    whatsapp_files=["data/samples/sample_whatsapp_debt.txt"],
    email_files=["data/samples/sample_email_thread.eml"],
    cheque_date=datetime(2024, 2, 15),
)

print(result.cheque_context.value)   # "legally_enforceable"
print(result.summary)
print(result.is_legally_enforceable) # True

# Full JSON report
import json
print(json.dumps(result.to_dict(), indent=2))
```

#### Analyse raw text (no files needed)

```python
result = orchestrator.analyse_text(
    whatsapp_text=open("chat_export.txt").read(),
    cheque_date=datetime(2024, 3, 14),
)
```

#### Explain a single message

```python
from src.agents.classifier_agent import ClassifierAgent

agent = ClassifierAgent()
scores = agent.explain("Here is the cheque for Invoice No. INV-2024-0145.")
# {"pre_existing_debt": 0.9}
```

---

## Project Structure

```
research/
├── src/
│   ├── __init__.py              # Exports Orchestrator
│   ├── orchestrator.py          # Pipeline coordinator
│   ├── agents/
│   │   ├── classifier_agent.py  # Agent A – intent classification
│   │   └── chronologist_agent.py# Agent B – timeline & legal conclusion
│   ├── ingestion/
│   │   ├── whatsapp_parser.py   # WhatsApp export parser
│   │   ├── email_parser.py      # RFC 2822 / mbox parser
│   │   └── pdf_parser.py        # PDF ledger parser (pdfminer.six)
│   └── models/
│       └── communication.py     # Data models (Communication, Intent, …)
├── tests/
│   ├── test_classifier_agent.py
│   ├── test_chronologist_agent.py
│   ├── test_ingestion.py
│   └── test_orchestrator.py
├── data/
│   └── samples/
│       ├── sample_whatsapp_debt.txt    # Scenario: pre-existing debt
│       ├── sample_whatsapp_advance.txt # Scenario: advance / security
│       └── sample_email_thread.eml     # Scenario: email payment reminder
├── requirements.txt
└── pytest.ini
```

---

## Running Tests

```bash
pytest tests/ -v
```

All 81 tests cover:
- Intent classification for every label (including edge cases)
- Security intent overriding advance intent
- Timeline construction and chronological sorting
- Timezone-aware vs naive datetime handling
- WhatsApp Android and iOS export formats
- Email single-message and mbox multi-message formats
- PDF date detection (multiple date formats)
- End-to-end orchestrator integration scenarios

---

## Extending the System

### Adding new classification patterns

```python
import re
from src.agents.classifier_agent import ClassifierAgent

agent = ClassifierAgent()
agent.PRE_EXISTING_DEBT_PATTERNS.append(
    (re.compile(r"debit note", re.IGNORECASE), 0.85)
)
```

### Plugging in a transformer-based classifier

Replace `ClassifierAgent._infer` with a call to any HuggingFace model that
returns `(Intent, confidence)` — the rest of the pipeline is unchanged.

---

## Research Context

This system addresses the **temporal and causal reasoning** gap in legal NLP:

- **Beyond NER**: Rather than simply extracting entities, the system infers
  *why* a cheque was issued by reasoning about the *sequence* of business events.
- **Causal chain reconstruction**: Agent B autonomously constructs a causal
  narrative (invoice → acknowledgement → cheque) that mirrors the evidence
  discovery performed by experienced litigators.
- **Explainability**: Every classification decision is backed by a specific
  matched pattern and confidence score, making the output suitable for
  presentation in court proceedings.

### Related Research Areas

- Temporal NLP & event ordering (TimeML, TORQUE benchmark)
- Legal NLP & argument mining
- Causal inference in unstructured text
- Multi-agent systems for document analysis