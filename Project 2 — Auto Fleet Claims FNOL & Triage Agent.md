# Project 2: Auto Fleet Claims FNOL & Triage Agent

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
A conversational multi-agent AI system for First Notice of Loss (FNOL) intake and claims triage for large commercial auto fleets — logistics companies, transport operators, and delivery services with 50–5,000+ vehicles. The system guides claims adjusters through structured incident reporting, automatically classifies incident severity (minor/moderate/major/catastrophic), estimates loss reserves, and routes claims to the appropriate handler — all through a sequential, chat-driven workflow.

### Why
Commercial auto fleet claims are high-volume, time-sensitive, and inconsistent. Large fleets experience hundreds of incidents per year across geographies, vehicle types, and incident categories. Today, FNOL intake is a manual phone-and-form process riddled with missed data fields, inconsistent severity assignments, and slow handler routing. Junior adjusters frequently under-classify severity or set inappropriate reserves, while senior adjusters waste time on repetitive data entry when they should be applying expert judgment. An AI agent system can:
- Reduce FNOL intake time from 45–60 minutes to under 10 minutes per claim
- Ensure every incident captures complete, validated data from day one
- Standardize severity classification and reserve estimation across all adjusters
- Route claims to the correct specialist handler immediately, eliminating reassignment delays
- Free experienced adjusters to focus on complex multi-vehicle and litigation-prone incidents

### How
Four specialized agents communicate in a strict sequential workflow:
1. **Claims Orchestrator Agent** — Converses with the adjuster, coordinates the entire FNOL workflow
2. **Incident Data Collector Agent** — Gathers vehicle, driver, police report, and weather data from mock external APIs
3. **Severity & Triage Agent** — Classifies severity, estimates reserves, and determines handler routing using Chain-of-Thought reasoning
4. **FNOL Report Generator Agent** — Produces formatted FNOL reports ready for the claims management system

Built with Google ADK + Gemini 2.5, monitored via Arize Phoenix and OpenTelemetry, deployed to Vertex AI Agent Engine.

### Timeline & Scope

| Attribute | Detail |
|-----------|--------|
| **Duration** | 7–8 working days |
| **Intern Level** | CS background, medium Python expertise |
| **Delivery** | Working demo with 2 scenarios (junior adjuster Maria + senior adjuster Tom) |

The project is organized into **MVP (Must-ship in 7 days)** and **Stretch (Day 8 if time permits)** scopes. The MVP alone covers all 6 tech topics and produces an impressive demo. Stretch goals add polish and advanced triage features.

### Key Tech Stack Coverage

| Tech Topic | How It's Used in This Project | Scope |
|------------|------------------------------|-------|
| Prompt Engineering | Zero-shot (general queries, validation), Few-shot (severity classification with examples, user profiling), CoT (reserve estimation, severity reasoning, handler routing), ReAct (data collection tool calls, orchestration tool calls) | **MVP** |
| Prompt Manager | Version and serve all prompt templates across 4 agents using a lightweight dict-based Python module with `str.format_map` | **MVP** |
| ADK | All 4 agents built with Google Agent Development Kit; UI via `adk web` | **MVP** |
| A2A | Agents publish Agent Cards; Orchestrator delegates tasks to sub-agents via A2A protocol; 1 real A2A endpoint in MVP | **MVP** |
| User Profiling | Profile adjuster's experience level (junior vs senior) and adapt response style — guided vs express intake, verbose vs concise output | **MVP** |
| Arize | Trace all agent interactions with OpenTelemetry; monitor severity classification accuracy; set up annotation queue for triage quality review | **MVP** |
| Agent Engine | Deploy agents to Vertex AI Agent Engine (or local `adk web` fallback) | **MVP** |

### What Makes This an Excellent Learning Project

1. **Real-world domain** — FNOL intake and claims triage is the most common workflow in insurance claims operations, processing millions of claims per year industry-wide
2. **Every tech topic integrated** — Not toy examples; each tech serves a genuine purpose in the FNOL pipeline (e.g., CoT for reserve reasoning isn't forced — it's how real actuaries think)
3. **Purely sequential workflow** — Intern masters strict step-by-step orchestration where each phase depends on the prior phase's output, a foundational pattern for production agent systems
4. **Conversational + task-based agents** — Orchestrator is chat-driven; sub-agents are fire-and-forget task executors, teaching both interaction paradigms
5. **4 prompt engineering techniques** — Each used where it genuinely fits: few-shot for severity classification (needs examples), CoT for reserve estimation (needs transparent reasoning), ReAct for data collection (needs tool calls)
6. **End-to-end observability** — Intern sees how production claims systems are monitored, with severity classification accuracy tracked through Arize annotation queues
7. **Deployment experience** — From local `adk web` development to cloud deployment on Vertex AI Agent Engine

---

## 2. Business Context

### 2.1 Industry Background

**Commercial Auto Fleet Insurance** covers vehicles owned or leased by businesses for commercial purposes — delivery vans, long-haul trucks, service vehicles, company cars, and specialized equipment carriers. For large enterprises (logistics companies like XPO, transport operators like Werner, delivery fleets like FedEx Ground contractors), a single fleet policy can cover 500–5,000+ vehicles with annual premiums of $2M–$50M+.

**First Notice of Loss (FNOL)** is the initial report filed when an insured incident occurs. It is the single most critical touchpoint in the claims lifecycle because:
- **Data quality at FNOL determines everything downstream** — incomplete reports cause rework, delays, and litigation risk
- **Severity classification at FNOL drives reserve setting** — under-reserving creates financial surprises; over-reserving ties up capital unnecessarily
- **Speed of handler assignment determines cycle time** — a major injury claim sitting in a general queue for 48 hours is a lawsuit waiting to happen

**Claims Triage** is the process of evaluating an FNOL to determine:
- **Severity tier** — How bad is this incident? (minor fender-bender vs. multi-vehicle fatality)
- **Reserve estimate** — How much should we set aside for this claim? ($2,000 vs. $2,000,000)
- **Handler routing** — Who should work this claim? (auto-pay bot vs. senior complex claims adjuster vs. special investigations unit)

For large commercial fleets, triage complexity is amplified by fleet-specific factors: driver DOT compliance records, vehicle maintenance history, fleet safety programs, and cargo liability exposure.

### 2.2 Current Process (Pain Points)

```
┌──────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌──────────────┐
│ Incident     │───►│ Phone Call    │───►│ Manual Form   │───►│ Supervisor    │───►│ Claim Queued │
│ Occurs       │    │ to Call       │    │ Entry         │    │ Reviews &     │    │ for Handler  │
│ (Fleet       │    │ Center        │    │ (20-30 min)   │    │ Triages       │    │ (24-72 hrs)  │
│  Vehicle)    │    │ (10-15 min    │    │               │    │ (15-30 min)   │    │              │
│              │    │  wait + call) │    │               │    │               │    │              │
└──────────────┘    └───────────────┘    └───────────────┘    └───────────────┘    └──────────────┘
```

**Pain Points:**
- **Slow** — 45–90 minutes from incident call to completed FNOL; another 24–72 hours before handler assignment
- **Incomplete data** — Adjusters forget to ask about police reports, weather conditions, passenger injuries; 30%+ of FNOLs require follow-up calls
- **Inconsistent severity** — Junior adjusters under-classify a "moderate" multi-vehicle as "minor" because no injuries were immediately reported; severity reclassification rate is 25%+
- **Reserve misses** — Initial reserve estimates swing wildly based on adjuster experience; junior adjusters set reserves 40-60% below actual on complex claims
- **Routing delays** — All claims enter a general queue; catastrophic claims sit alongside fender-benders; average time-to-right-handler is 2–3 business days
- **No external data** — Adjusters don't check weather at time of incident, driver history, or vehicle recall status during FNOL — these factors surface weeks later
- **Fleet context ignored** — A fleet's safety record, driver training compliance, and maintenance history should influence triage but are never considered at FNOL

### 2.3 Proposed Solution

```
┌──────────────┐    ┌──────────────────────────────────────────────────────┐    ┌──────────────┐
│ Adjuster     │───►│          AI Agent System (This Project)              │───►│ Complete     │
│ Reports      │    │                                                      │    │ FNOL Report  │
│ Incident     │    │  Orchestrator ─► Data Collector                      │    │ + Severity   │
│ via Chat     │    │       │          (vehicle, driver, police, weather)   │    │   Grade      │
│              │    │       │                                               │    │ + Reserve    │
│              │    │       ├────────► Severity & Triage                    │    │   Estimate   │
│              │    │       │          (classify, reserve, route)           │    │ + Handler    │
│              │    │       │                                               │    │   Assignment │
│              │    │       └────────► Report Generator                     │    │ (< 10 min)   │
│              │    │                  (formatted FNOL)                     │    │              │
└──────────────┘    └──────────────────────────────────────────────────────┘    └──────────────┘
```

**Target:** Reduce FNOL cycle from 45–90 minutes → under 10 minutes. Eliminate routing delay entirely — claim goes directly to the right handler with severity, reserves, and full FNOL report attached.

### 2.4 Target Enterprise Segment

| Attribute | Detail |
|-----------|--------|
| **Company Size** | Large commercial auto fleet insurers, fleet-specialist MGAs, self-insured fleet operators |
| **Fleet Types** | Long-haul trucking, local delivery, logistics/warehousing, passenger transport, utility/service vehicles |
| **Fleet Size** | 50–5,000+ vehicles per insured account |
| **Annual Claims Volume** | 200–10,000+ claims per fleet per year |
| **Geographic Focus** | United States (for DOT data, state-specific requirements) |
| **Lines of Business** | Commercial Auto Liability, Physical Damage, Cargo, Non-Trucking Liability |

---

## 3. Personas & User Stories

### 3.1 Personas

#### Persona 1: Junior Claims Adjuster — "Maria"

| Attribute | Detail |
|-----------|--------|
| **Name** | Maria Rodriguez |
| **Experience** | 1 year in commercial auto claims |
| **Daily Volume** | 4–6 new FNOLs per day |
| **Pain Points** | Unsure how to classify severity on multi-vehicle incidents, doesn't know what questions to ask about cargo damage, struggles with reserve estimation on anything beyond simple rear-ends, often forgets to collect police report or weather details |
| **Needs** | Step-by-step guided FNOL intake, explanation of severity classification reasoning, reserve range suggestions with rationale, checklists for required data fields |
| **Tech Comfort** | High (recent grad), comfortable with chat interfaces |
| **Typical Incidents** | Single-vehicle property damage, simple rear-ends, parking lot incidents |

#### Persona 2: Senior Claims Adjuster — "Tom"

| Attribute | Detail |
|-----------|--------|
| **Name** | Tom Henderson |
| **Experience** | 12+ years in commercial fleet claims, former field adjuster |
| **Daily Volume** | 8–12 FNOLs per day (handles complex claims for multiple fleet accounts) |
| **Pain Points** | Too much time on data entry for routine claims, wants to focus judgment on complex multi-vehicle and injury claims, hates repeating the same intake script for the 10th call today |
| **Needs** | Express/structured input mode (paste or rapid-fire key details), concise severity confirmation (not lengthy explanations), ability to override AI recommendations, speed above all |
| **Tech Comfort** | Medium, values efficiency over novelty, skeptical of AI "black boxes" |
| **Typical Incidents** | Multi-vehicle highway collisions, catastrophic cargo spills, fleet-wide weather events, fraud-suspicious claims |

### 3.2 User Stories

| ID | As a... | I want to... | So that... | Scope |
|----|---------|-------------|-----------|-------|
| US-01 | Junior Adjuster | be guided through FNOL intake step-by-step | I don't miss critical data fields like police report number or passenger injury details | **MVP** |
| US-02 | Junior Adjuster | see severity classification with reasoning | I can learn how to triage claims and explain decisions to my supervisor | **MVP** |
| US-03 | Junior Adjuster | get reserve range suggestions with rationale | I set appropriate reserves instead of guessing and getting corrected later | **MVP** |
| US-04 | Senior Adjuster | provide incident details in structured/express format | I can complete FNOL intake in under 3 minutes for routine claims | **MVP** |
| US-05 | Senior Adjuster | override severity or reserve recommendations | the system supports, not replaces, my expert judgment | Stretch |
| US-06 | Any Adjuster | have vehicle, driver, weather, and police data auto-collected | I don't spend time looking up information in separate systems | **MVP** |
| US-07 | Any Adjuster | see the claim auto-routed to the correct handler | claims reach the right person immediately instead of sitting in a general queue | **MVP** |
| US-08 | Any Adjuster | receive a formatted FNOL report I can submit to our claims system | I save 15–20 minutes of manual report writing per claim | **MVP** |
| US-09 | Any Adjuster | have the system adapt to my experience level | I get the right amount of guidance without feeling patronized or lost | **MVP** |

---

## 4. Functional Requirements

### 4.1 Core Features

#### F1: Conversational FNOL Intake
- The Claims Orchestrator Agent engages the adjuster in a structured conversation to collect all incident details
- Supports **guided mode** (step-by-step questions with explanations) for junior adjusters and **express mode** (accepts rapid structured input) for senior adjusters
- Validates inputs in real-time (e.g., valid VIN format, valid date/time, reasonable vehicle count)
- Allows corrections mid-conversation ("Actually, there were 3 vehicles involved, not 2")
- Collects all FNOL-required fields:
  - Date, time, and location of incident
  - Number of vehicles involved (insured + third-party)
  - Vehicle identification (VIN, year/make/model, unit number)
  - Driver information (name, license, CDL status)
  - Incident description (what happened, road/weather conditions)
  - Injuries (number of persons, severity indicators)
  - Police involvement (report filed? report number?)
  - Witnesses (name, contact)
  - Photos/documentation (noted, not uploaded in MVP)
- **Scope:** MVP

#### F2: Incident Data Collection
- Given incident details, the Incident Data Collector Agent fetches from mock external APIs:
  - **Vehicle data** — VIN decode (year, make, model, weight class, cargo type), fleet vehicle history, maintenance status, recall status
  - **Driver data** — CDL status, violations history, DOT compliance score, years of driving experience, prior claims
  - **Police report data** — Report status, filing officer, preliminary fault assessment, citation issued
  - **Weather data** — Conditions at incident time and location (rain, fog, ice, wind speed, visibility)
- Data sources are simulated APIs for the demo (see Section 15 for mock data schemas)
- **Scope:** MVP

#### F3: Severity Classification & Triage
- The Severity & Triage Agent evaluates **4 risk/severity categories** (MVP), expandable to 6 (Stretch):

| # | Category | Key Factors Assessed | Output | Scope |
|---|----------|---------------------|--------|-------|
| 1 | **Vehicle Damage Assessment** | Number of vehicles, damage descriptions, vehicle type/weight class, tow required, airbag deployment, total loss indicators | Damage severity score (1–10) + damage tier (cosmetic/moderate/heavy/total) | **MVP** |
| 2 | **Injury Assessment** | Number of injured persons, injury descriptions, hospitalization, EMS transport, fatality, seatbelt/helmet usage | Injury severity score (1–10) + bodily injury tier (none/minor/moderate/severe/catastrophic) | **MVP** |
| 3 | **Liability Determination** | Police report fault %, insured driver actions, traffic violations, road conditions, witness statements, comparative negligence indicators | Liability score (1–10) + liability category (clear insured-not-at-fault / shared / clear insured-at-fault / disputed) | **MVP** |
| 4 | **Coverage Verification** | Policy active status, vehicle on policy, driver authorized, coverage types (liability/collision/comprehensive/cargo), deductibles, coverage limits | Coverage confidence score (1–10) + coverage status (full/partial/excluded/pending-review) | **MVP** |
| 5 | **Subrogation Potential** | Third-party fault indicators, police citations, witness availability, damage documentation quality, recovery likelihood | Subrogation score (1–10) + recovery potential (high/medium/low/none) | Stretch |
| 6 | **Fraud Indicators** | Inconsistencies in timeline, prior claim patterns, known fraud rings, suspicious loss patterns, driver behavior anomalies | Fraud risk score (1–10) + flag level (clear/monitor/investigate) | Stretch |

- Results combined into **Overall Severity Grade**: Minor / Moderate / Major / Catastrophic
- Severity thresholds:
  - **Minor** — Property damage only, single vehicle, no injuries, clear liability, estimated loss < $10,000
  - **Moderate** — Multiple vehicles or minor injuries, moderate damage, estimated loss $10,000–$100,000
  - **Major** — Serious injuries, significant property damage, disputed liability, estimated loss $100,000–$500,000
  - **Catastrophic** — Fatalities, hazmat spill, fleet-wide event, estimated loss > $500,000
- **Scope:** MVP (4 categories) / Stretch (all 6)

#### F4: Reserve Estimation
- Based on severity classification and collected data, the system estimates initial loss reserves
- Reserve components:
  - **Vehicle damage reserve** — Based on damage assessment and vehicle value
  - **Bodily injury reserve** — Based on injury severity, number of claimants
  - **Liability/legal reserve** — Based on liability determination and litigation risk
  - **Cargo reserve** — Based on cargo type and damage (if applicable)
- Uses Chain-of-Thought reasoning to show estimation logic:
  ```
  Single vehicle rear-end, moderate bumper damage, no injuries:
  → Vehicle repair estimate: $3,500–$5,000 (based on truck class, damage description)
  → No BI reserve needed
  → Low litigation risk (clear liability, no injuries)
  → Recommended initial reserve: $5,000
  ```
- Provides reserve range (low/recommended/high) with confidence level
- **Scope:** MVP

#### F5: FNOL Report Generation
- The FNOL Report Generator Agent produces a structured report containing:
  - Incident summary (who, what, when, where)
  - Involved parties (vehicles, drivers, passengers, witnesses)
  - External data collected (vehicle details, driver history, weather, police report)
  - Severity classification with reasoning for each category
  - Reserve estimate with breakdown
  - Recommended handler and routing rationale
  - Outstanding items / follow-up actions needed
- Output format: Markdown (displayable in chat)
- **Scope:** MVP (Markdown in chat) / Stretch (also JSON export for claims system integration)

#### F6: Handler Routing
- Based on severity grade and triage results, the system recommends the appropriate claims handler:

| Severity | Handler Type | Queue | Target Response Time |
|----------|-------------|-------|---------------------|
| Minor | Auto-adjudication / desk adjuster | Fast-track queue | Same day |
| Moderate | Experienced desk adjuster | Standard queue | 24 hours |
| Major | Senior field adjuster + legal review | Priority queue | 4 hours |
| Catastrophic | Complex claims team + SIU + management | Immediate escalation | 1 hour |

- Routing logic considers: severity grade, injury presence, litigation risk, fleet account importance, regulatory requirements
- Uses Chain-of-Thought to explain routing decision
- **Scope:** MVP

### 4.2 Feature Interaction Map

```
User Chat ──► [F1: Intake] ──► [F2: Data Collection] ──► [F3: Severity & Triage] ──► [F4: Reserves] ──► [F5: Report] ──► [F6: Route]
                   │                                              │
                   ▼                                              ▼
           [User Profiling]                                (4 categories MVP,
           adapts all features                              6 if stretch)
```

---

## 5. Non-Functional Requirements

| # | Requirement | Target | Notes |
|---|------------|--------|-------|
| NFR-01 | Response time per agent call | < 10 seconds | Gemini 2.5 Flash for speed-critical data collection |
| NFR-02 | End-to-end FNOL time | < 10 minutes | From first message to complete FNOL report with routing |
| NFR-03 | Data completeness | 100% required fields | System does not proceed until all mandatory FNOL fields are captured |
| NFR-04 | Tracing coverage | 100% of LLM calls | All calls traced via OpenTelemetry → Arize |
| NFR-05 | Prompt template versioning | All prompts versioned | Via Prompt Manager module |
| NFR-06 | Graceful degradation | Fallback messaging | If external data APIs fail, continue with adjuster-provided data only |
| NFR-07 | Data persistence | JSON file-based | User profiles and claim records persist as JSON files across sessions |
| NFR-08 | Severity classification consistency | Same inputs → same severity | Deterministic triage given identical incident parameters |
| NFR-09 | Reserve estimation accuracy | Within 30% of expected range | For demo mock data; validated against pre-calculated expected reserves |

---

## 6. System Architecture

### 6.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          PRESENTATION LAYER                              │
│                                                                          │
│   ┌──────────────────────────────────────────────────────────────┐      │
│   │              ADK Web UI (built-in `adk web`)                  │      │
│   │   - Chat panel (conversational FNOL intake & results)        │      │
│   │   - Agent interaction log (visible agent delegation)         │      │
│   │   - Report display (Markdown FNOL report rendered in chat)   │      │
│   └──────────────────────────────┬───────────────────────────────┘      │
│                                  │                                      │
└──────────────────────────────────┼──────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                           AGENT LAYER (ADK)                              │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐    │
│   │            CLAIMS ORCHESTRATOR AGENT (Main Agent)               │    │
│   │   - Conversational FNOL intake                                  │    │
│   │   - Delegates tasks to sub-agents via A2A                       │    │
│   │   - Applies user profile adaptations (junior vs senior)        │    │
│   │   - Sequential workflow coordination                            │    │
│   └──────┬──────────────────┬──────────────────┬───────────────────┘    │
│          │ A2A              │ A2A              │ A2A                     │
│          ▼                  ▼                  ▼                         │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐              │
│   │ INCIDENT     │   │ SEVERITY &   │   │ FNOL REPORT  │              │
│   │ DATA         │   │ TRIAGE       │   │ GENERATOR    │              │
│   │ COLLECTOR    │   │ AGENT        │   │ AGENT        │              │
│   │ AGENT        │   │              │   │              │              │
│   │              │   │ - Vehicle    │   │ - Markdown   │              │
│   │ - Vehicle    │   │   Damage     │   │   FNOL       │              │
│   │   VIN decode │   │ - Injury     │   │ - Summary    │              │
│   │ - Driver     │   │ - Liability  │   │ - Detailed   │              │
│   │   history    │   │ - Coverage   │   │ - JSON       │              │
│   │ - Police     │   │ - Subrog.    │   │   export     │              │
│   │   report     │   │ - Fraud      │   │              │              │
│   │ - Weather    │   │              │   │              │              │
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
│  │ - Templates  │  │ - JSON       │  │ - Vehicle    │  │ - Traces   │ │
│  │ - Versions   │  │   files      │  │   (VIN)      │  │ - Evals    │ │
│  │ - format_map │  │ - Adjuster   │  │ - Driver     │  │ - Annot.   │ │
│  │              │  │   profiles   │  │   (CDL/DOT)  │  │   queues   │ │
│  │              │  │              │  │ - Police     │  │            │ │
│  │              │  │              │  │ - Weather    │  │            │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘ │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ Claims Data Store (JSON Files)                                    │   │
│  │ - fnol_records/        → completed FNOL claim JSON files          │   │
│  │ - user_profiles/       → adjuster profile JSON files              │   │
│  │ - mock_fleet_data/     → fleet, vehicle, driver mock databases    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
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

### 6.2 Tech Stack Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Agent Framework | Google ADK (Python) | Required tech stack; native Gemini integration |
| LLM (reasoning) | Gemini 2.5 Pro | Complex severity reasoning, reserve estimation, handler routing |
| LLM (speed) | Gemini 2.5 Flash | Data collection, report generation — speed-critical, lower complexity |
| Agent Communication | A2A Protocol | Required tech stack; demonstrates agent interoperability |
| UI | **ADK Web** (built-in `adk web`) | Zero build time; ADK provides chat UI out of the box. **No Streamlit.** |
| Storage | JSON files | Simplest possible for demo; no database setup. **No SQLite.** |
| Prompt Management | Custom Python module (dict-based, `str.format_map`) | Lightweight, demonstrates concept clearly. **No Jinja2.** |
| Monitoring | Arize Cloud + OpenTelemetry | Required tech stack; cloud-hosted observability |
| Deployment | Vertex AI Agent Engine / Local ADK | GCP if available, local `adk web` fallback |

---

## 7. Agent Specifications (ADK)

### 7.1 Claims Orchestrator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `claims_orchestrator` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Main conversational agent; guides adjuster through FNOL intake and coordinates the entire claims workflow |
| **Mode** | Conversational (chat-based) |

**Responsibilities:**
1. Greet the adjuster and determine their experience level (user profiling) on first interaction
2. Collect incident details through structured conversation (guided or express mode based on profile)
3. Validate inputs in real-time (VIN format, date/time reasonableness, required fields)
4. Delegate external data collection to the Incident Data Collector Agent (via A2A)
5. Delegate severity classification and triage to the Severity & Triage Agent (via A2A)
6. Delegate report generation to the FNOL Report Generator Agent (via A2A)
7. Present results to the adjuster with appropriate detail level (verbose for Maria, concise for Tom)
8. Handle follow-up questions, corrections, and re-assessments

**ADK Tools (Functions):**

```python
# Tool 1: Send task to Incident Data Collector Agent via A2A
def collect_incident_data(
    vehicle_vins: list[str],
    driver_names: list[str],
    incident_location: str,
    incident_datetime: str,
    police_report_number: str | None = None
) -> dict:
    """
    Sends a data collection task to the Incident Data Collector Agent.
    Returns structured external data including vehicle details (VIN decode),
    driver history (CDL, violations), police report status, and weather
    conditions at incident time/location.
    """
    pass

# Tool 2: Send task to Severity & Triage Agent via A2A
def classify_and_triage(
    incident_details: dict,
    external_data: dict
) -> dict:
    """
    Sends incident details and external data to the Severity & Triage Agent
    for severity classification, reserve estimation, and handler routing.
    Returns severity grade, category scores, reserve estimate, and
    recommended handler assignment.
    """
    pass

# Tool 3: Send task to FNOL Report Generator Agent via A2A
def generate_fnol_report(
    incident_details: dict,
    external_data: dict,
    triage_results: dict,
    user_profile: dict
) -> str:
    """
    Sends all collected data and triage results to the FNOL Report Generator.
    Returns a formatted FNOL report adapted to adjuster preferences (verbose
    for junior, concise for senior).
    """
    pass

# Tool 4: Get/update user profile
def manage_user_profile(user_id: str, action: str, data: dict = None) -> dict:
    """
    Retrieves or updates the claims adjuster's profile.
    Actions: 'get', 'update', 'get_or_create'
    Returns profile with experience_level, preferred_mode, claim_types_handled.
    """
    pass

# Tool 5: Get prompt from Prompt Manager
def get_prompt(template_name: str, version: str = "latest", variables: dict = None) -> str:
    """
    Retrieves a prompt template from the Prompt Manager,
    optionally rendering with provided variables via str.format_map.
    """
    pass
```

**System Instruction (Summary):**
```
You are an expert commercial auto fleet claims assistant specializing in First Notice
of Loss (FNOL) intake and triage. You help claims adjusters report fleet vehicle
incidents completely and accurately.

You adapt your communication style based on the adjuster's experience level:
- Junior adjusters (e.g., Maria): Guide step-by-step, explain terminology
  (e.g., "comparative negligence," "subrogation"), confirm each section before
  moving on, provide checklists
- Senior adjusters (e.g., Tom): Accept structured/express input, be concise,
  skip obvious explanations, confirm only key data points

Your workflow (strictly sequential):
1. Profile the adjuster (if first interaction)
2. Collect incident details through conversation
3. Trigger external data collection (vehicle, driver, police, weather)
4. Trigger severity classification, reserve estimation, and handler routing
5. Trigger FNOL report generation
6. Present results and route to handler
```

**Why Gemini Pro:** The Orchestrator must manage nuanced conversation — adapting tone for junior vs. senior adjusters, handling corrections, validating inputs, and maintaining context across a multi-turn FNOL intake. Pro's stronger instruction-following and contextual reasoning are essential here.

### 7.2 Incident Data Collector Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `incident_data_collector` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Gathers vehicle, driver, police, and weather data from external mock APIs |
| **Mode** | Non-conversational (task-based) |

**Responsibilities:**
1. Receive vehicle VINs, driver names, incident location/time from the Orchestrator
2. Call multiple mock APIs to gather external data
3. Structure and validate the collected data
4. Return comprehensive incident data package

**ADK Tools (Functions):**

```python
# Tool 1: VIN decode — vehicle characteristics
def decode_vin(vin: str) -> dict:
    """
    Simulated NHTSA VIN decode API call.
    Returns: year, make, model, body_class, weight_class, gvwr,
             engine_type, fuel_type, cargo_type_capability.
    Example: '1HGBH41JXMN109186' → {year: 2021, make: 'Freightliner',
             model: 'Cascadia', body_class: 'Truck-Tractor', ...}
    """
    pass

# Tool 2: Driver history lookup
def lookup_driver_history(driver_name: str, license_number: str = None) -> dict:
    """
    Simulated DOT/DMV driver record API call.
    Returns: cdl_status, cdl_class, endorsements, restrictions,
             violations_3yr (list), accidents_5yr (list), dot_compliance_score,
             years_experience, prior_claims (list).
    """
    pass

# Tool 3: Police report lookup
def lookup_police_report(report_number: str = None, incident_location: str = None,
                         incident_date: str = None) -> dict:
    """
    Simulated law enforcement records API call.
    Returns: report_status (filed/pending/not_filed), filing_officer,
             preliminary_fault_assessment, citations_issued (list),
             diagram_available, narrative_summary.
    """
    pass

# Tool 4: Weather conditions lookup
def lookup_weather_conditions(location: str, datetime_str: str) -> dict:
    """
    Simulated weather API call for conditions at incident time/location.
    Returns: temperature, precipitation (rain/snow/ice/none),
             visibility_miles, wind_speed_mph, road_conditions
             (dry/wet/icy/flooded), weather_advisory_active.
    """
    pass

# Tool 5: Fleet vehicle record lookup
def lookup_fleet_vehicle(vin: str, fleet_id: str = None) -> dict:
    """
    Simulated fleet management system API call.
    Returns: unit_number, fleet_company, last_maintenance_date,
             maintenance_status (current/overdue), mileage,
             active_recalls (list), telematics_equipped,
             dash_cam_available.
    """
    pass
```

**Why Gemini Flash:** Data collection is straightforward — call mock APIs based on inputs, structure the results, validate completeness. No complex reasoning required. Flash is faster and cheaper, keeping FNOL cycle time low.

### 7.3 Severity & Triage Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `severity_triage_agent` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Classifies incident severity across 4 categories (MVP), estimates reserves, recommends handler routing using Chain-of-Thought reasoning |
| **Mode** | Non-conversational (task-based) |

**Responsibilities:**
1. Receive incident details and collected external data from the Orchestrator
2. Evaluate 4 severity categories sequentially with Chain-of-Thought reasoning (MVP: Vehicle Damage, Injury, Liability, Coverage; Stretch adds Subrogation, Fraud)
3. Determine overall severity grade (Minor / Moderate / Major / Catastrophic)
4. Estimate initial loss reserves with breakdown by component
5. Recommend handler type and routing queue with justification
6. Return comprehensive triage results

**ADK Tools (Functions):**

```python
# Tool 1: Assess vehicle damage severity
def assess_vehicle_damage(
    vehicles_involved: list[dict],
    damage_descriptions: list[str],
    tow_required: bool,
    airbag_deployed: bool
) -> dict:
    """
    Evaluates vehicle damage severity using Chain-of-Thought.
    Returns: damage_score (1-10), damage_tier (cosmetic/moderate/heavy/total),
             estimated_repair_cost_range, total_loss_indicators, reasoning.
    """
    pass

# Tool 2: Assess injury severity
def assess_injury_severity(
    number_injured: int,
    injury_descriptions: list[str],
    hospitalized: bool,
    ems_transport: bool,
    fatality: bool
) -> dict:
    """
    Evaluates bodily injury severity using Chain-of-Thought.
    Returns: injury_score (1-10), injury_tier (none/minor/moderate/severe/
             catastrophic), estimated_bi_reserve_range, reasoning.
    """
    pass

# Tool 3: Determine liability
def determine_liability(
    police_report: dict,
    incident_description: str,
    driver_actions: str,
    weather_conditions: dict,
    witness_statements: list[str] | None = None
) -> dict:
    """
    Evaluates liability determination using Chain-of-Thought.
    Returns: liability_score (1-10), liability_category (insured-not-at-fault/
             shared/insured-at-fault/disputed), fault_percentage_estimate,
             litigation_risk (low/medium/high), reasoning.
    """
    pass

# Tool 4: Verify coverage
def verify_coverage(
    policy_status: str,
    vehicle_on_policy: bool,
    driver_authorized: bool,
    coverage_types: list[str],
    incident_type: str
) -> dict:
    """
    Evaluates coverage applicability using Chain-of-Thought.
    Returns: coverage_score (1-10), coverage_status (full/partial/excluded/
             pending-review), applicable_coverages, deductible_amounts,
             coverage_limit_concerns, reasoning.
    """
    pass

# Tool 5: Estimate reserves
def estimate_reserves(
    damage_assessment: dict,
    injury_assessment: dict,
    liability_assessment: dict,
    coverage_assessment: dict
) -> dict:
    """
    Calculates initial reserve estimate using Chain-of-Thought reasoning.
    Returns: vehicle_damage_reserve, bodily_injury_reserve,
             liability_legal_reserve, cargo_reserve, total_reserve,
             reserve_range (low/recommended/high), confidence_level, reasoning.
    """
    pass

# Tool 6: Determine handler routing
def determine_handler_routing(
    overall_severity: str,
    injury_present: bool,
    litigation_risk: str,
    total_reserve: float,
    special_flags: list[str] | None = None
) -> dict:
    """
    Determines appropriate claims handler using Chain-of-Thought.
    Returns: handler_type, queue_name, priority_level,
             target_response_time, routing_reasoning.
    """
    pass
```

**Why Gemini Pro:** Severity classification, reserve estimation, and handler routing are the highest-stakes decisions in this system. A misclassified severity or underestimated reserve has real financial consequences. Pro's superior Chain-of-Thought reasoning produces more accurate, transparent, and defensible triage decisions.

### 7.4 FNOL Report Generator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `fnol_report_generator` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Compiles all incident data, external data, and triage results into a formatted FNOL report |
| **Mode** | Non-conversational (task-based) |

**Responsibilities:**
1. Receive incident details, external data, triage results, and user profile from the Orchestrator
2. Generate FNOL report adapted to adjuster's experience level and format preferences
3. Produce both human-readable (Markdown) and machine-readable (JSON) outputs
4. Include all required FNOL fields for claims management system submission

**ADK Tools (Functions):**

```python
# Tool 1: Format FNOL as Markdown report
def format_fnol_markdown(
    incident_details: dict,
    external_data: dict,
    triage_results: dict,
    user_profile: dict
) -> str:
    """
    Generates a formatted Markdown FNOL report.
    Adapts detail level: verbose (junior) vs concise (senior).
    Includes: incident summary, involved parties, vehicle/driver data,
    severity classification, reserve estimate, handler routing,
    follow-up actions.
    """
    pass

# Tool 2: Format FNOL as JSON (for claims system)
def format_fnol_json(
    incident_details: dict,
    external_data: dict,
    triage_results: dict
) -> dict:
    """
    Generates a structured JSON FNOL record suitable for
    claims management system ingestion. Follows ACORD-like
    field naming conventions for insurance interoperability.
    """
    pass

# Tool 3: Generate follow-up action list
def generate_follow_up_actions(
    triage_results: dict,
    data_completeness: dict
) -> list[dict]:
    """
    Generates a prioritized list of follow-up actions needed.
    Returns list of: {action, priority (high/medium/low),
    assigned_to, due_date_relative, reason}.
    Example: 'Obtain signed police report' - high priority - 48 hours.
    """
    pass
```

**Why Gemini Flash:** Report generation is template-driven — assemble structured data into a formatted output. No complex reasoning required. Flash generates formatted text quickly, keeping total FNOL cycle time low.

---

## 8. A2A Communication Design

### 8.1 What is A2A?

Agent-to-Agent (A2A) is Google's open protocol for agents to discover each other's capabilities and exchange tasks. Key concepts:

- **Agent Card** — A JSON metadata file describing an agent's capabilities, URL, and supported skills
- **Task** — A unit of work one agent requests from another
- **Message** — Communication payloads within a task (text, data, files)
- **Artifact** — Output produced by a task (reports, data files)

### 8.1.1 Scope for 7-Day MVP

| A2A Feature | MVP (Days 1–7) | Stretch (Day 8) |
|-------------|---------------|-----------------|
| Agent Cards | ✅ Static JSON files for all 4 agents | Auto-served via A2A server `/.well-known/agent.json` |
| Agent discovery | ✅ Hardcoded URLs in Orchestrator | Dynamic discovery via registry |
| Task delegation | ✅ Use ADK's built-in sub-agent pattern (Orchestrator calls sub-agents directly) | Full A2A SDK client/server for all agents |
| A2A protocol demo | ✅ One working A2A client→server call (Orchestrator → Data Collector) to demonstrate the concept | All 3 sub-agents via A2A endpoints |
| Streaming | ❌ Not in MVP | SSE streaming for long report generation |

> **Why this approach?** ADK natively supports sub-agents — the Orchestrator can call the Data Collector, Severity & Triage, and Report Generator as sub-agents with zero A2A overhead. For the MVP, the intern uses this built-in mechanism for the actual FNOL workflow, then adds one real A2A server endpoint (on the Incident Data Collector) to demonstrate the A2A protocol. Agent Cards are created as static JSON files for all 4 agents — these are valuable artifacts that show the intern understands A2A discovery even if not all calls use the full protocol. This is practical: it works reliably and covers the learning objective.

### 8.2 Agent Cards

Each agent publishes an Agent Card at `/.well-known/agent.json`:

**Claims Orchestrator Agent Card:**
```json
{
  "name": "claims_orchestrator",
  "description": "Commercial auto fleet claims FNOL intake and triage assistant. Guides adjusters through incident reporting and coordinates triage workflow.",
  "url": "http://localhost:8001",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "fnol-intake",
      "name": "Fleet Claims FNOL Intake",
      "description": "Conversational first notice of loss intake for commercial auto fleet incidents. Collects incident details, triggers data collection, severity classification, and report generation.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Incident Data Collector Agent Card:**
```json
{
  "name": "incident_data_collector",
  "description": "Collects vehicle, driver, police, and weather data for auto fleet claims FNOL.",
  "url": "http://localhost:8002",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "incident-data-collection",
      "name": "Incident External Data Collection",
      "description": "Gathers VIN decode, driver CDL/DOT history, police report status, and weather conditions at time of incident given VINs, driver names, location, and datetime.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Severity & Triage Agent Card:**
```json
{
  "name": "severity_triage_agent",
  "description": "Classifies incident severity, estimates loss reserves, and determines handler routing for auto fleet claims.",
  "url": "http://localhost:8003",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "severity-classification",
      "name": "Claims Severity Classification & Triage",
      "description": "Evaluates vehicle damage, injury, liability, and coverage to produce severity grade, reserve estimate, and handler routing recommendation using Chain-of-Thought reasoning.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**FNOL Report Generator Agent Card:**
```json
{
  "name": "fnol_report_generator",
  "description": "Generates formatted FNOL reports from incident data, external data, and triage results.",
  "url": "http://localhost:8004",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "fnol-report-generation",
      "name": "FNOL Report Generation",
      "description": "Produces Markdown and JSON formatted FNOL reports from collected incident data, severity classification, reserve estimates, and handler routing. Adapts detail level to adjuster experience.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

### 8.3 A2A Task Flow

```
Claims Orchestrator          Data Collector         Severity & Triage      Report Generator
     │                            │                       │                      │
     │  [Adjuster provides        │                       │                      │
     │   incident details         │                       │                      │
     │   via conversation]        │                       │                      │
     │                            │                       │                      │
     │──── Create Task ──────────►│                       │                      │
     │     "Collect external      │                       │                      │
     │      data for VINs:        │                       │                      │
     │      [1FTNF..., 3C6TR...]  │                       │                      │
     │      Driver: John Smith    │                       │                      │
     │      Location: I-95 MM42   │                       │                      │
     │      Time: 2024-03-15      │                       │                      │
     │            14:30 EST"      │                       │                      │
     │                            │                       │                      │
     │                      ┌─────┤ (calls mock APIs)     │                      │
     │                      │ VIN decode                  │                      │
     │                      │ Driver history              │                      │
     │                      │ Police report               │                      │
     │                      │ Weather conditions          │                      │
     │                      │ Fleet vehicle record        │                      │
     │                      └─────┤                       │                      │
     │                            │                       │                      │
     │◄─── Task Complete ─────────│                       │                      │
     │     {vehicle_data,         │                       │                      │
     │      driver_data,          │                       │                      │
     │      police_data,          │                       │                      │
     │      weather_data}         │                       │                      │
     │                            │                       │                      │
     │──── Create Task ───────────────────────────────────►│                      │
     │     "Classify severity     │                       │                      │
     │      and triage with       │                       │                      │
     │      incident_details +    │                       │                      │
     │      external_data"        │                       │                      │
     │                            │            ┌──────────┤                      │
     │                            │            │ Vehicle  │                      │
     │                            │            │  Damage  │                      │
     │                            │            │ Injury   │                      │
     │                            │            │ Liability│                      │
     │                            │            │ Coverage │                      │
     │                            │            │ (Subrog.)│ Stretch              │
     │                            │            │ (Fraud)  │ Stretch              │
     │                            │            └──────────┤                      │
     │                            │                       │                      │
     │                            │  ┌── Reserve Estimation                     │
     │                            │  └── Handler Routing                        │
     │                            │                       │                      │
     │◄─── Task Complete ─────────────────────────────────│                      │
     │     {severity_grade,       │                       │                      │
     │      category_scores,      │                       │                      │
     │      reserve_estimate,     │                       │                      │
     │      handler_routing}      │                       │                      │
     │                            │                       │                      │
     │──── Create Task ──────────────────────────────────────────────────────────►│
     │     "Generate FNOL report  │                       │                      │
     │      with all data +       │                       │                      │
     │      triage results"       │                       │                      │
     │                            │                       │                      │
     │◄─── Task Complete ─────────────────────────────────────────────────────────│
     │     {fnol_report_markdown, │                       │                      │
     │      fnol_report_json,     │                       │                      │
     │      follow_up_actions}    │                       │                      │
     │                            │                       │                      │
     ▼ Present FNOL report +      │                       │                      │
       routing to adjuster        │                       │                      │
```

### 8.4 Implementation Guidance

**How to implement A2A in this project:**

1. **Install the A2A SDK:**
   ```bash
   pip install a2a-sdk
   ```

2. **Each agent runs as a separate A2A-compatible server** (separate port or separate Cloud Run service):
   ```
   claims_orchestrator     → http://localhost:8001
   incident_data_collector → http://localhost:8002
   severity_triage_agent   → http://localhost:8003
   fnol_report_generator   → http://localhost:8004
   ```

3. **The Orchestrator uses an A2A client** to discover and call other agents:
   ```python
   from a2a.client import A2AClient
   import json

   # Discover the incident data collector agent
   data_collector = A2AClient(url="http://localhost:8002")
   card = await data_collector.get_agent_card()

   # Send a data collection task
   task = await data_collector.send_task({
       "id": "fnol-data-001",
       "skill_id": "incident-data-collection",
       "message": {
           "role": "user",
           "parts": [{
               "text": json.dumps({
                   "vehicle_vins": ["1FTNF1CF9DKE12345", "3C6TRVDG8HE54321"],
                   "driver_names": ["John Smith", "Maria Garcia"],
                   "incident_location": "I-95 Northbound, Mile Marker 42, Jacksonville FL",
                   "incident_datetime": "2024-03-15T14:30:00-05:00",
                   "police_report_number": "JSO-2024-031542"
               })
           }]
       }
   })
   ```

4. **Each sub-agent implements a TaskHandler:**
   ```python
   from a2a.server import A2AServer, TaskHandler
   import json

   class IncidentDataCollectorHandler(TaskHandler):
       async def handle_task(self, task):
           """Collect external data for the reported incident."""
           input_data = json.loads(task.message.parts[0].text)

           # Call mock APIs for each data source
           vehicle_data = []
           for vin in input_data["vehicle_vins"]:
               vehicle_data.append(await self.decode_vin(vin))
               vehicle_data[-1].update(
                   await self.lookup_fleet_vehicle(vin)
               )

           driver_data = []
           for name in input_data["driver_names"]:
               driver_data.append(await self.lookup_driver_history(name))

           police_data = await self.lookup_police_report(
               input_data.get("police_report_number"),
               input_data["incident_location"],
               input_data["incident_datetime"]
           )

           weather_data = await self.lookup_weather_conditions(
               input_data["incident_location"],
               input_data["incident_datetime"]
           )

           return TaskResult(
               artifacts=[Artifact(parts=[TextPart(text=json.dumps({
                   "vehicle_data": vehicle_data,
                   "driver_data": driver_data,
                   "police_data": police_data,
                   "weather_data": weather_data
               }))])]
           )
   ```

5. **MVP Shortcut — Use ADK sub-agents for the actual workflow:**
   ```python
   from google.adk import Agent

   # In the Orchestrator agent definition, register sub-agents directly
   claims_orchestrator = Agent(
       name="claims_orchestrator",
       model="gemini-2.5-pro",
       instruction=ORCHESTRATOR_SYSTEM_PROMPT,
       sub_agents=[
           incident_data_collector,   # ADK sub-agent (direct call)
           severity_triage_agent,     # ADK sub-agent (direct call)
           fnol_report_generator,     # ADK sub-agent (direct call)
       ],
   )

   # The ONE real A2A endpoint (Data Collector) demonstrates the protocol
   # while the rest use the simpler, more reliable sub-agent pattern.
   ```

---

## 9. Workflow Design

### 9.1 Sequential Workflow — End-to-End FNOL

This is the primary workflow. Steps execute in **strict sequential order** because each depends on the previous step's output. Unlike Project 1 (which uses parallel risk assessment), this project demonstrates pure sequential orchestration.

```
┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ STEP 1  │───►│ STEP 2   │───►│ STEP 3   │───►│ STEP 4   │───►│ STEP 5   │───►│ STEP 6   │
│ Profile  │    │ Intake   │    │ Collect  │    │ Classify │    │ Generate │    │ Route to │
│ User     │    │ Incident │    │ External │    │ Severity │    │ FNOL     │    │ Handler  │
│          │    │ Details  │    │ Data     │    │ & Triage │    │ Report   │    │          │
│          │    │ via Chat │    │          │    │          │    │          │    │          │
│ Orch.    │    │ Orch.    │    │ Data     │    │ Severity │    │ Report   │    │ Orch.    │
│ Agent    │    │ Agent    │    │ Collector│    │ & Triage │    │ Generator│    │ Agent    │
│          │    │          │    │ Agent    │    │ Agent    │    │ Agent    │    │          │
└─────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
```

**Step Details:**

| Step | Agent | Input | Output | Prompt Type |
|------|-------|-------|--------|-------------|
| 1. Profile User | Orchestrator | Adjuster's first messages | User profile (junior/senior, preferences) | Few-shot (profiling classification examples) |
| 2. Intake Incident | Orchestrator | Multi-turn chat conversation | Structured incident details (date, location, vehicles, drivers, injuries, police report) | Zero-shot (guided questions) + ReAct (input validation tools) |
| 3. Collect External Data | Data Collector | VINs, driver names, location, datetime, police report # | External data package (vehicle decode, driver history, police report, weather) | ReAct (sequential mock API tool calls) |
| 4. Classify & Triage | Severity & Triage | Incident details + external data | Severity grade, 4 category scores, reserve estimate, handler routing | Chain-of-Thought (per category + reserve reasoning + routing logic) |
| 5. Generate Report | Report Generator | All data + triage results + user profile | Formatted FNOL report (Markdown + optional JSON) | Few-shot (report format examples) |
| 6. Route to Handler | Orchestrator | Triage results, severity grade | Handler assignment confirmation, follow-up actions | Chain-of-Thought (routing justification) |

**Why Strictly Sequential (not Parallel)?**
- Step 3 (Data Collection) **requires** Step 2's output — you can't collect vehicle data without VINs from intake
- Step 4 (Triage) **requires** Step 3's output — severity classification needs weather conditions, driver history, police fault assessment
- Step 5 (Report) **requires** Step 4's output — the FNOL report includes severity grade and reserve estimate
- Step 6 (Routing) **requires** Step 4's output — handler assignment depends on severity and reserve amount
- This is intentionally different from Project 1 (which has parallel risk assessment within a sequential outer workflow), giving interns exposure to a purely sequential pattern

### 9.2 Conversational Workflow — FNOL Intake Flow

The Claims Orchestrator's conversation follows a structured but flexible flow during Step 2:

```
START
  │
  ▼
[Greeting & Profiling] ◄──────────────────────────────────────── First session only
  "Welcome to Fleet Claims FNOL. I'm here to help you
   report a new incident."
  "Before we begin — a few quick questions to personalize
   your experience." (4 profiling questions)
  │
  ▼
[Incident Overview]
  Junior: "Let's start with the basics. When did this
           incident occur? (date and approximate time)"
  Senior: "Incident date/time, location, # vehicles?"
  │
  ▼
[Location Details]
  "Where did the incident occur?"
  (validates: street address or highway/mile marker + city/state)
  │
  ▼
[Vehicle Information] ◄─── Loop for each vehicle involved
  "Tell me about the vehicles involved."
  - VIN or Year/Make/Model
  - Unit number (fleet vehicle)
  - Damage description
  - Towed? Drivable?
  │
  ├── More vehicles? ──► [Loop back]
  │
  ▼
[Driver Information] ◄─── Loop for each driver
  "Now let's capture driver details."
  - Name and license number
  - CDL holder?
  - Insured driver or third-party?
  │
  ├── More drivers? ──► [Loop back]
  │
  ▼
[Injury Details]
  "Were there any injuries?"
  ├── No ──► [Skip to Police]
  └── Yes ──► "How many people were injured?"
              "Describe the injuries."
              "Was anyone transported to a hospital?"
              "Any fatalities?" (if severity indicators warrant)
  │
  ▼
[Police & Witnesses]
  "Was a police report filed?"
  ├── Yes ──► "Report number?" / "Issuing agency?"
  └── No  ──► "Are you planning to file one?"
  "Were there any witnesses?"
  │
  ▼
[Incident Description]
  "In your own words, what happened?"
  (free-text narrative from adjuster)
  │
  ▼
[Confirmation]
  "Here's a summary of what I've captured: [structured summary]"
  "Is everything correct, or would you like to make changes?"
  │
  ├── Correct ──────► [Proceed to Step 3: Data Collection]
  │
  └── Changes needed ──► [Navigate to specific section for correction]
```

### 9.3 Triage Decision Flow

The Severity & Triage Agent's internal logic for classifying severity and determining handler routing:

```
                         INCIDENT DETAILS + EXTERNAL DATA
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                 │
              ┌─────▼──────┐   ┌─────▼──────┐   ┌─────▼──────┐
              │ VEHICLE     │   │ INJURY     │   │ LIABILITY  │
              │ DAMAGE      │   │ ASSESSMENT │   │ DETERMIN.  │
              │ ASSESSMENT  │   │            │   │            │
              │             │   │ Score: ?   │   │ Score: ?   │
              │ Score: ?    │   │ Tier: ?    │   │ Category:? │
              │ Tier: ?     │   │            │   │            │
              └─────┬──────┘   └─────┬──────┘   └─────┬──────┘
                    │                │                 │
                    │          ┌─────▼──────┐         │
                    │          │ COVERAGE   │         │
                    │          │ VERIFICATN │         │
                    │          │            │         │
                    │          │ Score: ?   │         │
                    │          │ Status: ?  │         │
                    │          └─────┬──────┘         │
                    │                │                 │
                    └────────────────┼─────────────────┘
                                     │
                              ┌──────▼──────┐
                              │  COMBINE    │
                              │  Weighted   │
                              │  Assessment │
                              └──────┬──────┘
                                     │
                    ┌────────────────┬┼────────────────┐
                    │                ││                │
              ┌─────▼──────┐  ┌─────▼▼─────┐  ┌─────▼──────┐
              │ SEVERITY   │  │ RESERVE    │  │ HANDLER    │
              │ GRADE      │  │ ESTIMATION │  │ ROUTING    │
              │            │  │            │  │            │
              │ Minor?     │  │ Low:   $X  │  │ Auto-pay?  │
              │ Moderate?  │  │ Rec'd: $Y  │  │ Desk adj?  │
              │ Major?     │  │ High:  $Z  │  │ Field adj? │
              │ Catastroph │  │            │  │ Complex?   │
              └────────────┘  └────────────┘  └────────────┘


        SEVERITY DETERMINATION LOGIC:

        ┌─────────────────────────────────────────────────────────────────┐
        │ IF fatality OR hazmat_spill OR reserve > $500K                  │
        │    → CATASTROPHIC                                              │
        │ ELIF hospitalization OR multi-vehicle-major OR reserve > $100K  │
        │    → MAJOR                                                     │
        │ ELIF minor_injuries OR multi-vehicle OR reserve > $10K         │
        │    → MODERATE                                                  │
        │ ELSE                                                           │
        │    → MINOR                                                     │
        └─────────────────────────────────────────────────────────────────┘

        HANDLER ROUTING LOGIC:

        ┌──────────────┬──────────────────────────┬──────────────────────┐
        │ Severity     │ Handler Type             │ Queue / Response     │
        ├──────────────┼──────────────────────────┼──────────────────────┤
        │ MINOR        │ Auto-adjudicate or       │ Fast-track /         │
        │              │ desk adjuster            │ Same day             │
        ├──────────────┼──────────────────────────┼──────────────────────┤
        │ MODERATE     │ Experienced desk         │ Standard queue /     │
        │              │ adjuster                 │ 24 hours             │
        ├──────────────┼──────────────────────────┼──────────────────────┤
        │ MAJOR        │ Senior field adjuster    │ Priority queue /     │
        │              │ + legal review           │ 4 hours              │
        ├──────────────┼──────────────────────────┼──────────────────────┤
        │ CATASTROPHIC │ Complex claims team      │ Immediate            │
        │              │ + SIU + management       │ escalation / 1 hour  │
        └──────────────┴──────────────────────────┴──────────────────────┘
```

---

## 10. Prompt Engineering Strategy

### 10.1 Prompt Types Used

This project uses **4 distinct prompt engineering techniques**, each chosen for the right task in the FNOL and triage workflow:

#### Type 1: Zero-Shot Prompting
**Used for:** Simple, well-defined tasks where the model's pre-training knowledge is sufficient without examples.

**Where in this project:**
- General adjuster queries ("What does 'comparative negligence' mean?")
- Address and VIN format validation
- Incident overview collection (basic intake questions)

**Example (VIN Validation):**
```
You are a claims intake assistant for commercial auto fleet insurance.

The adjuster has provided the following Vehicle Identification Number (VIN):
{vin}

Validate this VIN:
1. Is it exactly 17 characters long?
2. Does it contain only valid characters (A-Z, 0-9, excluding I, O, Q)?
3. Does the check digit (position 9) pass the standard VIN algorithm?

If valid, confirm the VIN is properly formatted.
If invalid, explain what is wrong and ask the adjuster to re-enter.
```

**Example (General Query):**
```
You are a commercial auto fleet claims assistant. The adjuster is asking a
general question during FNOL intake.

Answer their question accurately and concisely, using insurance claims
terminology appropriate for a {experience_level} adjuster.

Adjuster question: {question}
```

#### Type 2: Few-Shot Prompting
**Used for:** Tasks where examples significantly improve consistency, classification accuracy, and output format.

**Where in this project:**
- User profiling (classifying adjuster experience level from conversation)
- Severity classification (grading incidents with consistent criteria)
- Report generation (consistent FNOL report format)

**Example (User Profiling — Adjuster Classification):**
```
Based on the adjuster's messages during onboarding, classify their experience
level as JUNIOR or SENIOR. Consider: terminology usage, comfort with
structured input, speed preference, and knowledge of claims processes.

Example 1:
Adjuster: "Hi, I'm pretty new to fleet claims. I've mostly handled personal
auto. Can you walk me through what I need to collect for a commercial fleet
incident?"
Classification: JUNIOR
Reasoning: Explicitly states being new to fleet claims, asks for guidance,
limited domain experience.

Example 2:
Adjuster: "New FNOL — 2-vehicle MVA on I-10, 2024 Peterbilt 579 rear-ended
a 2022 Sprinter van at a weigh station. No injuries, both drivable. VIN on
the Pete is 1XPWD49X1JD456789. Police on scene, report pending."
Classification: SENIOR
Reasoning: Uses industry shorthand (MVA, Pete), provides structured data
upfront, includes VIN immediately, concise reporting style.

Example 3:
Adjuster: "I've got a new claim to report. A delivery truck from one of our
fleet accounts was in an accident. I'm not sure exactly what all I need to
collect — is there a checklist?"
Classification: JUNIOR
Reasoning: Uncertain about data requirements, asks for checklist, no
technical shorthand.

Example 4:
Adjuster: "Multi-vehicle on I-95 MM 42, Jacksonville. 3 vehicles including
our insured's Cascadia. Driver CDL Class A, 2 yrs exp. One soft tissue BI
claim from the sedan driver. Police report JSO-2024-031542. Weather was rain,
reduced visibility. Need severity and routing."
Classification: SENIOR
Reasoning: Provides all key FNOL data upfront in structured format, uses
abbreviations (BI, MM, CDL), knows exactly what output they need.

Now classify:
Adjuster: "{adjuster_messages}"
Classification:
```

**Example (Severity Classification — Few-Shot):**
```
Classify the following auto fleet incident into a severity tier. Use ONLY
these tiers: MINOR, MODERATE, MAJOR, CATASTROPHIC.

Example 1:
Incident: Single fleet van backed into a bollard in a parking lot. Rear
bumper dented. No other vehicles. No injuries. Vehicle drivable.
Severity: MINOR
Reasoning: Single vehicle, property damage only, no injuries, low damage
estimate (~$1,500), no third-party involvement.

Example 2:
Incident: Fleet box truck rear-ended a sedan at a red light. Moderate
damage to both vehicles. Sedan driver reports neck pain, went to urgent
care. Fleet driver cited for following too closely.
Severity: MODERATE
Reasoning: Two vehicles, minor soft-tissue injury, fleet driver at fault,
moderate damage estimate (~$15,000 PD + $10,000 BI).

Example 3:
Incident: Fleet tractor-trailer jackknifed on I-40 in rain, struck 2
vehicles. Trailer overturned, cargo spilled. 3 people hospitalized including
fleet driver. Multiple citations. Highway closed 4 hours.
Severity: MAJOR
Reasoning: Multi-vehicle, multiple hospitalizations, cargo loss, highway
closure, significant liability exposure (~$250,000 estimated).

Example 4:
Incident: Head-on collision between fleet tanker truck and passenger bus on
a two-lane highway. Fuel spill requiring hazmat response. 2 fatalities,
12 injured. Fleet driver tested positive for fatigue (HOS violation).
Severity: CATASTROPHIC
Reasoning: Fatalities, hazmat spill, mass casualties, HOS violation = gross
negligence, regulatory investigation, estimated loss > $2M.

Now classify:
Incident: {incident_description}
Vehicles: {vehicles_involved}
Injuries: {injury_details}
Police: {police_report_summary}
Severity:
```

#### Type 3: Chain-of-Thought (CoT) Prompting
**Used for:** Complex reasoning where transparency, step-by-step logic, and defensible conclusions are critical — the core of triage decision-making.

**Where in this project:**
- All 4 severity category assessments (Vehicle Damage, Injury, Liability, Coverage)
- Reserve estimation (calculating appropriate loss reserves)
- Handler routing logic (matching severity to the right handler)

**Example (Reserve Estimation — CoT):**
```
You are an expert auto fleet claims reserve analyst. Based on the following
incident data and severity assessments, estimate the initial loss reserves.

INCIDENT DATA:
- Incident type: {incident_type}
- Vehicles involved: {vehicle_count}
- Vehicle details: {vehicle_details}
- Injuries reported: {injury_summary}
- Police report: {police_report_summary}
- Weather conditions: {weather_conditions}

SEVERITY ASSESSMENTS:
- Vehicle Damage Severity: {vehicle_damage_tier} (score: {vehicle_damage_score}/10)
- Injury Severity: {injury_tier} (score: {injury_score}/10)
- Liability Exposure: {liability_tier} (score: {liability_score}/10)
- Coverage Adequacy: {coverage_tier} (score: {coverage_score}/10)

Think step by step:

Step 1 — Vehicle Damage Costs:
Estimate the total property damage reserve. Consider:
- Number and type of vehicles (commercial fleet vehicles cost more to repair)
- Damage severity from the assessment
- Whether vehicles are drivable or total losses
- Cargo damage if applicable
- Towing and storage costs

Step 2 — Injury Costs:
Estimate bodily injury reserves. Consider:
- Number and severity of injuries
- Type of medical treatment (ER, hospital, surgery, rehab)
- Lost wages for injured parties
- Pain and suffering multiplier (1.5x-5x medical costs based on severity)

Step 3 — Liability Exposure:
Estimate third-party liability reserve. Consider:
- Fleet driver fault percentage
- Number of third-party claimants
- Jurisdiction and venue considerations
- Potential for litigation (attorney involvement)
- Regulatory fines or citations (DOT/FMCSA violations)

Step 4 — Coverage Adequacy:
Evaluate if the fleet policy limits are sufficient. Consider:
- Policy limits vs estimated total exposure
- Deductible amounts
- Sub-limits for specific coverage types (cargo, environmental)
- Excess/umbrella policy availability

Step 5 — Recommended Reserve with Breakdown:
Provide the final reserve recommendation in this exact JSON format:
{{"reserve_breakdown": {{
    "vehicle_damage": <amount>,
    "bodily_injury": <amount>,
    "liability_defense": <amount>,
    "cargo_damage": <amount>,
    "other_costs": <amount>,
    "total_reserve": <amount>
  }},
  "confidence": "<HIGH|MEDIUM|LOW>",
  "reserve_rationale": "<summary of key factors>",
  "escalation_needed": <true|false>,
  "escalation_reason": "<reason if applicable>"
}}
```

**Example (Handler Routing — CoT):**
```
You are a claims triage specialist for a large commercial auto fleet insurer.
Based on the severity assessment results, determine the appropriate claims
handler assignment.

SEVERITY ASSESSMENT:
- Overall Severity Tier: {overall_severity_tier}
- Vehicle Damage Score: {vehicle_damage_score}/10
- Injury Score: {injury_score}/10
- Liability Score: {liability_score}/10
- Coverage Score: {coverage_score}/10
- Total Estimated Reserve: ${total_reserve}
- Fleet Account Tier: {fleet_account_tier}

HANDLER ROUTING RULES:
- MINOR (reserve < $25K): Junior adjuster, standard processing
- MODERATE (reserve $25K-$100K): Senior adjuster, priority processing
- MAJOR (reserve $100K-$500K): Senior adjuster + supervisor review, high priority
- CATASTROPHIC (reserve > $500K): Complex claims unit + management notification, urgent

Think step by step:

Step 1: Determine the primary severity driver (which category scored highest?)
Step 2: Check for any automatic escalation triggers:
  - Fatality → always CATASTROPHIC routing
  - Hazmat involvement → environmental specialist required
  - DOT/FMCSA violation → regulatory specialist required
  - Attorney represented claimant → litigation unit involvement
  - Fleet account Tier 1 → account manager notification
Step 3: Match severity tier to handler level using the routing rules
Step 4: Identify any specialist resources needed (medical, environmental, legal, SIU)
Step 5: Recommend the handler assignment and priority level

Output JSON:
{{"handler_assignment": {{
    "handler_level": "<JUNIOR|SENIOR|COMPLEX_UNIT>",
    "priority": "<STANDARD|PRIORITY|HIGH|URGENT>",
    "specialists_needed": [...],
    "supervisor_review": <true|false>,
    "management_notification": <true|false>,
    "account_manager_notify": <true|false>,
    "routing_rationale": "<step-by-step reasoning>"
  }}
}}
```

#### Type 4: ReAct (Reasoning + Acting) Prompting
**Used for:** Tasks that require the agent to reason about what information it needs, take actions (tool calls) to gather that information, then reason about the results — the core of data collection.

**Where in this project:**
- Incident Data Collector agent (gathering incident details via tool calls)
- Vehicle lookup (checking VIN against fleet database)
- Weather and road condition lookup
- Police report retrieval

**Example (Incident Data Collection — ReAct):**
```
You are the Incident Data Collector agent for a commercial auto fleet claims
system. Your job is to gather all necessary incident data to support FNOL
filing and severity assessment.

You have access to these tools:
1. lookup_vehicle_by_vin(vin) — Returns vehicle details from fleet database
2. get_police_report(report_number) — Returns police report summary
3. get_weather_conditions(location, datetime) — Returns weather at time of incident
4. validate_driver_info(driver_id) — Returns driver record and CDL status

INCIDENT SO FAR:
{incident_summary}

MISSING DATA:
{missing_fields}

Think about what data you still need, then take action:

Thought: I need to verify the fleet vehicle details. The adjuster provided
VIN {vin}. Let me look it up in the fleet database.
Action: lookup_vehicle_by_vin("{vin}")
Observation: {{vehicle details returned}}

Thought: The adjuster mentioned a police report number {report_number}.
Let me retrieve it for the severity assessment.
Action: get_police_report("{report_number}")
Observation: {{police report summary returned}}

Thought: The incident occurred at {location} on {incident_datetime}.
Weather conditions may have been a contributing factor.
Action: get_weather_conditions("{location}", "{incident_datetime}")
Observation: {{weather data returned}}

Thought: I now have vehicle details, police report, and weather data.
Let me compile the complete incident data package.

Final Output: Compile all gathered data into a structured incident data
package in this JSON format:
{{"incident_data_package": {{
    "vehicle_details": {{...}},
    "police_report": {{...}},
    "weather_conditions": {{...}},
    "driver_info": {{...}},
    "data_completeness": "<COMPLETE|PARTIAL>",
    "missing_items": [...]
  }}
}}
```

### 10.2 Prompt Type Selection Matrix

This table maps every significant task in the system to its prompt type, with reasoning:

| # | Task | Agent | Prompt Type | Why |
|---|------|-------|-------------|-----|
| 1 | Welcome greeting & profiling questions | Claims Orchestrator | Zero-Shot | Simple conversational task, no examples needed |
| 2 | Adjuster experience classification | Claims Orchestrator | Few-Shot | Examples ensure consistent JUNIOR/SENIOR classification |
| 3 | Incident intake conversation | Claims Orchestrator | Zero-Shot | Open-ended data collection, model's conversation ability is sufficient |
| 4 | Workflow routing decisions | Claims Orchestrator | CoT | Needs step-by-step reasoning about which agent to invoke next |
| 5 | VIN validation | Incident Data Collector | Zero-Shot | Well-defined rule-based check |
| 6 | Vehicle lookup via fleet database | Incident Data Collector | ReAct | Requires reasoning about what to look up, then tool calls |
| 7 | Police report retrieval | Incident Data Collector | ReAct | Tool call with reasoning about report relevance |
| 8 | Weather/road condition lookup | Incident Data Collector | ReAct | Tool call with reasoning about contributing factors |
| 9 | Data completeness validation | Incident Data Collector | CoT | Step-by-step check of all required FNOL fields |
| 10 | Vehicle Damage severity scoring | Severity & Triage | CoT | Complex reasoning with transparent scoring logic |
| 11 | Injury severity scoring | Severity & Triage | CoT | Complex reasoning about medical impact and costs |
| 12 | Liability exposure scoring | Severity & Triage | CoT | Multi-factor legal reasoning needs transparency |
| 13 | Coverage adequacy scoring | Severity & Triage | CoT | Policy analysis requires step-by-step evaluation |
| 14 | Reserve estimation | Severity & Triage | CoT | Financial calculation needs defensible reasoning chain |
| 15 | Handler routing assignment | Severity & Triage | CoT | Rules-based but needs reasoning about escalation triggers |
| 16 | FNOL report generation | FNOL Report Generator | Few-Shot | Examples ensure consistent report formatting and structure |

---

## 11. Prompt Manager Design

### 11.1 What Is the Prompt Manager?

A lightweight Python module that manages all prompt templates across agents. It's deliberately simple — no heavy frameworks, no database — just enough to demonstrate the concept of centralized prompt management.

**Core capabilities:**
1. **Stores** prompt templates as Python string constants, organized by agent
2. **Versions** each template (v1, v2, etc.) so you can compare and iterate
3. **Renders** templates with variables using Python `str.format_map`
4. **Tracks** which prompt version was used per call (feeds into Arize tracing)

> **Design philosophy:** Keep it simple. A dict-based prompt manager is something an intern can build in 1-2 hours and understand completely. It demonstrates the *concept* of prompt management without getting lost in library complexity.

### 11.2 Directory Structure

```
prompt_manager/
├── __init__.py
├── manager.py                  # Core PromptManager class
└── templates/
    ├── __init__.py
    ├── orchestrator.py         # Claims Orchestrator prompts (system, profiling, intake)
    ├── data_collector.py       # Incident Data Collector prompts (ReAct collection)
    ├── severity_triage.py      # Severity & Triage prompts (CoT for each category)
    └── report_generator.py     # FNOL report generation prompts (few-shot format)
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
            agent: Agent name (e.g., "orchestrator", "severity_triage")
            name: Template name (e.g., "reserve_estimation_cot")
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
# prompt_manager/templates/severity_triage.py

# Each template is a Python string with {variable} placeholders.
# Multiple versions allow comparison and iteration.

SEVERITY_CLASSIFICATION_FEWSHOT_V1 = """
Classify the following auto fleet incident into a severity tier. Use ONLY
these tiers: MINOR, MODERATE, MAJOR, CATASTROPHIC.

Example 1:
Incident: Single fleet van backed into a bollard in a parking lot. Rear
bumper dented. No other vehicles. No injuries. Vehicle drivable.
Severity: MINOR
Reasoning: Single vehicle, property damage only, no injuries, low damage
estimate (~$1,500), no third-party involvement.

Example 2:
Incident: Fleet box truck rear-ended a sedan at a red light. Moderate
damage to both vehicles. Sedan driver reports neck pain, went to urgent
care. Fleet driver cited for following too closely.
Severity: MODERATE
Reasoning: Two vehicles, minor soft-tissue injury, fleet driver at fault,
moderate damage estimate (~$15,000 PD + $10,000 BI).

Now classify:
Incident: {incident_description}
Vehicles: {vehicles_involved}
Injuries: {injury_details}
Police: {police_report_summary}
Severity:
"""

# Version 2: More detailed examples (for A/B testing in Arize)
SEVERITY_CLASSIFICATION_FEWSHOT_V2 = """
... (refined prompt with MAJOR and CATASTROPHIC examples added) ...
"""


def register_all(manager):
    """Register all severity & triage templates."""
    manager.register("severity_triage", "severity_classification_fewshot", "v1", SEVERITY_CLASSIFICATION_FEWSHOT_V1)
    manager.register("severity_triage", "severity_classification_fewshot", "v2", SEVERITY_CLASSIFICATION_FEWSHOT_V2)
    # ... register reserve_estimation_cot, handler_routing_cot, etc.
```

### 11.3.2 Usage in Agent Code

```python
# Inside the Severity & Triage Agent:
prompt = prompt_manager.get_prompt(
    agent="severity_triage",
    name="severity_classification_fewshot",
    variables={
        "incident_description": incident_data["description"],
        "vehicles_involved": incident_data["vehicles_summary"],
        "injury_details": incident_data["injury_summary"],
        "police_report_summary": incident_data["police_report"],
    },
    version="v1"  # or "latest"
)

response = await model.generate_content(prompt)
```

### 11.4 Full CoT Template Example

The full Reserve Estimation CoT template (matching the pattern in Section 10.1) demonstrates the Chain-of-Thought pattern registered as a Prompt Manager constant:

```python
# prompt_manager/templates/severity_triage.py

RESERVE_ESTIMATION_COT_V1 = """
You are an expert auto fleet claims reserve analyst. Based on the following
incident data and severity assessments, estimate the initial loss reserves.

INCIDENT DATA:
- Incident type: {incident_type}
- Vehicles involved: {vehicle_count}
- Vehicle details: {vehicle_details}
- Injuries reported: {injury_summary}
- Police report: {police_report_summary}
- Weather conditions: {weather_conditions}

SEVERITY ASSESSMENTS:
- Vehicle Damage Severity: {vehicle_damage_tier} (score: {vehicle_damage_score}/10)
- Injury Severity: {injury_tier} (score: {injury_score}/10)
- Liability Exposure: {liability_tier} (score: {liability_score}/10)
- Coverage Adequacy: {coverage_tier} (score: {coverage_score}/10)

Think step by step:

Step 1 — Vehicle Damage Costs:
Estimate total property damage reserve considering vehicle types, damage
severity, drivability, cargo damage, towing, and storage.

Step 2 — Injury Costs:
Estimate bodily injury reserves considering number/severity of injuries,
treatment type, lost wages, and pain & suffering multiplier.

Step 3 — Liability Exposure:
Estimate third-party liability considering fault percentage, claimant count,
jurisdiction, litigation potential, and regulatory fines.

Step 4 — Coverage Adequacy:
Evaluate policy limits vs estimated exposure, deductibles, sub-limits, and
excess/umbrella availability.

Step 5 — Recommended Reserve with Breakdown:
Output JSON with reserve_breakdown, confidence, rationale, and escalation flag.
"""


def register_all(manager):
    manager.register("severity_triage", "reserve_estimation_cot", "v1", RESERVE_ESTIMATION_COT_V1)
```

> **Tip for the intern:** Start by writing the Vehicle Damage severity CoT prompt, test it in Gemini, and iterate until the output is consistent. Then copy the pattern for the other 3 categories. Don't try to write all 4 from scratch — template from the first.

---

## 12. User Profiling Design

### 12.1 What Is User Profiling?

The system builds a profile of each claims adjuster through conversation and adapts its behavior to match their expertise, preferences, and working style. A junior adjuster who's new to fleet claims gets guided help with terminology and checklists, while a senior adjuster gets rapid, data-dense responses.

### 12.2 Profile Attributes

Stored as a simple JSON file at `data/profiles/{user_id}.json`:

```json
{
  "user_id": "adjuster_001",
  "name": "Carlos Mendez",
  "experience_level": "junior",
  "specialization": ["personal_auto", "light_fleet"],
  "preferred_detail_level": "verbose",
  "preferred_report_format": "narrative",
  "session_count": 3,
  "last_interaction": "2026-04-18T10:30:00Z",
  "profiling_complete": true
}
```

> **Why JSON files instead of SQLite?** For a 7-day project, JSON files eliminate database setup time while still demonstrating persistence. Each user gets their own file. Simple to read, write, and debug.

### 12.3 Profiling Questions (Asked in First Session)

The Claims Orchestrator asks these questions **naturally in conversation** during first interaction:

| # | Question | Purpose | Maps To |
|---|----------|---------|---------|
| Q1 | "Welcome! Before we start, how long have you been handling commercial fleet claims?" | Determine experience level | `experience_level` |
| Q2 | "Do you mainly handle certain types of claims — like single-vehicle incidents, multi-vehicle collisions, cargo losses, or liability?" | Identify specialization | `specialization` |
| Q3 | "When I provide severity assessments and reserve estimates, do you prefer detailed reasoning for each factor, or a quick summary with the numbers?" | Set detail level | `preferred_detail_level` |
| Q4 | "For the FNOL report, do you prefer a narrative write-up or a structured data table format?" | Set format preference | `preferred_report_format` |

**Key Design Principle:** Questions are asked conversationally, not as a form. The agent weaves them into the natural flow:

```
Agent: "Welcome! I'm your fleet claims FNOL assistant. I'll help you capture
        incident details and triage severity. Before we start — how long have
        you been handling commercial fleet claims?"

User:  "About 6 months. I transferred from personal auto."

Agent: "Got it — welcome to fleet claims! There are some important differences
        from personal auto I'll point out as we go. Do you mainly handle
        certain types of fleet claims — like single-vehicle, multi-vehicle,
        or cargo losses?"

User:  "Mostly the smaller ones right now. Parking lot stuff, fender benders."

Agent: "Perfect — that's actually a great way to build your fleet claims
        foundation. Let's get started — do you have a new incident to report?"
```

### 12.4 Adaptive Behavior Based on Profile

| Behavior | Junior Mode | Senior Mode |
|----------|-------------|-------------|
| **Greeting** | Warm, explains the FNOL process | Brief, gets to incident details |
| **FNOL intake** | Guided questions one at a time, explains why each field matters | Accepts bulk structured input, minimal prompting |
| **Severity explanations** | Define terms (BI, PD, comparative negligence), explain scoring criteria | Show scores + key factors only |
| **Report style** | Narrative with definitions and context | Structured tables + bullet points |
| **Follow-up** | "Would you like me to explain the severity tiers?" | "Anything to override or adjust before I file?" |
| **Recommendations** | Detailed with reasoning and next-steps checklist | Brief list of action items |

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
                "preferred_report_format": "narrative",
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
| **Traces** | Every LLM call, tool invocation, agent-to-agent delegation | Debug failures, understand latency across the FNOL workflow |
| **Evaluations** | Severity classification accuracy, reserve estimate quality, report completeness | Ensure triage output quality |
| **Annotations** | Human review of severity assessments and reserve estimates | Build golden dataset, improve severity prompts |
| **Metrics** | Latency, token usage, cost per FNOL, error rates, severity distribution | Operational monitoring and cost control |

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
        model_id="auto-fleet-claims-fnol",                 # Project name in Arize
        model_version="1.0.0",
    )

    print("Arize tracing initialized successfully.")
```

#### Step 3: Instrument Agent Calls

```python
# In each agent, wrap LLM calls with OpenTelemetry spans

from opentelemetry import trace

tracer = trace.get_tracer("fleet-claims-agents")

class SeverityTriageAgent:
    async def assess_severity(self, incident_data: dict) -> dict:
        with tracer.start_as_current_span("severity_assessment") as span:
            # Add claims-specific attributes for Arize
            span.set_attribute("incident_type", incident_data["incident_type"])
            span.set_attribute("vehicle_count", incident_data["vehicle_count"])

            # Get prompt from Prompt Manager
            prompt = self.prompt_manager.get_prompt(
                agent="severity_triage",
                name="severity_classification_fewshot",
                variables=incident_data
            )

            # Track prompt version
            span.set_attribute("prompt_template", "severity_classification_fewshot")
            span.set_attribute("prompt_version", "v1")

            # Call Gemini
            response = await self.model.generate_content(prompt)

            # Track claims-specific output attributes
            span.set_attribute("severity_tier", result["overall_severity_tier"])
            span.set_attribute("reserve_amount", result["total_reserve"])
            span.set_attribute("handler_assigned", result["handler_level"])

            return result
```

#### Step 4: Set Up Annotation Queue

In the Arize dashboard:

1. **Create a new Space** → "Auto Fleet Claims FNOL"
2. **Go to Annotations** → Create new annotation queue
3. **Name:** "Severity Assessment Accuracy Review"
4. **Filter:** traces where `severity_tier` is set
5. **Annotation Schema:**
   - `tier_accurate` (boolean) — Is the AI severity tier correct?
   - `reserve_reasonable` (boolean) — Is the reserve estimate within acceptable range?
   - `reasoning_quality` (1-5) — How good is the CoT reasoning?
   - `corrected_tier` (string, optional) — What should the severity tier be?
   - `corrected_reserve` (float, optional) — What should the reserve amount be?
   - `notes` (text) — Reviewer comments

#### Step 5: Build Monitoring Dashboard

In Arize, create dashboard widgets for:

| Widget | Metric | Purpose |
|--------|--------|---------|
| Latency over time | p50, p95 agent response time | Performance monitoring |
| Token usage | Input + output tokens per FNOL | Cost tracking |
| Severity distribution | Histogram of severity tiers | Detect classification drift |
| Reserve estimate range | Distribution of total reserves | Identify outlier estimates |
| Error rate | % of failed agent calls | Reliability |
| Annotation agreement | Human vs AI severity tier match rate | Quality tracking |
| Prompt version comparison | Performance by prompt version | A/B testing prompts |

### 13.3 Arize Integration Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Claims       │     │ Incident     │     │ Severity &   │     │ FNOL Report  │
│ Orchestrator │     │ Data         │     │ Triage       │     │ Generator    │
│              │     │ Collector    │     │ Agent        │     │              │
│ [OTel Spans] │     │ [OTel Spans] │     │ [OTel Spans] │     │ [OTel Spans] │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                     │                    │                    │
       └─────────────────────┼────────────────────┼────────────────────┘
                             │                    │
                    ┌────────▼────────────────────▼┐
                    │      OpenTelemetry           │
                    │      Collector               │
                    └────────────┬─────────────────┘
                                │
                    ┌───────────▼──────────┐
                    │                      │
                    │    Arize Cloud        │
                    │                      │
                    │  ┌────────────────┐  │
                    │  │ Traces         │  │
                    │  │ Dashboard      │  │
                    │  │ Annotations    │  │
                    │  │ Evals          │  │
                    │  └────────────────┘  │
                    │                      │
                    └──────────────────────┘
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
    display_name="auto-fleet-claims-fnol",
    description="Multi-agent system for auto fleet claims FNOL and triage",
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
    query="New FNOL — 2-vehicle collision on I-95, our fleet Cascadia rear-ended a sedan. One minor injury reported. VIN 1FUJHHDR8CLBK1234."
)
print(response)
```

### 14.2 Option B: Local Deployment (Fallback)

If GCP access is not available:

```bash
# Run agents locally using ADK CLI
# Terminal 1: Incident Data Collector Agent
adk run incident_data_collector --port 8002

# Terminal 2: Severity & Triage Agent
adk run severity_triage_agent --port 8003

# Terminal 3: FNOL Report Generator Agent
adk run fnol_report_generator --port 8004

# Terminal 4: Claims Orchestrator Agent (main)
adk web claims_orchestrator --port 8001
```

The ADK web interface provides a chat UI for testing the conversational workflow.

### 14.3 Environment Setup Guide

```bash
# 1. Clone the project
git clone <repo_url>
cd auto-fleet-claims-fnol

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
adk web claims_orchestrator
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

### 15.1 Incident Data Schema

```json
{
  "incident_id": "INC-2026-00789",
  "reported_by": "adjuster_001",
  "reported_at": "2026-04-18T14:30:00Z",
  "incident_datetime": "2026-04-18T08:15:00Z",
  "incident_details": {
    "type": "multi_vehicle_collision",
    "description": "Fleet tractor-trailer rear-ended a sedan at a red light on US-90 eastbound near mile marker 12. Sedan pushed into crosswalk. Fleet driver cited for following too closely.",
    "location": {
      "street": "US-90 Eastbound MM 12",
      "city": "Houston",
      "state": "TX",
      "zip": "77001",
      "latitude": 29.7604,
      "longitude": -95.3698
    },
    "road_type": "highway",
    "speed_limit_mph": 55,
    "lane_configuration": "4-lane divided"
  },
  "vehicles_involved": [
    {
      "vehicle_number": 1,
      "role": "insured_fleet",
      "vin": "1FUJHHDR8CLBK1234",
      "year": 2023,
      "make": "Freightliner",
      "model": "Cascadia",
      "vehicle_type": "tractor_trailer",
      "fleet_unit_number": "FLT-4892",
      "damage_description": "Front bumper, hood, radiator damaged. Airbags deployed. Not drivable.",
      "drivable": false,
      "towed": true,
      "tow_destination": "Fleet maintenance yard",
      "estimated_damage": 35000,
      "cargo": {
        "type": "consumer_electronics",
        "value": 120000,
        "damage_description": "Partial cargo shift, some boxes crushed",
        "estimated_cargo_damage": 15000
      }
    },
    {
      "vehicle_number": 2,
      "role": "third_party",
      "vin": "1HGCV1F34PA012345",
      "year": 2022,
      "make": "Honda",
      "model": "Accord",
      "vehicle_type": "passenger_sedan",
      "damage_description": "Rear trunk crushed, rear axle displaced. Not drivable.",
      "drivable": false,
      "towed": true,
      "tow_destination": "City Collision Center",
      "estimated_damage": 18000
    }
  ],
  "driver_info": {
    "fleet_driver": {
      "driver_id": "DRV-2891",
      "name": "James Rodriguez",
      "cdl_class": "A",
      "cdl_expiry": "2027-11-15",
      "years_experience": 5,
      "hours_on_duty": 6.5,
      "hos_compliant": true,
      "drug_test_scheduled": true,
      "prior_incidents_3yr": 0
    },
    "third_party_driver": {
      "name": "Sarah Chen",
      "injury_reported": true,
      "attorney_represented": false
    }
  },
  "injuries": [
    {
      "person": "Sarah Chen (third-party driver)",
      "type": "soft_tissue",
      "description": "Neck pain, lower back pain. Went to ER, released same day.",
      "treatment": "emergency_room",
      "hospitalized": false,
      "severity": "minor"
    },
    {
      "person": "James Rodriguez (fleet driver)",
      "type": "contusion",
      "description": "Bruised chest from seatbelt. Declined transport. Cleared at fleet clinic.",
      "treatment": "clinic_visit",
      "hospitalized": false,
      "severity": "minor"
    }
  ],
  "police_report": {
    "report_number": "HPD-2026-041845",
    "responding_agency": "Houston PD",
    "citations_issued": [
      {
        "driver": "James Rodriguez (fleet)",
        "citation": "Following too closely",
        "code": "TX Trans. Code §545.062"
      }
    ],
    "fault_determination": "Fleet driver at fault",
    "diagram_available": true
  },
  "weather_conditions": {
    "condition": "clear",
    "temperature_f": 78,
    "visibility": "good",
    "road_surface": "dry",
    "contributing_factor": false
  },
  "fleet_info": {
    "fleet_account_id": "ACCT-FL-00234",
    "fleet_name": "Southwest Logistics Inc.",
    "fleet_size": 180,
    "account_tier": "tier_1",
    "policy_number": "CFA-2026-00234",
    "policy_limits": {
      "combined_single_limit": 1000000,
      "cargo_limit": 250000,
      "medical_payments": 10000,
      "uninsured_motorist": 1000000
    },
    "deductible": 5000,
    "fleet_safety_rating": "satisfactory"
  }
}
```

### 15.2 Severity Assessment Output Schema

```json
{
  "assessment_id": "SEV-2026-00456",
  "incident_id": "INC-2026-00789",
  "timestamp": "2026-04-18T14:35:00Z",
  "category_scores": [
    {
      "category": "vehicle_damage",
      "score": 6,
      "grade": "MODERATE",
      "weight": 0.25,
      "key_factors": [
        "Fleet tractor-trailer not drivable, airbags deployed ($35K est.)",
        "Third-party sedan rear crushed, not drivable ($18K est.)",
        "Cargo shift with $15K estimated damage"
      ],
      "mitigating_factors": [
        "Both vehicles towed to known facilities",
        "Cargo damage limited to partial shift"
      ],
      "reasoning": "Step 1: Fleet vehicle damage is significant — airbag deployment and non-drivable status indicate major front-end damage (~$35K). Step 2: Third-party sedan has structural rear damage (~$18K), likely approaching total loss threshold. Step 3: Cargo damage is moderate at ~$15K. Total PD exposure: ~$68K."
    },
    {
      "category": "injury",
      "score": 4,
      "grade": "MODERATE",
      "weight": 0.30,
      "key_factors": [
        "Third-party driver reports neck and back pain",
        "ER visit same day — soft tissue injury pattern",
        "Fleet driver minor contusion, cleared at clinic"
      ],
      "mitigating_factors": [
        "No hospitalization for either party",
        "Third-party not attorney-represented (yet)",
        "Injuries appear minor/soft tissue"
      ],
      "reasoning": "Step 1: Third-party soft tissue injury with ER visit typically resolves in 3-6 months with treatment (~$8K-$15K medical). Step 2: Pain and suffering multiplier of 2x-3x for soft tissue = $16K-$45K BI exposure. Step 3: Fleet driver injury is minimal, workers' comp applies. Total BI exposure: $25K-$60K."
    },
    {
      "category": "liability",
      "score": 7,
      "grade": "MAJOR",
      "weight": 0.25,
      "key_factors": [
        "Fleet driver cited for following too closely — clear fault",
        "Police report assigns fault to fleet driver",
        "Commercial vehicle rear-end collision = strong liability presumption"
      ],
      "mitigating_factors": [
        "Third-party not yet attorney-represented",
        "No HOS violation — driver was compliant",
        "Clean driver record — zero incidents in 3 years"
      ],
      "reasoning": "Step 1: Rear-end collision with citation creates near-certain liability for fleet. Step 2: Commercial vehicle involvement increases jury verdict potential. Step 3: No HOS violation or negligent entrustment factors. Step 4: Litigation risk is moderate — if third-party retains counsel, expect demand escalation."
    },
    {
      "category": "coverage",
      "score": 3,
      "grade": "LOW",
      "weight": 0.20,
      "key_factors": [
        "$1M CSL provides adequate coverage for this loss size",
        "$250K cargo limit covers the $15K cargo damage",
        "$10K med pay available for immediate medical costs"
      ],
      "mitigating_factors": [
        "Total estimated exposure ($150K-$200K) well within $1M limit",
        "Deductible is only $5K — manageable for Tier 1 account",
        "No umbrella/excess needed for this loss"
      ],
      "reasoning": "Step 1: Total estimated exposure ~$150K-$200K against $1M CSL — coverage is adequate at 5-6x the exposure. Step 2: Cargo sub-limit ($250K) easily covers $15K cargo loss. Step 3: Med pay ($10K) covers ER visit costs. No coverage adequacy concerns."
    }
  ],
  "overall_severity_tier": "MODERATE",
  "overall_score": 5.15,
  "reserve_breakdown": {
    "vehicle_damage": 68000,
    "bodily_injury": 45000,
    "liability_defense": 15000,
    "cargo_damage": 15000,
    "other_costs": 7000,
    "total_reserve": 150000
  },
  "handler_routing": {
    "handler_level": "SENIOR",
    "priority": "PRIORITY",
    "specialists_needed": [],
    "supervisor_review": false,
    "management_notification": false,
    "account_manager_notify": true,
    "routing_rationale": "MODERATE severity with $150K reserve requires senior adjuster. Tier 1 fleet account triggers account manager notification. No specialist resources needed — standard commercial auto collision."
  },
  "model_info": {
    "model": "gemini-2.5-pro",
    "prompt_versions": {
      "vehicle_damage_cot": "v1",
      "injury_severity_cot": "v1",
      "liability_exposure_cot": "v1",
      "coverage_adequacy_cot": "v1",
      "reserve_estimation_cot": "v1",
      "handler_routing_cot": "v1"
    },
    "total_tokens": 15800,
    "latency_seconds": 9.2
  }
}
```

### 15.3 Mock Data Strategy

Since this is a demo, all external data sources are **simulated**:

**The intern should create 5 diverse mock incidents:**

| # | Incident Type | Vehicles | Severity | Purpose |
|---|--------------|----------|----------|---------|
| 1 | Single-vehicle parking lot | 1 fleet van, bollard | MINOR | Low-complexity baseline |
| 2 | Two-vehicle intersection collision | 1 fleet truck, 1 sedan | MODERATE | Standard two-party claim with BI |
| 3 | Multi-vehicle highway pileup | 1 fleet tractor-trailer, 2 passenger vehicles | MAJOR | Complex multi-party, cargo loss |
| 4 | Theft/vandalism | 1 fleet vehicle stolen from depot | MODERATE | Non-collision claim type |
| 5 | Weather-related rollover | 1 fleet box truck, guardrail | MODERATE | Weather as contributing factor |

```python
# mock_apis/fleet_api.py

MOCK_INCIDENTS = {
    "INC-DEMO-001": {
        "type": "single_vehicle_parking",
        "description": "Fleet van backed into a bollard in warehouse parking lot.",
        "vehicles": [{"vin": "1GCGG25K671234567", "type": "cargo_van", "damage": "rear_bumper"}],
        "injuries": [],
        "severity_expected": "MINOR",
    },
    "INC-DEMO-002": {
        "type": "two_vehicle_intersection",
        "description": "Fleet box truck ran red light, T-boned sedan. Sedan driver neck pain.",
        "vehicles": [
            {"vin": "1FUJHHDR8CLBK1234", "type": "box_truck", "damage": "front_end"},
            {"vin": "1HGCV1F34PA012345", "type": "sedan", "damage": "passenger_side"},
        ],
        "injuries": [{"person": "third_party_driver", "type": "soft_tissue"}],
        "severity_expected": "MODERATE",
    },
    # ... INC-DEMO-003, 004, 005
}

MOCK_FLEET_VEHICLES = {
    "1GCGG25K671234567": {
        "year": 2024, "make": "Chevrolet", "model": "Express 2500",
        "type": "cargo_van", "fleet_unit": "FLT-1023",
        "fleet_account": "ACCT-FL-00234",
    },
    "1FUJHHDR8CLBK1234": {
        "year": 2023, "make": "Freightliner", "model": "Cascadia",
        "type": "tractor_trailer", "fleet_unit": "FLT-4892",
        "fleet_account": "ACCT-FL-00234",
    },
    # ... more VINs
}


def lookup_vehicle_by_vin(vin: str) -> dict:
    """Simulated fleet vehicle database lookup."""
    return MOCK_FLEET_VEHICLES.get(vin, {"error": "VIN not found in fleet database"})


def get_police_report(report_number: str) -> dict:
    """Simulated police report retrieval."""
    mock_reports = {
        "HPD-2026-041845": {
            "report_number": "HPD-2026-041845",
            "agency": "Houston PD",
            "fault": "Fleet driver at fault",
            "citations": ["Following too closely"],
        },
    }
    return mock_reports.get(report_number, {"error": "Report not found"})


def get_weather_conditions(location: str, datetime_str: str) -> dict:
    """Simulated weather data API."""
    return {
        "condition": "clear",
        "temperature_f": 78,
        "visibility": "good",
        "road_surface": "dry",
        "contributing_factor": False,
    }
```

---

## 16. Tech Stack & Tools

### 16.1 Complete Tech Stack

| Category | Technology | Version | Purpose |
|----------|-----------|---------|---------|
| **Language** | Python | 3.11+ | Primary development language |
| **Agent Framework** | Google ADK | Latest | Agent development and orchestration |
| **LLM** | Gemini 2.5 Flash | Latest | Data collection, report generation (task agents) |
| **LLM** | Gemini 2.5 Pro | Latest | Claims orchestration, severity reasoning (complex agents) |
| **Agent Communication** | A2A SDK | Latest | Inter-agent communication |
| **UI** | **ADK Web** (built-in) | Latest | Chat UI via `adk web` — zero build needed |
| **Prompt Templates** | Python strings (`str.format_map`) | N/A | Simple, no extra dependency |
| **Storage** | JSON files | Built-in | User profiles, mock data |
| **Monitoring** | Arize (Cloud) | Latest | Tracing, annotation, evaluation |
| **Tracing** | OpenTelemetry | 1.20+ | Distributed tracing framework |
| **Data Validation** | Pydantic | 2.0+ | Data model validation |
| **Config** | python-dotenv | 1.0+ | Environment variable management |
| **Deployment** | Vertex AI Agent Engine | Latest | Production deployment (GCP) |
| **Deployment (Alt)** | ADK CLI (`adk web`/`adk run`) | Latest | Local development and testing |

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
│ Setup  │───►│Incident│───►│Severity│───►│ Orch + │───►│ A2A +  │───►│ Arize  │───►│ Demo   │───►│Stretch │
│ + Mock │    │ Data   │    │& Triage│    │ Report │    │Profile │    │+Deploy │    │ Prep   │    │ Goals  │
│ Data   │    │Collect │    │ Agent  │    │ Agent  │    │        │    │        │    │        │    │        │
└────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘
 Foundation    1st Agent      2nd Agent     Wire it up    A2A + UX     Observe      Ship it!      Polish
```

### Day 1: Foundation + Mock Data
**Goal:** Project scaffolded, Gemini connected, mock incident data ready.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 1.1 | Set up Python project | Create venv, install `google-adk`, `google-genai`, `python-dotenv` | `pip list` shows packages |
| 1.2 | Get Gemini API key | Create key at ai.google.dev, save in `.env` | Test call returns response |
| 1.3 | Hello World agent | Create a minimal ADK agent, run with `adk web`, chat with it | Agent responds in browser |
| 1.4 | Create project structure | Set up folders: `agents/`, `prompt_manager/`, `mock_apis/`, `user_profile/`, `data/`, `telemetry/`, `a2a/` | Folders exist |
| 1.5 | Build mock data for 5 incidents | Create 5 mock incidents: single-vehicle parking lot, multi-vehicle intersection, highway pileup, theft/vandalism, weather damage | `lookup_incident("INC-001")` returns data |
| 1.6 | Build Prompt Manager | Implement the simple dict-based PromptManager class (Section 11) | `manager.get_prompt(...)` works |

**Key learning:** ADK basics, project structure, how Gemini API works.

**Suggested project structure after Day 1:**
```
auto-fleet-claims-fnol/
├── agents/
│   ├── __init__.py
│   ├── claims_orchestrator.py    # Skeleton only
│   ├── data_collector.py         # Skeleton only
│   ├── severity_triage.py        # Skeleton only
│   └── report_generator.py       # Skeleton only
├── prompt_manager/
│   ├── __init__.py
│   ├── manager.py                # PromptManager class (working)
│   └── templates/
│       ├── __init__.py
│       ├── orchestrator.py       # Prompt strings
│       ├── data_collector.py
│       ├── severity_triage.py
│       └── report_generator.py
├── mock_apis/
│   ├── __init__.py
│   ├── vehicle_api.py            # VIN lookup, fleet vehicle data
│   ├── police_report_api.py      # Mock police reports
│   ├── weather_api.py            # Weather conditions at incident
│   └── driver_api.py             # Driver history and records
├── user_profile/
│   ├── __init__.py
│   └── profile_store.py          # JSON file-based store
├── a2a/
│   └── agent_cards/              # Static agent card JSON files
├── data/
│   ├── profiles/                 # User profile JSON files
│   └── mock_incidents.json       # 5 mock incidents
├── telemetry/
│   ├── __init__.py
│   └── arize_setup.py            # Skeleton only
├── main.py
├── requirements.txt
├── .env.example
└── README.md
```

---

### Day 2: Incident Data Collector Agent (ReAct Pattern)
**Goal:** First real agent — collects incident data using tool calls.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 2.1 | Register data collection prompts | Add ReAct prompt templates to Prompt Manager | Prompts retrievable |
| 2.2 | Implement tool functions | Wire `lookup_vehicle_by_vin()`, `get_police_report()`, `get_weather_conditions()`, `get_driver_history()` as ADK tools | Tools return mock data |
| 2.3 | Build Data Collector Agent | Create ADK agent with system instruction + tools | Agent calls tools and returns structured incident data |
| 2.4 | Test with `adk web` | Run agent, provide an incident ID or VIN, verify data collection | Complete incident data package returned |

**Key learning:** ReAct prompting, ADK tool/function calling, how agents decide which tools to use.

**Prompt type practiced:** ReAct (Reasoning + Acting)

---

### Day 3: Severity & Triage Agent (Chain-of-Thought + Parallel Categories)
**Goal:** Second agent — evaluates 4 severity categories using CoT, estimates reserves, recommends handler.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 3.1 | Write CoT prompts | Create Chain-of-Thought prompt for each severity category: Vehicle Damage, Injury, Liability, Coverage | 4 prompt templates registered |
| 3.2 | Implement parallel assessment | Use `asyncio.gather()` to run 4 severity assessments simultaneously | All 4 run in parallel, results combined |
| 3.3 | Build Severity & Triage Agent | ADK agent that receives incident data and returns severity scores, reserve estimate, and handler recommendation | Agent returns 4 scores + overall tier + reserve + handler |
| 3.4 | Test with sample incidents | Feed mock incident data, verify severity scores and triage routing | Scores 1-10 with explanations, reserve and handler assigned |

**Key learning:** Chain-of-Thought prompting, parallel execution with asyncio, structured LLM output.

**Prompt type practiced:** Chain-of-Thought (CoT)

> **Tip:** Start with the Vehicle Damage prompt. Get it working well, then copy the pattern for Injury, Liability, and Coverage. Don't write all 4 from scratch — template from the first one.

---

### Day 4: Orchestrator + Report Generator (Sequential Workflow)
**Goal:** Wire everything together — Orchestrator manages the end-to-end sequential FNOL flow.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 4.1 | Build FNOL Report Generator Agent | ADK agent that takes incident data + severity scores → produces formatted FNOL report | Formatted FNOL report output |
| 4.2 | Write report generation prompts | Few-shot prompt with 1-2 example FNOL report snippets for consistent formatting | Report format is consistent |
| 4.3 | Build Claims Orchestrator Agent | Main conversational agent that: greets adjuster → profiles user → collects incident details via chat → delegates to sub-agents → presents results | End-to-end flow works |
| 4.4 | Wire sub-agent delegation | Orchestrator calls sub-agents in sequence: Profile → Intake → Data Collector → Severity & Triage → Report Generator → Route | Sequential: Intake → Data → Severity → Report → Route |
| 4.5 | Test end-to-end | Run `adk web`, chat as a claims adjuster, get a complete FNOL report and handler routing | Full demo flow works |

**Key learning:** ADK sub-agent pattern, sequential workflow design, few-shot prompting.

**Prompt types practiced:** Zero-shot (intake), Few-shot (report format), ReAct (orchestration tool calls)

> **This is the most critical day.** By end of Day 4, you should have a working end-to-end FNOL demo. Everything after this adds polish and tech coverage.

---

### Day 5: A2A Agent Cards + User Profiling
**Goal:** Add A2A protocol artifacts and user-adaptive behavior.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 5.1 | Create Agent Cards | Write agent card JSON for all 4 agents (see Section 8) | 4 JSON files in `a2a/agent_cards/` |
| 5.2 | Implement A2A demo endpoint | Add A2A server to Data Collector agent — serves agent card and accepts one task | `curl localhost:8002/.well-known/agent.json` works |
| 5.3 | A2A client call from Orchestrator | Orchestrator makes at least one call via A2A client (to Data Collector) | A2A task delegation works for data collection |
| 5.4 | Build user profiling | Add profiling questions to Orchestrator's first-turn conversation | Agent asks experience level, adapts style |
| 5.5 | Implement profile persistence | Save/load user profiles using JSON file store (Section 12) | Profile persists across sessions |
| 5.6 | Write profiling prompt (few-shot) | Few-shot prompt to classify user as junior/senior adjuster from their messages | Classification works |
| 5.7 | Implement adaptive behavior | Orchestrator adjusts verbosity based on profile: guided + verbose (junior) vs. express + concise (senior) | Two demo scenarios show different styles |

**Key learning:** A2A protocol, Agent Cards, user profiling, adaptive prompt engineering.

**Prompt type practiced:** Few-shot (user classification)

---

### Day 6: Arize Integration + Deployment
**Goal:** Add observability and deploy.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 6.1 | Set up Arize account | Sign up at arize.com, create Space, get API key | Dashboard accessible |
| 6.2 | Initialize tracing | Add `arize-otel` setup code (Section 13), instrument main agent calls | Traces appear in Arize dashboard |
| 6.3 | Add span attributes | Tag traces with: `severity_tier`, `reserve_amount`, `handler_assigned`, agent name, prompt version | Traces are filterable in Arize |
| 6.4 | Set up annotation queue | Create an annotation queue for severity classification accuracy review (Section 13) | Severity assessments can be reviewed/annotated |
| 6.5 | Deploy or finalize local | **Option A:** Deploy to Vertex AI Agent Engine if GCP access available. **Option B:** Document `adk web` local deployment as the production setup. | Agent accessible for demo |
| 6.6 | Test deployed agent | Run 2-3 test incidents, verify traces flow to Arize | Traces visible in Arize for test runs |

**Key learning:** OpenTelemetry tracing, LLM observability, annotation workflows, cloud deployment.

---

### Day 7: Integration Testing + Demo Preparation
**Goal:** Polish, test all scenarios, prepare for the demo.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 7.1 | Test Scenario 1: Junior Adjuster Maria | Run the junior adjuster guided fleet van scenario (Section 18.1) end-to-end | Guided, detailed flow works |
| 7.2 | Test Scenario 2: Senior Adjuster Tom | Run the senior adjuster multi-vehicle highway pileup scenario (Section 18.2) end-to-end | Concise, fast flow works |
| 7.3 | Test with all 5 mock incidents | Run FNOL intake on all 5 mock incidents, verify reasonable severity and routing | No crashes, severity scores make sense |
| 7.4 | Verify Arize traces | Check Arize dashboard has traces for all test runs, annotation queue works | Traces + annotations visible |
| 7.5 | Write demo script | Document the exact steps/inputs for the 2 demo scenarios | Demo script ready |
| 7.6 | Rehearse demo (dry run) | Run through both scenarios as if presenting | Smooth, no errors |
| 7.7 | Update README | Document how to set up and run the project | Someone else can run it |

---

### Day 8 (Stretch): Polish & Enhancements

If time allows, these additions make the demo even more impressive:

| # | Enhancement | Effort | Impact |
|---|------------|--------|--------|
| S1 | Add Subrogation & Fraud severity categories | 2-3 hours | Shows 6 parallel severity assessments instead of 4 |
| S2 | Full A2A for all sub-agents | 2-3 hours | All 3 sub-agents use A2A protocol |
| S3 | PDF export in FNOL Report Generator | 1-2 hours | Production-ready report output |
| S4 | Build Arize monitoring dashboard widgets | 1-2 hours | Latency/cost/severity distribution widgets |
| S5 | Prompt v2 and A/B comparison in Arize | 1-2 hours | Demonstrates prompt iteration |
| S6 | Add more mock incidents (total 8-10) | 1-2 hours | More diverse demo scenarios |

---

### Daily Checkpoint Validation

Use this checklist to verify each day's work is solid before moving on:

| Day | Checkpoint Question | Must Answer YES |
|-----|-------------------|----------------|
| 1 | Can I chat with a basic ADK agent in the browser? | ✅ |
| 2 | Does the Data Collector agent return structured incident data when given a VIN or incident ID? | ✅ |
| 3 | Does the Severity & Triage agent run 4 assessments in parallel and return scored results with reserve and handler? | ✅ |
| 4 | Can I chat with the Orchestrator and get a complete FNOL report + handler routing end-to-end? | ✅ |
| 5 | Are Agent Cards created? Does user profiling change the Orchestrator's response style? | ✅ |
| 6 | Do traces appear in Arize? Is the annotation queue set up? | ✅ |
| 7 | Can I smoothly run both demo scenarios (junior Maria + senior Tom) without errors? | ✅ |

---

## 18. Sample Interactions

### 18.1 Scenario: Junior Adjuster Maria — Guided Fleet Van Incident

```
AGENT:  Welcome to the Fleet Claims FNOL system! I'm here to help you
        file First Notice of Loss reports for your fleet incidents.

        Before we get started, I'd like to understand your background
        so I can tailor the experience. How long have you been working
        as a claims adjuster?

MARIA:  Hi! I'm fairly new — about a year.

AGENT:  Thanks, Maria! Since you're relatively new, I'll make sure to
        walk you through each step and explain things along the way.

        What types of fleet vehicles do you typically handle claims for?

MARIA:  Mostly delivery vans and small trucks. We have about 200 vehicles
        in our regional fleet.

AGENT:  Got it — delivery fleet, 200 vehicles. That helps me calibrate.

        Now, do you have an incident to report today?

MARIA:  Yes. One of our fleet vans rear-ended a sedan in a parking lot
        this morning.

AGENT:  I'm sorry to hear that. Let me guide you through the FNOL intake.
        I'll need some details about the incident.

        First, let's identify the vehicles involved.

        🚛 **Your Fleet Vehicle:**
        Do you have the VIN or unit number for the fleet van?

MARIA:  The VIN is 1FTFW1ET5DFC10235.

AGENT:  Let me pull up that vehicle...

        [Looking up VIN 1FTFW1ET5DFC10235...]

        ✅ Found it:

        | Detail | Value |
        |--------|-------|
        | Vehicle | 2023 Ford Transit 250 Cargo Van |
        | Unit # | FV-0892 |
        | Fleet | Metro Delivery Services |
        | Policy # | CAP-2026-04412 |
        | Driver Assigned | James Cooper |

        Does this look correct?

MARIA:  Yes, that's right. James was driving.

AGENT:  Great. Now let's get the details about the other vehicle — the sedan.
        Do you have any information about it?

MARIA:  It was a blue Toyota Camry, probably 2020 or 2021. The driver's
        name was Susan Park. I don't have her VIN.

AGENT:  That's fine — we'll capture what we have.

        Now I need some incident details:
        - **When** did this happen?
        - **Where** exactly in the parking lot?
        - **Was a police report filed?**
        - **Were there any injuries?**

MARIA:  It happened around 9:15 AM today at the Westfield Mall parking
        lot on level 2. James said he was backing out of a spot and
        didn't see the Camry. A police report was filed — case number
        is PD-2026-08834. No serious injuries, but Susan said her neck
        was sore.

AGENT:  Thank you — that's very helpful. Let me collect the supporting
        data now. This will take just a moment...

        [Delegating to Incident Data Collector Agent...]

        📋 Gathering police report PD-2026-08834...
        🌤️ Checking weather conditions at time of incident...
        🚗 Pulling driver history for James Cooper...

        ✅ Data collection complete! Here's the summary:

        | Data Source | Key Findings |
        |-------------|-------------|
        | Police Report | Backing collision, James cited for failure to yield |
        | Weather | Clear, 62°F, dry pavement, good visibility |
        | Driver History | James Cooper — 3 years commercial, 1 minor incident 2024 |

        Now I'm running the severity assessment. This evaluates 4 risk
        categories to determine how to classify and route this claim...

        [Delegating to Severity & Triage Agent — 4 parallel assessments...]

        ✅ Severity assessment complete!

        📊 **Severity Assessment**

        | Category | Score | Assessment |
        |----------|-------|-----------|
        | 🚗 Vehicle Damage | 3/10 | Low-speed parking lot impact, cosmetic/minor structural |
        | 🏥 Injury | 2/10 | Soft tissue complaint only, no EMS transport |
        | ⚖️ Liability | 6/10 | Our driver cited, clear fault — but low severity limits exposure |
        | 📋 Coverage | 1/10 | Standard fleet policy, no coverage gaps |

        **Overall Severity: MINOR-MODERATE**

        💰 **Reserve Estimate:**

        | Component | Amount |
        |-----------|--------|
        | Vehicle Damage (fleet van) | $3,500 |
        | Vehicle Damage (Camry) | $5,200 |
        | Medical (soft tissue) | $4,000 |
        | Legal/Admin | $1,300 |
        | **Total Reserve** | **$14,000** |

        🎯 **Handler Routing: Standard Claims Team**

        This claim is classified as MINOR-MODERATE and routed to your
        Standard Claims Team for processing. No special unit required.

MARIA:  What does the liability score of 6 mean? That seems high for
        a parking lot fender-bender.

AGENT:  Great question! Here's the detailed breakdown:

        **Liability Score: 6/10 (MODERATE-HIGH)**

        Even though the damage is minor, the *liability* score reflects
        how clearly fault is established and what that means for exposure:

        1️⃣ **Clear fault determination (raises score)**
        James was cited by police for "failure to yield while backing."
        This means our fleet driver is clearly at fault — there's no
        comparative negligence argument. In claims terms, we're looking
        at 100% liability.

        2️⃣ **Third-party injury claim (raises score)**
        Susan reported neck soreness. Even minor soft tissue complaints
        can escalate — "whiplash" claims from parking lot incidents are
        among the most common bodily injury claims in auto insurance.
        The score accounts for this escalation potential.

        3️⃣ **Low speed mitigates (lowers score)**
        Parking lot speeds (typically <10 mph) limit the severity of
        both vehicle damage and injury. This keeps the score from going
        higher.

        4️⃣ **Single third party (lowers score)**
        Only one other vehicle and one claimant. Multi-vehicle or
        multi-claimant incidents score much higher.

        **Bottom line:** The 6/10 isn't about the *damage* — it's about
        the *legal exposure*. Clear fault + injury complaint = moderate
        liability risk, even for a minor collision.

MARIA:  That makes sense, thanks for explaining. Can you generate the
        FNOL report now?

AGENT:  Of course!

        [Delegating to FNOL Report Generator Agent...]

        ✅ FNOL report generated!

        ---
        # FIRST NOTICE OF LOSS REPORT
        ## Incident: FV-0892 — Parking Lot Backing Collision

        **FNOL #:** FNOL-2026-04-18-0892
        **Date Filed:** April 18, 2026
        **Filed By:** Maria (Claims Adjuster)
        **Incident Date:** April 18, 2026 — 9:15 AM

        **Fleet Vehicle:** 2023 Ford Transit 250 (VIN: 1FTFW1ET5DFC10235)
        **Driver:** James Cooper
        **Third Party:** Susan Park — 2020/2021 Toyota Camry

        **Severity:** MINOR-MODERATE
        **Total Reserve:** $14,000
        **Handler:** Standard Claims Team

        ... [Full formatted FNOL report continues] ...
        ---

        The report is ready for your claims management system.
        Is there anything you'd like to adjust, or do you have another
        incident to report?
```

### 18.2 Scenario: Senior Adjuster Tom — Multi-Vehicle Highway Pileup

```
AGENT:  Welcome back, Tom! I see you've processed 847 fleet claims
        with us. What do you have today?

TOM:    Multi-vehicle on I-94 westbound near mile marker 187. Our
        Kenworth T680 VIN 1XKYD49X1MJ442891 hit 2 civilian vehicles
        in heavy rain. 3 injuries reported — one transported by EMS.
        Our driver cited for following too closely. Need FNOL.

AGENT:  Copy. Running full pipeline on the I-94 incident...

        [Data Collection — VIN lookup, police report, weather, driver history...]
        [Severity & Triage — 4 parallel assessments...]

        ⚠️ **ALERT — MAJOR SEVERITY**

        📊 **Severity Assessment**

        | Category | Score | Flag |
        |----------|-------|------|
        | 🚗 Vehicle Damage | 7/10 | ⚠️ Class 8 truck + 2 civilian vehicles, highway speed |
        | 🏥 Injury | 7/10 | ⚠️ 3 injuries, 1 EMS transport — potential serious BI |
        | ⚖️ Liability | 8/10 | ⚠️ Driver cited, multi-party, rain conditions |
        | 📋 Coverage | 3/10 | Fleet policy active, commercial auto limits apply |

        **Overall Severity: MAJOR**

        💰 **Reserve Estimate:**

        | Component | Amount |
        |-----------|--------|
        | Vehicle Damage (Kenworth) | $85,000 |
        | Vehicle Damage (2 civilian) | $55,000 |
        | Medical (3 claimants) | $150,000 |
        | Legal/Defense | $40,000 |
        | General Damages | $20,000 |
        | **Total Reserve** | **$350,000** |

        🎯 **Handler Routing: Complex Claims — Litigation Unit**

        Multi-party injury with EMS transport triggers Litigation Unit
        assignment. Subrogation potential flagged for rain/road conditions.

TOM:    Override reserve to $400K. Add subrogation flag — road
        maintenance may be a contributing factor. Also flag for
        potential DOT hours-of-service review on the driver.

AGENT:  Acknowledged. Updating:

        ✏️ **Adjustments Applied:**
        - Reserve overridden: $350,000 → **$400,000** (adjuster override)
        - Subrogation flag: **ACTIVE** — road maintenance / contributing factors
        - DOT flag: **Hours-of-service review requested** for driver

        These overrides will be noted in the FNOL report with your
        adjuster authority.

        [Generating FNOL Report...]

        ✅ FNOL report generated.

        ---
        # FIRST NOTICE OF LOSS REPORT
        ## Incident: KW-T680 — Multi-Vehicle Highway Collision (I-94)

        **FNOL #:** FNOL-2026-04-18-1893
        **Date Filed:** April 18, 2026
        **Filed By:** Tom (Senior Claims Adjuster)
        **Incident Date:** April 18, 2026

        **Fleet Vehicle:** 2019 Kenworth T680 (VIN: 1XKYD49X1MJ442891)
        **Third Parties:** 2 civilian vehicles, 3 injury claimants
        **Police Report:** Driver cited — following too closely

        **Severity:** MAJOR
        **Total Reserve:** $400,000 (adjuster override from $350K)
        **Handler:** Complex Claims — Litigation Unit
        **Flags:** Subrogation (road maintenance), DOT hours-of-service

        ... [Full formatted FNOL report continues] ...
        ---

        Report filed. Litigation Unit notified. Anything else?

TOM:    No, that's it. Good job on the severity — scores look right.

AGENT:  Thanks, Tom. FNOL-2026-04-18-1893 is closed and routed.
```

---

## 19. Acceptance Criteria

### 19.1 MVP — Must Demo (Days 1-7)

These are the **non-negotiable** deliverables for the demo:

| # | Criterion | Verification | Tech Topic Covered |
|---|-----------|-------------|-------------------|
| AC-01 | Conversational FNOL intake works for at least 2 incident types | Demo with parking lot collision + highway pileup | ADK |
| AC-02 | Data Collector Agent returns structured incident data via tool calls | JSON output from `lookup_vehicle_by_vin`, `get_police_report`, `get_weather_conditions`, `get_driver_history` | ADK + ReAct prompting |
| AC-03 | Severity assessment runs **4 categories in parallel** (Vehicle Damage, Injury, Liability, Coverage) | Visible parallel execution, all 4 scores returned | ADK + CoT prompting + Parallel |
| AC-04 | Severity scores include Chain-of-Thought reasoning | Each score has step-by-step explanation | Prompt Engineering (CoT) |
| AC-05 | FNOL Report Generator produces formatted report | Readable FNOL report with all sections | ADK + Few-shot prompting |
| AC-06 | User profiling detects junior vs senior adjuster | Different output styles for Maria (guided) vs Tom (express) | User Profiling |
| AC-07 | Agent Cards (JSON) created for all 4 agents | Files exist with valid schema | A2A |
| AC-08 | At least 1 A2A call works (Orchestrator → Data Collector) | A2A task delegation demonstrated | A2A |
| AC-09 | Prompt Manager serves versioned templates | Templates loaded via manager, version tracked | Prompt Manager |
| AC-10 | Arize traces visible in dashboard | All LLM calls traced | Arize |
| AC-11 | Arize annotation queue for severity classification review set up | Severity assessments can be reviewed/annotated for accuracy | Arize |
| AC-12 | Running locally via `adk web` **OR** deployed to Agent Engine | Accessible demo | Agent Engine / Deployment |

**Count: 6/6 tech topics covered in MVP** ✅

### 19.2 Stretch — Nice to Show (Day 8)

| # | Criterion | Tech Topic |
|---|-----------|-----------|
| AC-13 | 6 parallel severity categories (add Subrogation, Fraud) | ADK + Parallel |
| AC-14 | All 3 sub-agents callable via full A2A protocol | A2A |
| AC-15 | PDF export in FNOL Report Generator | ADK |
| AC-16 | Arize dashboard with latency and cost widgets | Arize |
| AC-17 | Prompt v1 vs v2 comparison visible in Arize | Prompt Manager + Arize |
| AC-18 | 8+ mock incidents for diverse scenarios | Mock Data |

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

### 20.6 Auto Fleet Claims Domain
- NAIC Commercial Auto Insurance definitions and classifications
- ISO commercial auto filing classifications and rating procedures
- FNOL best practices for commercial fleet operations (industry general knowledge)

---

*Document Version: 1.0*
*Created: April 18, 2026*
*Author: AI Planning Assistant*
*Status: Ready for Review*
*Scope: 7-day MVP + 1-day Stretch*
