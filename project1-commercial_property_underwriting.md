# Project 1: Commercial Property Underwriting Risk Assessment Agent

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
17. [Development Phases](#17-development-phases)
18. [Sample Interactions](#18-sample-interactions)
19. [Acceptance Criteria](#19-acceptance-criteria)
20. [References & Resources](#20-references--resources)

---

## 1. Executive Summary

### What
A multi-agent AI system that assists commercial property underwriters at mid-to-large enterprise insurance companies in evaluating risk for commercial properties (office buildings, warehouses, manufacturing plants, retail complexes).

### Why
Commercial property underwriting is manual, time-consuming, and inconsistent. Underwriters spend hours collecting property data, analyzing multiple risk factors, and compiling reports. An AI agent system can:
- Reduce underwriting cycle time from days to minutes
- Ensure consistent risk evaluation criteria
- Surface risk factors human underwriters might miss
- Generate professional underwriting reports automatically

### How
Four specialized agents communicate via Google's A2A protocol:
1. **Orchestrator Agent** — Converses with the underwriter, coordinates the workflow
2. **Property Data Collector Agent** — Gathers property and location data from external sources
3. **Risk Assessment Agent** — Evaluates multiple risk factors in parallel
4. **Report Generator Agent** — Produces structured underwriting reports

Built with Google ADK + Gemini 2.5, monitored via Arize, deployed to Vertex AI Agent Engine.

### Timeline & Scope

| Attribute | Detail |
|-----------|--------|
| **Duration** | 7-8 working days |
| **Intern Level** | CS background, medium Python expertise |
| **Delivery** | Working demo with 2 scenarios (junior + senior underwriter) |

The project is organized into **MVP (Must-ship in 7 days)** and **Stretch (Day 8 if time permits)** scopes. The MVP alone covers all 6 tech topics and produces an impressive demo. Stretch goals add polish.

### Key Tech Stack Coverage

| Tech Topic | How It's Used in This Project | Scope |
|------------|------------------------------|-------|
| Prompt Engineering | Zero-shot (general queries), Few-shot (risk scoring), CoT (complex risk reasoning), ReAct (tool-calling data collection) | MVP |
| Prompt Manager | Version and serve all prompt templates across 4 agents using a lightweight Python module | MVP |
| ADK | All 4 agents built with Google Agent Development Kit, UI via `adk web` | MVP |
| A2A | Agents publish Agent Cards; Orchestrator delegates tasks to sub-agents via A2A protocol | MVP |
| User Profiling | Profile underwriter's experience level and adapt response style (verbose vs. concise) | MVP |
| Arize | Trace all agent interactions with OpenTelemetry; set up annotation queue for risk assessment quality review | MVP |
| Agent Engine | Deploy agents to Vertex AI Agent Engine (or local `adk web` fallback) | MVP |

### What Makes This an Excellent Learning Project

1. **Real-world domain** — Insurance underwriting is a genuine enterprise AI use case
2. **Every tech topic integrated** — Not toy examples; each tech serves a real purpose in the workflow
3. **Sequential + Parallel workflows** — Intern learns both patterns in one project
4. **Conversational + Non-conversational agents** — Orchestrator is chat-based, sub-agents are task-based
5. **4 prompt engineering techniques** — Each used where it genuinely fits (not forced)
6. **End-to-end observability** — Intern sees how production AI systems are monitored and debugged
7. **Deployment experience** — From local dev to cloud deployment

---

## 2. Business Context

### 2.1 Industry Background

**Commercial Property Insurance** covers physical assets owned or leased by businesses — buildings, equipment, inventory, and improvements. For mid-to-large enterprises, a single policy can cover properties worth $10M–$500M+.

**Underwriting** is the process of evaluating the risk associated with insuring a property and determining appropriate premium pricing. An underwriter must assess:

- **Physical risk** — Construction type, age, fire protection systems, electrical/plumbing condition
- **Location risk** — Natural catastrophe exposure (flood, earthquake, windstorm), crime rates, fire department proximity
- **Occupancy risk** — Business type, hazardous materials, foot traffic, manufacturing processes
- **Loss history** — Prior claims on the property and by the insured
- **Protection** — Sprinklers, alarms, security, business continuity plans

### 2.2 Current Process (Pain Points)

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ Submission   │───►│ Manual Data  │───►│ Spreadsheet   │───►│ Word Doc     │
│ Received     │    │ Collection   │    │ Risk Analysis │    │ Report       │
│ (Email/PDF)  │    │ (2-4 hours)  │    │ (3-6 hours)   │    │ (1-2 hours)  │
└─────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

**Pain Points:**
- **Slow** — 1-2 days per submission for a thorough underwriting review
- **Inconsistent** — Junior vs. senior underwriters apply different criteria
- **Data scattered** — Property data, hazard maps, loss history in separate systems
- **Reporting drudgery** — Hours spent on repetitive report formatting
- **Missed risks** — Cognitive overload leads to overlooked factors

### 2.3 Proposed Solution

```
┌─────────────┐    ┌──────────────────────────────────────────┐    ┌──────────────┐
│ Underwriter  │───►│        AI Agent System (This Project)    │───►│ Underwriting │
│ Provides     │    │                                          │    │ Report       │
│ Property     │    │  Orchestrator ─► Data Collector          │    │ + Risk Score │
│ Details      │    │       │         (parallel APIs)          │    │ + Premium    │
│ via Chat     │    │       ├──────► Risk Assessor             │    │   Guidance   │
│              │    │       │         (parallel factors)        │    │ (Minutes)    │
│              │    │       └──────► Report Generator           │    │              │
└─────────────┘    └──────────────────────────────────────────┘    └──────────────┘
```

**Target:** Reduce underwriting cycle from 1-2 days → 15-30 minutes for standard commercial properties.

### 2.4 Target Enterprise Segment

| Attribute | Detail |
|-----------|--------|
| **Company Size** | Mid-to-large insurance carriers, reinsurers, MGAs |
| **Property Types** | Office buildings, warehouses, manufacturing plants, retail complexes, mixed-use developments |
| **Total Insured Value (TIV)** | $5M – $500M per submission |
| **Geographic Focus** | United States (for hazard data availability) |
| **Lines of Business** | Commercial Property, may include Business Interruption |

---

## 3. Personas & User Stories

### 3.1 Personas

#### Persona 1: Junior Underwriter — "Priya"
| Attribute | Detail |
|-----------|--------|
| **Experience** | 1-2 years in commercial property |
| **Pain Points** | Not sure which risk factors to prioritize, takes long to compile reports, needs guidance on scoring |
| **Needs** | Step-by-step guidance, explanation of risk factors, templates, double-checking |
| **Tech Comfort** | High (recent grad), comfortable with new tools |

#### Persona 2: Senior Underwriter — "James"
| Attribute | Detail |
|-----------|--------|
| **Experience** | 10+ years, handles complex/large accounts |
| **Pain Points** | Too much repetitive data gathering, wants to focus on judgment calls, not formatting |
| **Needs** | Quick data, concise summaries, override ability, minimal hand-holding |
| **Tech Comfort** | Medium, prefers efficiency over novelty |

### 3.2 User Stories

| ID | As a... | I want to... | So that... | Scope |
|----|---------|-------------|-----------|-------|
| US-01 | Junior Underwriter | enter property details through a guided conversation | I don't miss critical information | **MVP** |
| US-02 | Junior Underwriter | get explanations for each risk factor score | I can learn and justify decisions to my manager | **MVP** |
| US-03 | Senior Underwriter | provide minimal input and get a complete risk summary fast | I can process more submissions per day | **MVP** |
| US-04 | Senior Underwriter | override AI risk scores with my judgment | the system supports, not replaces, my expertise | Stretch |
| US-05 | Any Underwriter | get property data auto-populated from address | I don't manually look up construction, hazard, and location data | **MVP** |
| US-06 | Any Underwriter | see all risk factors scored in parallel | I get a comprehensive view without waiting for sequential analysis | **MVP** |
| US-07 | Any Underwriter | generate a formatted underwriting report | I can share it with my manager or attach to the submission file | **MVP** |
| US-08 | Any Underwriter | review my assessment history | I can reference prior evaluations for similar properties | Stretch |
| US-09 | Any Underwriter | have the system adapt to my experience level | junior users get more guidance, seniors get concise output | **MVP** |

---

## 4. Functional Requirements

### 4.1 Core Features

#### F1: Conversational Property Intake
- The Orchestrator Agent engages the underwriter in a structured conversation to collect property details
- Supports guided mode (step-by-step questions) for all users and express mode (bulk input) for senior users
- Validates inputs in real-time (e.g., valid address, reasonable TIV)
- Allows corrections mid-conversation ("Actually, it's a 3-story building, not 2")
- **Scope:** MVP

#### F2: Automated Property Data Collection
- Given an address, the Data Collector Agent fetches:
  - **Property characteristics** — Construction type, year built, square footage, number of stories
  - **Natural hazard exposure** — Flood zone, earthquake fault proximity, wildfire risk, windstorm zone
  - **Location context** — Fire station distance, fire protection class, crime index, nearest hydrant
  - **Loss history** — Prior claims on property
- Data sources are simulated APIs for the demo (see Section 15 for mock data schemas)
- **Scope:** MVP

#### F3: Parallel Risk Factor Assessment
- The Risk Assessment Agent evaluates **4 risk categories simultaneously** (MVP), expandable to 6 (Stretch):

| # | Risk Category | Key Factors Assessed | Score Range | Scope |
|---|--------------|---------------------|-------------|-------|
| 1 | **Fire Risk** | Construction type, fire protection (sprinklers, alarms), cooking operations, electrical age | 1-10 | **MVP** |
| 2 | **Natural Catastrophe** | Flood zone, earthquake proximity, windstorm exposure, wildfire zone | 1-10 | **MVP** |
| 3 | **Occupancy & Operations** | Business type, hazardous materials, foot traffic, 24/7 operations | 1-10 | **MVP** |
| 4 | **Building Condition** | Age, renovation history, roof condition, HVAC/plumbing/electrical | 1-10 | **MVP** |
| 5 | **Protection & Security** | Sprinkler coverage, alarm systems, security guards, CCTV, access control | 1-10 | Stretch |
| 6 | **Loss History** | Prior claims count, total loss amount, largest single loss, claim trends | 1-10 | Stretch |

- MVP delivers **4 parallel risk categories** (Fire, NatCat, Occupancy, Building) — enough to demonstrate parallel execution convincingly
- Stretch adds 2 more (Protection, Loss History) — same pattern, minimal additional effort
- Results combined into an **Overall Risk Score** (weighted average)
- Risk grade: Low (1-3), Medium (4-6), High (7-8), Very High (9-10)
- **Scope:** MVP (4 categories) / Stretch (all 6)

#### F4: Premium Range Recommendation
- Based on overall risk score and TIV, the system suggests a premium range
- Formula (simplified for demo):
  ```
  Base Rate = lookup(property_type, construction_class)
  Adjusted Rate = Base Rate × (1 + risk_score_adjustment)
  Premium = TIV × Adjusted Rate / 1000
  ```
- Shows breakdown of how each risk factor influences final premium
- **Scope:** MVP

#### F5: Underwriting Report Generation
- The Report Generator Agent produces a structured report containing:
  - Property summary
  - Risk factor analysis (all assessed categories with scores and explanations)
  - Overall risk grade
  - Premium recommendation with rationale
  - Key concerns and recommendations
- Output format: Markdown (displayable in chat)
- **Scope:** MVP (Markdown in chat) / Stretch (also JSON export)

#### F6: User Profile-Adaptive Behavior
- System profiles the underwriter (see Section 12) and adapts:
  - **Verbose mode** (junior): Detailed explanations, definitions, step-by-step guidance
  - **Concise mode** (senior): Brief summaries, data tables, skip obvious explanations
- Profile stored as a JSON file (simple, no database needed)
- **Scope:** MVP

### 4.2 Feature Interaction Map

```
User Chat ──► [F1: Intake] ──► [F2: Data Collection] ──► [F3: Risk Assessment] ──► [F4: Premium] ──► [F5: Report]
                   │                                              │
                   ▼                                              ▼
              [F6: Profiling]                              (4 parallel risk       MVP
              adapts all features                          factor assessments)    ──────
                                                           (6 if stretch)        Stretch
```

---

## 5. Non-Functional Requirements

| # | Requirement | Target | Notes |
|---|------------|--------|-------|
| NFR-01 | Response time per agent call | < 10 seconds | Gemini 2.5 Flash for speed-critical calls |
| NFR-02 | End-to-end assessment time | < 5 minutes | From first input to complete report |
| NFR-03 | Concurrent risk assessments | 4 parallel (MVP), 6 parallel (Stretch) | Risk categories assessed simultaneously |
| NFR-04 | Tracing coverage | 100% of LLM calls | All calls traced via Arize |
| NFR-05 | Prompt template versioning | All prompts versioned | Via Prompt Manager |
| NFR-06 | Graceful degradation | Fallback to sequential | If parallel execution fails |
| NFR-07 | Data persistence | JSON file-based | User profiles persist across sessions |

---

## 6. System Architecture

### 6.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          PRESENTATION LAYER                              │
│                                                                          │
│   ┌──────────────────────────────────────────────────────────────┐     │
│   │              ADK Web UI (built-in)                            │     │
│   │   - Chat panel (conversational intake & results)             │     │
│   │   - Agent interaction log (visible agent delegation)         │     │
│   │   - Report display (Markdown rendered in chat)               │     │
│   └──────────────────────────────┬───────────────────────────────┘     │
│                                  │                                     │
└──────────────────────────────────┼─────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           AGENT LAYER (ADK)                              │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │              ORCHESTRATOR AGENT (Main Agent)                    │    │
│   │   - Manages conversation flow                                  │    │
│   │   - Delegates tasks to sub-agents via A2A                      │    │
│   │   - Applies user profile adaptations                           │    │
│   │   - Sequential workflow coordination                           │    │
│   └──────┬──────────────────┬──────────────────┬───────────────────┘    │
│          │ A2A              │ A2A              │ A2A                     │
│          ▼                  ▼                  ▼                         │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐              │
│   │ DATA         │   │ RISK         │   │ REPORT       │              │
│   │ COLLECTOR    │   │ ASSESSMENT   │   │ GENERATOR    │              │
│   │ AGENT        │   │ AGENT        │   │ AGENT        │              │
│   │              │   │              │   │              │              │
│   │ - Property   │   │ - Fire       │   │ - Markdown   │              │
│   │   lookup     │   │ - NatCat     │   │ - JSON       │              │
│   │ - Hazard     │   │ - Occupancy  │   │ - Summary    │              │
│   │   data       │   │ - Building   │   │ - Detailed   │              │
│   │ - Location   │   │ - Protection │   │              │              │
│   │   context    │   │ - Loss Hx    │   │              │              │
│   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘              │
│          │                  │                   │                       │
└──────────┼──────────────────┼───────────────────┼───────────────────────┘
           │                  │                   │
           ▼                  ▼                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          SERVICES LAYER                                  │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Prompt       │  │ User Profile │  │ Mock Data    │  │ Arize      │ │
│  │ Manager      │  │ Store        │  │ APIs         │  │ Telemetry  │ │
│  │              │  │              │  │              │  │            │ │
│  │ - Templates  │  │ - JSON       │  │ - Property   │  │ - Traces   │ │
│  │ - Versions   │  │   files      │  │ - Hazard     │  │ - Evals    │ │
│  │              │  │ - Profile    │  │ - Location   │  │ - Annot.   │ │
│  │              │  │   history    │  │ - Loss Hx    │  │            │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        DEPLOYMENT LAYER                                  │
│                                                                          │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────┐ │
│  │ Vertex AI Agent Engine       │  │ Local ADK Server (Fallback)     │ │
│  │ - Serverless scaling         │  │ - adk web / adk run             │ │
│  │ - Cloud Logging              │  │ - Local testing                 │ │
│  └──────────────────────────────┘  └─────────────────────────────────┘ │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Technology Decisions

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Agent Framework | Google ADK (Python) | Required tech stack; native Gemini integration |
| LLM | Gemini 2.5 Flash (speed) / Pro (complex reasoning) | Flash for data collection, Pro for risk assessment |
| Agent Communication | A2A Protocol | Required tech stack; demonstrates agent interoperability |
| UI | **ADK Web** (built-in `adk web`) | Zero build time; ADK provides chat UI out of the box |
| Storage | JSON files | Simplest possible for demo; no database setup |
| Prompt Management | Custom Python module (dict-based) | Lightweight, demonstrates concept clearly |
| Monitoring | Arize (Cloud) | Required tech stack |
| Tracing | OpenTelemetry + Arize SDK | Industry standard, Arize-compatible |
| Deployment | Vertex AI Agent Engine / Local ADK | GCP if available, local fallback |

---

## 7. Agent Specifications (ADK)

### 7.1 Orchestrator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `underwriting_orchestrator` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Main conversational agent; coordinates the entire underwriting workflow |
| **Mode** | Conversational (chat-based) |

**Responsibilities:**
1. Greet the underwriter, determine experience level (user profiling)
2. Collect property submission details through structured conversation
3. Delegate data collection to the Data Collector Agent (via A2A)
4. Delegate risk assessment to the Risk Assessment Agent (via A2A)
5. Delegate report generation to the Report Generator Agent (via A2A)
6. Present results back to the underwriter with appropriate detail level
7. Handle follow-up questions, overrides, and re-assessments

**ADK Tools (Functions):**

```python
# Tool 1: Send task to Data Collector Agent via A2A
def collect_property_data(address: str, property_type: str) -> dict:
    """
    Sends a data collection task to the Property Data Collector Agent.
    Returns structured property data including hazards, location context,
    and building characteristics.
    """
    pass

# Tool 2: Send task to Risk Assessment Agent via A2A
def assess_risk(property_data: dict) -> dict:
    """
    Sends property data to the Risk Assessment Agent for parallel
    risk factor evaluation. Returns risk scores for all categories
    and an overall risk grade.
    """
    pass

# Tool 3: Send task to Report Generator Agent via A2A
def generate_report(property_data: dict, risk_assessment: dict, user_profile: dict) -> str:
    """
    Sends assessment results to the Report Generator Agent.
    Returns a formatted underwriting report adapted to user preferences.
    """
    pass

# Tool 4: Get/update user profile
def manage_user_profile(user_id: str, action: str, data: dict = None) -> dict:
    """
    Retrieves or updates the underwriter's profile.
    Actions: 'get', 'update', 'get_or_create'
    """
    pass

# Tool 5: Get prompt from Prompt Manager
def get_prompt(template_name: str, version: str = "latest", variables: dict = None) -> str:
    """
    Retrieves a prompt template from the Prompt Manager,
    optionally rendering with provided variables.
    """
    pass
```

**System Instruction (Summary):**
```
You are an expert commercial property underwriting assistant. You help underwriters
evaluate risk for commercial properties. You adapt your communication style based
on the underwriter's experience level:
- Junior underwriters: Provide detailed explanations, define terminology, guide step-by-step
- Senior underwriters: Be concise, use tables, skip obvious context

Your workflow:
1. Profile the underwriter (if first interaction)
2. Collect property details through conversation
3. Trigger data collection, risk assessment, and report generation
4. Present results and handle follow-ups
```

### 7.2 Property Data Collector Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `property_data_collector` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Fetches and structures property/location data from external sources |
| **Mode** | Non-conversational (task-based) |

**Responsibilities:**
1. Receive a property address and type from the Orchestrator
2. Call multiple mock APIs to gather data
3. Structure and validate the collected data
4. Return comprehensive property data package

**ADK Tools (Functions):**

```python
# Tool 1: Property characteristics lookup
def lookup_property_characteristics(address: str) -> dict:
    """
    Simulated API call to property database.
    Returns: construction_type, year_built, stories, sqft, roof_type, etc.
    """
    pass

# Tool 2: Natural hazard lookup
def lookup_natural_hazards(latitude: float, longitude: float) -> dict:
    """
    Simulated API call to hazard data provider.
    Returns: flood_zone, earthquake_risk, wildfire_risk, windstorm_zone, etc.
    """
    pass

# Tool 3: Location context lookup
def lookup_location_context(address: str) -> dict:
    """
    Simulated API call to location services.
    Returns: fire_station_distance, fire_protection_class, crime_index, etc.
    """
    pass

# Tool 4: Loss history lookup
def lookup_loss_history(property_id: str, insured_name: str) -> dict:
    """
    Simulated API call to claims database.
    Returns: prior_claims list with dates, amounts, types.
    """
    pass
```

**Why Gemini Flash:** Data collection is straightforward — call APIs, structure results. Doesn't need complex reasoning. Flash is faster and cheaper.

### 7.3 Risk Assessment Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `risk_assessment_agent` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Evaluates 4 risk categories in parallel (MVP), expandable to 6 (Stretch); produces risk scores with reasoning |
| **Mode** | Non-conversational (task-based) |

**Responsibilities:**
1. Receive structured property data from the Orchestrator
2. Spawn 4 parallel risk assessment sub-tasks (MVP: Fire, NatCat, Occupancy, Building; Stretch adds Protection, Loss History)
3. Each sub-task uses Chain-of-Thought prompting for transparent reasoning
4. Combine results into overall risk score and grade
5. Return comprehensive risk assessment

**Parallel Risk Assessment Sub-Tasks:**

```
Property Data ──┬──► [Fire Risk Assessment]        ──┐
                ├──► [NatCat Risk Assessment]       ──┤──► Combine ──► Overall Score
                ├──► [Occupancy Risk Assessment]    ──┤
                └──► [Building Risk Assessment]     ──┘
                     (Stretch: + Protection, Loss History)
```

**Each sub-task uses Chain-of-Thought prompting:**

```
Given the following property data:
{property_data}

Assess the FIRE RISK for this commercial property. Think step by step:

1. What is the construction type and how does it affect fire risk?
2. What fire protection systems are in place?
3. Are there any high-risk occupancy factors?
4. What is the fire department response capability?

Based on your analysis, provide:
- Fire Risk Score (1-10, where 10 = highest risk)
- Key risk factors identified
- Mitigating factors
- Recommendations
```

**Why Gemini Pro:** Risk assessment requires complex reasoning across multiple factors. Pro produces better Chain-of-Thought reasoning and more nuanced scores.

### 7.4 Report Generator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `report_generator` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Compiles all data and assessments into a formatted underwriting report |
| **Mode** | Non-conversational (task-based) |

**Responsibilities:**
1. Receive property data, risk assessment results, and user profile
2. Generate report adapted to user's experience level and format preferences
3. Produce both human-readable (Markdown) and machine-readable (JSON) outputs

**ADK Tools (Functions):**

```python
# Tool 1: Format as Markdown report
def format_markdown_report(data: dict) -> str:
    """Generates a formatted Markdown underwriting report."""
    pass

# Tool 2: Format as JSON
def format_json_report(data: dict) -> dict:
    """Generates a structured JSON report for system consumption."""
    pass

# Tool 3: Calculate premium range
def calculate_premium_range(tiv: float, risk_score: float, property_type: str) -> dict:
    """
    Calculates recommended premium range based on TIV and risk score.
    Returns: min_premium, recommended_premium, max_premium, rate_per_thousand
    """
    pass
```

---

## 8. A2A Communication Design

### 8.1 What is A2A?

Agent-to-Agent (A2A) is Google's open protocol for agents to discover each other's capabilities and exchange tasks. Key concepts:

- **Agent Card** — A JSON metadata file describing an agent's capabilities, URL, and supported skills
- **Task** — A unit of work one agent requests from another
- **Message** — Communication payloads within a task (text, data, files)
- **Artifact** — Output produced by a task (reports, data files)

### 8.1.1 Scope for 7-Day MVP

| A2A Feature | MVP (Days 1-7) | Stretch (Day 8) |
|-------------|---------------|-----------------|
| Agent Cards | ✅ Static JSON files for all 4 agents | Auto-served via A2A server |
| Agent discovery | ✅ Hardcoded URLs in Orchestrator | Dynamic discovery |
| Task delegation | ✅ Use ADK's built-in sub-agent pattern (Orchestrator calls sub-agents directly) | Full A2A SDK client/server |
| A2A protocol demo | ✅ One working A2A client→server call (Orchestrator → Data Collector) to demonstrate the concept | All 3 sub-agents via A2A |

> **Why this approach?** ADK natively supports sub-agents — the Orchestrator can call Data Collector, Risk Assessor, and Report Generator as sub-agents with zero A2A overhead. For the MVP, the intern uses this built-in mechanism for the actual workflow, then adds one real A2A server endpoint (on the Data Collector) to demonstrate the A2A protocol. Agent Cards are created as static JSON files for all agents — these are valuable artifacts that show the intern understands A2A even if not all calls use the full protocol. This is practical: it works reliably and covers the learning objective.

### 8.2 Agent Cards

Each agent publishes an Agent Card at `/.well-known/agent.json`:

**Orchestrator Agent Card:**
```json
{
  "name": "underwriting_orchestrator",
  "description": "Commercial property underwriting assistant. Coordinates risk assessment workflow.",
  "url": "http://localhost:8001",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "property-underwriting",
      "name": "Commercial Property Underwriting",
      "description": "Evaluates commercial property risk through conversational interaction",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Data Collector Agent Card:**
```json
{
  "name": "property_data_collector",
  "description": "Collects property, hazard, and location data for underwriting.",
  "url": "http://localhost:8002",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "property-data-collection",
      "name": "Property Data Collection",
      "description": "Gathers property characteristics, hazard data, and location context given an address",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Risk Assessment Agent Card:**
```json
{
  "name": "risk_assessment_agent",
  "description": "Evaluates risk categories for commercial properties in parallel.",
  "url": "http://localhost:8003",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "risk-assessment",
      "name": "Commercial Property Risk Assessment",
      "description": "Parallel risk evaluation across fire, NatCat, occupancy, building, protection, and loss history",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Report Generator Agent Card:**
```json
{
  "name": "report_generator",
  "description": "Generates formatted underwriting reports from risk assessment data.",
  "url": "http://localhost:8004",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "report-generation",
      "name": "Underwriting Report Generation",
      "description": "Produces Markdown and JSON reports from property and risk data",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

### 8.3 A2A Task Flow

```
Orchestrator                 Data Collector       Risk Assessor        Report Generator
     │                            │                    │                      │
     │──── Create Task ──────────►│                    │                      │
     │     "Collect data for      │                    │                      │
     │      123 Industrial Ave"   │                    │                      │
     │                            │                    │                      │
     │◄─── Task Complete ─────────│                    │                      │
     │     {property_data}        │                    │                      │
     │                            │                    │                      │
     │──── Create Task ───────────────────────────────►│                      │
     │     "Assess risk with      │                    │                      │
     │      this property_data"   │                    │                      │
     │                            │         ┌──────────┤ (4 parallel, MVP)    │
     │                            │         │  Fire    │                      │
     │                            │         │  NatCat  │                      │
     │                            │         │  Occup.  │                      │
     │                            │         │  Bldg    │                      │
     │                            │         │  Protect │                      │
     │                            │         │  Loss Hx │                      │
     │                            │         └──────────┤                      │
     │◄─── Task Complete ─────────────────────────────│                      │
     │     {risk_scores}          │                    │                      │
     │                            │                    │                      │
     │──── Create Task ──────────────────────────────────────────────────────►│
     │     "Generate report with  │                    │                      │
     │      property_data +       │                    │                      │
     │      risk_scores"          │                    │                      │
     │                            │                    │                      │
     │◄─── Task Complete ─────────────────────────────────────────────────────│
     │     {report_markdown,      │                    │                      │
     │      report_json}          │                    │                      │
     │                            │                    │                      │
     ▼ Present to underwriter     │                    │                      │
```

### 8.4 Implementation Guidance

**How to implement A2A in this project:**

1. **Install the A2A SDK:**
   ```bash
   pip install a2a-sdk
   ```

2. **Each agent runs as a separate A2A-compatible server** (separate port or separate Cloud Run service)

3. **The Orchestrator uses an A2A client** to discover and call other agents:
   ```python
   from a2a.client import A2AClient

   # Discover the data collector agent
   data_collector = A2AClient(url="http://localhost:8002")
   card = await data_collector.get_agent_card()

   # Send a task
   task = await data_collector.send_task({
       "id": "task-001",
       "skill_id": "property-data-collection",
       "message": {
           "role": "user",
           "parts": [{"text": json.dumps({"address": "123 Industrial Ave", "type": "warehouse"})}]
       }
   })
   ```

4. **Each sub-agent implements a TaskHandler:**
   ```python
   from a2a.server import A2AServer, TaskHandler

   class DataCollectorHandler(TaskHandler):
       async def handle_task(self, task):
           # Parse input, call mock APIs, return structured data
           input_data = json.loads(task.message.parts[0].text)
           property_data = await self.collect_all_data(input_data["address"])
           return TaskResult(
               artifacts=[Artifact(parts=[TextPart(text=json.dumps(property_data))])]
           )
   ```

---

## 9. Workflow Design

### 9.1 Sequential Workflow — End-to-End Underwriting

This is the primary workflow. Steps execute in strict order because each depends on the previous step's output.

```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ STEP 1  │───►│ STEP 2   │───►│ STEP 3   │───►│ STEP 4   │───►│ STEP 5   │───►│ STEP 6   │
│ Profile  │    │ Intake   │    │ Data     │    │ Risk     │    │ Premium  │    │ Report   │
│ User     │    │ Property │    │ Collect  │    │ Assess   │    │ Calc     │    │ Generate │
│          │    │ Details  │    │          │    │          │    │          │    │          │
│ Orch.    │    │ Orch.    │    │ Collector│    │ Assessor │    │ Reporter │    │ Reporter │
│ Agent    │    │ Agent    │    │ Agent    │    │ Agent    │    │ Agent    │    │ Agent    │
└─────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

**Step Details:**

| Step | Agent | Input | Output | Prompt Type |
|------|-------|-------|--------|-------------|
| 1. Profile User | Orchestrator | User's first message | User profile (level, prefs) | Few-shot (profiling examples) |
| 2. Intake Property | Orchestrator | Chat conversation | Structured property input | Zero-shot + ReAct (validation tools) |
| 3. Collect Data | Data Collector | Address + property type | Full property data package | ReAct (tool calls to mock APIs) |
| 4. Assess Risk | Risk Assessor | Property data package | 4 risk scores + overall grade | Chain-of-Thought (per risk factor) |
| 5. Calculate Premium | Report Generator | TIV + risk score | Premium range | Zero-shot (formula application) |
| 6. Generate Report | Report Generator | All data + user profile | Formatted report | Few-shot (report examples) |

### 9.2 Parallel Workflow — Risk Factor Assessment

Within Step 4 above, the Risk Assessment Agent runs 4 assessments **simultaneously** (MVP):

```
                              Property Data
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
              ┌─────▼────┐  ┌─────▼────┐  ┌─────▼────┐
              │ Fire Risk │  │ NatCat   │  │ Occupancy│
              │ Score: 7  │  │ Score: 4 │  │ Score: 5 │
              └─────┬────┘  └─────┬────┘  └─────┬────┘
                    │              │              │
                    │        ┌─────▼────┐        │
                    │        │ Building │        │
                    │        │ Score: 6 │        │
                    │        └─────┬────┘        │
                    │              │              │
                    └──────────────┼──────────────┘
                                   │
                              ┌────▼────┐
                              │ COMBINE │
                              │ Weighted│
                              │ Average │
                              │         │
                              │ Overall │
                              │ Score:  │
                              │  5.5    │
                              │ MEDIUM  │
                              └─────────┘
```

> **Stretch:** Add Protection Risk and Loss History Risk for 6 total parallel assessments.

**ADK Implementation — Parallel Execution:**

```python
import asyncio
from google.adk import Agent

class RiskAssessmentAgent:
    async def assess_all_risks(self, property_data: dict) -> dict:
        """Run all risk assessments in parallel using asyncio (4 MVP, 6 stretch)."""

        # Define the assessment coroutines (MVP: 4 categories)
        tasks = [
            self.assess_fire_risk(property_data),
            self.assess_natcat_risk(property_data),
            self.assess_occupancy_risk(property_data),
            self.assess_building_risk(property_data),
            # Stretch: uncomment for 6 categories
            # self.assess_protection_risk(property_data),
            # self.assess_loss_history_risk(property_data),
        ]

        # Execute all in parallel
        results = await asyncio.gather(*tasks)

        # Combine into overall score (weighted)
        weights = {
            "fire": 0.30, "natcat": 0.25, "occupancy": 0.20,
            "building": 0.25,
            # Stretch weights (re-balance when adding 2 more):
            # "fire": 0.20, "natcat": 0.20, "occupancy": 0.15,
            # "building": 0.15, "protection": 0.15, "loss_history": 0.15
        }

        overall_score = sum(
            result["score"] * weights[result["category"]]
            for result in results
        )

        return {
            "individual_scores": results,
            "overall_score": round(overall_score, 1),
            "risk_grade": self.score_to_grade(overall_score)
        }

    def score_to_grade(self, score: float) -> str:
        if score <= 3: return "LOW"
        elif score <= 6: return "MEDIUM"
        elif score <= 8: return "HIGH"
        else: return "VERY HIGH"
```

### 9.3 Conversational Workflow — Intake Flow

The Orchestrator Agent's conversation follows a structured but flexible flow:

```
START
  │
  ▼
[Greeting & Profiling]
  "Welcome! I'm your underwriting assistant."
  "Before we start, are you familiar with commercial property underwriting,
   or would you like me to explain things as we go?"
  │
  ▼
[Property Address]
  "What's the address of the property you'd like to evaluate?"
  (validates address format)
  │
  ▼
[Property Type]
  "What type of commercial property is this?"
  Options: Office, Warehouse, Manufacturing, Retail, Mixed-Use, Other
  │
  ▼
[Key Details]
  "Can you provide the following details?"
  - Total Insured Value (TIV)
  - Insured's business name
  - Occupancy description
  - Any known hazards or concerns?
  │
  ▼
[Confirmation]
  "Here's what I have so far: [summary]. Shall I proceed with the assessment?"
  │
  ├── Yes ──► [Trigger Data Collection + Assessment]
  │
  └── No / Corrections ──► [Loop back to relevant step]
```

### 9.4 Non-Conversational Workflow — Batch Mode

For senior underwriters who want to submit multiple properties:

```
Input: JSON file with multiple property submissions
  │
  ▼
[Parse & Validate] ──► For each property:
  │                          │
  ▼                          ├──► [Data Collection]
[Queue Properties]           ├──► [Risk Assessment (parallel)]
  │                          └──► [Report Generation]
  ▼
[Output: Batch report with all properties assessed]
```

This mode skips the conversational intake — the underwriter provides a structured input file.

---

## 10. Prompt Engineering Strategy

### 10.1 Prompt Types Used

This project uses **4 distinct prompt engineering techniques**, each chosen for the right task:

#### Type 1: Zero-Shot Prompting
**Used for:** Simple, well-defined tasks where the model's training is sufficient.

**Where in this project:**
- General underwriter queries ("What is a protection class?")
- Address validation
- Premium calculation narrative

**Example:**
```
You are a commercial property underwriting assistant.
The user is asking about the property at {address}.
Answer their question concisely and accurately.

User question: {question}
```

#### Type 2: Few-Shot Prompting
**Used for:** Tasks where examples significantly improve consistency and format.

**Where in this project:**
- User profiling (classifying underwriter experience level)
- Report generation (consistent format)
- Risk grade explanation

**Example (User Profiling):**
```
Based on the user's messages, classify their experience level as JUNIOR or SENIOR.

Example 1:
User: "Hi, I'm new to commercial property. Can you walk me through how to evaluate a warehouse?"
Classification: JUNIOR
Reasoning: Explicitly states being new, asks for walkthrough guidance.

Example 2:
User: "Need a quick risk assessment on a Class 6 fire-resistive warehouse, 50K sqft, TIV $25M."
Classification: SENIOR
Reasoning: Uses technical terminology (Class 6, fire-resistive), provides structured data upfront, requests speed.

Example 3:
User: "I have a mixed-use property submission. What should I look at first?"
Classification: JUNIOR
Reasoning: Asks for guidance on prioritization, suggesting limited experience with the assessment process.

Now classify:
User: "{user_message}"
Classification:
```

#### Type 3: Chain-of-Thought (CoT) Prompting
**Used for:** Complex reasoning where transparency and step-by-step logic are critical.

**Where in this project:**
- All risk factor assessments (core of the system)
- Overall risk grade determination
- Premium recommendation justification

**Example (Fire Risk Assessment):**
```
You are an expert commercial property fire risk assessor.

Given the following property data:
- Construction Type: {construction_type}
- Year Built: {year_built}
- Sprinkler System: {sprinkler_status}
- Fire Alarm: {alarm_status}
- Occupancy: {occupancy_type}
- Fire Protection Class: {protection_class}
- Distance to Fire Station: {fire_station_distance}

Assess the fire risk for this property. Think step by step:

Step 1: Evaluate the construction type's inherent fire resistance.
  - Frame construction = high risk, fire-resistive = low risk

Step 2: Assess the fire protection systems.
  - Full sprinkler with monitoring = strong mitigation
  - No sprinkler = significant risk increase

Step 3: Consider the occupancy risk.
  - Manufacturing with chemicals = high risk
  - Standard office = lower risk

Step 4: Evaluate the fire response capability.
  - Protection class 1-3 = excellent, 8-10 = poor
  - Distance affects response time

Step 5: Synthesize all factors into a final assessment.

Provide your output as:
- Fire Risk Score: [1-10]
- Key Risk Factors: [list]
- Mitigating Factors: [list]
- Recommendations: [list]
- Reasoning: [your step-by-step analysis]
```

#### Type 4: ReAct (Reasoning + Acting) Prompting
**Used for:** Tasks requiring tool use — the model reasons about what data it needs, calls a tool, observes the result, and reasons again.

**Where in this project:**
- Data Collector Agent (calls multiple APIs based on what's needed)
- Orchestrator Agent (decides which sub-agent to call next)

**Example (Data Collection with ReAct):**
```
You are a property data collection agent. Your job is to gather comprehensive
data about a commercial property for underwriting.

You have the following tools:
- lookup_property_characteristics(address) - returns building details
- lookup_natural_hazards(lat, lon) - returns hazard zone data
- lookup_location_context(address) - returns fire dept, crime, etc.
- lookup_loss_history(property_id, insured_name) - returns claims history

For the property at: {address}
Insured: {insured_name}

Think about what data you need, call the appropriate tools, and compile
a comprehensive property data package.

Thought: I need to start by getting the basic property characteristics...
Action: lookup_property_characteristics("{address}")
Observation: {result}

Thought: Now I have the basic details. I need hazard data. Let me use the
coordinates from the property lookup...
Action: lookup_natural_hazards({lat}, {lon})
...
```

### 10.2 Prompt Type Selection Matrix

| Task | Prompt Type | Why This Type |
|------|-------------|---------------|
| Greeting / general chat | Zero-shot | Simple, no examples needed |
| User profiling classification | Few-shot | Examples ensure consistent classification |
| Property detail validation | Zero-shot | Straightforward validation logic |
| Property data collection | ReAct | Requires sequential tool calls with reasoning |
| Fire risk assessment | Chain-of-Thought | Complex multi-factor reasoning |
| NatCat risk assessment | Chain-of-Thought | Complex multi-factor reasoning |
| Occupancy risk assessment | Chain-of-Thought | Complex multi-factor reasoning |
| Building risk assessment | Chain-of-Thought | Complex multi-factor reasoning |
| Protection risk assessment | Chain-of-Thought | Complex multi-factor reasoning |
| Loss history assessment | Chain-of-Thought | Pattern analysis requiring reasoning |
| Overall risk synthesis | Chain-of-Thought | Combining 6 scores requires justification |
| Premium calculation | Zero-shot | Formula application, straightforward |
| Report generation | Few-shot | Examples ensure consistent format |
| Follow-up Q&A | Zero-shot | Contextual answers from assessment data |

---

## 11. Prompt Manager Design

### 11.1 What Is the Prompt Manager?

A lightweight Python module that manages all prompt templates across agents. It's deliberately simple — no heavy frameworks, no database — just enough to demonstrate the concept of centralized prompt management.

**Core capabilities:**
1. **Stores** prompt templates as Python string constants, organized by agent
2. **Versions** each template (v1, v2, etc.) so you can compare and iterate
3. **Renders** templates with variables using Python f-strings
4. **Tracks** which prompt version was used per call (feeds into Arize tracing)

> **Design philosophy:** Keep it simple. A dict-based prompt manager is something an intern can build in 1-2 hours and understand completely. It demonstrates the *concept* of prompt management without getting lost in library complexity.

### 11.2 Directory Structure

```
prompt_manager/
├── __init__.py
├── manager.py              # Core PromptManager class
└── templates/
    ├── __init__.py
    ├── orchestrator.py     # All Orchestrator prompts (system, profiling, intake)
    ├── data_collector.py   # Data Collector prompts (ReAct collection)
    ├── risk_assessor.py    # Risk Assessment prompts (CoT for each category)
    └── report_generator.py # Report generation prompts (few-shot format)
```

### 11.3 Core Implementation

```python
# prompt_manager/manager.py

from datetime import datetime


class PromptManager:
    """
    Simple dict-based prompt manager.
    Stores versioned prompt templates for all agents.
    Tracks usage for Arize tracing integration.
    """

    def __init__(self):
        self._templates = {}   # {agent: {template_name: {version: template_str}}}
        self._usage_log = []   # For Arize tracing

    def register(self, agent: str, name: str, version: str, template: str):
        """Register a prompt template."""
        self._templates.setdefault(agent, {}).setdefault(name, {})[version] = template

    def get_prompt(self, agent: str, name: str, variables: dict = None, version: str = "latest") -> str:
        """
        Retrieve and render a prompt template.

        Args:
            agent: Agent name (e.g., "orchestrator", "risk_assessor")
            name: Template name (e.g., "fire_risk_cot")
            variables: Dict of variables to render into the template
            version: Specific version or "latest"

        Returns:
            Rendered prompt string
        """
        agent_templates = self._templates.get(agent, {})
        template_versions = agent_templates.get(name, {})

        if version == "latest":
            version = sorted(template_versions.keys())[-1] if template_versions else "v1"

        template_str = template_versions.get(version, "")

        # Render with variables using str.format_map
        rendered = template_str.format_map(variables or {})

        # Log usage (sent to Arize as span attributes)
        self._usage_log.append({
            "agent": agent,
            "template": name,
            "version": version,
            "timestamp": datetime.utcnow().isoformat(),
        })

        return rendered

    def list_templates(self, agent: str = None) -> dict:
        """List all registered templates with their versions."""
        if agent:
            return {agent: {
                name: list(versions.keys())
                for name, versions in self._templates.get(agent, {}).items()
            }}
        return {
            a: {name: list(vers.keys()) for name, vers in templates.items()}
            for a, templates in self._templates.items()
        }

    def get_usage_log(self) -> list:
        """Return prompt usage log for Arize tracing integration."""
        return self._usage_log
```

### 11.3.1 Registering Templates (Example)

```python
# prompt_manager/templates/risk_assessor.py

# Each template is a Python string with {variable} placeholders.
# Multiple versions allow comparison and iteration.

FIRE_RISK_COT_V1 = """
You are an expert commercial property fire risk assessor.

## Property Data
- Address: {address}
- Construction Type: {construction_type} (Class {construction_class})
- Sprinkler System: {sprinkler_status}
- Fire Alarm: {alarm_status}
- Fire Protection Class: {fire_protection_class}

Assess the fire risk. Think step by step:
1. Construction fire resistance (ISO Classes: 1=Frame/highest → 6=Fire Resistive/lowest)
2. Fire protection systems adequacy
3. Occupancy fire hazard
4. Fire response capability

Output JSON:
{{"category": "fire", "score": <1-10>, "grade": "<LOW|MEDIUM|HIGH>",
  "key_risk_factors": [...], "mitigating_factors": [...],
  "recommendations": [...], "reasoning": "<step-by-step>"}}
"""

# Version 2: More detailed instructions (for A/B testing in Arize)
FIRE_RISK_COT_V2 = """
... (refined prompt with better scoring calibration) ...
"""


def register_all(manager):
    """Register all risk assessor templates."""
    manager.register("risk_assessor", "fire_risk_cot", "v1", FIRE_RISK_COT_V1)
    manager.register("risk_assessor", "fire_risk_cot", "v2", FIRE_RISK_COT_V2)
    # ... register natcat, occupancy, building prompts similarly
```

### 11.3.2 Usage in Agent Code

```python
# Inside any agent:
prompt = prompt_manager.get_prompt(
    agent="risk_assessor",
    name="fire_risk_cot",
    variables={
        "address": property_data["address"],
        "construction_type": property_data["construction_type"],
        "construction_class": property_data["construction_class"],
        "sprinkler_status": property_data["sprinkler_status"],
        "alarm_status": property_data["alarm_status"],
        "fire_protection_class": property_data["fire_protection_class"],
    },
    version="v1"  # or "latest"
)

response = await model.generate_content(prompt)
```

### 11.4 Full CoT Prompt Template Example

The full fire risk CoT template (registered as `FIRE_RISK_COT_V1` in Section 11.3.1 above) demonstrates the Chain-of-Thought pattern. Each risk category follows the same structure — the intern creates one template per risk category, each with:

1. **Role context** — "You are an expert [category] risk assessor"
2. **Property data injection** — All relevant fields for that category
3. **Step-by-step reasoning instructions** — 3-5 assessment steps specific to the category
4. **Structured output format** — JSON with score, grade, factors, recommendations, and reasoning

> **Tip for the intern:** Start by writing the fire risk prompt manually, testing it in Gemini, and iterating until the output is good. Then templatize it and repeat for the other categories. Don't try to write all templates at once.

---

## 12. User Profiling Design

### 12.1 What Is User Profiling?

The system builds a profile of each underwriter through conversation and adapts its behavior to match their expertise, preferences, and working style.

### 12.2 Profile Attributes

Stored as a simple JSON file at `data/profiles/{user_id}.json`:

```json
{
  "user_id": "underwriter_001",
  "name": "Priya Sharma",
  "experience_level": "junior",
  "specialization": ["warehouse", "manufacturing"],
  "preferred_detail_level": "verbose",
  "session_count": 5,
  "last_interaction": "2026-04-18T10:30:00Z",
  "profiling_complete": true
}
```

> **Why JSON files instead of SQLite?** For a 7-day project, JSON files eliminate database setup time while still demonstrating persistence. Each user gets their own file. Simple to read, write, and debug.

### 12.3 Profiling Questions (Asked in First Session)

The Orchestrator asks these questions **naturally in conversation** during first interaction:

| # | Question | Purpose | Maps To |
|---|----------|---------|---------|
| Q1 | "Welcome! Before we start, how long have you been working in commercial property underwriting?" | Determine experience level | `experience_level` |
| Q2 | "Do you specialize in any particular property types — like office, warehouse, manufacturing?" | Identify specialization | `specialization` |
| Q3 | "When you review risk assessments, do you prefer detailed explanations for each factor, or a concise summary with just the scores?" | Set detail level | `preferred_detail_level` |
| Q4 | "For the underwriting report, do you like a narrative write-up, or data tables with key metrics?" | Set format preference | `preferred_format` |

**Key Design Principle:** Questions are asked conversationally, not as a form. The agent weaves them into the natural flow:

```
Agent: "Welcome! I'm your underwriting assistant for commercial properties.
        I'd like to tailor my responses to your style — how long have you
        been working in commercial property underwriting?"

User:  "About 8 years, mostly warehouses and distribution centers."

Agent: "Great, 8 years — you clearly know your way around commercial property!
        I'll keep things concise and data-focused. Do you have a property
        you'd like me to assess today?"
```

### 12.4 Adaptive Behavior Based on Profile

| Behavior | Junior Mode | Senior Mode |
|----------|-------------|-------------|
| **Greeting** | Warm, explains the process | Brief, gets to business |
| **Property intake** | Guided questions one at a time | Accepts bulk structured input |
| **Risk explanations** | Define terms, explain scoring criteria | Show scores + key factors only |
| **Report style** | Narrative with definitions and context | Tables + bullet points |
| **Follow-up** | "Would you like me to explain any of these?" | "Anything to override or adjust?" |
| **Recommendations** | Detailed with reasoning | Brief list |

### 12.5 Implementation

```python
# user_profile/profile_store.py

import json
import os
from datetime import datetime


class UserProfileStore:
    """JSON file-backed user profile storage. One file per user."""

    def __init__(self, profiles_dir: str = "data/profiles"):
        self.profiles_dir = profiles_dir
        os.makedirs(profiles_dir, exist_ok=True)

    def _path(self, user_id: str) -> str:
        return os.path.join(self.profiles_dir, f"{user_id}.json")

    def get_profile(self, user_id: str) -> dict | None:
        path = self._path(user_id)
        if os.path.exists(path):
            with open(path) as f:
                return json.load(f)
        return None

    def save_profile(self, user_id: str, profile: dict):
        profile["last_interaction"] = datetime.utcnow().isoformat()
        with open(self._path(user_id), "w") as f:
            json.dump(profile, f, indent=2)

    def get_or_create(self, user_id: str) -> dict:
        profile = self.get_profile(user_id)
        if not profile:
            profile = {
                "user_id": user_id,
                "experience_level": "unknown",
                "preferred_detail_level": "verbose",  # default to verbose
                "profiling_complete": False,
                "session_count": 0,
            }
            self.save_profile(user_id, profile)
        return profile
```

---

## 13. Arize Integration

### 13.1 What We're Monitoring

| Aspect | What to Track | Why |
|--------|--------------|-----|
| **Traces** | Every LLM call, tool invocation, agent-to-agent message | Debug failures, understand latency |
| **Evaluations** | Risk score quality, report completeness, prompt effectiveness | Ensure output quality |
| **Annotations** | Human review of risk assessments | Build golden dataset, improve prompts |
| **Metrics** | Latency, token usage, cost per assessment, error rates | Operational monitoring |

### 13.2 Setup Steps

#### Step 1: Install Arize SDK

```bash
pip install arize-otel opentelemetry-sdk opentelemetry-api
```

#### Step 2: Initialize Tracing

```python
# telemetry/arize_setup.py

import os
from arize_otel import register_otel, Endpoints

def init_arize_tracing():
    """Initialize Arize OpenTelemetry tracing."""

    # Set environment variables (or pass directly)
    os.environ["ARIZE_SPACE_ID"] = "your-space-id"       # From Arize dashboard
    os.environ["ARIZE_API_KEY"] = "your-api-key"          # From Arize settings

    # Register OpenTelemetry with Arize endpoint
    register_otel(
        endpoints=Endpoints.ARIZE,
        space_id=os.environ["ARIZE_SPACE_ID"],
        api_key=os.environ["ARIZE_API_KEY"],
        model_id="commercial-property-underwriting",       # Project name in Arize
        model_version="1.0.0",
    )

    print("Arize tracing initialized successfully.")
```

#### Step 3: Instrument Agent Calls

```python
# In each agent, wrap LLM calls with OpenTelemetry spans

from opentelemetry import trace

tracer = trace.get_tracer("underwriting-agents")

class RiskAssessmentAgent:
    async def assess_fire_risk(self, property_data: dict) -> dict:
        with tracer.start_as_current_span("fire_risk_assessment") as span:
            # Add attributes for Arize
            span.set_attribute("risk_category", "fire")
            span.set_attribute("property_address", property_data["address"])
            span.set_attribute("construction_type", property_data["construction_type"])

            # Get prompt from Prompt Manager
            prompt = self.prompt_manager.get_prompt(
                agent="risk_assessor",
                template_name="fire_risk_cot",
                variables={"property": property_data}
            )

            # Track prompt version
            span.set_attribute("prompt_template", "fire_risk_cot")
            span.set_attribute("prompt_version", "v1")

            # Call Gemini
            response = await self.model.generate_content(prompt)

            # Track output
            span.set_attribute("risk_score", result["score"])
            span.set_attribute("risk_grade", result["grade"])

            return result
```

#### Step 4: Set Up Annotation Queue

In the Arize dashboard:

1. **Create a new Space** → "Commercial Property Underwriting"
2. **Go to Annotations** → Create new annotation queue
3. **Name:** "Risk Assessment Quality Review"
4. **Filter:** traces where `risk_category` is set
5. **Annotation Schema:**
   - `score_accurate` (boolean) — Is the AI risk score reasonable?
   - `reasoning_quality` (1-5) — How good is the CoT reasoning?
   - `corrected_score` (int, optional) — What should the score be?
   - `notes` (text) — Reviewer comments

#### Step 5: Build Monitoring Dashboard

In Arize, create dashboard widgets for:

| Widget | Metric | Purpose |
|--------|--------|---------|
| Latency over time | p50, p95 agent response time | Performance monitoring |
| Token usage | Input + output tokens per assessment | Cost tracking |
| Risk score distribution | Histogram of risk scores | Detect scoring drift |
| Error rate | % of failed agent calls | Reliability |
| Annotation agreement | Human vs AI score correlation | Quality tracking |
| Prompt version comparison | Performance by prompt version | A/B testing prompts |

### 13.3 Arize Integration Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Orchestrator │     │ Risk         │     │ Report       │
│ Agent        │     │ Assessor     │     │ Generator    │
│              │     │              │     │              │
│ [OTel Spans] │     │ [OTel Spans] │     │ [OTel Spans] │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                     │                    │
       └─────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  OpenTelemetry  │
                    │  Collector      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │                 │
                    │   Arize Cloud   │
                    │                 │
                    │  ┌───────────┐  │
                    │  │ Traces    │  │
                    │  │ Dashboard │  │
                    │  │ Annotate  │  │
                    │  │ Evals     │  │
                    │  └───────────┘  │
                    │                 │
                    └─────────────────┘
```

---

## 14. Deployment Plan

### 14.1 Option A: Vertex AI Agent Engine (Preferred)

If the intern has GCP access:

#### Step 1: Set Up GCP Project

```bash
# Install Google Cloud SDK
# Create a new GCP project or use existing
gcloud config set project YOUR_PROJECT_ID

# Enable required APIs
gcloud services enable aiplatform.googleapis.com
gcloud services enable cloudbuild.googleapis.com
```

#### Step 2: Package the Agent

```python
# deploy/deploy_to_agent_engine.py

from google.cloud import aiplatform

aiplatform.init(
    project="your-project-id",
    location="us-central1",
)

# Create Agent Engine instance
agent_engine = aiplatform.Agent.create(
    display_name="commercial-property-underwriting",
    description="Multi-agent system for commercial property risk assessment",
    # Agent configuration
    agent_config={
        "model": "gemini-2.5-pro",
        "tools": [...],
        "system_instruction": "...",
    }
)

print(f"Agent deployed: {agent_engine.resource_name}")
```

#### Step 3: Test the Deployed Agent

```python
# Send a test query to the deployed agent
response = agent_engine.query(
    query="I need to assess a 50,000 sqft warehouse at 123 Industrial Ave, Houston TX. TIV $25M."
)
print(response)
```

### 14.2 Option B: Local Deployment (Fallback)

If GCP access is not available:

```bash
# Run agents locally using ADK CLI
# Terminal 1: Data Collector Agent
adk run property_data_collector --port 8002

# Terminal 2: Risk Assessment Agent
adk run risk_assessment_agent --port 8003

# Terminal 3: Report Generator Agent
adk run report_generator --port 8004

# Terminal 4: Orchestrator Agent (main)
adk web underwriting_orchestrator --port 8001
```

The ADK web interface provides a chat UI for testing the conversational workflow.

### 14.3 Environment Setup Guide

```bash
# 1. Clone the project
git clone <repo_url>
cd commercial-property-underwriting

# 2. Create virtual environment
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set environment variables
cp .env.example .env
# Edit .env with your API keys:
#   GOOGLE_API_KEY=your-gemini-api-key
#   ARIZE_SPACE_ID=your-arize-space-id
#   ARIZE_API_KEY=your-arize-api-key

# 5. Run the application
adk web underwriting_orchestrator
# Opens browser at http://localhost:8000
```

### 14.4 Requirements File

```
# requirements.txt
google-adk>=0.3.0
google-genai>=1.0.0
a2a-sdk>=0.2.0
arize-otel>=0.5.0
opentelemetry-sdk>=1.20.0
opentelemetry-api>=1.20.0
pydantic>=2.0.0
python-dotenv>=1.0.0
```

> **Note:** No Streamlit, no Jinja2, no SQLite driver needed. We use ADK's built-in web UI, Python f-strings for templates, and JSON files for storage. This keeps the dependency list lean and setup fast.

---

## 15. Data Models

### 15.1 Property Data Schema

```json
{
  "property_id": "PROP-2026-00123",
  "address": {
    "street": "123 Industrial Avenue",
    "city": "Houston",
    "state": "TX",
    "zip": "77001",
    "latitude": 29.7604,
    "longitude": -95.3698
  },
  "characteristics": {
    "property_type": "warehouse",
    "construction_type": "joisted_masonry",
    "construction_class": 3,
    "year_built": 1998,
    "year_renovated": 2015,
    "stories": 1,
    "sqft": 50000,
    "roof_type": "metal_deck",
    "roof_age_years": 9,
    "electrical_update_year": 2015,
    "plumbing_update_year": 2015,
    "hvac_update_year": 2018
  },
  "protection": {
    "sprinkler_status": "full_wet_system",
    "sprinkler_coverage_pct": 100,
    "fire_alarm": "central_station_monitored",
    "security_system": "cctv_with_guards",
    "access_control": "badge_entry",
    "backup_power": true
  },
  "occupancy": {
    "business_name": "GlobalTech Distribution LLC",
    "business_type": "wholesale_distribution",
    "occupancy_description": "Electronics and consumer goods distribution",
    "hazardous_materials": false,
    "operates_24_7": true,
    "employee_count": 150,
    "annual_revenue": 45000000
  },
  "location_context": {
    "fire_protection_class": 3,
    "fire_station_distance_miles": 1.2,
    "nearest_hydrant_feet": 300,
    "crime_index": 42,
    "flood_zone": "X",
    "earthquake_zone": "low",
    "wildfire_risk": "minimal",
    "windstorm_zone": "moderate",
    "coastal_distance_miles": 45
  },
  "loss_history": {
    "prior_claims": [
      {
        "date": "2022-03-15",
        "type": "water_damage",
        "amount": 45000,
        "description": "Roof leak during heavy rain, inventory damage"
      },
      {
        "date": "2019-08-22",
        "type": "theft",
        "amount": 12000,
        "description": "Break-in, electronics stolen from dock area"
      }
    ],
    "total_claims_5yr": 2,
    "total_loss_amount_5yr": 57000,
    "claims_free_years": 3
  },
  "insurance_details": {
    "total_insured_value": 25000000,
    "building_value": 15000000,
    "contents_value": 8000000,
    "business_income_value": 2000000,
    "current_carrier": "National Insurance Co.",
    "current_premium": 62000,
    "policy_effective_date": "2026-07-01",
    "deductible": 25000
  }
}
```

### 15.2 Risk Assessment Output Schema

```json
{
  "assessment_id": "RA-2026-00456",
  "property_id": "PROP-2026-00123",
  "timestamp": "2026-04-18T14:30:00Z",
  "individual_scores": [
    {
      "category": "fire",
      "score": 4,
      "grade": "MEDIUM",
      "weight": 0.20,
      "key_risk_factors": [
        "Joisted masonry construction (Class 3) has moderate fire resistance",
        "24/7 operations increase ignition opportunity window"
      ],
      "mitigating_factors": [
        "Full wet sprinkler system with 100% coverage",
        "Central station monitored fire alarm",
        "Fire Protection Class 3 — excellent fire department response"
      ],
      "recommendations": [
        "Verify sprinkler inspection records are current",
        "Confirm electrical system updated in 2015 renovation"
      ],
      "reasoning": "Step 1: Construction Class 3 (joisted masonry) provides moderate fire resistance..."
    },
    {
      "category": "natural_catastrophe",
      "score": 3,
      "grade": "LOW",
      "weight": 0.20,
      "key_risk_factors": ["Moderate windstorm zone (Gulf Coast proximity)"],
      "mitigating_factors": ["Flood Zone X (minimal flood risk)", "Low earthquake zone", "45 miles from coast"],
      "recommendations": ["Review windstorm deductible adequacy"],
      "reasoning": "Step 1: Flood Zone X indicates minimal flood risk..."
    }
    // ... 4 more categories
  ],
  "overall_score": 4.1,
  "overall_grade": "MEDIUM",
  "premium_recommendation": {
    "base_rate_per_thousand": 2.10,
    "adjusted_rate_per_thousand": 2.48,
    "recommended_premium": 62000,
    "premium_range": {
      "min": 55000,
      "max": 72000
    },
    "rate_justification": "Base rate for warehouse Class 3, adjusted +18% for moderate overall risk"
  },
  "model_info": {
    "model": "gemini-2.5-pro",
    "prompt_versions": {
      "fire_risk_cot": "v1",
      "natcat_risk_cot": "v1"
    },
    "total_tokens": 12450,
    "latency_seconds": 8.3
  }
}
```

### 15.3 Mock Data Strategy

Since this is a demo, all external data sources are **simulated**:

```python
# mock_apis/property_api.py

MOCK_PROPERTIES = {
    "123 Industrial Ave, Houston, TX": {
        "construction_type": "joisted_masonry",
        "construction_class": 3,
        "year_built": 1998,
        # ... full property data
    },
    "456 Commerce Blvd, Chicago, IL": {
        "construction_type": "fire_resistive",
        "construction_class": 6,
        "year_built": 2010,
        # ...
    },
    # 8-10 sample properties covering different types and risk profiles
}

def lookup_property_characteristics(address: str) -> dict:
    """Simulated property data API."""
    # Fuzzy match the address
    for key, data in MOCK_PROPERTIES.items():
        if key.lower() in address.lower() or address.lower() in key.lower():
            return data
    # Generate reasonable defaults for unknown addresses
    return generate_default_property(address)
```

**The intern should create 8-10 diverse mock properties:**

| # | Address | Type | Risk Profile | Purpose |
|---|---------|------|-------------|---------|
| 1 | 123 Industrial Ave, Houston TX | Warehouse | Medium | Standard case |
| 2 | 456 Commerce Blvd, Chicago IL | Office | Low | Low-risk baseline |
| 3 | 789 Factory Rd, Detroit MI | Manufacturing | High | High-risk with hazmat |
| 4 | 101 Retail Plaza, Miami FL | Retail Complex | Medium-High | Coastal/windstorm |
| 5 | 222 Tech Park Dr, San Jose CA | Office Campus | Medium | Earthquake zone |
| 6 | 333 Harbor Way, New Orleans LA | Warehouse | High | Flood zone |
| 7 | 444 Mill Creek Rd, Portland OR | Manufacturing | Medium | Wildfire zone |
| 8 | 555 Downtown St, New York NY | Mixed-Use | Medium | Dense urban, old building |

---

## 16. Tech Stack & Tools

### 16.1 Complete Tech Stack

| Category | Technology | Version | Purpose |
|----------|-----------|---------|---------|
| **Language** | Python | 3.11+ | Primary development language |
| **Agent Framework** | Google ADK | Latest | Agent development and orchestration |
| **LLM** | Gemini 2.5 Flash | Latest | Data collection, report generation |
| **LLM** | Gemini 2.5 Pro | Latest | Risk assessment, complex reasoning |
| **Agent Communication** | A2A SDK | Latest | Inter-agent communication |
| **UI** | **ADK Web** (built-in) | Latest | Chat UI via `adk web` — zero build needed |
| **Prompt Templates** | Python strings (f-string) | N/A | Simple, no extra dependency |
| **Storage** | JSON files | Built-in | User profiles, mock data |
| **Monitoring** | Arize (Cloud) | Latest | Tracing, annotation, evaluation |
| **Tracing** | OpenTelemetry | 1.20+ | Distributed tracing framework |
| **Data Validation** | Pydantic | 2.0+ | Data model validation |
| **Config** | python-dotenv | 1.0+ | Environment variable management |
| **Deployment** | Vertex AI Agent Engine | Latest | Production deployment (GCP) |
| **Deployment (Alt)** | ADK CLI (adk web/run) | Latest | Local development and testing |

### 16.2 Development Tools

| Tool | Purpose | Installation |
|------|---------|-------------|
| VS Code | IDE | Download from code.visualstudio.com |
| Python 3.11+ | Runtime | python.org or pyenv |
| Git | Version control | git-scm.com |
| Google Cloud SDK | GCP deployment (optional) | cloud.google.com/sdk |

---

## 17. Development Plan — 7-Day Sprint + 1-Day Stretch

### Overview

```
Day 1          Day 2          Day 3          Day 4          Day 5          Day 6          Day 7         [Day 8]
┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
│ Setup  │───►│ Data   │───►│ Risk   │───►│ Orch + │───►│ A2A +  │───►│ Arize  │───►│ Demo   │───►│Stretch │
│ + Mock │    │Collect │    │Assess  │    │ Report │    │Profile │    │+Deploy │    │ Prep   │    │ Goals  │
│ Data   │    │ Agent  │    │ Agent  │    │ Agent  │    │        │    │        │    │        │    │        │
└────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘
 Foundation    1st Agent      2nd Agent     Wire it up    A2A + UX     Observe      Ship it!      Polish
```

### Day 1: Foundation + Mock Data
**Goal:** Project scaffolded, Gemini connected, mock data ready.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 1.1 | Set up Python project | Create venv, install `google-adk`, `google-genai`, `python-dotenv` | `pip list` shows packages |
| 1.2 | Get Gemini API key | Create key at ai.google.dev, save in `.env` | Test call returns response |
| 1.3 | Hello World agent | Create a minimal ADK agent, run with `adk web`, chat with it | Agent responds in browser |
| 1.4 | Create project structure | Set up folders: `agents/`, `prompt_manager/`, `mock_apis/`, `user_profile/`, `data/` | Folders exist |
| 1.5 | Build mock data APIs | Create 4-5 mock properties with full data (see Section 15.3) | `lookup_property("123 Industrial Ave")` returns data |
| 1.6 | Build Prompt Manager | Implement the simple dict-based PromptManager class (Section 11.3) | `manager.get_prompt(...)` works |

**Key learning:** ADK basics, project structure, how Gemini API works.

**Suggested project structure after Day 1:**
```
commercial-property-underwriting/
├── agents/
│   ├── __init__.py
│   ├── orchestrator.py       # Skeleton only
│   ├── data_collector.py     # Skeleton only
│   ├── risk_assessor.py      # Skeleton only
│   └── report_generator.py   # Skeleton only
├── prompt_manager/
│   ├── __init__.py
│   ├── manager.py             # PromptManager class (working)
│   └── templates/
│       ├── __init__.py
│       ├── orchestrator.py    # Prompt strings
│       ├── data_collector.py
│       ├── risk_assessor.py
│       └── report_generator.py
├── mock_apis/
│   ├── __init__.py
│   ├── property_api.py        # 4-5 mock properties
│   ├── hazard_api.py
│   └── location_api.py
├── user_profile/
│   ├── __init__.py
│   └── profile_store.py       # JSON file-based store
├── a2a/
│   └── agent_cards/           # Static agent card JSON files
├── data/
│   ├── profiles/              # User profile JSON files
│   └── mock_properties.json
├── telemetry/
│   ├── __init__.py
│   └── arize_setup.py         # Skeleton only
├── main.py
├── requirements.txt
├── .env.example
└── README.md
```

---

### Day 2: Data Collector Agent (ReAct Pattern)
**Goal:** First real agent — collects property data using tool calls.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 2.1 | Register data collection prompts | Add ReAct prompt templates to Prompt Manager | Prompts retrievable |
| 2.2 | Implement tool functions | Wire `lookup_property_characteristics()`, `lookup_natural_hazards()`, `lookup_location_context()` as ADK tools | Tools return mock data |
| 2.3 | Build Data Collector Agent | Create ADK agent with system instruction + tools | Agent calls tools and returns structured property data |
| 2.4 | Test with `adk web` | Run agent, provide an address, verify data collection | Complete property data package returned |

**Key learning:** ReAct prompting, ADK tool/function calling, how agents decide which tools to use.

**Prompt type practiced:** ReAct (Reasoning + Acting)

---

### Day 3: Risk Assessment Agent (Chain-of-Thought + Parallel)
**Goal:** Second agent — evaluates 4 risk categories in parallel using CoT prompts.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 3.1 | Write CoT prompts | Create Chain-of-Thought prompt for each risk category: Fire, NatCat, Occupancy, Building | 4 prompt templates registered |
| 3.2 | Implement parallel assessment | Use `asyncio.gather()` to run 4 risk assessments simultaneously | All 4 run in parallel, results combined |
| 3.3 | Build Risk Assessment Agent | ADK agent that receives property data and returns scored risk assessment | Agent returns 4 scores + overall grade |
| 3.4 | Test with sample property | Feed mock property data, verify scores are reasonable | Scores 1-10 with explanations |

**Key learning:** Chain-of-Thought prompting, parallel execution with asyncio, structured LLM output.

**Prompt type practiced:** Chain-of-Thought (CoT)

> **Tip:** Start with the Fire Risk prompt. Get it working well, then copy the pattern for NatCat, Occupancy, and Building. Don't write all 4 from scratch — template from the first one.

---

### Day 4: Orchestrator + Report Generator (Sequential Workflow)
**Goal:** Wire everything together — Orchestrator manages the end-to-end sequential flow.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 4.1 | Build Report Generator Agent | ADK agent that takes property data + risk scores → produces Markdown report | Formatted report output |
| 4.2 | Write report generation prompts | Few-shot prompt with 1-2 example report snippets for consistent formatting | Report format is consistent |
| 4.3 | Build Orchestrator Agent | Main agent that: collects property details via chat → calls Data Collector → calls Risk Assessor → calls Report Generator | End-to-end flow works |
| 4.4 | Wire sub-agent delegation | Orchestrator calls sub-agents as ADK sub-agents (built-in pattern) | Sequential: Intake → Data → Risk → Report |
| 4.5 | Test end-to-end | Run `adk web`, chat as an underwriter, get a complete assessment and report | Full demo flow works |

**Key learning:** ADK sub-agent pattern, sequential workflow design, few-shot prompting.

**Prompt types practiced:** Zero-shot (intake), Few-shot (report format), ReAct (orchestration tool calls)

> **This is the most critical day.** By end of Day 4, you should have a working end-to-end demo. Everything after this adds polish and tech coverage.

---

### Day 5: A2A Agent Cards + User Profiling
**Goal:** Add A2A protocol artifacts and user-adaptive behavior.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 5.1 | Create Agent Cards | Write agent card JSON for all 4 agents (see Section 8.2) | 4 JSON files in `a2a/agent_cards/` |
| 5.2 | Implement A2A demo endpoint | Add A2A server to Data Collector agent — serves agent card and accepts one task | `curl localhost:8002/.well-known/agent.json` works |
| 5.3 | A2A client call from Orchestrator | Orchestrator makes at least one call via A2A client (to Data Collector) | A2A task delegation works for data collection |
| 5.4 | Build user profiling | Add profiling questions to Orchestrator's first-turn conversation | Agent asks experience level, adapts style |
| 5.5 | Implement profile persistence | Save/load user profiles using JSON file store (Section 12.5) | Profile persists across sessions |
| 5.6 | Write profiling prompt (few-shot) | Few-shot prompt to classify user as junior/senior from their messages | Classification works |
| 5.7 | Implement adaptive behavior | Orchestrator adjusts verbosity based on profile: verbose (junior) vs. concise (senior) | Two demo scenarios show different styles |

**Key learning:** A2A protocol, Agent Cards, user profiling, adaptive prompt engineering.

**Prompt type practiced:** Few-shot (user classification)

---

### Day 6: Arize Integration + Deployment
**Goal:** Add observability and deploy.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 6.1 | Set up Arize account | Sign up at arize.com, create Space, get API key | Dashboard accessible |
| 6.2 | Initialize tracing | Add `arize-otel` setup code (Section 13.2), instrument main agent calls | Traces appear in Arize dashboard |
| 6.3 | Add span attributes | Tag traces with: agent name, prompt version, risk category, property address | Traces are filterable in Arize |
| 6.4 | Set up annotation queue | Create an annotation queue for risk assessment quality review (Section 13.2 Step 4) | Risk assessments can be reviewed/annotated |
| 6.5 | Deploy or finalize local | **Option A:** Deploy to Vertex AI Agent Engine if GCP access available. **Option B:** Document `adk web` local deployment as the production setup. | Agent accessible for demo |
| 6.6 | Test deployed agent | Run 2-3 test assessments, verify traces flow to Arize | Traces visible in Arize for test runs |

**Key learning:** OpenTelemetry tracing, LLM observability, annotation workflows, cloud deployment.

---

### Day 7: Integration Testing + Demo Preparation
**Goal:** Polish, test all scenarios, prepare for the demo.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 7.1 | Test Scenario 1: Junior Underwriter | Run the junior underwriter warehouse scenario (Section 18.1) end-to-end | Guided, detailed flow works |
| 7.2 | Test Scenario 2: Senior Underwriter | Run the senior underwriter factory scenario (Section 18.2) end-to-end | Concise, fast flow works |
| 7.3 | Test with all mock properties | Run assessment on all 4-5 mock properties, verify reasonable results | No crashes, scores make sense |
| 7.4 | Verify Arize traces | Check Arize dashboard has traces for all test runs, annotation queue works | Traces + annotations visible |
| 7.5 | Write demo script | Document the exact steps/inputs for the 2 demo scenarios | Demo script ready |
| 7.6 | Rehearse demo (dry run) | Run through both scenarios as if presenting | Smooth, no errors |
| 7.7 | Update README | Document how to set up and run the project | Someone else can run it |

---

### Day 8 (Stretch): Polish & Enhancements

If time allows, these additions make the demo even more impressive:

| # | Enhancement | Effort | Impact |
|---|------------|--------|--------|
| S1 | Add 2 more risk categories (Protection, Loss History) | 2-3 hours | Shows 6 parallel assessments instead of 4 |
| S2 | Full A2A for all sub-agents | 2-3 hours | All 3 sub-agents use A2A protocol |
| S3 | Add JSON export to Report Generator | 1 hour | Machine-readable output |
| S4 | Build Arize monitoring dashboard | 1-2 hours | Latency/cost/score distribution widgets |
| S5 | Prompt v2 and A/B comparison in Arize | 1-2 hours | Demonstrates prompt iteration |
| S6 | Add more mock properties (total 8-10) | 1-2 hours | More diverse demo scenarios |

---

### Daily Checkpoint Validation

Use this checklist to verify each day's work is solid before moving on:

| Day | Checkpoint Question | Must Answer YES |
|-----|-------------------|----------------|
| 1 | Can I chat with a basic ADK agent in the browser? | ✅ |
| 2 | Does the Data Collector agent return structured property data when given an address? | ✅ |
| 3 | Does the Risk Assessor run 4 assessments in parallel and return scored results? | ✅ |
| 4 | Can I chat with the Orchestrator and get a complete assessment + report end-to-end? | ✅ |
| 5 | Are Agent Cards created? Does user profiling change the Orchestrator's response style? | ✅ |
| 6 | Do traces appear in Arize? Is the annotation queue set up? | ✅ |
| 7 | Can I smoothly run both demo scenarios (junior + senior) without errors? | ✅ |

---

## 18. Sample Interactions

### 18.1 Scenario: Junior Underwriter — Guided Warehouse Assessment

```
AGENT:  Welcome! I'm your commercial property underwriting assistant.
        I can help you evaluate risk and generate underwriting reports.

        Before we start, how long have you been working in commercial
        property underwriting?

USER:   Hi! I'm fairly new — just about a year.

AGENT:  Thanks, great to have you here! I'll make sure to explain things
        clearly as we go. Do you primarily work with any particular type
        of commercial property?

USER:   Mostly warehouses so far.

AGENT:  Perfect — warehouses are a great starting point. They have some
        unique risk factors I'll walk you through.

        Do you have a property you'd like me to assess today?

USER:   Yes, it's a warehouse at 123 Industrial Ave, Houston, TX.
        TIV is $25 million.

AGENT:  Got it! Let me collect some details about that property...

        📍 123 Industrial Ave, Houston, TX

        I'm pulling property data now — this includes building
        characteristics, hazard exposure, and location context.

        [Delegating to Data Collector Agent...]

        ✅ Data collected! Here's what I found:

        | Detail | Value |
        |--------|-------|
        | Type | Warehouse |
        | Construction | Joisted Masonry (Class 3) |
        | Year Built | 1998 (renovated 2015) |
        | Size | 50,000 sqft, 1 story |
        | Sprinklers | Full wet system, 100% coverage |
        | Fire Protection Class | 3 |
        | Flood Zone | X (minimal risk) |
        | Occupancy | Electronics distribution, 24/7 |

        Does this look correct? Any corrections before I run the risk assessment?

USER:   That looks right. Go ahead.

AGENT:  Running the risk assessment now. I'm evaluating 4 risk categories
        simultaneously...

        [Delegating to Risk Assessment Agent — 4 parallel assessments...]

        ✅ Assessment complete! Here are the results:

        📊 **Risk Assessment Summary**

        | Category | Score | Grade | Key Factor |
        |----------|-------|-------|------------|
        | 🔥 Fire | 4/10 | MEDIUM | Good sprinklers offset Class 3 construction |
        | 🌊 Natural Catastrophe | 3/10 | LOW | Flood Zone X, minimal earthquake risk |
        | 🏭 Occupancy | 5/10 | MEDIUM | 24/7 operations, electronics increase value risk |
        | 🏗️ Building Condition | 3/10 | LOW | Well-maintained, recent renovation |

        **Overall Risk Score: 3.8 / 10 — LOW-MEDIUM**

        💡 What does this mean? A LOW-MEDIUM score means this property is
        a relatively good risk. The building is well-maintained with strong
        fire protection systems. The main risk factors are the 24/7
        operations and moderate coastal windstorm exposure.

        💰 **Premium Recommendation: $55,000 – $65,000**

        Would you like me to generate the full underwriting report, or
        do you have questions about any of the risk factors?

USER:   Can you explain the fire risk score more?

AGENT:  Of course! Here's the detailed fire risk analysis:

        **Fire Risk Score: 4/10 (MEDIUM)**

        The assessment considered 4 factors:

        1️⃣ **Construction (moderate risk)**
        The building is Joisted Masonry — that's ISO Construction Class 3.
        "Joisted masonry" means the exterior walls are masonry (brick/stone)
        but the roof and floor structures are combustible wood. This is
        middle-of-the-road for fire resistance — better than frame (Class 1)
        but not as good as non-combustible (Class 4) or fire-resistive (Class 6).

        2️⃣ **Fire Protection (strong mitigation)**
        Full wet sprinkler system with 100% coverage is the gold standard.
        Combined with a central station monitored fire alarm, this significantly
        reduces both the chance of a fire getting out of control and the
        potential loss severity.

        3️⃣ **Occupancy (moderate concern)**
        Electronics distribution operations run 24/7, which means more
        opportunity for an ignition event. However, the goods themselves
        aren't highly flammable.

        4️⃣ **Fire Response (excellent)**
        Fire Protection Class 3 with a fire station only 1.2 miles away
        means fast response times. Classes 1-4 are considered excellent.

        **Bottom line:** The strong protection systems (sprinklers + alarm)
        do the heavy lifting here, keeping what would be a higher risk
        (due to construction class) down to a manageable MEDIUM score.

        Any other questions, or shall I generate the report?

USER:   Generate the report please.

AGENT:  [Delegating to Report Generator Agent...]

        ✅ Report generated! Here's your underwriting report:

        ---
        # UNDERWRITING REPORT
        ## 123 Industrial Ave, Houston, TX

        **Date:** April 18, 2026
        **Underwriter:** [Your Name]
        **Overall Risk Grade:** LOW-MEDIUM (3.4/10)
        **Recommended Premium:** $55,000 – $65,000

        ... [Full formatted report continues] ...
        ---

        The report is also saved as JSON for your records system.
        Is there anything you'd like to adjust or any other property to assess?
```

### 18.2 Scenario: Senior Underwriter — Quick Assessment

```
AGENT:  Welcome back! I see you've assessed 45 properties with us.
        What property would you like to evaluate today?

USER:   789 Factory Rd, Detroit MI. Manufacturing plant, TIV $40M.
        Let me know if the hazmat flag comes up.

AGENT:  Running full assessment on 789 Factory Rd, Detroit MI...

        [Data Collection + Risk Assessment running...]

        ⚠️ **ALERT — High Risk Property**

        | Category | Score | Flag |
        |----------|-------|------|
        | Fire | 8/10 | ⚠️ Hazmat + aging electrical |
        | NatCat | 2/10 | ✅ |
        | Occupancy | 7/10 | ⚠️ Chemical manufacturing |
        | Building | 6/10 | ⚠️ Built 1975, partial renovation |

        **Overall: 5.8 / 10 — MEDIUM-HIGH**
        **Premium Range: $145,000 – $185,000**

        ⚡ Yes, hazmat flag is raised — chemical solvents stored on-site.
        Recommend CERC rider and environmental liability review.

        Generate report or adjust?
```

---

## 19. Acceptance Criteria

### 19.1 MVP — Must Demo (Days 1-7)

These are the **non-negotiable** deliverables for the demo:

| # | Criterion | Verification | Tech Topic Covered |
|---|-----------|-------------|-------------------|
| AC-01 | Conversational intake works for at least 2 property types | Demo with warehouse + manufacturing | ADK |
| AC-02 | Data Collector Agent returns structured property data via tool calls | JSON output matches schema | ADK + ReAct prompting |
| AC-03 | Risk Assessment runs **4 categories in parallel** (Fire, NatCat, Occupancy, Building) | Visible parallel execution, all 4 scores returned | ADK + CoT prompting + Parallel |
| AC-04 | Risk scores include Chain-of-Thought reasoning | Each score has step-by-step explanation | Prompt Engineering (CoT) |
| AC-05 | Report Generator produces formatted Markdown report | Readable report with all sections | ADK + Few-shot prompting |
| AC-06 | User profiling detects junior vs senior underwriter | Different output styles for each (demo both) | User Profiling |
| AC-07 | Agent Cards (JSON) created for all 4 agents | Files exist with valid schema | A2A |
| AC-08 | At least 1 A2A call works (Orchestrator → Data Collector) | A2A task delegation demonstrated | A2A |
| AC-09 | Prompt Manager serves versioned templates | Templates loaded via manager, version tracked | Prompt Manager |
| AC-10 | Arize traces visible in dashboard | All LLM calls traced | Arize |
| AC-11 | Arize annotation queue set up | Risk assessments can be reviewed | Arize |
| AC-12 | Running locally via `adk web` **OR** deployed to Agent Engine | Accessible demo | Agent Engine / Deployment |

**Count: 6/6 tech topics covered in MVP** ✅

### 19.2 Stretch — Nice to Show (Day 8)

| # | Criterion | Tech Topic |
|---|-----------|-----------|
| AC-13 | 6 parallel risk categories (add Protection, Loss History) | ADK + Parallel |
| AC-14 | All 3 sub-agents callable via full A2A protocol | A2A |
| AC-15 | JSON export in Report Generator | ADK |
| AC-16 | Arize dashboard with latency and cost widgets | Arize |
| AC-17 | Prompt v1 vs v2 comparison visible in Arize | Prompt Manager + Arize |
| AC-18 | 8+ mock properties for diverse scenarios | Mock Data |

---

## 20. References & Resources

### 20.1 Google ADK
- ADK documentation: https://google.github.io/adk-docs/
- ADK GitHub: https://github.com/google/adk-python
- ADK quickstart: https://google.github.io/adk-docs/get-started/quickstart/

### 20.2 A2A Protocol
- A2A specification: https://github.com/google/A2A
- A2A Python SDK: https://github.com/google/A2A/tree/main/samples/python

### 20.3 Gemini
- Gemini API: https://ai.google.dev/docs
- Google Gen AI SDK: https://github.com/googleapis/python-genai

### 20.4 Arize
- Arize docs: https://docs.arize.com/
- Arize LLM tracing: https://docs.arize.com/arize/llm-large-language-models/llm-traces
- OpenTelemetry for LLMs: https://docs.arize.com/arize/llm-large-language-models

### 20.5 Vertex AI Agent Engine
- Agent Engine docs: https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/overview

### 20.6 Insurance Domain
- ISO Construction Classes: 1 (Frame), 2 (Joisted Masonry), 3 (Non-Combustible), 4 (Masonry Non-Combustible), 5 (Modified Fire Resistive), 6 (Fire Resistive)
- ISO Fire Protection Classes: 1 (Best) to 10 (Unprotected)
- FEMA Flood Zones: A, AE (high risk), X (minimal risk), V (coastal)

---

*Document Version: 2.0 (7-Day Sprint Edition)*
*Created: April 18, 2026*
*Author: AI Planning Assistant*
*Status: Ready for Review*
*Scope: 7-day MVP + 1-day Stretch*
