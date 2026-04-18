# Project 5: Commercial Liability Policy Comparison & Recommendation Agent

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
A conversational multi-agent system that helps insurance brokers compare commercial liability policies (General Liability, Professional Liability / E&O, Cyber Liability) for mid-to-large businesses. The broker chats with the system, describes client needs, and the system searches a mock policy database in **parallel**, scores matches, and presents side-by-side comparisons with a recommendation.

### How — 4 Agents

| Agent | Model | Role |
|-------|-------|------|
| **Comparison Orchestrator** | Gemini 2.5 Pro | Conversational front-end; collects client requirements, coordinates sub-agents |
| **Policy Search Agent** | Gemini 2.5 Flash | Searches mock policy JSON database, filters by coverage type / limits / industry |
| **Policy Scoring Agent** | Gemini 2.5 Pro | Scores each policy against client requirements using CoT reasoning; runs **parallel** scoring across policies |
| **Comparison Report Agent** | Gemini 2.5 Flash | Generates side-by-side comparison table + recommendation narrative |

### Workflow Pattern
**Sequential + Parallel**: Requirements gathering → parallel policy search across liability lines → parallel scoring of matched policies → report generation.

### Timeline
7 days MVP, Day 8 stretch. CS intern, medium Python.

### Tech Coverage (All 6 in MVP)

| # | Tech | How Used |
|---|------|----------|
| 1 | **ADK** | 4 agents built with Google ADK; UI via `adk web` |
| 2 | **A2A** | Agent Cards for all 4 agents; 1+ real A2A endpoint (Orchestrator → Policy Search) |
| 3 | **Prompt Engineering** | Zero-shot (requirement extraction), Few-shot (policy scoring with examples), CoT (scoring rationale), ReAct (tool-calling for search & scoring) |
| 4 | **Prompt Manager** | Dict-based Python module, `str.format_map()`, versioned templates |
| 5 | **User Profiling** | Detect broker experience (junior vs senior); adapt detail level |
| 6 | **Arize** | OTel tracing on all LLM calls; annotation queue for scoring quality |
| 7 | **Agent Engine** | Deploy to Vertex AI Agent Engine or local `adk web` fallback |

---

## 2. Business Context

### Problem
Commercial liability insurance is complex — brokers must compare policies across carriers for General Liability (GL), Professional Liability (PL/E&O), and Cyber Liability. Each policy has different coverage limits, exclusions, deductibles, premium structures, and endorsements. Today brokers manually read 50+ page policy documents and build comparison spreadsheets — slow and error-prone.

### Insurance Domain Primer

**3 Liability Types (MVP):**

| Type | What It Covers | Key Terms |
|------|---------------|-----------|
| **General Liability (GL)** | Bodily injury, property damage, advertising injury to third parties | Per-occurrence limit, aggregate limit, products-completed ops |
| **Professional Liability (PL/E&O)** | Errors, omissions, negligent acts in professional services | Claims-made vs occurrence, retroactive date, consent-to-settle |
| **Cyber Liability** | Data breaches, cyber attacks, business interruption from cyber events | First-party vs third-party, breach notification costs, regulatory fines |

**Key Comparison Dimensions:**
- Coverage limits (per-occurrence, aggregate)
- Deductibles / self-insured retentions
- Premium (annual)
- Exclusions (what's NOT covered)
- Endorsements (add-ons)
- Carrier AM Best rating
- Claims process / reputation

### Target Clients
Mid-to-large enterprises: tech companies (need Cyber + PL), manufacturers (need GL), consulting firms (need PL + GL), healthcare (need all 3).

---

## 3. Personas & User Stories

### Persona 1: Sarah — Junior Broker (1 year experience)

| Attribute | Value |
|-----------|-------|
| Experience | 1 year |
| Comfort | Knows basics, unsure about exclusion nuances |
| Needs | Guided requirement gathering, detailed explanations, explicit recommendations |
| Behavior | Asks follow-up questions, wants verbose comparisons |

### Persona 2: Marcus — Senior Broker (12 years experience)

| Attribute | Value |
|-----------|-------|
| Experience | 12 years |
| Comfort | Expert in liability lines, knows carrier reputations |
| Needs | Fast comparison, concise output, skip explanations |
| Behavior | Provides complete requirements upfront, wants data tables |

### User Stories

| ID | Story | Persona |
|----|-------|---------|
| US-01 | As a broker, I want to describe my client's business and coverage needs so the system finds matching policies | Both |
| US-02 | As a junior broker, I want guided questions to help me articulate requirements I might miss | Sarah |
| US-03 | As a senior broker, I want to paste requirements in one message and get results fast | Marcus |
| US-04 | As a broker, I want policies scored against my client's specific needs with clear rationale | Both |
| US-05 | As a broker, I want a side-by-side comparison table I can share with my client | Both |
| US-06 | As a broker, I want a recommendation with reasoning so I can advise my client | Both |

---

## 4. Functional Requirements

### MVP (Days 1-7)

| ID | Requirement |
|----|-------------|
| FR-01 | Conversational requirement gathering — collect: industry, revenue, employee count, coverage types needed, desired limits, budget range |
| FR-02 | Search mock policy database (JSON files) filtering by coverage type, industry eligibility, limit ranges |
| FR-03 | **Parallel search** across liability types (if client needs GL + Cyber, search both simultaneously) |
| FR-04 | Score each matched policy (0-100) against client requirements using weighted criteria |
| FR-05 | **Parallel scoring** — score all matched policies concurrently |
| FR-06 | Generate side-by-side comparison table (max 5 policies) |
| FR-07 | Generate recommendation with CoT reasoning |
| FR-08 | Adapt output verbosity based on broker profile (junior=detailed, senior=concise) |
| FR-09 | Support at least 3 client scenarios in demo |
| FR-10 | All data stored in JSON files (no database) |

### Stretch (Day 8)

| ID | Requirement |
|----|-------------|
| FR-11 | Add Umbrella/Excess liability as 4th line |
| FR-12 | Policy gap analysis — identify what's NOT covered |
| FR-13 | Premium negotiation talking points |

---

## 5. Non-Functional Requirements

| ID | Requirement |
|----|-------------|
| NFR-01 | Response time < 30s for complete comparison cycle |
| NFR-02 | All LLM calls traced via OpenTelemetry → Arize |
| NFR-03 | Mock policy database: 15+ policies across 3 liability types and 5+ carriers |
| NFR-04 | Runs locally via `adk web` or deployed to Agent Engine |
| NFR-05 | User profiles persisted in JSON files between sessions |

---

## 6. System Architecture

### High-Level Flow

```
Broker (via adk web)
    │
    ▼
┌─────────────────────────┐
│  Comparison Orchestrator │  (Gemini Pro — conversational)
│  - Requirement gathering │
│  - User profiling        │
│  - Coordinates workflow  │
└────────┬────────────────┘
         │
    ┌────┴────┐  (parallel fan-out by liability type)
    ▼         ▼
┌────────┐ ┌────────┐
│ Policy │ │ Policy │  (Gemini Flash — one call per liability type)
│ Search │ │ Search │
│ (GL)   │ │ (Cyber)│
└───┬────┘ └───┬────┘
    └────┬─────┘
         ▼
┌─────────────────────────┐
│   Policy Scoring Agent  │  (Gemini Pro — parallel scoring)
│   - Score each policy   │
│   - CoT reasoning       │
└────────┬────────────────┘
         ▼
┌─────────────────────────┐
│ Comparison Report Agent │  (Gemini Flash)
│ - Side-by-side table    │
│ - Recommendation        │
└─────────────────────────┘
```

### Directory Structure

```
project-5-liability-comparison/
├── agents/
│   ├── __init__.py
│   ├── orchestrator.py          # Comparison Orchestrator (root agent)
│   ├── policy_search.py         # Policy Search Agent
│   ├── policy_scoring.py        # Policy Scoring Agent
│   └── report_generator.py      # Comparison Report Agent
├── prompts/
│   ├── __init__.py
│   └── prompt_manager.py        # Dict-based prompt manager
├── profiles/
│   ├── __init__.py
│   └── user_profile_manager.py  # JSON-based user profiling
├── a2a/
│   ├── agent_cards/
│   │   ├── orchestrator_card.json
│   │   ├── policy_search_card.json
│   │   ├── policy_scoring_card.json
│   │   └── report_generator_card.json
│   └── a2a_client.py            # A2A task delegation
├── data/
│   ├── policies/
│   │   ├── gl_policies.json     # General Liability mock policies
│   │   ├── pl_policies.json     # Professional Liability mock policies
│   │   └── cyber_policies.json  # Cyber Liability mock policies
│   ├── carriers.json            # Carrier info (AM Best ratings etc.)
│   └── user_profiles/           # Persisted broker profiles
│       └── {user_id}.json
├── tracing/
│   └── arize_setup.py           # OTel + Arize configuration
├── agent.yaml                   # ADK agent config (root agent)
└── requirements.txt
```

---

## 7. Agent Specifications (ADK)

### 7.1 Comparison Orchestrator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `comparison_orchestrator` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Conversational front-end; gathers requirements, profiles broker, coordinates sub-agents |

**ADK Tools:**

```python
def search_policies(
    coverage_types: list[str],  # ["general_liability", "cyber_liability"]
    industry: str,
    min_limit: float,
    max_premium: float | None = None
) -> dict:
    """
    Delegates to Policy Search Agent. Searches mock JSON database
    for matching policies. Runs parallel searches if multiple coverage types.
    Returns: {"general_liability": [...policies], "cyber_liability": [...policies]}
    """

def score_policies(
    client_requirements: dict,
    candidate_policies: list[dict]
) -> list[dict]:
    """
    Delegates to Policy Scoring Agent. Scores each policy 0-100
    against client needs. Returns policies with scores + reasoning.
    """

def generate_comparison(
    scored_policies: list[dict],
    client_requirements: dict,
    user_profile: dict
) -> str:
    """
    Delegates to Comparison Report Agent. Returns formatted
    side-by-side comparison + recommendation.
    """

def manage_user_profile(user_id: str, action: str, data: dict = None) -> dict:
    """Get/update broker profile. Actions: 'get', 'update', 'get_or_create'"""

def get_prompt(template_name: str, version: str = "v1", variables: dict = None) -> str:
    """Retrieve rendered prompt from Prompt Manager."""
```

**ADK Config (`agent.yaml`):**

```yaml
name: comparison_orchestrator
model: gemini-2.5-pro
description: >
  Conversational agent that helps insurance brokers compare commercial
  liability policies. Gathers client requirements, searches policies,
  scores matches, and presents side-by-side comparisons.
tools:
  - search_policies
  - score_policies
  - generate_comparison
  - manage_user_profile
  - get_prompt
instruction: |
  You are a commercial liability policy comparison assistant for insurance brokers.
  
  WORKFLOW:
  1. Greet the broker and determine their experience level (ask 1-2 profiling questions)
  2. Gather client requirements: industry, size, revenue, coverage types needed, limits, budget
  3. Call search_policies with the requirements
  4. Call score_policies on the results
  5. Call generate_comparison to produce the final report
  6. Present results adapted to broker profile (detailed for junior, concise for senior)
  
  RULES:
  - Always confirm requirements before searching
  - If <3 policies found, suggest broadening criteria
  - Maximum 5 policies in final comparison
sub_agents:
  - policy_search
  - policy_scoring
  - report_generator
```

### 7.2 Policy Search Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `policy_search` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Searches mock JSON policy files, filters and returns candidates |

**ADK Tools:**

```python
def load_policies(coverage_type: str) -> list[dict]:
    """Load policies from data/policies/{coverage_type}_policies.json"""

def filter_policies(
    policies: list[dict],
    industry: str,
    min_limit: float,
    max_premium: float | None
) -> list[dict]:
    """Filter policies by eligibility criteria. Returns matching subset."""

def get_carrier_info(carrier_id: str) -> dict:
    """Load carrier details from data/carriers.json"""
```

**Key behavior:** When Orchestrator requests multiple coverage types, this agent runs **parallel** searches (one per liability type) using ADK's parallel tool execution.

### 7.3 Policy Scoring Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `policy_scoring` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Scores each candidate policy against client requirements using CoT reasoning |

**ADK Tools:**

```python
def score_single_policy(
    policy: dict,
    requirements: dict,
    scoring_weights: dict
) -> dict:
    """
    Score one policy against requirements. Uses CoT prompt for transparent scoring.
    Returns: {policy_id, overall_score, category_scores: {coverage: X, price: X, 
    carrier_strength: X, exclusions: X}, reasoning: "step-by-step..."}
    """
```

**Scoring Criteria (MVP — 4 categories):**

| Category | Weight | What's Scored |
|----------|--------|---------------|
| Coverage Fit | 35% | Do limits/coverage match client needs? |
| Price Value | 25% | Premium relative to coverage provided |
| Carrier Strength | 20% | AM Best rating, claims reputation |
| Exclusion Risk | 20% | Are important exposures excluded? |

**Key behavior:** Runs **parallel** scoring across all candidate policies. Each policy scored independently.

### 7.4 Comparison Report Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `report_generator` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Produces side-by-side comparison table and recommendation narrative |

**ADK Tools:**

```python
def build_comparison_table(scored_policies: list[dict]) -> str:
    """Build markdown side-by-side comparison table. Max 5 policies."""

def build_recommendation(
    scored_policies: list[dict],
    client_requirements: dict,
    user_profile: dict
) -> str:
    """
    Generate recommendation narrative. Adapts verbosity:
    - Junior broker: detailed with explanations of exclusions, caveats
    - Senior broker: concise with key differentiators only
    """
```

---

## 8. A2A Communication Design

### 8.1 Agent Cards

Each agent has a static JSON Agent Card in `a2a/agent_cards/`. Example for Policy Search:

```json
{
  "name": "policy_search",
  "description": "Searches commercial liability policy database by coverage type, industry, and limit requirements",
  "url": "http://localhost:8001/policy_search",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "search_gl_policies",
      "name": "Search General Liability Policies",
      "description": "Find GL policies matching client criteria"
    },
    {
      "id": "search_pl_policies",
      "name": "Search Professional Liability Policies",
      "description": "Find PL/E&O policies matching client criteria"
    },
    {
      "id": "search_cyber_policies",
      "name": "Search Cyber Liability Policies",
      "description": "Find Cyber policies matching client criteria"
    }
  ]
}
```

### 8.2 A2A Task Flow

**MVP:** Orchestrator → Policy Search via real A2A (1 endpoint minimum).

```python
# a2a/a2a_client.py — simplified A2A task sender
import json
import uuid

class A2AClient:
    def __init__(self, agent_card_path: str):
        with open(agent_card_path) as f:
            self.card = json.load(f)

    def send_task(self, skill_id: str, input_data: dict) -> dict:
        """
        Create A2A task message and send to agent.
        MVP: direct function call simulating A2A protocol.
        Stretch: real HTTP A2A endpoint.
        """
        task = {
            "id": str(uuid.uuid4()),
            "skill_id": skill_id,
            "input": input_data,
            "status": "pending"
        }
        # MVP: call agent function directly
        # Stretch: POST to self.card["url"]
        return self._execute_local(task)
```

**Stretch:** All 3 sub-agents callable via full A2A HTTP endpoints.

### 8.3 Agent Card Creation (All 4)

Create these files in `a2a/agent_cards/`:
- `orchestrator_card.json` — skills: gather_requirements, run_comparison
- `policy_search_card.json` — skills: search_gl, search_pl, search_cyber
- `policy_scoring_card.json` — skills: score_policies
- `report_generator_card.json` — skills: generate_comparison, generate_recommendation

---

## 9. Workflow Design

### 9.1 Main Workflow — Sequential + Parallel

```
Step 1: REQUIREMENT GATHERING (Sequential, Conversational)
  Orchestrator ←→ Broker (multi-turn chat)
  Output: client_requirements dict

Step 2: POLICY SEARCH (Parallel fan-out)
  Orchestrator → Policy Search Agent
    ├── search GL policies (if requested)    ─┐
    ├── search PL policies (if requested)    ─┤ PARALLEL
    └── search Cyber policies (if requested) ─┘
  Output: candidate_policies grouped by type

Step 3: POLICY SCORING (Parallel fan-out)
  Orchestrator → Policy Scoring Agent
    ├── score policy A ─┐
    ├── score policy B ─┤ PARALLEL
    ├── score policy C ─┤
    └── score policy D ─┘
  Output: scored_policies with reasoning

Step 4: REPORT GENERATION (Sequential)
  Orchestrator → Comparison Report Agent
  Output: comparison_table + recommendation

Step 5: PRESENTATION (Sequential, Conversational)
  Orchestrator → Broker (adapted to profile)
```

### 9.2 State Machine

```
INIT → PROFILING → GATHERING → SEARCHING → SCORING → REPORTING → PRESENTED → (REFINE?)
                                                                       │
                                                                       └→ Broker asks "what about higher limits?"
                                                                          → back to GATHERING
```

### 9.3 Error Handling

| Scenario | Action |
|----------|--------|
| Zero policies found | Suggest broadening criteria (higher budget, lower limits) |
| Only 1 policy found | Still show comparison with scoring, note limited options |
| Scoring timeout | Return partial results with disclaimer |

---

## 10. Prompt Engineering Strategy

### 10.1 Prompt Types Used

| Type | Where Used | Why |
|------|-----------|-----|
| **Zero-shot** | Requirement extraction from broker input | Structured extraction doesn't need examples |
| **Few-shot** | Policy scoring (with 2-3 example scores) | Scoring needs calibration examples |
| **CoT** | Scoring rationale, recommendation reasoning | Transparent decision-making |
| **ReAct** | Orchestrator tool calls, search tool calls | Agent needs Reasoning + Action loop |

### 10.2 Key Prompts

#### Requirement Extraction (Zero-shot)

```python
REQUIREMENT_EXTRACTION = {
    "v1": """Extract structured requirements from the broker's message.

Input: {broker_message}

Return JSON:
{{
  "industry": "string",
  "company_size": "small|mid|large",
  "annual_revenue": number_or_null,
  "employee_count": number_or_null,
  "coverage_types_needed": ["general_liability", "professional_liability", "cyber_liability"],
  "desired_limits": {{"per_occurrence": number, "aggregate": number}},
  "max_annual_premium": number_or_null,
  "special_requirements": ["string"]
}}

If any field is unclear, set to null."""
}
```

#### Policy Scoring (Few-shot + CoT)

```python
POLICY_SCORING = {
    "v1": """Score this policy against client requirements.

CLIENT REQUIREMENTS:
{requirements_json}

POLICY TO SCORE:
{policy_json}

SCORING WEIGHTS:
- Coverage Fit (35%): Do limits and coverage match needs?
- Price Value (25%): Premium relative to coverage?
- Carrier Strength (20%): AM Best rating, reputation?
- Exclusion Risk (20%): Are key exposures excluded?

EXAMPLES:
---
Example 1: Tech company needs $2M GL + $1M Cyber
Policy: Carrier A GL - $2M limit, $5,000 premium, A+ rated, excludes cyber
Scores: Coverage=85 (limits match), Price=70 (fair), Carrier=90 (A+), Exclusions=60 (no cyber bundled)
Overall: 85*0.35 + 70*0.25 + 90*0.20 + 60*0.20 = 77.25
---
Example 2: Manufacturer needs $5M GL
Policy: Carrier B GL - $3M limit, $3,200 premium, A rated, excludes pollution
Scores: Coverage=50 (under limit), Price=80 (cheap), Carrier=75 (A), Exclusions=40 (pollution concern for mfg)
Overall: 50*0.35 + 80*0.25 + 75*0.20 + 40*0.20 = 60.50
---

Now score the given policy. Think step-by-step for each category, then compute overall.

Return JSON:
{{
  "policy_id": "string",
  "category_scores": {{
    "coverage_fit": {{"score": int, "reasoning": "..."}},
    "price_value": {{"score": int, "reasoning": "..."}},
    "carrier_strength": {{"score": int, "reasoning": "..."}},
    "exclusion_risk": {{"score": int, "reasoning": "..."}}
  }},
  "overall_score": float,
  "summary": "1-2 sentence summary"
}}"""
}
```

#### Recommendation (CoT)

```python
RECOMMENDATION = {
    "v1": """Based on the scored policies below, recommend the best option for this client.

CLIENT: {client_summary}
SCORED POLICIES:
{scored_policies_json}
BROKER PROFILE: {profile_json}

Think step-by-step:
1. Which policy best fits the coverage needs?
2. Which offers best value for money?
3. Are there any dealbreaker exclusions?
4. What's the overall best choice and why?

Adapt your output:
- If broker is junior (experience < 3 years): explain exclusion implications, define terms
- If broker is senior (experience >= 3 years): skip basics, focus on differentiators

Return your recommendation."""
}
```

#### Profiling (Zero-shot)

```python
BROKER_PROFILING = {
    "v1": """Based on the broker's responses, determine their experience level.

Conversation so far:
{conversation_history}

Indicators of JUNIOR broker:
- Asks what coverage types mean
- Unsure about appropriate limits
- Requests explanations of terms
- Hesitant or vague about client needs

Indicators of SENIOR broker:
- Provides specific limits and coverage types immediately
- Uses technical terms (aggregate, SIR, claims-made)
- Mentions carrier preferences
- Provides complete requirements in few messages

Return JSON:
{{
  "experience_level": "junior" or "senior",
  "confidence": 0.0-1.0,
  "signals": ["list of observed indicators"]
}}"""
}
```

---

## 11. Prompt Manager Design

### 11.1 Implementation

```python
# prompts/prompt_manager.py

from typing import Optional

# All templates stored as versioned dicts
TEMPLATES = {
    "requirement_extraction": {
        "v1": """Extract structured requirements...""",  # full text above
        "v2": None  # future version
    },
    "policy_scoring": {
        "v1": """Score this policy against client requirements..."""
    },
    "recommendation": {
        "v1": """Based on the scored policies below..."""
    },
    "broker_profiling": {
        "v1": """Based on the broker's responses..."""
    },
    "comparison_table": {
        "v1": """Generate a markdown comparison table..."""
    },
    "search_filter": {
        "v1": """Given these requirements, determine search filters..."""
    }
}

class PromptManager:
    def __init__(self):
        self._templates = TEMPLATES
        self._usage_log = []  # for Arize tracing

    def get_prompt(
        self,
        template_name: str,
        version: str = "v1",
        variables: Optional[dict] = None
    ) -> str:
        """Retrieve and render a prompt template."""
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

# In orchestrator
prompt = pm.get_prompt("requirement_extraction", "v1", {
    "broker_message": user_input
})

# In scoring agent
prompt = pm.get_prompt("policy_scoring", "v1", {
    "requirements_json": json.dumps(reqs),
    "policy_json": json.dumps(policy)
})
```

**Key rules:**
- NO Jinja2 — use `str.format_map()` only
- Templates are Python string constants (not loaded from files)
- Version tracked in Arize spans via `prompt_version` attribute

---

## 12. User Profiling Design

### 12.1 Profile Schema

```json
{
  "user_id": "sarah_broker_01",
  "name": "Sarah Chen",
  "experience_level": "junior",
  "years_experience": 1,
  "preferred_detail": "verbose",
  "specialty_lines": ["general_liability"],
  "interaction_count": 0,
  "created_at": "2026-04-18T10:00:00Z",
  "updated_at": "2026-04-18T10:00:00Z"
}
```

### 12.2 Profile Manager

```python
# profiles/user_profile_manager.py

import json
import os
from datetime import datetime

PROFILES_DIR = "data/user_profiles"

class UserProfileManager:
    def get_or_create(self, user_id: str, name: str = "") -> dict:
        path = os.path.join(PROFILES_DIR, f"{user_id}.json")
        if os.path.exists(path):
            with open(path) as f:
                return json.load(f)
        profile = {
            "user_id": user_id,
            "name": name,
            "experience_level": "unknown",
            "years_experience": None,
            "preferred_detail": "verbose",  # default to verbose until profiled
            "specialty_lines": [],
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

| Attribute | Junior (Sarah) | Senior (Marcus) |
|-----------|----------------|-----------------|
| Requirement gathering | Guided, one question at a time | Accepts bulk input |
| Scoring output | Full reasoning for every category | Summary scores + key differentiators |
| Recommendation | Explains terms, caveats, next steps | Bottom line + action items |
| Comparison table | Includes term definitions | Raw data table |

### 12.4 Profiling Trigger
On first interaction, Orchestrator asks 1-2 lightweight questions:
1. "How long have you been working in commercial liability?"
2. If answer suggests experience, ask: "Which liability lines do you specialize in?"

Then sets `experience_level` based on the profiling prompt response. Profile persists in JSON for future sessions.

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
    """Configure OTel tracing to export to Arize Cloud."""
    provider = TracerProvider()

    exporter = OTLPSpanExporter(
        endpoint="https://otlp.arize.com:443",
        headers={
            "space_id": space_id,
            "api_key": api_key
        }
    )
    provider.add_span_processor(BatchSpanProcessor(exporter))
    trace.set_tracer_provider(provider)
    return trace.get_tracer("liability-comparison-agent")
```

### 13.2 What Gets Traced

| Span | Attributes |
|------|-----------|
| `orchestrator.gather_requirements` | `user_id`, `experience_level`, `coverage_types_requested` |
| `policy_search.search` | `coverage_type`, `filters_applied`, `policies_found_count` |
| `policy_scoring.score` | `policy_id`, `overall_score`, `prompt_version`, `model` |
| `report_generator.compare` | `policies_count`, `recommendation_policy_id` |
| Each LLM call | `model`, `prompt_template`, `prompt_version`, `token_count`, `latency_ms` |

### 13.3 Annotation Queue

Set up a **"Scoring Quality Review"** annotation queue in Arize Cloud:
- **Purpose:** Review whether policy scores are reasonable and well-calibrated
- **What gets queued:** All `policy_scoring.score` spans
- **Annotator labels:** `accurate`, `over_scored`, `under_scored`, `bad_reasoning`
- **Use case:** Intern reviews 10+ scoring examples, labels quality, identifies prompt improvement opportunities

### 13.4 Dashboard Widgets (MVP)

1. Scoring distribution histogram (are scores well-spread or clustered?)
2. Latency by agent (which agent is slowest?)
3. Prompt version comparison (v1 vs v2 scoring accuracy if time)
4. Error rate by agent

---

## 14. Deployment Plan

### 14.1 Local Development (Primary)

```bash
# Install dependencies
pip install google-adk google-genai opentelemetry-api opentelemetry-sdk \
    opentelemetry-exporter-otlp arize-otel

# Set environment variables
export GOOGLE_API_KEY="your-gemini-api-key"
export ARIZE_SPACE_ID="your-space-id"
export ARIZE_API_KEY="your-api-key"

# Run locally
adk web
```

### 14.2 Agent Engine Deployment

```python
from google.cloud import aiplatform
from google.cloud.aiplatform import agent_engines

# Deploy root agent (Orchestrator)
app = agent_engines.create(
    display_name="liability-comparison-agent",
    agent_engine_config={
        "agent": "agents/orchestrator.py",
        "requirements": "requirements.txt"
    }
)
print(f"Deployed: {app.resource_name}")
```

**Fallback:** If Agent Engine has issues (permissions, quota), demo runs locally via `adk web`.

### 14.3 Environment Variables

| Variable | Purpose |
|----------|---------|
| `GOOGLE_API_KEY` | Gemini API access |
| `ARIZE_SPACE_ID` | Arize Cloud project |
| `ARIZE_API_KEY` | Arize Cloud auth |
| `DEPLOYMENT_MODE` | `local` or `agent_engine` |

---

## 15. Data Models

### 15.1 Client Requirements

```json
{
  "client_name": "TechCorp Solutions",
  "industry": "technology",
  "sub_industry": "saas_provider",
  "annual_revenue": 25000000,
  "employee_count": 200,
  "coverage_types_needed": ["general_liability", "cyber_liability"],
  "desired_limits": {
    "per_occurrence": 2000000,
    "aggregate": 5000000
  },
  "max_annual_premium": 50000,
  "special_requirements": ["data_breach_coverage", "regulatory_defense"]
}
```

### 15.2 Policy (Mock Data)

```json
{
  "policy_id": "GL-CARRIER-A-001",
  "carrier": "Atlantic Mutual Insurance",
  "carrier_am_best": "A+",
  "coverage_type": "general_liability",
  "policy_form": "occurrence",
  "limits": {
    "per_occurrence": 2000000,
    "general_aggregate": 4000000,
    "products_completed_ops": 2000000,
    "personal_advertising_injury": 1000000
  },
  "deductible": 5000,
  "annual_premium": 12500,
  "eligible_industries": ["technology", "consulting", "healthcare", "manufacturing"],
  "exclusions": ["pollution", "employment_practices", "cyber"],
  "endorsements_available": ["waiver_of_subrogation", "additional_insured"],
  "key_features": ["Broad form coverage", "Worldwide territory", "24/7 claims hotline"],
  "claims_process_rating": 4.2
}
```

### 15.3 Scored Policy (Output)

```json
{
  "policy_id": "GL-CARRIER-A-001",
  "carrier": "Atlantic Mutual Insurance",
  "overall_score": 78.5,
  "category_scores": {
    "coverage_fit": {"score": 85, "reasoning": "Limits match client needs at $2M/$4M"},
    "price_value": {"score": 80, "reasoning": "$12,500 premium is competitive for $2M GL"},
    "carrier_strength": {"score": 90, "reasoning": "A+ AM Best, 4.2 claims rating"},
    "exclusion_risk": {"score": 55, "reasoning": "Cyber exclusion is a concern since client also needs cyber coverage — check for gaps"}
  },
  "summary": "Strong GL option with good carrier backing, but watch for cyber gap"
}
```

### 15.4 Comparison Report (Output)

```json
{
  "report_id": "RPT-20260418-001",
  "client_name": "TechCorp Solutions",
  "comparison_table": "| Feature | Atlantic Mutual GL | Pacific Shield GL | ... |",
  "recommendation": {
    "top_pick": "GL-CARRIER-A-001",
    "reasoning": "Best coverage fit with strong carrier...",
    "caveats": ["Cyber exclusion — ensure separate cyber policy covers gap"],
    "runner_up": "GL-CARRIER-B-002"
  },
  "generated_at": "2026-04-18T14:30:00Z",
  "broker_profile_used": "junior"
}
```

### 15.5 Mock Policy Database

Create **15+ mock policies** across 3 JSON files:

| File | Count | Carriers |
|------|-------|----------|
| `gl_policies.json` | 6 policies | Atlantic Mutual, Pacific Shield, Continental, Meridian, Summit, Guardian |
| `pl_policies.json` | 5 policies | Atlantic Mutual, Pacific Shield, Continental, Apex, ProShield |
| `cyber_policies.json` | 5 policies | CyberFirst, Pacific Shield, DataGuard, Continental, TechShield |

**Carrier diversity:** Mix of A++, A+, A, A- rated carriers with varying premiums, limits, and exclusions to make comparisons meaningful.

---

## 16. Tech Stack & Tools

| Component | Technology |
|-----------|-----------|
| Language | Python 3.11+ |
| Agent Framework | Google ADK (`google-adk`) |
| LLM | Gemini 2.5 Pro (orchestrator, scoring) / Flash (search, reporting) |
| Agent Communication | A2A Protocol (Agent Cards + task delegation) |
| Prompt Management | Dict-based Python module with `str.format_map()` |
| User Profiles | JSON files in `data/user_profiles/` |
| Mock Data | JSON files in `data/policies/` |
| Observability | OpenTelemetry + Arize Cloud |
| Deployment | Vertex AI Agent Engine / local `adk web` |
| UI | ADK Web (built-in) |
| Version Control | Git |

---

## 17. Development Plan — 7-Day Sprint + 1-Day Stretch

### Day 1: Setup + Mock Data
- [ ] Initialize project structure (directories, `requirements.txt`, `agent.yaml`)
- [ ] Install ADK, Gemini SDK, OTel, Arize packages
- [ ] Create all 15+ mock policies across 3 JSON files
- [ ] Create `carriers.json` with 6 carrier profiles
- [ ] Verify Gemini API key works with a test call
- [ ] **Checkpoint:** Can load and print mock policies from JSON

### Day 2: Prompt Manager + Core Prompts
- [ ] Build `prompts/prompt_manager.py` (PromptManager class)
- [ ] Write all 6 prompt templates (v1): extraction, scoring, recommendation, profiling, comparison_table, search_filter
- [ ] Unit test: render each template with sample variables
- [ ] **Checkpoint:** All prompts renderable, no format errors

### Day 3: Agents — Search + Scoring
- [ ] Build `policy_search.py` — load JSON, filter by criteria, return matches
- [ ] Build `policy_scoring.py` — score a single policy with CoT prompt
- [ ] Test search with sample requirements → returns filtered policies
- [ ] Test scoring with sample policy → returns score + reasoning JSON
- [ ] **Checkpoint:** Both agents return valid JSON independently

### Day 4: Agents — Report + Orchestrator
- [ ] Build `report_generator.py` — comparison table + recommendation
- [ ] Build `orchestrator.py` — conversational flow + tool definitions
- [ ] Wire Orchestrator → sub-agents via ADK tool calls
- [ ] Test end-to-end: requirement → search → score → report
- [ ] **Checkpoint:** Full pipeline works via `adk web`

### Day 5: Parallel Execution + User Profiling
- [ ] Implement parallel search (multiple coverage types simultaneously)
- [ ] Implement parallel scoring (all policies scored concurrently)
- [ ] Build `user_profile_manager.py` + profiling flow
- [ ] Wire profiling into Orchestrator (adapt output based on profile)
- [ ] Test with Sarah (junior) and Marcus (senior) personas
- [ ] **Checkpoint:** Parallel execution visible, profiles persist in JSON

### Day 6: A2A + Arize
- [ ] Create 4 Agent Card JSONs
- [ ] Implement A2A client — Orchestrator → Policy Search via A2A task
- [ ] Set up `arize_setup.py` with OTel tracing
- [ ] Add trace spans to all agents (attributes: model, prompt_version, etc.)
- [ ] Set up annotation queue in Arize Cloud for scoring review
- [ ] **Checkpoint:** Traces visible in Arize, Agent Cards loadable

### Day 7: Integration + Demo Prep
- [ ] End-to-end demo run: 3 client scenarios (tech company, manufacturer, consulting firm)
- [ ] Test junior broker flow (Sarah) — guided, verbose
- [ ] Test senior broker flow (Marcus) — fast, concise
- [ ] Deploy to Agent Engine (or confirm local `adk web` fallback)
- [ ] Fix any integration bugs
- [ ] **Checkpoint:** Demo-ready with 3 scenarios passing

### Day 8 (Stretch):
- [ ] Add Umbrella/Excess liability as 4th line
- [ ] Policy gap analysis feature
- [ ] Full A2A for all sub-agents
- [ ] Prompt v2 experiments + Arize comparison dashboard

---

## 18. Sample Interactions

### 18.1 Junior Broker (Sarah) — Guided Flow

```
BROKER: Hi, I need to compare liability policies for a client.

AGENT: Welcome! I'd love to help you find the right liability coverage.
       Before we start, could you tell me how long you've been working
       in commercial liability?

BROKER: About a year. I'm still learning the ropes.

AGENT: Great — I'll make sure to explain things clearly as we go.
       Let's gather your client's details. What industry is your client in?

BROKER: They're a tech company, SaaS provider.

AGENT: Got it. And roughly what's their annual revenue and employee count?

BROKER: Around $25 million revenue, about 200 employees.

AGENT: Thanks. For a SaaS company that size, they'll typically need:
       - **General Liability (GL)** — covers bodily injury and property damage claims
       - **Cyber Liability** — covers data breaches and cyber incidents
       
       Does your client need both, or are there others?

BROKER: Both of those. Maybe Professional Liability too?

AGENT: Good thinking — Professional Liability (E&O) covers errors in their
       software services. Let's include all three.
       
       What coverage limits are they looking for? For a $25M tech company,
       common limits are $1M-$2M per occurrence.

BROKER: Let's go with $2M per occurrence.

AGENT: And is there a budget range for annual premium?

BROKER: Under $50K total if possible.

AGENT: Perfect. Let me search across all three liability types now...
       
       [Searching GL policies... ✓]
       [Searching PL policies... ✓]  (parallel)
       [Searching Cyber policies... ✓]
       
       Found 8 policies across 5 carriers. Now scoring each against
       TechCorp's needs...
       
       [Scoring 8 policies... ✓] (parallel)
       
       Here's your comparison:

       ## General Liability — Top 3
       
       | Feature | Atlantic Mutual | Pacific Shield | Continental |
       |---------|----------------|----------------|-------------|
       | Limit (Occ/Agg) | $2M / $4M | $2M / $4M | $2M / $5M |
       | Premium | $12,500 | $14,200 | $11,800 |
       | AM Best | A+ | A | A+ |
       | Deductible | $5,000 | $2,500 | $10,000 |
       | Score | 78.5 | 74.2 | 76.1 |
       
       [Similar tables for PL and Cyber]
       
       ## My Recommendation
       
       For TechCorp, I'd recommend:
       - **GL:** Atlantic Mutual ($12,500) — best coverage fit with A+ carrier
       - **Cyber:** CyberFirst ($18,000) — strongest cyber-specific coverage
       - **PL:** Apex Professional ($15,500) — best E&O for SaaS companies
       
       **Total estimated premium: $46,000** (within budget)
       
       ⚠️ **One thing to watch:** Atlantic Mutual GL excludes cyber events,
       but that's fine since you'll have separate cyber coverage from CyberFirst.
       
       Want me to dig deeper into any of these?
```

### 18.2 Senior Broker (Marcus) — Express Flow

```
BROKER: Compare GL and Cyber policies for a $100M manufacturer,
        500 employees, $5M per occurrence, budget $80K.

AGENT: Running comparison for manufacturer, $5M limits...

       [Parallel search + scoring complete]

       | Policy | Type | Carrier | Limit | Premium | Score |
       |--------|------|---------|-------|---------|-------|
       | GL-001 | GL | Continental | $5M/$10M | $32,000 | 82.3 |
       | GL-004 | GL | Summit | $5M/$10M | $28,500 | 79.8 |
       | CY-002 | Cyber | DataGuard | $5M/$5M | $42,000 | 85.1 |
       | CY-005 | Cyber | TechShield | $5M/$5M | $38,000 | 76.9 |

       **Pick:** Continental GL + DataGuard Cyber = $74,000 (under budget).
       Continental has pollution exclusion — verify that's acceptable for mfg.

BROKER: What about Guardian for GL?

AGENT: Guardian GL maxes at $3M per-occurrence — under your $5M requirement.
       Would need an umbrella policy on top. Want me to check?
```

---

## 19. Acceptance Criteria

### 19.1 MVP — Must Demo (Days 1-7)

| # | Criterion | Verification | Tech Topic |
|---|-----------|-------------|------------|
| AC-01 | Conversational requirement gathering works | Demo 2+ client scenarios | ADK |
| AC-02 | Policy Search returns filtered results from JSON | Correct filtering by type/industry/limits | ADK + ReAct |
| AC-03 | **Parallel search** across liability types | Visible parallel execution for multi-type requests | ADK + Parallel |
| AC-04 | Policy scoring with CoT reasoning | Each score has step-by-step explanation | Prompt Engineering (CoT + Few-shot) |
| AC-05 | **Parallel scoring** across policies | Multiple policies scored concurrently | ADK + Parallel |
| AC-06 | Side-by-side comparison table generated | Readable markdown table, max 5 policies | ADK |
| AC-07 | Recommendation with reasoning | Clear top pick + rationale | Prompt Engineering (CoT) |
| AC-08 | User profiling detects junior vs senior | Different output for Sarah vs Marcus | User Profiling |
| AC-09 | Agent Cards (JSON) for all 4 agents | Files exist with valid schema | A2A |
| AC-10 | 1+ real A2A call (Orchestrator → Search) | A2A task delegation works | A2A |
| AC-11 | Prompt Manager serves versioned templates | Templates loaded via manager, version tracked | Prompt Manager |
| AC-12 | Arize traces visible in dashboard | All LLM calls traced with attributes | Arize |
| AC-13 | Annotation queue for scoring review | Queue set up, scores annotatable | Arize |
| AC-14 | Running via `adk web` or Agent Engine | Accessible demo | Agent Engine |

**Count: 6/6 tech topics covered in MVP** ✅

### 19.2 Stretch (Day 8)

| # | Criterion | Tech Topic |
|---|-----------|-----------|
| AC-15 | 4th liability line (Umbrella/Excess) | ADK |
| AC-16 | All sub-agents via full A2A HTTP | A2A |
| AC-17 | Policy gap analysis | ADK + Prompt Engineering |
| AC-18 | Prompt v1 vs v2 comparison in Arize | Prompt Manager + Arize |

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

### Commercial Liability Domain
- ISO Commercial General Liability coverage forms
- Cyber liability policy structures and first-party vs third-party coverage
- Professional Liability / E&O claims-made policy mechanics

---

*Document Version: 1.0 | Created: April 18, 2026 | Status: Ready for Implementation*
