# Project 6: Multi-Line Submission Intake & Document Processing Agent

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Business Context](#2-business-context)
3. [Personas & User Stories](#3-personas--user-stories)
4. [Functional Requirements](#4-functional-requirements)
5. [Non-Functional Requirements](#5-non-functional-requirements)
6. [System Architecture](#6-system-architecture)
7. [Agent Specifications (ADK)](#7-agent-specifications-adk)
8. [A2A Communication Design](#8-a2a-communication-design)
9. [Workflow Design](#9-workflow-design)
10. [Prompt Engineering Strategy](#10-prompt-engineering-strategy)
11. [Prompt Manager Design](#11-prompt-manager-design)
12. [User Profiling Design](#12-user-profiling-design)
13. [Arize Integration](#13-arize-integration)
14. [Deployment Plan](#14-deployment-plan)
15. [Data Models](#15-data-models)
16. [Tech Stack & Tools](#16-tech-stack--tools)
17. [Development Plan — 7-Day Sprint + 1-Day Stretch](#17-development-plan--7-day-sprint--1-day-stretch)
18. [Sample Interactions](#18-sample-interactions)
19. [Acceptance Criteria](#19-acceptance-criteria)
20. [References & Resources](#20-references--resources)

---

## 1. Executive Summary

### What
A document processing pipeline agent that handles incoming insurance submissions across **multiple lines** (Auto, Property, Commercial Liability) for large enterprises. It parses mock submission documents in **parallel**, extracts structured data, validates completeness, classifies by line of business, and routes to the appropriate underwriting queue. Includes a **conversational chatbot** for submission status queries.

### How — 4 Agents

| Agent | Model | Role |
|-------|-------|------|
| **Intake Orchestrator** | Gemini 2.5 Pro | Pipeline controller (non-conversational batch mode) + status chatbot (conversational mode) |
| **Document Parser Agent** | Gemini 2.5 Flash | Extracts structured data from mock submission documents (text/JSON). Runs in **parallel** for multi-doc submissions |
| **Validation & Classification Agent** | Gemini 2.5 Pro | Validates extracted data completeness, classifies line of business, flags missing fields using CoT |
| **Routing & Summary Agent** | Gemini 2.5 Flash | Routes submission to correct underwriting queue, generates intake summary report |

### Workflow Pattern
**Parallel + Sequential**: Parallel document parsing → sequential validation & classification → routing & summary. Plus a separate conversational mode for status queries.

### Dual Mode Operation
1. **Pipeline Mode (Non-conversational):** User submits a batch of documents → system processes all in parallel → returns structured results
2. **Status Chat Mode (Conversational):** User asks "What's the status of submission SUB-001?" → system looks up and responds

### Timeline
7 days MVP, Day 8 stretch. CS intern, medium Python.

### Tech Coverage (All 6 in MVP)

| # | Tech | How Used |
|---|------|----------|
| 1 | **ADK** | 4 agents built with Google ADK; dual-mode orchestrator; UI via `adk web` |
| 2 | **A2A** | Agent Cards for all 4 agents; 1+ real A2A endpoint (Orchestrator → Document Parser) |
| 3 | **Prompt Engineering** | Zero-shot (data extraction), Few-shot (line-of-business classification with examples), CoT (validation reasoning, completeness checking), ReAct (tool-calling for parsing & routing) |
| 4 | **Prompt Manager** | Dict-based Python module, `str.format_map()`, versioned templates |
| 5 | **User Profiling** | Detect operations clerk vs senior underwriter; adapt status responses and summary detail |
| 6 | **Arize** | OTel tracing on all LLM calls; annotation queue for classification quality review |
| 7 | **Agent Engine** | Deploy to Vertex AI Agent Engine or local `adk web` fallback |

---

## 2. Business Context

### Problem
Large insurers receive hundreds of submissions weekly across multiple lines of business. Submissions arrive as bundles of documents — applications, loss runs, financial statements, supplemental questionnaires. Today, operations clerks manually read each document, identify the line of business, extract key data, check for missing information, and route to the right underwriting team. This is slow (30-60 min per submission), error-prone (15-20% missing field rate), and creates bottlenecks.

### Insurance Domain Primer

**3 Lines of Business (MVP):**

| Line | Typical Submission Docs | Key Data to Extract |
|------|------------------------|-------------------|
| **Commercial Auto** | Fleet schedule, driver list, loss history, MVRs | Vehicle count, fleet size, driver ages, accident history, radius of operation |
| **Commercial Property** | Property schedule, building details, loss history | Building values, construction type, occupancy, square footage, protection class |
| **General Liability** | Application, revenue info, operations description | Industry/SIC code, revenue, employee count, prior claims, operations summary |

**Underwriting Queues:**
- Auto Queue → Commercial auto underwriters
- Property Queue → Property underwriters
- GL Queue → Casualty/liability underwriters
- Mixed Queue → Multi-line underwriter (when submission spans 2+ lines)

### What Makes This Unique Among the 6 Projects
- **Only multi-line project** — handles Auto + Property + Commercial in one system
- **Document processing focus** — structured data extraction from unstructured text
- **Dual interaction mode** — pipeline (batch) + conversational (status queries)
- **Operations domain** — not underwriting, claims, or sales — it's intake operations

---

## 3. Personas & User Stories

### Persona 1: Priya — Junior Operations Clerk (6 months)

| Attribute | Value |
|-----------|-------|
| Experience | 6 months in intake operations |
| Comfort | Can identify document types, unsure about completeness requirements |
| Needs | Detailed status reports, clear missing-field explanations, step-by-step guidance |

### Persona 2: David — Senior Underwriting Operations Manager (15 years)

| Attribute | Value |
|-----------|-------|
| Experience | 15 years, manages intake team |
| Comfort | Expert in all lines, knows what underwriters need |
| Needs | Batch processing summaries, quick status lookups, routing accuracy stats |

### User Stories

| ID | Story | Persona |
|----|-------|---------|
| US-01 | As a clerk, I want to submit a bundle of documents and have the system extract key data automatically | Priya |
| US-02 | As a clerk, I want the system to tell me exactly which fields are missing so I can request them | Priya |
| US-03 | As a manager, I want to submit a batch of submissions and get a summary of all classifications and routings | David |
| US-04 | As a clerk, I want to ask "What's the status of SUB-001?" and get an answer via chat | Both |
| US-05 | As a manager, I want to see routing accuracy and processing stats | David |
| US-06 | As a clerk, I want the system to correctly classify which line of business each document belongs to | Priya |

---

## 4. Functional Requirements

### MVP (Days 1-7)

| ID | Requirement |
|----|-------------|
| FR-01 | Accept mock submission documents (JSON/text files representing applications, schedules, loss runs) |
| FR-02 | **Parallel document parsing** — parse multiple documents in a submission simultaneously |
| FR-03 | Extract structured data from each document (key fields per line of business) |
| FR-04 | Validate completeness — check for required fields, flag what's missing |
| FR-05 | Classify line of business (Auto, Property, GL, or Multi-line) |
| FR-06 | Route to correct underwriting queue based on classification |
| FR-07 | Generate intake summary with extracted data, validation results, and routing |
| FR-08 | **Pipeline mode** — process a batch of submissions non-conversationally |
| FR-09 | **Status chat mode** — answer "What's the status of submission X?" conversationally |
| FR-10 | Adapt summary detail based on user profile (clerk=detailed, manager=concise) |
| FR-11 | All data stored in JSON files (no database) |
| FR-12 | Support at least 3 mock submissions in demo (one per line of business) |

### Stretch (Day 8)

| ID | Requirement |
|----|-------------|
| FR-13 | Multi-line submission detection (single submission spanning Auto + Property) |
| FR-14 | Priority scoring (urgent vs routine based on effective date, premium size) |
| FR-15 | Batch analytics — processing time, completeness rates, routing distribution |

---

## 5. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 | Process a single submission (3-5 docs) in < 30 seconds |
| NFR-02 | All LLM calls traced via OpenTelemetry → Arize |
| NFR-03 | Mock submission database: 5+ submissions with 3-5 documents each |
| NFR-04 | Runs locally via `adk web` or deployed to Agent Engine |
| NFR-05 | User profiles persisted in JSON files between sessions |

---

## 6. System Architecture

### High-Level Flow

```
User submits documents (via adk web)
    │
    ▼
┌──────────────────────────────┐
│     Intake Orchestrator      │  (Gemini Pro — dual mode)
│  Mode 1: Pipeline (batch)    │
│  Mode 2: Status chat         │
└──────────┬───────────────────┘
           │
     ┌─────┴──────┐   (parallel fan-out per document)
     ▼            ▼
┌──────────┐ ┌──────────┐
│ Doc      │ │ Doc      │  (Gemini Flash — one per document)
│ Parser   │ │ Parser   │
│ (doc 1)  │ │ (doc 2)  │
└────┬─────┘ └────┬─────┘
     └──────┬─────┘
            ▼
┌──────────────────────────────┐
│  Validation & Classification │  (Gemini Pro — sequential)
│  - Check completeness        │
│  - Classify line of business │
│  - Flag missing fields       │
└──────────┬───────────────────┘
           ▼
┌──────────────────────────────┐
│    Routing & Summary Agent   │  (Gemini Flash)
│  - Route to UW queue         │
│  - Generate intake report    │
└──────────────────────────────┘
```

### Directory Structure

```
project-6-multiline-intake/
├── agents/
│   ├── __init__.py
│   ├── orchestrator.py          # Intake Orchestrator (root agent, dual-mode)
│   ├── document_parser.py       # Document Parser Agent
│   ├── validator.py             # Validation & Classification Agent
│   └── router.py                # Routing & Summary Agent
├── prompts/
│   ├── __init__.py
│   └── prompt_manager.py        # Dict-based prompt manager
├── profiles/
│   ├── __init__.py
│   └── user_profile_manager.py  # JSON-based user profiling
├── a2a/
│   ├── agent_cards/
│   │   ├── orchestrator_card.json
│   │   ├── document_parser_card.json
│   │   ├── validator_card.json
│   │   └── router_card.json
│   └── a2a_client.py            # A2A task delegation
├── data/
│   ├── submissions/             # Mock submission bundles
│   │   ├── SUB-001/             # Auto fleet submission
│   │   │   ├── application.json
│   │   │   ├── fleet_schedule.json
│   │   │   └── loss_history.json
│   │   ├── SUB-002/             # Property submission
│   │   │   ├── application.json
│   │   │   ├── property_schedule.json
│   │   │   └── building_details.json
│   │   ├── SUB-003/             # GL submission
│   │   │   ├── application.json
│   │   │   ├── revenue_info.json
│   │   │   └── operations_description.json
│   │   ├── SUB-004/             # Multi-line (Auto + Property)
│   │   │   ├── application.json
│   │   │   ├── fleet_schedule.json
│   │   │   └── property_schedule.json
│   │   └── SUB-005/             # GL with missing fields
│   │       ├── application.json
│   │       └── operations_description.json
│   ├── processed/               # Output — processed submission results
│   │   └── {submission_id}.json
│   └── user_profiles/
│       └── {user_id}.json
├── tracing/
│   └── arize_setup.py           # OTel + Arize configuration
├── agent.yaml                   # ADK agent config (root agent)
└── requirements.txt
```

---

## 7. Agent Specifications (ADK)

### 7.1 Intake Orchestrator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `intake_orchestrator` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Dual-mode: pipeline controller + status chatbot |

**ADK Tools:**

```python
def process_submission(submission_id: str) -> dict:
    """
    Pipeline mode: Process an entire submission.
    1. Load all documents from data/submissions/{submission_id}/
    2. Fan out to Document Parser (parallel, one per doc)
    3. Send parsed data to Validator & Classifier
    4. Send to Router for queue assignment + summary
    Returns: full intake result with routing decision.
    """

def process_batch(submission_ids: list[str]) -> list[dict]:
    """
    Batch mode: Process multiple submissions.
    Calls process_submission for each. Returns list of results.
    """

def get_submission_status(submission_id: str) -> dict:
    """
    Chat mode: Look up processed result from data/processed/{submission_id}.json
    Returns status, routing, any missing fields.
    """

def parse_documents(submission_id: str, document_paths: list[str]) -> list[dict]:
    """
    Delegates to Document Parser Agent. Runs PARALLEL parsing.
    Returns list of extracted data dicts, one per document.
    """

def validate_and_classify(submission_id: str, parsed_data: list[dict]) -> dict:
    """
    Delegates to Validation & Classification Agent.
    Returns: {line_of_business, completeness_score, missing_fields, validation_notes}
    """

def route_and_summarize(
    submission_id: str, parsed_data: list[dict],
    validation_result: dict, user_profile: dict
) -> dict:
    """
    Delegates to Routing & Summary Agent.
    Returns: {queue, summary_report, priority}
    """

def manage_user_profile(user_id: str, action: str, data: dict = None) -> dict:
    """Get/update user profile. Actions: 'get', 'update', 'get_or_create'"""

def get_prompt(template_name: str, version: str = "v1", variables: dict = None) -> str:
    """Retrieve rendered prompt from Prompt Manager."""
```

**ADK Config (`agent.yaml`):**

```yaml
name: intake_orchestrator
model: gemini-2.5-pro
description: >
  Dual-mode agent for insurance submission intake.
  Pipeline mode: processes document bundles automatically.
  Chat mode: answers submission status queries.
tools:
  - process_submission
  - process_batch
  - get_submission_status
  - parse_documents
  - validate_and_classify
  - route_and_summarize
  - manage_user_profile
  - get_prompt
instruction: |
  You are an insurance submission intake assistant with two modes:

  PIPELINE MODE (user says "process submission SUB-XXX" or "process batch"):
  1. Load documents from the submission folder
  2. Parse all documents in parallel via parse_documents
  3. Validate completeness and classify line of business via validate_and_classify
  4. Route to underwriting queue and generate summary via route_and_summarize
  5. Save result to data/processed/
  6. Return the intake summary

  CHAT MODE (user asks about status, or general questions):
  1. Look up submission via get_submission_status
  2. Respond conversationally, adapting detail to user profile

  On first interaction, ask the user their role to set up profiling.
sub_agents:
  - document_parser
  - validator
  - router
```

### 7.2 Document Parser Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `document_parser` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Extracts structured data from mock submission documents |

**ADK Tools:**

```python
def load_document(file_path: str) -> str:
    """Load a mock document (JSON/text) from disk."""

def extract_fields(document_content: str, document_type: str) -> dict:
    """
    Extract structured fields from document content.
    Uses zero-shot extraction prompt.
    document_type: 'application', 'fleet_schedule', 'property_schedule',
                   'loss_history', 'building_details', 'revenue_info',
                   'operations_description'
    Returns: {document_type, extracted_fields: {field: value}, confidence: float}
    """
```

**Key behavior:** Runs in **parallel** — Orchestrator sends one `extract_fields` call per document, all execute concurrently.

### 7.3 Validation & Classification Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `validator` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Validates completeness, classifies line of business using CoT |

**ADK Tools:**

```python
def validate_completeness(parsed_data: list[dict], line_of_business: str) -> dict:
    """
    Check if all required fields are present for the given line.
    Returns: {completeness_score: float, missing_fields: [...], validation_notes: str}
    """

def classify_line_of_business(parsed_data: list[dict]) -> dict:
    """
    Determine which line of business this submission belongs to.
    Uses few-shot + CoT prompt with example classifications.
    Returns: {primary_line: str, secondary_lines: [...], confidence: float, reasoning: str}
    """
```

**Required Fields per Line (for validation):**

| Line | Required Fields |
|------|----------------|
| Auto | Vehicle count, fleet schedule, driver list, loss history (3yr), radius of operation |
| Property | Building values, construction type, occupancy, square footage, protection class |
| GL | Industry/SIC code, annual revenue, employee count, operations description, prior claims |

### 7.4 Routing & Summary Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `router` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Routes to underwriting queue, generates intake summary |

**ADK Tools:**

```python
def determine_routing(
    line_of_business: str, validation_result: dict
) -> dict:
    """
    Route submission to correct queue.
    Rules: auto→Auto Queue, property→Property Queue, gl→GL Queue,
           multi-line→Mixed Queue, incomplete→Hold Queue (with missing field list)
    Returns: {queue: str, routing_reason: str, priority: 'routine'|'urgent'}
    """

def generate_intake_summary(
    submission_id: str, parsed_data: list[dict],
    validation_result: dict, routing: dict, user_profile: dict
) -> str:
    """
    Generate intake summary report.
    Adapts detail: clerk=verbose with field-by-field breakdown,
    manager=concise with key metrics and routing decision.
    """
```

---

## 8. A2A Communication Design

### 8.1 Agent Cards

Each agent has a static JSON Agent Card. Example for Document Parser:

```json
{
  "name": "document_parser",
  "description": "Extracts structured data from insurance submission documents including applications, schedules, and loss runs",
  "url": "http://localhost:8001/document_parser",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "parse_application",
      "name": "Parse Application Form",
      "description": "Extract fields from an insurance application"
    },
    {
      "id": "parse_schedule",
      "name": "Parse Schedule Document",
      "description": "Extract fleet or property schedule data"
    },
    {
      "id": "parse_loss_history",
      "name": "Parse Loss History",
      "description": "Extract loss run data and claims history"
    }
  ]
}
```

### 8.2 A2A Task Flow

**MVP:** Orchestrator → Document Parser via real A2A (1 endpoint minimum).

```python
# a2a/a2a_client.py
import json
import uuid

class A2AClient:
    def __init__(self, agent_card_path: str):
        with open(agent_card_path) as f:
            self.card = json.load(f)

    def send_task(self, skill_id: str, input_data: dict) -> dict:
        task = {
            "id": str(uuid.uuid4()),
            "skill_id": skill_id,
            "input": input_data,
            "status": "pending"
        }
        # MVP: direct function call simulating A2A protocol
        # Stretch: POST to self.card["url"]
        return self._execute_local(task)
```

### 8.3 Agent Cards to Create

- `orchestrator_card.json` — skills: process_submission, get_status
- `document_parser_card.json` — skills: parse_application, parse_schedule, parse_loss_history
- `validator_card.json` — skills: validate_completeness, classify_line
- `router_card.json` — skills: route_submission, generate_summary

**Stretch:** All 3 sub-agents callable via full A2A HTTP endpoints.

---

## 9. Workflow Design

### 9.1 Pipeline Mode — Parallel + Sequential

```
Step 1: LOAD SUBMISSION (Sequential)
  Orchestrator loads all docs from data/submissions/{id}/
  Output: list of document paths + content

Step 2: DOCUMENT PARSING (Parallel fan-out)
  Orchestrator → Document Parser Agent
    ├── parse doc 1 (application.json)     ─┐
    ├── parse doc 2 (fleet_schedule.json)  ─┤ PARALLEL
    └── parse doc 3 (loss_history.json)    ─┘
  Output: list of parsed_data dicts

Step 3: VALIDATION & CLASSIFICATION (Sequential)
  Orchestrator → Validation & Classification Agent
    1. classify_line_of_business(parsed_data)
    2. validate_completeness(parsed_data, line)
  Output: {line_of_business, completeness_score, missing_fields}

Step 4: ROUTING & SUMMARY (Sequential)
  Orchestrator → Routing & Summary Agent
    1. determine_routing(line, validation_result)
    2. generate_intake_summary(all_data, user_profile)
  Output: {queue, summary_report}

Step 5: PERSIST RESULT
  Save to data/processed/{submission_id}.json
```

### 9.2 Chat Mode — Status Query

```
User: "What's the status of SUB-001?"
  │
  ▼
Orchestrator: get_submission_status("SUB-001")
  │
  ▼
Load data/processed/SUB-001.json → format response based on user profile
```

### 9.3 State Machine

```
Pipeline: LOADED → PARSING → PARSED → VALIDATING → VALIDATED → ROUTING → COMPLETE
Chat: QUERY → LOOKUP → RESPOND
```

### 9.4 Error Handling

| Scenario | Action |
|----------|--------|
| Document can't be parsed | Mark as "parse_failed", continue with remaining docs |
| All required fields missing | Route to Hold Queue, list all missing fields |
| Unknown line of business | Flag for manual classification, route to Mixed Queue |
| Submission ID not found (chat mode) | "Submission not found. Available: SUB-001, SUB-002..." |

---

## 10. Prompt Engineering Strategy

### 10.1 Prompt Types Used

| Type | Where Used | Why |
|------|-----------|-----|
| **Zero-shot** | Document field extraction | Structured extraction from clear document formats |
| **Few-shot** | Line-of-business classification (with 3 examples) | Classification needs calibration examples |
| **CoT** | Validation reasoning, completeness assessment | Step-by-step field checking needs transparency |
| **ReAct** | Orchestrator tool calls, parser tool calls | Agent needs Reasoning + Action loop |

### 10.2 Key Prompts

#### Document Extraction (Zero-shot)

```python
DOCUMENT_EXTRACTION = {
    "v1": """Extract structured data from this insurance submission document.

Document Type: {document_type}
Document Content:
{document_content}

Extract ALL relevant fields. Return JSON:
{{
  "document_type": "{document_type}",
  "extracted_fields": {{
    "field_name": "value",
    ...
  }},
  "confidence": 0.0-1.0,
  "notes": "any extraction issues or ambiguities"
}}

For {document_type}, key fields to look for:
- application: insured_name, business_type, industry, revenue, employee_count, effective_date, requested_limits
- fleet_schedule: vehicle_count, vehicle_types, model_years, VINs, garaging_locations
- property_schedule: building_addresses, construction_types, square_footage, building_values, occupancy
- loss_history: claim_count, total_incurred, largest_loss, loss_years, claim_descriptions
- revenue_info: annual_revenue, revenue_by_segment, revenue_trend
- operations_description: primary_operations, locations, hazards, safety_programs"""
}
```

#### Line-of-Business Classification (Few-shot + CoT)

```python
LOB_CLASSIFICATION = {
    "v1": """Classify this submission's line of business based on the extracted documents.

Extracted data from all documents:
{parsed_data_json}

EXAMPLES:
---
Example 1: Documents contain fleet_schedule (50 vehicles), driver_list, auto loss history
Classification: commercial_auto
Reasoning: Fleet schedule and driver list are auto-specific. No property or GL docs.
---
Example 2: Documents contain property_schedule (3 buildings), building_details, property loss history
Classification: commercial_property
Reasoning: Property schedule and building details indicate property line. No auto or GL docs.
---
Example 3: Documents contain application (consulting firm), revenue_info, operations_description, fleet_schedule
Classification: multi_line (general_liability + commercial_auto)
Reasoning: Operations description + revenue = GL indicators. Fleet schedule = auto indicator. Two lines present.
---

Now classify the given submission. Think step-by-step:
1. What document types are present?
2. Which line-specific indicators exist?
3. Is this single-line or multi-line?

Return JSON:
{{
  "primary_line": "commercial_auto|commercial_property|general_liability|multi_line",
  "secondary_lines": [],
  "confidence": 0.0-1.0,
  "reasoning": "step-by-step reasoning"
}}"""
}
```

#### Completeness Validation (CoT)

```python
COMPLETENESS_VALIDATION = {
    "v1": """Validate the completeness of this {line_of_business} submission.

Extracted data:
{parsed_data_json}

Required fields for {line_of_business}:
{required_fields_json}

Think step-by-step:
1. List each required field
2. Check if it was extracted (present and non-null)
3. Rate the quality of each extracted value (complete, partial, missing)
4. Calculate overall completeness score

Return JSON:
{{
  "completeness_score": 0.0-1.0,
  "field_status": {{
    "field_name": {{"status": "complete|partial|missing", "value": "extracted value or null", "note": "..."}}
  }},
  "missing_fields": ["list of missing field names"],
  "validation_notes": "overall assessment"
}}"""
}
```

#### Routing Decision (Zero-shot)

```python
ROUTING_DECISION = {
    "v1": """Determine the underwriting queue for this submission.

Line of Business: {line_of_business}
Completeness Score: {completeness_score}
Missing Fields: {missing_fields_json}

Routing Rules:
- commercial_auto → "Auto Queue"
- commercial_property → "Property Queue"
- general_liability → "GL Queue"
- multi_line → "Mixed Queue"
- completeness_score < 0.7 → "Hold Queue" (regardless of line)

Return JSON:
{{
  "queue": "queue name",
  "routing_reason": "why this queue",
  "priority": "routine|urgent",
  "action_needed": "none|request_missing_info|manual_review"
}}"""
}
```

#### User Profiling (Zero-shot)

```python
USER_PROFILING = {
    "v1": """Determine this user's role and experience from their input.

User message: {user_message}
Conversation history: {conversation_history}

Indicators of OPERATIONS CLERK (junior):
- Asks what fields are required
- Processes one submission at a time
- Needs explanations of validation results

Indicators of OPERATIONS MANAGER (senior):
- Submits batches
- Asks about stats and routing accuracy
- Uses technical insurance terms
- Wants concise summaries

Return JSON:
{{
  "role": "clerk" or "manager",
  "confidence": 0.0-1.0,
  "signals": ["observed indicators"]
}}"""
}
```

---

## 11. Prompt Manager Design

### 11.1 Implementation

```python
# prompts/prompt_manager.py

from typing import Optional

TEMPLATES = {
    "document_extraction": {
        "v1": """Extract structured data from this insurance submission document..."""
    },
    "lob_classification": {
        "v1": """Classify this submission's line of business..."""
    },
    "completeness_validation": {
        "v1": """Validate the completeness of this {line_of_business} submission..."""
    },
    "routing_decision": {
        "v1": """Determine the underwriting queue for this submission..."""
    },
    "user_profiling": {
        "v1": """Determine this user's role and experience..."""
    },
    "intake_summary": {
        "v1": """Generate an intake summary report for submission {submission_id}..."""
    }
}

class PromptManager:
    def __init__(self):
        self._templates = TEMPLATES
        self._usage_log = []

    def get_prompt(
        self,
        template_name: str,
        version: str = "v1",
        variables: Optional[dict] = None
    ) -> str:
        template = self._templates[template_name][version]
        rendered = template.format_map(variables or {})
        self._usage_log.append({
            "template": template_name,
            "version": version,
            "variables_keys": list((variables or {}).keys())
        })
        return rendered

    def list_templates(self) -> list[str]:
        return list(self._templates.keys())

    def get_versions(self, template_name: str) -> list[str]:
        return [v for v, t in self._templates[template_name].items() if t is not None]

    def get_usage_log(self) -> list[dict]:
        return self._usage_log
```

### 11.2 Usage Pattern

```python
pm = PromptManager()

# In document parser
prompt = pm.get_prompt("document_extraction", "v1", {
    "document_type": "fleet_schedule",
    "document_content": doc_text
})

# In validator
prompt = pm.get_prompt("completeness_validation", "v1", {
    "line_of_business": "commercial_auto",
    "parsed_data_json": json.dumps(parsed),
    "required_fields_json": json.dumps(required_fields["commercial_auto"])
})
```

**Key rules:** NO Jinja2 — `str.format_map()` only. Templates are Python string constants. Version tracked in Arize spans.

---

## 12. User Profiling Design

### 12.1 Profile Schema

```json
{
  "user_id": "priya_clerk_01",
  "name": "Priya Patel",
  "role": "clerk",
  "experience_level": "junior",
  "preferred_detail": "verbose",
  "interaction_count": 0,
  "created_at": "2026-04-18T10:00:00Z",
  "updated_at": "2026-04-18T10:00:00Z"
}
```

### 12.2 Profile Manager

```python
# profiles/user_profile_manager.py

import json, os
from datetime import datetime

PROFILES_DIR = "data/user_profiles"

class UserProfileManager:
    def get_or_create(self, user_id: str, name: str = "") -> dict:
        path = os.path.join(PROFILES_DIR, f"{user_id}.json")
        if os.path.exists(path):
            with open(path) as f:
                return json.load(f)
        profile = {
            "user_id": user_id, "name": name,
            "role": "unknown", "experience_level": "unknown",
            "preferred_detail": "verbose",
            "interaction_count": 0,
            "created_at": datetime.utcnow().isoformat(),
            "updated_at": datetime.utcnow().isoformat()
        }
        self._save(profile)
        return profile

    def update(self, user_id: str, updates: dict) -> dict:
        profile = self.get_or_create(user_id)
        profile.update(updates)
        profile["updated_at"] = datetime.utcnow().isoformat()
        profile["interaction_count"] += 1
        self._save(profile)
        return profile

    def _save(self, profile: dict):
        os.makedirs(PROFILES_DIR, exist_ok=True)
        path = os.path.join(PROFILES_DIR, f"{profile['user_id']}.json")
        with open(path, "w") as f:
            json.dump(profile, f, indent=2)
```

### 12.3 Adaptive Behavior

| Attribute | Clerk (Priya) | Manager (David) |
|-----------|---------------|-----------------|
| Pipeline output | Field-by-field breakdown, explains missing fields | Summary table: submission ID, line, queue, score |
| Status query | Full details with next steps | One-line status + queue |
| Validation results | "Revenue field is missing — ask the broker for annual revenue" | "Missing: revenue. Completeness: 80%" |
| Batch results | Not applicable (processes one at a time) | Summary stats: 5 processed, 3 auto, 1 property, 1 hold |

### 12.4 Profiling Trigger
On first interaction, Orchestrator asks: "Are you processing submissions as an operations clerk, or reviewing as a manager?" Then persists role in JSON profile.

---

## 13. Arize Integration

### 13.1 Setup

```python
# tracing/arize_setup.py

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

def setup_arize_tracing(space_id: str, api_key: str):
    provider = TracerProvider()
    exporter = OTLPSpanExporter(
        endpoint="https://otlp.arize.com:443",
        headers={"space_id": space_id, "api_key": api_key}
    )
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    return trace.get_tracer("multiline-intake-agent")
```

### 13.2 What Gets Traced

| Span | Key Attributes |
|------|---------------|
| `orchestrator.process_submission` | `submission_id`, `mode`, `user_role`, `documents_count` |
| `document_parser.extract` | `document_type`, `fields_extracted_count`, `confidence` |
| `validator.classify` | `line_of_business`, `classification_confidence`, `prompt_version` |
| `validator.validate` | `completeness_score`, `missing_fields_count` |
| `router.route` | `queue`, `priority`, `action_needed` |
| Each LLM call | `model`, `prompt_template`, `prompt_version`, `token_count`, `latency_ms` |

### 13.3 Annotation Queue

**"Classification Quality Review"** annotation queue:
- **What gets queued:** All `validator.classify` spans
- **Annotator labels:** `correct_classification`, `wrong_line`, `should_be_multiline`, `ambiguous`
- **Purpose:** Review whether LOB classification is accurate across submissions

### 13.4 Dashboard Widgets (MVP)

1. Classification accuracy (via annotation results)
2. Completeness score distribution across submissions
3. Routing distribution (which queues get most submissions?)
4. Latency by agent

---

## 14. Deployment Plan

### 14.1 Local Development

```bash
pip install google-adk google-genai opentelemetry-api opentelemetry-sdk \
    opentelemetry-exporter-otlp arize-otel

export GOOGLE_API_KEY="your-gemini-api-key"
export ARIZE_SPACE_ID="your-space-id"
export ARIZE_API_KEY="your-api-key"

adk web
```

### 14.2 Agent Engine Deployment

```python
from google.cloud import aiplatform
from google.cloud.aiplatform import agent_engines

app = agent_engines.create(
    display_name="multiline-intake-agent",
    agent_engine_config={
        "agent": "agents/orchestrator.py",
        "requirements": "requirements.txt"
    }
)
print(f"Deployed: {app.resource_name}")
```

**Fallback:** If Agent Engine has issues, demo runs locally via `adk web`.

### 14.3 Environment Variables

| Variable | Purpose |
|----------|---------|
| `GOOGLE_API_KEY` | Gemini API access |
| `ARIZE_SPACE_ID` | Arize Cloud project |
| `ARIZE_API_KEY` | Arize Cloud auth |
| `DEPLOYMENT_MODE` | `local` or `agent_engine` |

---

## 15. Data Models

### 15.1 Mock Submission Document (Auto Application)

```json
{
  "document_type": "application",
  "submission_id": "SUB-001",
  "insured_name": "Pacific Freight Lines Inc.",
  "business_type": "trucking_and_logistics",
  "industry_sic": "4213",
  "annual_revenue": 45000000,
  "employee_count": 350,
  "effective_date": "2026-07-01",
  "requested_coverage": ["commercial_auto"],
  "requested_limits": {
    "combined_single_limit": 1000000,
    "aggregate": 5000000
  },
  "years_in_business": 18,
  "prior_carrier": "Old Republic Insurance"
}
```

### 15.2 Mock Fleet Schedule

```json
{
  "document_type": "fleet_schedule",
  "submission_id": "SUB-001",
  "vehicle_count": 85,
  "vehicles": [
    {
      "vin": "1HGBH41JXMN109186",
      "year": 2023,
      "make": "Freightliner",
      "model": "Cascadia",
      "type": "tractor",
      "value": 145000,
      "garaging_zip": "90001"
    }
  ],
  "radius_of_operation": "500_miles",
  "states_operated": ["CA", "NV", "AZ", "OR", "WA"]
}
```

### 15.3 Mock Property Schedule

```json
{
  "document_type": "property_schedule",
  "submission_id": "SUB-002",
  "properties": [
    {
      "address": "1200 Industrial Blvd, Chicago, IL 60601",
      "construction_type": "fire_resistive",
      "occupancy": "warehouse_distribution",
      "square_footage": 75000,
      "year_built": 2005,
      "building_value": 8500000,
      "contents_value": 3200000,
      "protection_class": 3,
      "sprinkler": true
    }
  ]
}
```

### 15.4 Parsed Data Output (from Document Parser)

```json
{
  "document_type": "fleet_schedule",
  "submission_id": "SUB-001",
  "extracted_fields": {
    "vehicle_count": 85,
    "vehicle_types": ["tractor", "trailer", "cargo_van"],
    "avg_vehicle_age": 3.2,
    "total_fleet_value": 12325000,
    "primary_garaging_state": "CA",
    "radius_of_operation": "500_miles",
    "states_operated": ["CA", "NV", "AZ", "OR", "WA"]
  },
  "confidence": 0.95,
  "notes": "All fields extracted cleanly"
}
```

### 15.5 Processed Submission Result

```json
{
  "submission_id": "SUB-001",
  "status": "complete",
  "insured_name": "Pacific Freight Lines Inc.",
  "documents_processed": 3,
  "classification": {
    "primary_line": "commercial_auto",
    "confidence": 0.97,
    "reasoning": "Fleet schedule with 85 vehicles + driver list + auto loss history"
  },
  "validation": {
    "completeness_score": 0.92,
    "missing_fields": ["driver_mvr_reports"],
    "field_count": {"complete": 11, "partial": 1, "missing": 1}
  },
  "routing": {
    "queue": "Auto Queue",
    "priority": "routine",
    "action_needed": "request_missing_info"
  },
  "summary": "Auto fleet submission for Pacific Freight Lines (85 vehicles). Mostly complete, missing MVR reports. Routed to Auto Queue.",
  "processed_at": "2026-04-18T14:30:00Z"
}
```

### 15.6 Mock Data Plan

| Submission | Line | Docs | Scenario |
|-----------|------|------|----------|
| SUB-001 | Auto | application, fleet_schedule, loss_history | Complete auto fleet, 85 vehicles |
| SUB-002 | Property | application, property_schedule, building_details | 2-building warehouse/office, complete |
| SUB-003 | GL | application, revenue_info, operations_description | Tech consulting firm, complete |
| SUB-004 | Multi-line | application, fleet_schedule, property_schedule | Auto + Property combined (stretch) |
| SUB-005 | GL | application, operations_description | Missing revenue_info — tests incomplete validation |

---

## 16. Tech Stack & Tools

| Component | Technology |
|-----------|-----------|
| Language | Python 3.11+ |
| Agent Framework | Google ADK (`google-adk`) |
| LLM | Gemini 2.5 Pro (orchestrator, validator) / Flash (parser, router) |
| Agent Communication | A2A Protocol (Agent Cards + task delegation) |
| Prompt Management | Dict-based Python module with `str.format_map()` |
| User Profiles | JSON files in `data/user_profiles/` |
| Mock Data | JSON files in `data/submissions/` |
| Observability | OpenTelemetry + Arize Cloud |
| Deployment | Vertex AI Agent Engine / local `adk web` |
| UI | ADK Web (built-in) |
| Version Control | Git |

---

## 17. Development Plan — 7-Day Sprint + 1-Day Stretch

### Day 1: Setup + Mock Data
- [ ] Initialize project structure (directories, `requirements.txt`, `agent.yaml`)
- [ ] Install ADK, Gemini SDK, OTel, Arize packages
- [ ] Create 5 mock submission folders with 3-5 documents each (JSON files)
- [ ] Verify Gemini API key works with a test call
- [ ] **Checkpoint:** Can load and print documents from each submission folder

### Day 2: Prompt Manager + Core Prompts
- [ ] Build `prompts/prompt_manager.py`
- [ ] Write all 6 prompt templates (v1): extraction, classification, validation, routing, profiling, summary
- [ ] Unit test: render each template with sample variables
- [ ] **Checkpoint:** All prompts renderable, no format errors

### Day 3: Document Parser + Validator
- [ ] Build `document_parser.py` — load docs, extract fields with zero-shot prompt
- [ ] Build `validator.py` — classify LOB (few-shot+CoT), validate completeness (CoT)
- [ ] Test: parse SUB-001 docs → get extracted fields JSON
- [ ] Test: classify SUB-001 → "commercial_auto" with reasoning
- [ ] **Checkpoint:** Parser and validator work independently

### Day 4: Router + Orchestrator
- [ ] Build `router.py` — routing rules + summary generation
- [ ] Build `orchestrator.py` — pipeline mode + chat mode
- [ ] Wire Orchestrator → sub-agents via ADK tool calls
- [ ] Test end-to-end: SUB-001 → parse → validate → route → summary
- [ ] **Checkpoint:** Full pipeline works via `adk web`

### Day 5: Parallel Execution + User Profiling
- [ ] Implement parallel document parsing (all docs in a submission parsed concurrently)
- [ ] Build `user_profile_manager.py` + profiling flow
- [ ] Wire profiling into Orchestrator (adapt output: clerk vs manager)
- [ ] Test with Priya (clerk) and David (manager) personas
- [ ] Test status chat mode: "What's the status of SUB-001?"
- [ ] **Checkpoint:** Parallel parsing visible, profiles persist, both modes work

### Day 6: A2A + Arize
- [ ] Create 4 Agent Card JSONs
- [ ] Implement A2A client — Orchestrator → Document Parser via A2A task
- [ ] Set up `arize_setup.py` with OTel tracing
- [ ] Add trace spans to all agents with proper attributes
- [ ] Set up annotation queue for classification quality review
- [ ] **Checkpoint:** Traces visible in Arize, Agent Cards loadable

### Day 7: Integration + Demo Prep
- [ ] Demo run: process all 5 submissions (3 single-line, 1 multi-line, 1 incomplete)
- [ ] Test clerk flow (Priya) — verbose output, field explanations
- [ ] Test manager flow (David) — batch summary, concise
- [ ] Test chat mode status queries
- [ ] Deploy to Agent Engine (or confirm local `adk web` fallback)
- [ ] **Checkpoint:** Demo-ready with all scenarios passing

### Day 8 (Stretch):
- [ ] Multi-line submission detection and split routing
- [ ] Priority scoring (urgent vs routine)
- [ ] Batch analytics dashboard in Arize
- [ ] Prompt v2 experiments + Arize comparison

---

## 18. Sample Interactions

### 18.1 Pipeline Mode — Junior Clerk (Priya)

```
CLERK: Hi, I need to process a new submission.

AGENT: Welcome! Before we start, are you processing submissions as an
       operations clerk, or reviewing as a manager?

CLERK: I'm a clerk, still pretty new.

AGENT: Got it — I'll walk you through everything in detail.
       Which submission would you like to process?

CLERK: SUB-001

AGENT: Processing submission SUB-001...

       📄 Loading documents:
       - application.json
       - fleet_schedule.json
       - loss_history.json

       ⚡ Parsing all 3 documents in parallel...

       ✅ Parsing complete. Here's what I extracted:

       **Application:**
       - Insured: Pacific Freight Lines Inc.
       - Industry: Trucking & Logistics (SIC 4213)
       - Revenue: $45,000,000
       - Employees: 350
       - Requested: Commercial Auto, $1M CSL / $5M aggregate

       **Fleet Schedule:**
       - 85 vehicles (tractors, trailers, cargo vans)
       - Total fleet value: $12,325,000
       - Primary state: California
       - Radius: 500 miles (5-state operation)

       **Loss History:**
       - 3-year history: 12 claims, $892,000 total incurred
       - Largest loss: $185,000 (jackknife incident, 2024)

       🔍 Classification: **Commercial Auto** (97% confidence)
       Reasoning: Fleet schedule with 85 vehicles + driver list + auto
       loss history all point to commercial auto line.

       ✅ Completeness: **92%** (11/12 required fields present)
       ⚠️ Missing: Driver MVR reports — you'll need to request these
       from the broker before underwriting can proceed.

       📋 Routing: **Auto Queue** (routine priority)
       Action needed: Request missing MVR reports from broker.

       Anything else you'd like me to process?
```

### 18.2 Batch Mode — Senior Manager (David)

```
MANAGER: Process batch: SUB-001, SUB-002, SUB-003, SUB-005

AGENT: Processing 4 submissions...

       | ID | Insured | Line | Completeness | Queue | Priority |
       |----|---------|------|-------------|-------|----------|
       | SUB-001 | Pacific Freight Lines | Auto | 92% | Auto Queue | Routine |
       | SUB-002 | Midwest Distribution | Property | 100% | Property Queue | Routine |
       | SUB-003 | NovaTech Consulting | GL | 95% | GL Queue | Routine |
       | SUB-005 | GreenLeaf Services | GL | 68% | Hold Queue | Needs info |

       Summary: 3 routed, 1 on hold (SUB-005 missing revenue_info).
```

### 18.3 Chat Mode — Status Query

```
CLERK: What's the status of SUB-002?

AGENT: **SUB-002 — Midwest Distribution Corp.**
       - Line: Commercial Property
       - Status: ✅ Complete (100% fields present)
       - Queue: Property Queue
       - Priority: Routine
       - Documents: application, property_schedule, building_details
       - Processed: April 18, 2026 at 2:30 PM

       This submission is ready for the property underwriting team.
       No action needed from your side.
```

---

## 19. Acceptance Criteria

### 19.1 MVP — Must Demo (Days 1-7)

| # | Criterion | Verification | Tech Topic |
|---|-----------|-------------|------------|
| AC-01 | Pipeline mode processes a submission end-to-end | Demo SUB-001 through full pipeline | ADK |
| AC-02 | **Parallel document parsing** works | Parse 3+ docs concurrently in one submission | ADK + Parallel |
| AC-03 | Field extraction returns structured JSON | Correct fields from application, schedule, loss history | ADK + Zero-shot prompting |
| AC-04 | LOB classification with CoT reasoning | Correct line assignment with step-by-step explanation | Prompt Engineering (Few-shot + CoT) |
| AC-05 | Completeness validation identifies missing fields | SUB-005 correctly flags missing revenue_info | Prompt Engineering (CoT) |
| AC-06 | Routing puts submissions in correct queues | Auto→Auto Queue, GL incomplete→Hold Queue | ADK |
| AC-07 | Status chat mode answers queries | "What's the status of SUB-001?" works | ADK (dual-mode) |
| AC-08 | User profiling detects clerk vs manager | Different output for Priya vs David | User Profiling |
| AC-09 | Agent Cards (JSON) for all 4 agents | Files exist with valid schema | A2A |
| AC-10 | 1+ real A2A call (Orchestrator → Parser) | A2A task delegation works | A2A |
| AC-11 | Prompt Manager serves versioned templates | Templates loaded via manager, version tracked | Prompt Manager |
| AC-12 | Arize traces visible in dashboard | All LLM calls traced with attributes | Arize |
| AC-13 | Annotation queue for classification review | Queue set up, classifications annotatable | Arize |
| AC-14 | Running via `adk web` or Agent Engine | Accessible demo | Agent Engine |

**Count: 6/6 tech topics covered in MVP** ✅

### 19.2 Stretch (Day 8)

| # | Criterion | Tech Topic |
|---|-----------|-----------|
| AC-15 | Multi-line submission detection + split routing | ADK + Prompt Engineering |
| AC-16 | All sub-agents via full A2A HTTP | A2A |
| AC-17 | Priority scoring (urgent vs routine) | ADK |
| AC-18 | Batch analytics in Arize dashboard | Arize |

---

## 20. References & Resources

### Google ADK
- ADK docs: https://google.github.io/adk-docs/
- ADK GitHub: https://github.com/google/adk-python
- ADK quickstart: https://google.github.io/adk-docs/get-started/quickstart/

### A2A Protocol
- A2A spec: https://github.com/google/A2A
- A2A Python samples: https://github.com/google/A2A/tree/main/samples/python

### Gemini
- Gemini API: https://ai.google.dev/docs
- Google Gen AI SDK: https://github.com/googleapis/python-genai

### Arize
- Arize docs: https://docs.arize.com/
- LLM tracing: https://docs.arize.com/arize/llm-large-language-models/llm-traces

### Vertex AI Agent Engine
- Agent Engine: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview

### Insurance Submission Intake Domain
- ACORD forms and submission standards (general industry knowledge)
- Multi-line commercial insurance submission workflows
- Underwriting queue management and triage practices

---

*Document Version: 1.0 | Created: April 18, 2026 | Status: Ready for Implementation*
