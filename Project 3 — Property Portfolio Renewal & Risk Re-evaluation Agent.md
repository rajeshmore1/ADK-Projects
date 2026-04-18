# Project 3: Property Portfolio Renewal & Risk Re-evaluation Agent

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
A batch-processing multi-agent AI system that reviews an entire property portfolio at renewal time. It simultaneously re-evaluates risk for multiple properties in parallel, detects changes between current-year and prior-year data (new hazards, code violations, occupancy shifts), generates per-property renewal recommendations (renew as-is / adjust / refer / non-renew), and produces an aggregated portfolio summary. After batch processing completes, a conversational review interface lets the underwriter query results, drill into specific properties, and override recommendations.

### Why
Property portfolio renewal is one of the most time-intensive cycles in mid-market insurance. Underwriting managers must review 10–50 properties per portfolio at annual renewal, comparing each property's current risk profile against last year's data to detect material changes. Today, this is a manual, spreadsheet-driven process that takes days of analyst time per portfolio. Common failures:
- **Changed risks missed** — A property that added a restaurant tenant (higher fire risk) goes unnoticed until a loss occurs
- **Inconsistent re-evaluation** — Junior analysts apply different criteria than senior managers, producing uneven renewal decisions
- **No portfolio-level view** — Each property reviewed in isolation; aggregated risk trends are invisible
- **Slow turnaround** — Manual review of a 20-property portfolio takes 3–5 business days, delaying broker responses and risking non-renewal deadlines

An AI agent system can:
- Process an entire portfolio of 10–50 properties in minutes instead of days
- Detect material changes between current and prior year automatically
- Apply consistent risk re-evaluation criteria across every property
- Generate portfolio-level analytics (aggregate risk shift, exception count, renewal mix)
- Provide a conversational interface for the underwriter to query and override results

### How
Four specialized agents communicate via Google's A2A protocol:
1. **Portfolio Orchestrator Agent** — Accepts a portfolio JSON, fans out parallel property reviews, aggregates results, then enters conversational review mode for the underwriter
2. **Property Re-evaluation Agent** — Re-evaluates risk for a single property by comparing current vs. prior-year data and scoring 4 risk categories
3. **Change Detection Agent** — Compares current-year vs. prior-year property data field-by-field, identifies material changes that affect risk
4. **Renewal Recommendation Agent** — Generates a renewal decision (renew as-is / renew with adjustments / refer to senior / non-renew) based on change severity and risk re-evaluation

Built with Google ADK + Gemini 2.5, monitored via Arize Cloud and OpenTelemetry, deployed to Vertex AI Agent Engine.

### Timeline & Scope

| Attribute | Detail |
|-----------|--------|
| **Duration** | 7–8 working days |
| **Intern Level** | CS background, medium Python expertise |
| **Delivery** | Working demo with 3 portfolios (Midwest Office, Southeast Retail, Northeast Mixed) and 2 personas (junior analyst Aisha + senior manager Robert) |

The project is organized into **MVP (Must-ship in 7 days)** and **Stretch (Day 8 if time permits)** scopes. The MVP alone covers all 6 tech topics and produces an impressive demo. Stretch goals add polish and advanced risk categories.

### Key Tech Stack Coverage

| Tech Topic | How It's Used in This Project | Scope |
|------------|------------------------------|-------|
| Prompt Engineering | Zero-shot (review chat queries, data validation), Few-shot (renewal classification with 4 decision examples, user profiling), CoT (risk re-evaluation comparing current vs. prior year with step-by-step delta analysis), ReAct (data collection for updated property info, tool calls during review chat) | **MVP** |
| Prompt Manager | Version and serve all prompt templates across 4 agents using a lightweight dict-based Python module with `str.format_map` | **MVP** |
| ADK | All 4 agents built with Google Agent Development Kit; batch pipeline orchestration + conversational review UI via `adk web` | **MVP** |
| A2A | Agents publish Agent Cards; Orchestrator delegates tasks to sub-agents via A2A protocol; 1 real A2A endpoint in MVP | **MVP** |
| User Profiling | Profile the underwriter reviewing the portfolio (junior analyst vs. senior manager) and adapt conversational review output — guided walkthrough vs. exception-only reporting | **MVP** |
| Arize | Trace all agent interactions with OpenTelemetry; monitor renewal classification accuracy and change detection completeness; set up annotation queue for re-evaluation quality review | **MVP** |
| Agent Engine | Deploy agents to Vertex AI Agent Engine (or local `adk web` fallback) | **MVP** |

### What Makes This an Excellent Learning Project

1. **Parallel batch processing** — The intern learns to fan out N concurrent agent tasks using `asyncio.gather`, a fundamentally different pattern from the sequential workflows in Projects 1 and 2 — this is how real enterprise AI batch systems work
2. **Dual interaction mode** — The system combines non-conversational batch pipeline (fire-and-forget processing of the entire portfolio) with a conversational review chat (underwriter queries results), teaching both paradigms in a single project
3. **Change detection over time** — Unlike point-in-time assessments (Projects 1 and 2), this project compares current vs. prior-year data, introducing temporal reasoning — detecting what changed, why it matters, and how it shifts risk
4. **Portfolio-level analytics** — The intern builds aggregated views (total risk shift, exception count, renewal decision mix, properties requiring attention) on top of individual property results, learning to think at both granular and aggregate levels
5. **Mid-market focus** — Smaller portfolios (10–50 properties) provide a realistic scope for demo data while keeping batch sizes manageable for an 8-day project
6. **Every tech topic serves a genuine purpose** — Parallel fan-out needs ADK sub-agents + A2A, change detection needs CoT reasoning, renewal classification needs few-shot examples, review chat needs user profiling — none of it is forced
7. **Renewal decision framework** — The 4-category renewal classification (renew as-is / adjust / refer / non-renew) mirrors real underwriting workflows and produces a decision the intern can explain to stakeholders

---

## 2. Business Context

### 2.1 Industry Background

**Property Portfolio Renewal** is the annual cycle in which an insurance carrier reviews all properties covered under an existing policy (or book of business) to decide whether to renew, modify, or non-renew coverage. For mid-market commercial insurers, a single account may contain a portfolio of 10–50 properties — a regional office chain, a retail strip mall group, or a mixed-use real estate portfolio.

**Mid-market segment** refers to commercial property accounts with total insured values (TIV) between $5M and $100M and portfolios of 10–50 locations. These accounts are large enough to require portfolio-level analysis but small enough that carriers often cannot justify dedicating a full underwriting team to each renewal. Instead, a single portfolio analyst or underwriting manager handles the review — often juggling 5–10 portfolio renewals simultaneously during peak renewal season (Q4/Q1).

**Renewal underwriting** differs fundamentally from new business underwriting (Project 1):
- **Comparative analysis** — The underwriter already has last year's data. The question is not "What is the risk?" but "How has the risk changed?"
- **Decision framework** — Instead of accept/decline, the decision set is: renew as-is, renew with adjustments (premium increase, coverage restrictions, sublimits), refer to senior underwriter, or non-renew
- **Retention pressure** — Carriers want to retain profitable accounts. Non-renewal is a last resort because acquiring a new account costs 5–10x more than retaining an existing one
- **Batch context** — All properties in the portfolio are reviewed together to produce a single portfolio-level renewal recommendation to the broker

Key renewal triggers that this system detects:
- **Occupancy changes** — A vacant building is now occupied, or a tenant changed from office to restaurant
- **Natural catastrophe zone reclassification** — Updated flood or windstorm maps move a property into a higher-risk zone
- **Building code violations** — New inspection findings since last year
- **Fire protection changes** — Sprinkler system added or removed, alarm certification lapsed
- **Loss history deterioration** — Multiple claims in the past year vs. prior years of clean history
- **Construction or renovation** — Building age-related deterioration or recent improvements

### 2.2 Current Process (Pain Points)

```
┌──────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌──────────────┐
│ Renewal      │───►│ Pull Prior    │───►│ Gather        │───►│ Compare Each  │───►│ Draft Renewal│
│ Trigger      │    │ Year Data     │    │ Current Year  │    │ Property      │    │ Terms &      │
│ (60-90 days  │    │ Per Property  │    │ Updates       │    │ One by One    │    │ Compile      │
│  before      │    │ (Manual, 1-2  │    │ (Emails,      │    │ in            │    │ Portfolio    │
│  expiry)     │    │  hrs/property)│    │  inspections, │    │ Spreadsheets  │    │ Report       │
│              │    │               │    │  1-3 hrs ea.) │    │ (2-4 hrs ea.) │    │ (4-8 hours)  │
└──────────────┘    └───────────────┘    └───────────────┘    └───────────────┘    └──────────────┘
                                                                                          │
                                                                                          ▼
                                                                                   3-5 business days
                                                                                   per 20-property
                                                                                   portfolio
```

**Pain Points:**
- **Slow** — A 20-property portfolio takes 3–5 business days to review manually, with each property requiring 4–8 hours of analyst time for data pull, comparison, and recommendation drafting
- **Sequential bottleneck** — Properties are reviewed one at a time because a single analyst cannot hold multiple complex comparisons in working memory simultaneously
- **Changes missed** — Material changes between years (new hazard zone, occupancy shift, code violation) are buried in spreadsheets and frequently overlooked — 15–20% of renewal portfolios have at least one missed change that affects pricing
- **Inconsistent criteria** — Junior analysts focus on different risk factors than senior managers, producing uneven renewal decisions across the portfolio
- **No portfolio view** — Each property is reviewed in isolation. The analyst has no aggregated view of how the portfolio's overall risk profile shifted until they manually compile everything at the end
- **Renewal deadline pressure** — Brokers expect renewal terms 30–60 days before expiry. Delayed portfolio reviews mean rushed decisions, lost accounts, or inadequate pricing
- **Repetitive reporting** — 60% of mid-market portfolio properties are stable low-risk renewals that still require the same manual review effort as high-change properties

### 2.3 Proposed Solution

```
┌──────────────┐    ┌──────────────────────────────────────────────────────────┐    ┌──────────────┐
│ Underwriter  │───►│          AI Agent System (This Project)                  │───►│ Portfolio    │
│ Submits      │    │                                                          │    │ Renewal      │
│ Portfolio    │    │  PHASE 1: Batch Processing (Parallel)                    │    │ Report       │
│ JSON         │    │                                                          │    │ + Per-Prop   │
│ (10-50       │    │  Orchestrator fans out N parallel pipelines:             │    │   Decisions  │
│  properties) │    │    Property 1 ─► Change Detect → Re-eval → Recommend    │    │ + Portfolio  │
│              │    │    Property 2 ─► Change Detect → Re-eval → Recommend    │    │   Summary    │
│              │    │    Property 3 ─► Change Detect → Re-eval → Recommend    │    │ (Minutes)    │
│              │    │    ...                                                    │    │              │
│              │    │    Property N ─► Change Detect → Re-eval → Recommend    │    │              │
│              │    │                                                          │    │              │
│              │    │  PHASE 2: Conversational Review                          │    │              │
│              │    │    Underwriter queries results via chat                   │    │              │
│              │    │    Drill into specific properties                         │    │              │
│              │    │    Override recommendations                               │    │              │
│              │    │    Export final portfolio report                          │    │              │
│              │    │                                                          │    │              │
└──────────────┘    └──────────────────────────────────────────────────────────┘    └──────────────┘
```

**Target:** Reduce portfolio renewal review from 3–5 business days → under 30 minutes for batch processing + 15–30 minutes of conversational review. A 20-property portfolio that currently takes 40+ analyst-hours is completed in under 1 hour.

### 2.4 Target Enterprise Segment

| Attribute | Detail |
|-----------|--------|
| **Company Size** | Mid-market property insurers, regional carriers, managing general agents (MGAs) |
| **Portfolio Scope** | 10–50 commercial properties per portfolio (mid-market accounts) |
| **Property Types** | Office buildings, retail centers, mixed-use properties, light industrial |
| **Total Insured Value (TIV)** | $5M–$100M per portfolio |
| **Geographic Focus** | United States (for hazard zone data, building code frameworks) |
| **Lines of Business** | Commercial Property, may include Business Interruption and Inland Marine |
| **Peak Renewal Volume** | Q4/Q1, with 5–10 portfolio renewals per analyst per month |

---

## 3. Personas & User Stories

### 3.1 Personas

#### Persona 1: Junior Portfolio Analyst — "Aisha"

| Attribute | Detail |
|-----------|--------|
| **Name** | Aisha Patel |
| **Experience** | 1.5 years in commercial property underwriting, recently promoted from underwriting assistant to portfolio analyst |
| **Daily Volume** | Handles 2–3 smaller mid-market portfolios simultaneously (10–20 properties each) |
| **Pain Points** | Not confident identifying which property changes are material vs. cosmetic, struggles to interpret risk trend changes over time, takes too long comparing current vs. prior year data in spreadsheets, unsure when a property should be referred to senior underwriter vs. handled independently |
| **Needs** | Detailed explanations of what changed and why it matters, guided portfolio walkthrough (property by property), term explanations for risk concepts (e.g., "protection class," "loss development factor"), clear thresholds for when to escalate to senior |
| **Tech Comfort** | High (recent grad), comfortable with chat interfaces and JSON data formats |
| **Typical Portfolios** | Midwest office portfolios for regional corporate tenants, low-to-moderate risk, 10–20 properties each |

#### Persona 2: Senior Underwriting Manager — "Robert"

| Attribute | Detail |
|-----------|--------|
| **Name** | Robert Chen |
| **Experience** | 15+ years in commercial property underwriting, manages portfolio-level decisions for the mid-market segment, supervises 3 junior analysts including Aisha |
| **Daily Volume** | Oversees 8–12 portfolio renewals per month, personally reviews complex or high-change portfolios (5–8 properties per portfolio), signs off on all non-renew and refer decisions |
| **Pain Points** | Too much time reviewing stable low-change properties that don't need his attention, wants to see only the exceptions and material changes, hates scrolling through 20-property reports when only 3 properties have issues, needs portfolio-level risk trend data for management reporting |
| **Needs** | Exception-only reporting (show only properties with material changes), summary dashboards with portfolio-level aggregates, quick drill-down into flagged properties, ability to override AI recommendations, export-ready portfolio report for broker submission |
| **Tech Comfort** | Medium, values speed and efficiency over novelty, wants results not explanations |
| **Typical Portfolios** | Large or complex mid-market portfolios (30–50 properties), mixed-use and multi-state accounts, portfolios flagged by junior analysts for senior review |

### 3.2 User Stories

| ID | As a... | I want to... | So that... | Scope |
|----|---------|-------------|-----------|-------|
| US-01 | Junior Analyst | submit a portfolio JSON and have all properties reviewed automatically in parallel | I don't spend 3–5 days doing manual property-by-property comparison | **MVP** |
| US-02 | Junior Analyst | see a detailed explanation of what changed for each property since last year | I can understand which changes are material and why they affect risk | **MVP** |
| US-03 | Junior Analyst | ask questions about specific properties in a conversational review chat | I can drill into confusing results without leaving the system | **MVP** |
| US-04 | Senior Manager | see an exception-only portfolio summary showing only flagged properties | I can focus my time on the 3–5 properties that actually need attention instead of reviewing all 20+ | **MVP** |
| US-05 | Senior Manager | override AI renewal recommendations with my own judgment | the system supports, not replaces, my expertise — especially for refer/non-renew decisions | Stretch |
| US-06 | Any Underwriter | get a renewal recommendation for each property (renew as-is, adjust, refer, non-renew) | I have a clear starting point for my renewal decision, not just raw data | **MVP** |
| US-07 | Any Underwriter | see portfolio-level analytics (total risk shift, exception count, renewal decision mix) | I can report portfolio health to management and brokers | **MVP** |
| US-08 | Any Underwriter | have the system adapt to my experience level during the review chat | junior users get guided walkthroughs and senior users get concise exception summaries | **MVP** |
| US-09 | Any Underwriter | export the final portfolio renewal report as a formatted document | I can share it with brokers and attach it to the renewal submission file | Stretch |

---

## 4. Functional Requirements

### 4.1 Core Features

#### F1: Portfolio Batch Intake
- The Portfolio Orchestrator Agent accepts a portfolio JSON file containing 10–50 properties, each with current-year and prior-year data
- Validates the input: checks required fields per property, confirms current and prior year data are present, validates data types and ranges
- Queues all properties for parallel processing
- Provides a batch status indicator (e.g., "Processing 20 properties... 15/20 complete")
- **Scope:** MVP

#### F2: Change Detection (Current vs. Prior Year)
- The Change Detection Agent compares current-year vs. prior-year property data for each property
- Detects changes across all tracked fields:
  - **Occupancy** — Tenant type, vacancy status, business operations
  - **Building condition** — Roof age, HVAC status, renovation, code violations
  - **Natural hazard exposure** — Updated flood/wind/earthquake zone classifications
  - **Fire protection** — Sprinkler status, alarm certifications, fire protection class
  - **TIV changes** — Significant increases or decreases in total insured value
- Classifies each detected change by severity:
  - **Material** — Directly affects risk score or renewal terms (e.g., new flood zone, sprinkler removal)
  - **Notable** — Worth reviewing but may not change terms (e.g., minor TIV increase, tenant name change)
  - **Cosmetic** — No risk impact (e.g., address format update, contact info change)
- Returns a structured change report for each property
- **Scope:** MVP

#### F3: Parallel Risk Re-evaluation
- The Property Re-evaluation Agent assesses **4 risk categories** (MVP), expandable to 6 (Stretch), incorporating detected changes:

| # | Risk Category | What's Compared (Current vs. Prior) | Re-evaluation Output | Scope |
|---|--------------|-------------------------------------|---------------------|-------|
| 1 | **Fire Risk Change** | Sprinkler status, alarm certification, cooking operations, electrical age, fire protection class | Fire risk delta score (improved / stable / deteriorated) + reasoning | **MVP** |
| 2 | **Natural Catastrophe Change** | Flood zone, earthquake proximity, windstorm zone (re-mapped?), wildfire risk level | NatCat risk delta (improved / stable / deteriorated) + reasoning | **MVP** |
| 3 | **Occupancy Change** | Tenant type, vacancy rate, business operations, hazardous materials, foot traffic | Occupancy risk delta (improved / stable / deteriorated) + reasoning | **MVP** |
| 4 | **Building Condition Change** | Roof age/condition, HVAC, plumbing/electrical age, renovation status, code violations | Building risk delta (improved / stable / deteriorated) + reasoning | **MVP** |
| 5 | **Protection System Change** | Sprinkler coverage %, alarm system type, security measures, CCTV, access control | Protection risk delta (improved / stable / deteriorated) + reasoning | Stretch |
| 6 | **Loss History Trend** | Prior-year claims vs. current-year claims, frequency change, severity change, large loss occurrence | Loss trend delta (improving / stable / worsening) + reasoning | Stretch |

- Each category produces a **delta score** (not just an absolute score): how much did this risk factor change from last year?
- Delta classifications: **Improved** (risk decreased), **Stable** (no material change), **Deteriorated** (risk increased)
- Results combined into **Overall Risk Trend**: Improved / Stable / Deteriorated
- **Scope:** MVP (4 categories) / Stretch (all 6)

#### F4: Renewal Recommendation Generation
- The Renewal Recommendation Agent produces a renewal decision for each property based on change detection and risk re-evaluation results
- **Four renewal decision categories:**

| Decision | Criteria | Action Required |
|----------|---------|-----------------|
| **Renew As-Is** | Risk profile stable or improved. No material changes detected. All risk categories stable or improved. | None — renew at current terms and premium |
| **Renew with Adjustments** | Material changes detected that require premium or terms modification. Risk deteriorated in 1–2 categories but within appetite. | Adjust premium (increase/decrease), modify coverage terms, add sublimits or exclusions |
| **Refer to Senior** | Complex changes requiring human judgment. Mixed signals (some categories improved, some deteriorated). Multiple material changes. | Senior underwriter reviews before renewal decision is finalized |
| **Non-Renew** | Significant risk deterioration beyond appetite. Multiple categories deteriorated. Unresolved code violations. Catastrophic exposure increase. | Do not renew coverage. Provide notice per state requirements. |

- Each recommendation includes: decision, confidence level, supporting evidence, suggested adjustments (if applicable), and a one-paragraph rationale
- **Scope:** MVP

#### F5: Portfolio Summary Report
- After all properties are processed, the Orchestrator aggregates results into a portfolio-level summary:
  - **Renewal decision mix** — e.g., "14 Renew As-Is, 4 Renew with Adjustments, 1 Refer, 1 Non-Renew"
  - **Portfolio risk trend** — Aggregate risk shift (e.g., "Portfolio risk score shifted from 4.2 → 4.8, net deterioration")
  - **Material changes count** — Number of properties with material changes, by category
  - **Exception properties** — Top properties requiring attention, ranked by risk change magnitude
  - **Premium impact estimate** — Directional portfolio premium change (e.g., "+$45K estimated across adjusted properties")
- Output format: Markdown (displayed in chat) and JSON (for export)
- **Scope:** MVP

#### F6: Conversational Review Interface
- After batch processing completes, the Portfolio Orchestrator enters conversational review mode
- The underwriter can:
  - **Query portfolio summary** — "How many properties have material changes?"
  - **Drill into specific properties** — "Tell me about the flood zone change on property #12"
  - **Compare properties** — "Which properties had the biggest risk increase?"
  - **Override recommendations** — "Change property #7 from 'Refer' to 'Renew with Adjustments'" (Stretch)
  - **Ask general questions** — "What does protection class 8 mean?"
  - **Export report** — "Generate the final portfolio renewal report" (Stretch)
- Adapts response style based on user profile (Aisha gets detailed explanations, Robert gets concise summaries)
- **Scope:** MVP (query, drill-down, profile-adaptive) / Stretch (override, export)

### 4.2 Feature Interaction Map

```
                    PHASE 1: BATCH (Non-Conversational)
                    ═══════════════════════════════════

Portfolio JSON ──► [F1: Batch Intake & Validate]
                        │
                        ├──► Property 1 ──► [F2: Change Detection] ──► [F3: Risk Re-eval] ──► [F4: Recommendation]
                        ├──► Property 2 ──► [F2: Change Detection] ──► [F3: Risk Re-eval] ──► [F4: Recommendation]
                        ├──► ...
                        └──► Property N ──► [F2: Change Detection] ──► [F3: Risk Re-eval] ──► [F4: Recommendation]
                                │
                                ▼
                        [F5: Portfolio Summary Report]


                    PHASE 2: REVIEW (Conversational)
                    ═════════════════════════════════

                        [F6: Conversational Review Interface]
                               │
                               ├── Query portfolio summary
                               ├── Drill into properties
                               ├── Override recommendations (Stretch)
                               └── [User Profiling] adapts all review responses
```

---

## 5. Non-Functional Requirements

| # | Requirement | Target | Notes |
|---|------------|--------|-------|
| NFR-01 | Per-property processing time | < 15 seconds | Change Detection + Re-evaluation + Recommendation per property |
| NFR-02 | Full portfolio batch time (20 properties) | < 3 minutes | Parallel processing, not sequential |
| NFR-03 | Parallel property throughput | 10–50 concurrent property pipelines | Limited by Gemini API rate limits in practice; use `asyncio.gather` |
| NFR-04 | Conversational review response time | < 10 seconds | For drill-down queries after batch completes |
| NFR-05 | Tracing coverage | 100% of LLM calls | All calls traced via OpenTelemetry → Arize |
| NFR-06 | Prompt template versioning | All prompts versioned | Via Prompt Manager module |
| NFR-07 | Graceful degradation | Fallback to sequential | If parallel execution fails, process properties one at a time |
| NFR-08 | Data persistence | JSON file-based | Portfolios, results, and user profiles persist as JSON files across sessions |
| NFR-09 | Change detection completeness | 100% of tracked fields compared | System compares every field between current and prior year |
| NFR-10 | Renewal classification consistency | Same inputs → same decision | Deterministic renewal recommendation given identical property data |

---

## 6. System Architecture

### 6.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            PRESENTATION LAYER                                │
│                                                                              │
│   ┌──────────────────────────────────────────────────────────────────┐      │
│   │                ADK Web UI (built-in `adk web`)                    │      │
│   │   - Batch trigger: submit portfolio JSON path in chat             │      │
│   │   - Batch progress: "Processing 20 properties... 15/20 done"     │      │
│   │   - Portfolio summary display (Markdown rendered in chat)         │      │
│   │   - Conversational review (drill-down queries, overrides)        │      │
│   │   - Agent interaction log (visible agent delegation)             │      │
│   └──────────────────────────────┬───────────────────────────────────┘      │
│                                  │                                          │
└──────────────────────────────────┼──────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           AGENT LAYER (ADK)                                  │
│                                                                              │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │         PORTFOLIO ORCHESTRATOR AGENT (Main Agent, Dual-Mode)       │    │
│   │                                                                    │    │
│   │   MODE 1 — Batch Processing:                                       │    │
│   │     - Accepts portfolio JSON, validates all properties             │    │
│   │     - Fans out N parallel property pipelines                       │    │
│   │     - Collects results, generates portfolio summary                │    │
│   │                                                                    │    │
│   │   MODE 2 — Conversational Review:                                  │    │
│   │     - Profiles underwriter (Aisha vs Robert)                       │    │
│   │     - Answers queries about portfolio results                      │    │
│   │     - Supports drill-down, comparison, override                    │    │
│   └──┬─────────────────────┬─────────────────────┬─────────────────────┘    │
│      │ A2A (per property)  │ A2A (per property)  │ A2A (per property)       │
│      │ × N parallel        │ × N parallel        │ × N parallel             │
│      ▼                     ▼                     ▼                          │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                   │
│  │ CHANGE       │   │ PROPERTY     │   │ RENEWAL      │                   │
│  │ DETECTION    │   │ RE-EVALUATION│   │ RECOMMENDATION│                  │
│  │ AGENT        │   │ AGENT        │   │ AGENT        │                   │
│  │              │   │              │   │              │                   │
│  │ - Compare    │   │ - Fire Risk  │   │ - Renew As-Is│                   │
│  │   current vs │   │   Change     │   │ - Adjust     │                   │
│  │   prior year │   │ - NatCat     │   │ - Refer      │                   │
│  │ - Classify   │   │   Change     │   │ - Non-Renew  │                   │
│  │   material/  │   │ - Occupancy  │   │              │                   │
│  │   notable/   │   │   Change     │   │ - Confidence │                   │
│  │   cosmetic   │   │ - Building   │   │ - Rationale  │                   │
│  │              │   │   Change     │   │              │                   │
│  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                   │
│         │                  │                   │                            │
└─────────┼──────────────────┼───────────────────┼────────────────────────────┘
          │                  │                   │
          ▼                  ▼                   ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                            SERVICES LAYER                                    │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐     │
│  │ Prompt       │  │ User Profile │  │ Mock Data    │  │ Arize      │     │
│  │ Manager      │  │ Store        │  │ Store        │  │ Telemetry  │     │
│  │              │  │              │  │              │  │            │     │
│  │ - Templates  │  │ - JSON       │  │ - 3 Portfol. │  │ - Traces   │     │
│  │ - Versions   │  │   files      │  │ - Current yr │  │ - Evals    │     │
│  │ - format_map │  │ - Analyst    │  │ - Prior yr   │  │ - Annot.   │     │
│  │              │  │   profiles   │  │ - 18-21 prop │  │   queues   │     │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘     │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ Portfolio Data Store (JSON Files)                                     │   │
│  │ - portfolios/            → input portfolio JSON files                 │   │
│  │ - results/               → per-property change/re-eval/recommendation │   │
│  │ - reports/               → generated portfolio summary reports         │   │
│  │ - user_profiles/         → analyst profile JSON files                 │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                          DEPLOYMENT LAYER                                    │
│                                                                              │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────────┐ │
│  │ Vertex AI Agent Engine       │  │ Local ADK Server (Fallback)         │ │
│  │ - Serverless scaling         │  │ - adk web / adk run                 │ │
│  │ - Cloud Logging              │  │ - Local testing                     │ │
│  └──────────────────────────────┘  └─────────────────────────────────────┘ │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Tech Stack Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Agent Framework | Google ADK (Python) | Required tech stack; native Gemini integration, sub-agent support for parallel fan-out |
| LLM (reasoning) | Gemini 2.5 Pro | Risk re-evaluation (CoT), portfolio orchestration (complex coordination) |
| LLM (speed) | Gemini 2.5 Flash | Change detection (field comparison), renewal classification (structured output) — speed-critical in batch |
| Agent Communication | A2A Protocol | Required tech stack; demonstrates agent interoperability for parallel task delegation |
| UI | **ADK Web** (built-in `adk web`) | Zero build time; ADK provides chat UI for both batch trigger and conversational review. **No Streamlit.** |
| Storage | JSON files | Simplest possible for demo; portfolios, results, profiles as JSON files. **No SQLite.** |
| Prompt Management | Custom Python module (dict-based, `str.format_map`) | Lightweight, demonstrates concept clearly. **No Jinja2.** |
| Monitoring | Arize Cloud + OpenTelemetry | Required tech stack; cloud-hosted observability for batch and review traces |
| Deployment | Vertex AI Agent Engine / Local ADK | GCP if available, local `adk web` fallback |

---

## 7. Agent Specifications (ADK)

### 7.1 Portfolio Orchestrator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `portfolio_orchestrator` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Dual-mode agent: (1) batch processing orchestration — accepts portfolio JSON, fans out parallel property reviews, aggregates results; (2) conversational review — answers underwriter queries about results |
| **Mode** | Dual: Non-conversational (batch trigger) + Conversational (review chat) |

**Responsibilities:**
1. Accept portfolio JSON input (via chat message or file path) and validate all properties
2. Fan out N parallel property pipelines — for each property, invoke Change Detection → Re-evaluation → Recommendation
3. Collect and aggregate all per-property results into a portfolio summary
4. Transition to conversational review mode after batch completes
5. Profile the underwriter on first conversational interaction (user profiling)
6. Answer queries about portfolio results with appropriate detail level (verbose for Aisha, concise for Robert)
7. Support drill-down into specific properties, property comparison, and recommendation overrides (Stretch)

**ADK Tools (Functions):**

```python
# Tool 1: Load and validate portfolio JSON
def load_portfolio(portfolio_path: str) -> dict:
    """
    Loads a portfolio JSON file, validates all properties have required
    current-year and prior-year fields.
    Returns: validated portfolio dict with property count and any validation errors.
    """
    pass

# Tool 2: Process a single property through the full pipeline (called N times in parallel)
def process_property(
    property_id: str,
    current_year_data: dict,
    prior_year_data: dict
) -> dict:
    """
    Runs the full per-property pipeline:
    1. Change Detection Agent → detect changes
    2. Property Re-evaluation Agent → re-evaluate risk
    3. Renewal Recommendation Agent → generate renewal decision
    Returns aggregated results for this property.
    """
    pass

# Tool 3: Generate portfolio summary report
def generate_portfolio_summary(
    portfolio_name: str,
    property_results: list[dict]
) -> dict:
    """
    Aggregates per-property results into a portfolio-level summary.
    Returns: renewal decision mix, risk trend, exception count,
    material changes summary, premium impact estimate.
    """
    pass

# Tool 4: Get/update user profile
def manage_user_profile(user_id: str, action: str, data: dict = None) -> dict:
    """
    Retrieves or updates the underwriter's profile.
    Actions: 'get', 'update', 'get_or_create'
    Returns profile with experience_level, preferred_mode, review_preferences.
    """
    pass

# Tool 5: Get prompt from Prompt Manager
def get_prompt(template_name: str, version: str = "latest", variables: dict = None) -> str:
    """
    Retrieves a prompt template from the Prompt Manager,
    optionally rendering with provided variables via str.format_map.
    """
    pass

# Tool 6: Query portfolio results (for conversational review)
def query_results(
    portfolio_id: str,
    query_type: str,
    property_id: str = None
) -> dict:
    """
    Queries stored portfolio results during conversational review.
    query_type: 'summary', 'property_detail', 'exceptions', 'comparison'
    Returns filtered results matching the query.
    """
    pass
```

**System Instruction (Summary):**
```
You are an expert property portfolio renewal assistant for mid-market commercial
property insurance. You operate in two modes:

MODE 1 — BATCH PROCESSING:
When the user provides a portfolio JSON file or path, you:
1. Load and validate the portfolio
2. Process all properties in parallel (change detection → re-evaluation → recommendation)
3. Aggregate results into a portfolio summary
4. Present the portfolio summary and transition to review mode

MODE 2 — CONVERSATIONAL REVIEW:
After batch processing (or when loaded results exist), you:
1. Profile the underwriter (if first interaction)
2. Answer queries about portfolio results
3. Support drill-down into specific properties
4. Present results adapted to the underwriter's experience level:
   - Junior analysts (e.g., Aisha): Detailed explanations of each change,
     guided property-by-property walkthrough, define terminology
   - Senior managers (e.g., Robert): Exception-only summary, quick drill-down,
     skip explanations for standard concepts

Always provide structured, evidence-based responses with specific property
data and risk scores referenced.
```

**Why Gemini Pro:** The Orchestrator manages the most complex coordination in the system — parallel fan-out of N property pipelines, result aggregation, and nuanced conversational review with profile-adaptive responses. Pro's instruction-following and contextual reasoning are essential for both modes.

### 7.2 Property Re-evaluation Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `property_reevaluation_agent` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Re-evaluates risk for a single property by comparing current-year data to prior-year data across 4 risk categories (MVP) using Chain-of-Thought reasoning |
| **Mode** | Non-conversational (task-based); runs in parallel — one instance per property |

**Responsibilities:**
1. Receive current-year and prior-year property data along with detected changes from the Change Detection Agent
2. Evaluate 4 risk categories (MVP: Fire Risk Change, NatCat Change, Occupancy Change, Building Condition Change)
3. For each category, compare current vs. prior year and produce a **delta score** (improved / stable / deteriorated) with Chain-of-Thought reasoning
4. Combine category deltas into an overall risk trend
5. Return comprehensive risk re-evaluation results

**ADK Tools (Functions):**

```python
# Tool 1: Assess fire risk change
def assess_fire_risk_change(
    current_fire_data: dict,
    prior_fire_data: dict,
    detected_changes: list[dict]
) -> dict:
    """
    Compares current vs prior year fire risk factors using CoT.
    Returns: fire_risk_delta (improved/stable/deteriorated),
    current_score (1-10), prior_score (1-10), delta_magnitude,
    key_changes, reasoning.
    """
    pass

# Tool 2: Assess natural catastrophe risk change
def assess_natcat_risk_change(
    current_natcat_data: dict,
    prior_natcat_data: dict,
    detected_changes: list[dict]
) -> dict:
    """
    Compares current vs prior year NatCat exposure using CoT.
    Returns: natcat_risk_delta, current_score, prior_score,
    delta_magnitude, zone_reclassifications, reasoning.
    """
    pass

# Tool 3: Assess occupancy risk change
def assess_occupancy_risk_change(
    current_occupancy_data: dict,
    prior_occupancy_data: dict,
    detected_changes: list[dict]
) -> dict:
    """
    Compares current vs prior year occupancy risk using CoT.
    Returns: occupancy_risk_delta, current_score, prior_score,
    delta_magnitude, tenant_changes, reasoning.
    """
    pass

# Tool 4: Assess building condition risk change
def assess_building_condition_change(
    current_building_data: dict,
    prior_building_data: dict,
    detected_changes: list[dict]
) -> dict:
    """
    Compares current vs prior year building condition using CoT.
    Returns: building_risk_delta, current_score, prior_score,
    delta_magnitude, code_violations, reasoning.
    """
    pass

# Tool 5: Combine category deltas into overall risk trend
def combine_risk_deltas(
    fire_result: dict,
    natcat_result: dict,
    occupancy_result: dict,
    building_result: dict
) -> dict:
    """
    Combines individual risk category deltas into an overall risk trend.
    Returns: overall_risk_trend (improved/stable/deteriorated),
    current_overall_score, prior_overall_score, delta_summary.
    """
    pass
```

**System Instruction (Summary):**
```
You are an expert property risk re-evaluation specialist. Your job is to
compare a property's CURRENT year data against its PRIOR year data and
determine how risk has changed.

For each risk category, provide:
1. Prior year score (1-10)
2. Current year score (1-10)
3. Delta direction: IMPROVED, STABLE, or DETERIORATED
4. Delta magnitude: how significant is the change (minor/moderate/significant)
5. Key changes that drove the delta
6. Step-by-step reasoning (Chain-of-Thought)

Focus on CHANGES, not absolute risk. A property can have high absolute risk
but stable delta (no change from last year = renew as-is). Conversely, a
property with moderate absolute risk but significant deterioration needs
attention.
```

**Why Gemini Pro:** Risk re-evaluation is the highest-complexity task — comparing multi-factor data across two time periods, reasoning about which changes are material, and producing defensible delta assessments. Pro produces more nuanced Chain-of-Thought reasoning with better contextual understanding of how individual changes interact.

### 7.3 Change Detection Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `change_detection_agent` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Compares current-year vs. prior-year property data field-by-field, identifies and classifies changes by severity (material / notable / cosmetic) |
| **Mode** | Non-conversational (task-based); runs in parallel — one instance per property |

**Responsibilities:**
1. Receive current-year and prior-year data for a single property
2. Compare every tracked field between the two years
3. Identify all differences and classify each by severity (material / notable / cosmetic)
4. Flag changes that directly affect risk evaluation
5. Return a structured change report

**ADK Tools (Functions):**

```python
# Tool 1: Compare property data fields
def compare_property_fields(
    current_data: dict,
    prior_data: dict
) -> list[dict]:
    """
    Field-by-field comparison of current vs prior year data.
    Returns list of changes: [{field, prior_value, current_value,
    change_type, severity (material/notable/cosmetic)}]
    """
    pass

# Tool 2: Classify change severity
def classify_change_severity(
    field_name: str,
    prior_value: str,
    current_value: str,
    property_context: dict
) -> dict:
    """
    Determines if a specific field change is material, notable, or cosmetic.
    Uses domain rules: e.g., flood zone change = always material,
    contact info change = always cosmetic.
    Returns: severity, risk_impact, reasoning.
    """
    pass

# Tool 3: Generate change summary
def generate_change_summary(
    property_id: str,
    all_changes: list[dict]
) -> dict:
    """
    Summarizes all detected changes for a single property.
    Returns: total_changes, material_count, notable_count, cosmetic_count,
    material_changes (detailed list), overall_change_assessment.
    """
    pass
```

**System Instruction (Summary):**
```
You are a property data change detection specialist. Your job is to compare
a property's current-year data against its prior-year data and identify
every change.

For each change found, classify its severity:
- MATERIAL: Directly affects risk score or renewal terms.
  Examples: flood zone reclassification, sprinkler system removed, occupancy
  type changed from office to restaurant, new building code violation.
- NOTABLE: Worth reviewing but may not change terms.
  Examples: minor TIV increase (<10%), tenant name change with same
  occupancy type, building age crossed a threshold year.
- COSMETIC: No risk impact.
  Examples: contact information updated, address formatting changed,
  policy number reformatted.

Be exhaustive — compare EVERY field. Do not skip fields just because they
seem unimportant. A missed material change is a serious underwriting error.
```

**Why Gemini Flash:** Change detection is primarily a structured comparison task — compare field A (prior) vs. field A (current), classify the difference. It's rule-driven with clear thresholds, not complex reasoning. Flash is faster, which is critical when running N property comparisons in parallel during batch processing.

### 7.4 Renewal Recommendation Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `renewal_recommendation_agent` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Generates a renewal decision for each property (renew as-is / renew with adjustments / refer to senior / non-renew) based on change detection results and risk re-evaluation |
| **Mode** | Non-conversational (task-based); runs in parallel — one instance per property |

**Responsibilities:**
1. Receive change detection results and risk re-evaluation results for a single property
2. Apply renewal decision logic to classify the recommendation
3. Generate a confidence level, supporting evidence, and rationale
4. If "Renew with Adjustments," suggest specific adjustments (premium change, coverage modifications)
5. Return structured renewal recommendation

**ADK Tools (Functions):**

```python
# Tool 1: Classify renewal decision
def classify_renewal_decision(
    change_summary: dict,
    risk_reevaluation: dict,
    property_context: dict
) -> dict:
    """
    Applies renewal decision logic using Few-Shot classification.
    Returns: decision (renew_as_is/renew_with_adjustments/refer_to_senior/
    non_renew), confidence (high/medium/low), primary_drivers.
    """
    pass

# Tool 2: Generate adjustment recommendations
def generate_adjustments(
    decision: str,
    change_summary: dict,
    risk_reevaluation: dict
) -> dict:
    """
    If decision is 'renew_with_adjustments', generates specific suggestions.
    Returns: premium_adjustment_pct, coverage_modifications,
    sublimits_recommended, exclusions_recommended, rationale.
    """
    pass

# Tool 3: Generate recommendation rationale
def generate_rationale(
    decision: str,
    change_summary: dict,
    risk_reevaluation: dict,
    adjustments: dict = None
) -> str:
    """
    Produces a one-paragraph human-readable rationale for the renewal
    decision, suitable for broker communication.
    """
    pass
```

**System Instruction (Summary):**
```
You are a property renewal recommendation specialist. Based on detected
changes and risk re-evaluation, recommend one of four renewal decisions:

1. RENEW AS-IS: Risk profile stable or improved. No material changes.
2. RENEW WITH ADJUSTMENTS: Material changes exist but risk remains within
   appetite after premium/terms modification.
3. REFER TO SENIOR: Complex or mixed signal changes requiring human
   judgment. You are NOT confident in an automated decision.
4. NON-RENEW: Significant risk deterioration beyond carrier appetite.
   Multiple categories deteriorated significantly.

Decision rules:
- 0 material changes + all categories stable/improved → RENEW AS-IS
- 1-2 material changes + risk within appetite → RENEW WITH ADJUSTMENTS
- 3+ material changes OR mixed signals → REFER TO SENIOR
- 2+ categories with significant deterioration → NON-RENEW

For each recommendation, provide:
- Decision category
- Confidence level (HIGH / MEDIUM / LOW)
- Top 3 supporting evidence points
- One-paragraph rationale
- Suggested adjustments (if applicable)
```

**Why Gemini Flash:** Renewal classification is a structured decision with clear rules and a finite set of outcomes (4 categories). It benefits more from speed (critical in batch processing N properties) than from deep reasoning. The few-shot examples in the prompt provide sufficient context for accurate classification.

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
| Task delegation | ✅ Use ADK's built-in sub-agent pattern (Orchestrator calls sub-agents directly in parallel) | Full A2A SDK client/server for all agents |
| A2A protocol demo | ✅ One working A2A client→server call (Orchestrator → Change Detection) to demonstrate the concept | All 3 sub-agents via A2A endpoints |
| Parallel fan-out | ✅ `asyncio.gather` with ADK sub-agent calls per property | Full A2A parallel tasking with streaming status |
| Streaming | ❌ Not in MVP | SSE streaming for batch progress updates |

> **Why this approach?** ADK natively supports sub-agents — the Orchestrator can call Change Detection, Property Re-evaluation, and Renewal Recommendation as sub-agents with zero A2A overhead. For the MVP, the intern uses this built-in mechanism for the actual batch workflow, then adds one real A2A server endpoint (on the Change Detection Agent) to demonstrate the A2A protocol. Agent Cards are created as static JSON files for all 4 agents — these are valuable artifacts that show the intern understands A2A discovery even if not all calls use the full protocol. The parallel fan-out pattern (N properties × 3 agents) makes this project's A2A design notably more complex than Projects 1 and 2.

### 8.2 Agent Cards

Each agent publishes an Agent Card at `/.well-known/agent.json`:

**Portfolio Orchestrator Agent Card:**
```json
{
  "name": "portfolio_orchestrator",
  "description": "Property portfolio renewal orchestrator. Accepts portfolio JSON, fans out parallel property reviews, aggregates results, and provides conversational review interface for underwriters.",
  "url": "http://localhost:8001",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "portfolio-batch-review",
      "name": "Portfolio Batch Renewal Review",
      "description": "Processes a portfolio of 10-50 properties in parallel: detects changes, re-evaluates risk, generates renewal recommendations, and produces portfolio summary.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    },
    {
      "id": "portfolio-conversational-review",
      "name": "Portfolio Conversational Review",
      "description": "Interactive review of batch processing results. Supports drill-down queries, property comparisons, and recommendation overrides.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Change Detection Agent Card:**
```json
{
  "name": "change_detection_agent",
  "description": "Compares current-year vs. prior-year property data to detect material, notable, and cosmetic changes.",
  "url": "http://localhost:8002",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "property-change-detection",
      "name": "Property Change Detection",
      "description": "Field-by-field comparison of current-year vs. prior-year property data. Classifies changes as material (risk-affecting), notable (review-worthy), or cosmetic (no risk impact).",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Property Re-evaluation Agent Card:**
```json
{
  "name": "property_reevaluation_agent",
  "description": "Re-evaluates risk for a single property by comparing current vs. prior year data across fire, NatCat, occupancy, and building risk categories.",
  "url": "http://localhost:8003",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "property-risk-reevaluation",
      "name": "Property Risk Re-evaluation",
      "description": "Compares current-year vs. prior-year property data across 4 risk categories (fire, NatCat, occupancy, building) to produce risk delta scores and overall risk trend using Chain-of-Thought reasoning.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Renewal Recommendation Agent Card:**
```json
{
  "name": "renewal_recommendation_agent",
  "description": "Generates renewal decisions (renew as-is, adjust, refer, non-renew) based on change detection and risk re-evaluation results.",
  "url": "http://localhost:8004",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "renewal-recommendation",
      "name": "Property Renewal Recommendation",
      "description": "Classifies renewal decision into 4 categories (renew as-is, renew with adjustments, refer to senior, non-renew) with confidence level, evidence, rationale, and suggested adjustments.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

### 8.3 A2A Task Flow — Parallel Fan-Out

This project's A2A flow is fundamentally different from Projects 1 and 2 because the Orchestrator sends N parallel task groups (one per property), each containing 3 sequential sub-tasks.

```
Portfolio Orchestrator          Change Detection       Re-evaluation         Recommendation
     │                               │                      │                      │
     │  [Load portfolio JSON:        │                      │                      │
     │   20 properties]              │                      │                      │
     │                               │                      │                      │
     │═══ PARALLEL FAN-OUT (N properties simultaneously) ═════════════════════════│
     │                               │                      │                      │
     │── Property 1: Create Task ──►│                      │                      │
     │── Property 2: Create Task ──►│                      │                      │
     │── Property 3: Create Task ──►│                      │                      │
     │── ...                         │                      │                      │
     │── Property N: Create Task ──►│                      │                      │
     │     "Detect changes for       │                      │                      │
     │      property X: current      │                      │                      │
     │      vs prior year data"      │                      │                      │
     │                               │                      │                      │
     │◄── Property 1: Complete ──────│                      │                      │
     │◄── Property 2: Complete ──────│                      │                      │
     │◄── ...                        │                      │                      │
     │◄── Property N: Complete ──────│                      │                      │
     │    {change_reports × N}       │                      │                      │
     │                               │                      │                      │
     │── Property 1: Create Task ────────────────────────►│                      │
     │── Property 2: Create Task ────────────────────────►│                      │
     │── ...                         │                      │                      │
     │── Property N: Create Task ────────────────────────►│                      │
     │     "Re-evaluate risk for     │                      │                      │
     │      property X with changes" │                      │                      │
     │                               │                      │                      │
     │◄── Property 1: Complete ──────────────────────────│                      │
     │◄── Property 2: Complete ──────────────────────────│                      │
     │◄── ...                        │                      │                      │
     │◄── Property N: Complete ──────────────────────────│                      │
     │    {reevaluation_results × N} │                      │                      │
     │                               │                      │                      │
     │── Property 1: Create Task ──────────────────────────────────────────────►│
     │── Property 2: Create Task ──────────────────────────────────────────────►│
     │── ...                         │                      │                      │
     │── Property N: Create Task ──────────────────────────────────────────────►│
     │     "Generate renewal         │                      │                      │
     │      recommendation for       │                      │                      │
     │      property X"              │                      │                      │
     │                               │                      │                      │
     │◄── Property 1: Complete ────────────────────────────────────────────────│
     │◄── Property 2: Complete ────────────────────────────────────────────────│
     │◄── ...                        │                      │                      │
     │◄── Property N: Complete ────────────────────────────────────────────────│
     │    {recommendations × N}      │                      │                      │
     │                               │                      │                      │
     │═══ AGGREGATE RESULTS ══════════════════════════════════════════════════════│
     │                               │                      │                      │
     ▼ Generate Portfolio Summary    │                      │                      │
     ▼ Enter Conversational Review   │                      │                      │
```

### 8.4 Implementation Guidance

**How to implement A2A in this project:**

1. **Install the A2A SDK:**
   ```bash
   pip install a2a-sdk
   ```

2. **Each agent runs as a separate A2A-compatible server** (separate port or separate Cloud Run service):
   ```
   portfolio_orchestrator       → http://localhost:8001
   change_detection_agent       → http://localhost:8002
   property_reevaluation_agent  → http://localhost:8003
   renewal_recommendation_agent → http://localhost:8004
   ```

3. **The Orchestrator uses an A2A client with parallel fan-out** to process N properties simultaneously:
   ```python
   from a2a.client import A2AClient
   import asyncio
   import json

   # Discover the change detection agent
   change_detector = A2AClient(url="http://localhost:8002")
   card = await change_detector.get_agent_card()

   # Fan-out: send N change detection tasks in parallel
   async def detect_changes_for_property(property_data):
       task = await change_detector.send_task({
           "id": f"change-detect-{property_data['property_id']}",
           "skill_id": "property-change-detection",
           "message": {
               "role": "user",
               "parts": [{
                   "text": json.dumps({
                       "property_id": property_data["property_id"],
                       "current_year": property_data["current_year"],
                       "prior_year": property_data["prior_year"]
                   })
               }]
           }
       })
       return task

   # Process ALL properties in parallel
   portfolio = load_portfolio("midwest_office_portfolio.json")
   change_tasks = await asyncio.gather(*[
       detect_changes_for_property(prop)
       for prop in portfolio["properties"]
   ])
   ```

4. **Each sub-agent implements a TaskHandler that processes one property:**
   ```python
   from a2a.server import A2AServer, TaskHandler
   import json

   class ChangeDetectionHandler(TaskHandler):
       async def handle_task(self, task):
           """Detect changes for a single property."""
           input_data = json.loads(task.message.parts[0].text)

           current = input_data["current_year"]
           prior = input_data["prior_year"]

           # Compare every field between current and prior year
           changes = self.compare_all_fields(current, prior)

           # Classify each change
           classified = [
               self.classify_severity(c) for c in changes
           ]

           return TaskResult(
               artifacts=[Artifact(parts=[TextPart(text=json.dumps({
                   "property_id": input_data["property_id"],
                   "total_changes": len(classified),
                   "material_changes": [c for c in classified if c["severity"] == "material"],
                   "notable_changes": [c for c in classified if c["severity"] == "notable"],
                   "cosmetic_changes": [c for c in classified if c["severity"] == "cosmetic"],
               }))])]
           )
   ```

5. **MVP Shortcut — Use ADK sub-agents with `asyncio.gather` for parallel fan-out:**
   ```python
   from google.adk import Agent
   import asyncio

   # Define the sub-agents
   change_detection_agent = Agent(
       name="change_detection_agent",
       model="gemini-2.5-flash",
       instruction=CHANGE_DETECTION_PROMPT,
   )
   property_reevaluation_agent = Agent(
       name="property_reevaluation_agent",
       model="gemini-2.5-pro",
       instruction=REEVALUATION_PROMPT,
   )
   renewal_recommendation_agent = Agent(
       name="renewal_recommendation_agent",
       model="gemini-2.5-flash",
       instruction=RECOMMENDATION_PROMPT,
   )

   # Orchestrator uses sub-agents and fans out in parallel
   portfolio_orchestrator = Agent(
       name="portfolio_orchestrator",
       model="gemini-2.5-pro",
       instruction=ORCHESTRATOR_SYSTEM_PROMPT,
       sub_agents=[
           change_detection_agent,
           property_reevaluation_agent,
           renewal_recommendation_agent,
       ],
   )

   # The parallel fan-out happens in a tool function:
   async def process_portfolio_batch(portfolio: dict) -> dict:
       """Process all properties in parallel."""
       async def process_single_property(prop):
           # Sequential within each property: detect → re-eval → recommend
           changes = await change_detection_agent.run(prop)
           reevaluation = await property_reevaluation_agent.run({**prop, "changes": changes})
           recommendation = await renewal_recommendation_agent.run({
               **prop, "changes": changes, "reevaluation": reevaluation
           })
           return {
               "property_id": prop["property_id"],
               "changes": changes,
               "reevaluation": reevaluation,
               "recommendation": recommendation,
           }

       # Fan out ALL properties simultaneously
       results = await asyncio.gather(*[
           process_single_property(prop)
           for prop in portfolio["properties"]
       ])
       return {"property_results": results}
   ```

---

## 9. Workflow Design

### 9.1 Parallel Workflow — Portfolio Batch Processing

This is the **PRIMARY workflow** for this project. Unlike Projects 1 and 2 which are sequential, this project processes multiple properties simultaneously — a parallel batch pattern that mirrors real enterprise data pipelines.

```
┌────────────────┐
│ PORTFOLIO JSON │
│ (N Properties) │
└───────┬────────┘
        │
        ▼
┌────────────────┐
│ STEP 1:        │
│ VALIDATE &     │
│ QUEUE          │
│ (Orchestrator) │
└───────┬────────┘
        │
        ╔═══════════════════════════════════════════════════════════════╗
        ║            PARALLEL FAN-OUT (N concurrent pipelines)         ║
        ╠═══════════════════════════════════════════════════════════════╣
        ║                                                               ║
        ║  Property 1 ─► [Change] ─► [Re-eval] ─► [Recommend] ─► ✓    ║
        ║  Property 2 ─► [Change] ─► [Re-eval] ─► [Recommend] ─► ✓    ║
        ║  Property 3 ─► [Change] ─► [Re-eval] ─► [Recommend] ─► ✓    ║
        ║  Property 4 ─► [Change] ─► [Re-eval] ─► [Recommend] ─► ✓    ║
        ║  ...                                                          ║
        ║  Property N ─► [Change] ─► [Re-eval] ─► [Recommend] ─► ✓    ║
        ║                                                               ║
        ╚═══════════════════════════════════════════════════════════════╝
        │
        ▼
┌────────────────┐
│ STEP 3:        │
│ AGGREGATE      │
│ RESULTS &      │
│ GENERATE       │
│ PORTFOLIO      │
│ SUMMARY        │
│ (Orchestrator) │
└───────┬────────┘
        │
        ▼
┌────────────────┐
│ STEP 4:        │
│ ENTER          │
│ CONVERSATIONAL │
│ REVIEW MODE    │
│ (Orchestrator) │
└────────────────┘
```

**Batch Processing Step Details:**

| Step | Agent | Input | Output | Prompt Type |
|------|-------|-------|--------|-------------|
| 1. Validate & Queue | Orchestrator | Portfolio JSON file | Validated property list, batch job queued | Zero-shot (validation rules) |
| 2. Per-Property Pipeline (× N parallel) | Change Detection → Re-evaluation → Recommendation | Current + prior year data per property | Change report + risk delta + renewal decision per property | See Section 9.2 |
| 3. Aggregate Results | Orchestrator | N per-property results | Portfolio summary: decision mix, risk trend, exceptions, premium impact | Zero-shot (aggregation logic) |
| 4. Enter Review Mode | Orchestrator | Portfolio summary + per-property results | Transition to conversational mode, profile user | Few-shot (user profiling) |

**Why Parallel Fan-Out (not Sequential)?**
- Each property's review pipeline (Change → Re-eval → Recommend) is **independent** of other properties — Property 5's flood zone change has zero impact on Property 12's occupancy shift
- Sequential processing of 20 properties × 15 seconds each = 5 minutes. Parallel processing = ~15 seconds total (bounded by the slowest single property, not the sum)
- This is how real batch systems work — insurance carriers process thousands of renewals in parallel during peak season
- The intern learns `asyncio.gather` for concurrent execution, a production-critical Python pattern

### 9.2 Per-Property Pipeline — Sequential Within Each Property

Within each parallel lane, the 3 agents execute **sequentially** because each depends on the prior step's output:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ STEP 2a:        │────►│ STEP 2b:        │────►│ STEP 2c:        │
│ CHANGE          │     │ RISK            │     │ RENEWAL         │
│ DETECTION       │     │ RE-EVALUATION   │     │ RECOMMENDATION  │
│                 │     │                 │     │                 │
│ Agent: Change   │     │ Agent: Re-eval  │     │ Agent: Renewal  │
│ Model: Flash    │     │ Model: Pro      │     │ Model: Flash    │
│                 │     │                 │     │                 │
│ Input:          │     │ Input:          │     │ Input:          │
│ - Current year  │     │ - Current year  │     │ - Change report │
│ - Prior year    │     │ - Prior year    │     │ - Risk deltas   │
│                 │     │ - Change report │     │ - Property      │
│ Output:         │     │                 │     │   context       │
│ - Change report │     │ Output:         │     │                 │
│   (material/    │     │ - Risk deltas   │     │ Output:         │
│    notable/     │     │   per category  │     │ - Decision      │
│    cosmetic)    │     │ - Overall trend │     │ - Confidence    │
│                 │     │ - CoT reasoning │     │ - Rationale     │
│                 │     │                 │     │ - Adjustments   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

**Per-Property Step Details:**

| Step | Agent | Prompt Type | Duration (est.) | Why Sequential |
|------|-------|-------------|-----------------|----------------|
| 2a. Change Detection | Change Detection (Flash) | Zero-shot (field comparison) | ~3 seconds | Independent — first step, no dependencies |
| 2b. Risk Re-evaluation | Property Re-evaluation (Pro) | Chain-of-Thought (per category) | ~8 seconds | **Requires** change report from 2a — re-evaluation must know what changed to assess delta |
| 2c. Renewal Recommendation | Renewal Recommendation (Flash) | Few-shot (decision classification) | ~3 seconds | **Requires** both change report and risk deltas — decision logic depends on severity of changes AND magnitude of risk shifts |

**Total per property: ~14 seconds** (sequential within the pipeline)
**Total for 20 properties (parallel): ~14 seconds** (all pipelines run simultaneously)

### 9.3 Conversational Review Workflow — Post-Batch Review Chat

After batch processing completes, the Orchestrator transitions to conversational review mode. The underwriter interacts with results through a chat interface.

```
START (Batch Complete)
  │
  ▼
[Portfolio Summary Presented]
  "Portfolio 'Midwest Office Portfolio' — 6 properties reviewed.
   Summary: 4 Renew As-Is, 1 Renew with Adjustments, 1 Refer to Senior.
   Batch completed in 12 seconds."
  │
  ▼
[Profiling] ◄──────────────────────────────────────── First review session only
  "Before we review, a few quick questions to personalize your experience."
  (4 profiling questions — see Section 12)
  │
  ▼
[Review Mode — Adaptive]
  │
  ├── JUNIOR PATH (Aisha):
  │   "Let's walk through the portfolio property by property.
  │    Starting with the flagged properties..."
  │   → Guided walkthrough, explanations, term definitions
  │
  └── SENIOR PATH (Robert):
      "2 properties flagged for attention: Property #3 (Refer) and
       Property #5 (Adjust). All others stable. Want details?"
      → Exception-only, drill-down on demand
  │
  ▼
[Query Loop]
  Underwriter asks questions:
  │
  ├── "What changed on property #3?" ──► [Drill-Down]
  │   → Show change report, risk deltas, recommendation rationale
  │
  ├── "Which properties had the biggest risk increase?" ──► [Comparison]
  │   → Rank properties by risk delta magnitude
  │
  ├── "What does protection class 8 mean?" ──► [General Query]
  │   → Zero-shot explanation adapted to experience level
  │
  ├── "Change property #5 from Refer to Adjust" ──► [Override] (Stretch)
  │   → Update recommendation, log override
  │
  └── "Generate final report" ──► [Export] (Stretch)
      → Compile portfolio report for broker
  │
  ▼
[Session End]
  "Portfolio review complete. Results saved to results/midwest_office_portfolio.json"
```

### 9.4 Portfolio Aggregation Logic

The Orchestrator applies the following aggregation logic after all per-property results are collected:

**Renewal Decision Mix:**
```
For each property result:
    count[result.decision] += 1

Output:
    Renew As-Is:                {count["renew_as_is"]}  ({pct}%)
    Renew with Adjustments:     {count["renew_with_adjustments"]}  ({pct}%)
    Refer to Senior:            {count["refer_to_senior"]}  ({pct}%)
    Non-Renew:                  {count["non_renew"]}  ({pct}%)
```

**Portfolio Risk Trend:**
```
prior_avg = mean([prop.prior_overall_score for prop in results])
current_avg = mean([prop.current_overall_score for prop in results])
portfolio_delta = current_avg - prior_avg

If portfolio_delta < -0.5: "IMPROVED"
Elif portfolio_delta > 0.5: "DETERIORATED"
Else: "STABLE"

Output:
    Prior Year Average Score:   {prior_avg:.1f}
    Current Year Average Score: {current_avg:.1f}
    Portfolio Risk Trend:       {trend}  (delta: {portfolio_delta:+.1f})
```

**Exception Properties (sorted by risk change magnitude):**
```
exceptions = [prop for prop in results if prop.has_material_changes]
exceptions.sort(key=lambda p: abs(p.risk_delta), reverse=True)

Output:
    Top exceptions requiring attention:
    1. Property #12 — Risk delta +2.3 (deteriorated) — 4 material changes — REFER
    2. Property #7  — Risk delta +1.5 (deteriorated) — 2 material changes — ADJUST
    3. Property #19 — Risk delta -1.8 (improved)     — 1 material change  — AS-IS
```

**Premium Impact Estimate:**
```
For each "Renew with Adjustments" property:
    premium_impact += property.estimated_premium_change

Output:
    Estimated portfolio premium change: +${total_impact:,.0f}
    Properties with premium increase:  {count_increase}
    Properties with premium decrease:  {count_decrease}
```

---

## 10. Prompt Engineering Strategy

### 10.1 Prompt Types Used

This project uses **4 distinct prompt engineering techniques**, each chosen for the right task in the portfolio renewal workflow. The renewal context (comparing current vs. prior year data) creates unique prompt patterns not seen in Projects 1 or 2.

#### Type 1: Zero-Shot Prompting
**Used for:** Simple, well-defined tasks where the model's pre-training knowledge is sufficient without examples.

**Where in this project:**
- General queries in the conversational review chat ("What does protection class 8 mean?")
- Portfolio JSON validation (checking required fields, data types)
- Field-level data comparison in change detection (straightforward diff)
- Portfolio aggregation narration (summarizing computed statistics)

**Example (Review Chat — General Query):**
```
You are a property portfolio renewal assistant. The underwriter is asking a
general question during their review of the portfolio results.

Answer accurately and concisely, using commercial property insurance terminology
appropriate for a {experience_level} underwriter.

Context: The underwriter is reviewing the "{portfolio_name}" portfolio containing
{property_count} commercial properties at renewal time.

Underwriter question: {question}
```

**Example (Portfolio Validation):**
```
You are a property portfolio data validator. Validate the following portfolio
JSON for completeness and correctness before batch processing.

Required fields per property:
- property_id (string, unique)
- address (string, non-empty)
- property_type (one of: office, retail, warehouse, mixed_use, industrial)
- tiv (number, > 0)
- current_year (object with risk fields)
- prior_year (object with risk fields)

Required risk fields (in both current_year and prior_year):
- construction_type, year_built, stories, sqft
- sprinkler_status, alarm_status, fire_protection_class
- flood_zone, earthquake_zone, windstorm_zone
- occupancy_type, vacancy_rate, hazardous_materials
- roof_condition, hvac_status, code_violations

Portfolio to validate:
{portfolio_json}

Return: validation_status (valid/invalid), errors (list), warnings (list).
```

#### Type 2: Few-Shot Prompting
**Used for:** Tasks where examples significantly improve consistency, classification accuracy, and output format — particularly the 4-category renewal decision classification.

**Where in this project:**
- Renewal recommendation classification (4 decision categories with examples)
- User profiling (classifying underwriter experience level)
- Change severity classification (material vs. notable vs. cosmetic)

**Example (Renewal Recommendation Classification — Few-Shot with all 4 categories):**
```
Based on the detected changes and risk re-evaluation results, classify the
renewal recommendation into one of 4 categories: RENEW_AS_IS,
RENEW_WITH_ADJUSTMENTS, REFER_TO_SENIOR, or NON_RENEW.

Example 1 — RENEW AS-IS:
Property: 200 Lakeview Office Park, Chicago IL — Class A office building
Changes detected: 0 material, 1 notable (minor TIV increase +5%), 1 cosmetic
Risk re-evaluation: Fire (stable), NatCat (stable), Occupancy (stable), Building (improved — new HVAC)
Overall risk trend: STABLE (prior: 3.2, current: 3.0, delta: -0.2)
Decision: RENEW_AS_IS
Confidence: HIGH
Rationale: No material changes detected. Risk profile remains low and stable
with a slight improvement from HVAC upgrade. TIV increase is routine annual
adjustment. Recommend renewal at current terms with minor TIV adjustment.

Example 2 — RENEW WITH ADJUSTMENTS:
Property: 450 Gulf Coast Retail Plaza, Tampa FL — Strip retail center
Changes detected: 2 material (flood zone reclassified AE→VE, new restaurant tenant), 1 notable
Risk re-evaluation: Fire (deteriorated — restaurant tenant), NatCat (deteriorated — higher flood zone), Occupancy (stable), Building (stable)
Overall risk trend: DETERIORATED (prior: 4.5, current: 6.2, delta: +1.7)
Decision: RENEW_WITH_ADJUSTMENTS
Confidence: HIGH
Rationale: Two material changes directly impact risk: flood zone upgrade from
AE to VE increases NatCat exposure significantly, and new restaurant tenant
elevates fire risk. Recommend premium increase of 15-20%, add flood sublimit,
require restaurant tenant fire suppression certification.

Example 3 — REFER TO SENIOR:
Property: 800 Industrial Ave, Newark NJ — Mixed-use warehouse/office
Changes detected: 3 material (2 code violations unresolved, occupancy shift warehouse→light manufacturing, new hazmat storage), 2 notable
Risk re-evaluation: Fire (deteriorated — hazmat), NatCat (stable), Occupancy (significantly deteriorated), Building (deteriorated — code violations)
Overall risk trend: DETERIORATED (prior: 5.0, current: 7.8, delta: +2.8)
Decision: REFER_TO_SENIOR
Confidence: MEDIUM
Rationale: Multiple material changes create a complex risk picture. Occupancy
shift to light manufacturing with hazmat storage fundamentally changes the risk
profile. Unresolved code violations compound building risk. The combination of
changes across 3 categories requires senior underwriter judgment on whether the
account remains within appetite.

Example 4 — NON-RENEW:
Property: 125 Riverside Warehouse, Biloxi MS — Vacant industrial warehouse
Changes detected: 4 material (building condemned, full vacancy, sprinkler system decommissioned, 3 fire losses in past year)
Risk re-evaluation: Fire (significantly deteriorated), NatCat (deteriorated — hurricane zone), Occupancy (significantly deteriorated — vacant), Building (significantly deteriorated — condemned)
Overall risk trend: SIGNIFICANTLY DETERIORATED (prior: 5.5, current: 9.2, delta: +3.7)
Decision: NON_RENEW
Confidence: HIGH
Rationale: This property has deteriorated beyond carrier risk appetite. Building
condemned with no remediation plan, full vacancy, sprinkler decommissioned, and
3 fire losses in the past year indicate an unacceptable risk. All 4 risk
categories show significant deterioration. Recommend non-renewal with
appropriate notice per Mississippi state requirements (60 days).

Now classify:
Property: {property_summary}
Changes detected: {change_summary}
Risk re-evaluation: {risk_reevaluation_summary}
Overall risk trend: {overall_risk_trend}
Decision:
```

**Example (User Profiling — Analyst Classification):**
```
Based on the underwriter's messages during the review session, classify their
experience level as JUNIOR or SENIOR. Consider: terminology usage, questions
asked, comfort with data interpretation, and portfolio management experience.

Example 1:
Underwriter: "Hi, I'm Aisha. I just got this portfolio to review and I'm not
sure where to start. Can you walk me through what all these changes mean?"
Classification: JUNIOR
Reasoning: Asks for guidance on where to start, uncertain about interpreting
changes, requests walkthrough.

Example 2:
Underwriter: "Pull up the exceptions on the Southeast Retail portfolio. I need
to see which properties flagged for NatCat zone reclassification since last
year's renewal."
Classification: SENIOR
Reasoning: Knows to ask for exceptions, uses specific risk terminology, targets
a specific change category, expects concise data.

Example 3:
Underwriter: "I have a portfolio to review for renewal. There are a few properties
where I think the risk might have changed but I'm not really sure how to tell
which changes are important versus which ones I can ignore."
Classification: JUNIOR
Reasoning: Uncertain about change materiality, asks indirectly for guidance
on prioritization.

Example 4:
Underwriter: "Show me the delta summary. I want to see the properties sorted by
risk increase — anything with more than +1.0 delta gets my direct review,
the rest I'll approve as-is."
Classification: SENIOR
Reasoning: Uses quantitative thresholds, efficient review strategy, comfortable
making batch approve decisions, uses "delta" terminology.

Now classify:
Underwriter: "{underwriter_messages}"
Classification:
```

#### Type 3: Chain-of-Thought (CoT) Prompting
**Used for:** Complex reasoning where transparency and step-by-step logic are critical — the core of risk re-evaluation. This project's CoT is unique because it reasons about **changes over time** (current vs. prior year), not just point-in-time assessment.

**Where in this project:**
- All 4 risk category re-evaluations (Fire Risk Change, NatCat Change, Occupancy Change, Building Condition Change)
- Change impact analysis (assessing how a detected change affects risk)
- Overall risk trend synthesis (combining 4 category deltas)

**Example (Fire Risk Re-evaluation — CoT with Current vs. Prior Year Comparison):**
```
You are an expert property fire risk re-evaluation specialist. Compare the
CURRENT YEAR and PRIOR YEAR fire risk data for this property and determine
how fire risk has changed.

PROPERTY: {property_name} — {property_address}

PRIOR YEAR FIRE DATA:
- Construction Type: {prior_construction_type}
- Sprinkler System: {prior_sprinkler_status}
- Fire Alarm: {prior_alarm_status}
- Fire Protection Class: {prior_protection_class}
- Cooking Operations: {prior_cooking_operations}
- Electrical System Age: {prior_electrical_age} years
- Distance to Fire Station: {prior_fire_station_distance} miles

CURRENT YEAR FIRE DATA:
- Construction Type: {current_construction_type}
- Sprinkler System: {current_sprinkler_status}
- Fire Alarm: {current_alarm_status}
- Fire Protection Class: {current_protection_class}
- Cooking Operations: {current_cooking_operations}
- Electrical System Age: {current_electrical_age} years
- Distance to Fire Station: {current_fire_station_distance} miles

DETECTED CHANGES (from Change Detection Agent):
{detected_fire_changes}

Think step by step to re-evaluate fire risk:

Step 1: Score the PRIOR YEAR fire risk (1-10).
  Evaluate each factor as it was last year.
  Consider construction type, protection systems, occupancy risk, response capability.

Step 2: Score the CURRENT YEAR fire risk (1-10).
  Evaluate each factor as it is now.
  Apply the same criteria used in Step 1 for consistency.

Step 3: Calculate the delta.
  Delta = Current Score - Prior Score
  If delta < 0: IMPROVED (risk decreased)
  If delta = 0: STABLE (no change)
  If delta > 0: DETERIORATED (risk increased)

Step 4: Assess the delta magnitude.
  |delta| < 1: Minor change (likely cosmetic or seasonal)
  |delta| 1-2: Moderate change (worth noting in review)
  |delta| > 2: Significant change (drives renewal decision)

Step 5: Identify the primary driver(s) of the change.
  Which specific changes between prior and current year caused the delta?
  Are these changes reversible or permanent?

Provide your output as:
- Prior Year Fire Risk Score: [1-10]
- Current Year Fire Risk Score: [1-10]
- Fire Risk Delta: [number, + or -]
- Delta Direction: [IMPROVED / STABLE / DETERIORATED]
- Delta Magnitude: [MINOR / MODERATE / SIGNIFICANT]
- Primary Change Drivers: [list]
- Step-by-Step Reasoning: [your analysis from Steps 1-5]
```

**Example (Change Impact Analysis — CoT):**
```
You are a property risk analyst. A material change has been detected for
a property in a renewal portfolio. Analyze the impact of this change on
the property's overall risk profile.

PROPERTY: {property_name}
CHANGE DETECTED: {change_description}
  - Field: {field_name}
  - Prior Value: {prior_value}
  - Current Value: {current_value}
  - Change Severity: MATERIAL

Think step by step:

Step 1: What risk category does this change primarily affect?
  (Fire Risk, NatCat Risk, Occupancy Risk, Building Condition)

Step 2: How does this specific change shift risk?
  - Does it increase or decrease exposure?
  - By how much (minor / moderate / significant)?
  - Is this change common in this property type?

Step 3: Are there secondary risk impacts?
  - Does this change affect other risk categories indirectly?
  - Example: A new restaurant tenant (occupancy change) also increases
    fire risk and may affect building condition through grease trap requirements.

Step 4: What is the expected renewal impact?
  - Does this change alone warrant a premium adjustment?
  - Should coverage terms be modified?
  - Is additional information or inspection needed?

Output:
- Primary Risk Category Affected: [category]
- Risk Direction: [INCREASE / DECREASE / NEUTRAL]
- Risk Magnitude: [MINOR / MODERATE / SIGNIFICANT]
- Secondary Impacts: [list or "none"]
- Recommended Renewal Action: [description]
- Reasoning: [step-by-step analysis]
```

#### Type 4: ReAct (Reasoning + Acting) Prompting
**Used for:** Tasks requiring the agent to reason about what data it needs, take actions (tool calls) to gather that information, then reason about the results — primarily in data collection and the conversational review interface.

**Where in this project:**
- Gathering updated property information during batch processing (calling mock data APIs)
- Conversational review tool calls (querying results, drilling into specific properties)
- Cross-property comparison queries during review

**Example (Updated Property Data Collection — ReAct):**
```
You are a property data collection agent for portfolio renewal processing.
Your job is to gather the most current data for a property that is up for
renewal, supplementing the data provided in the portfolio JSON.

You have the following tools:
1. lookup_current_hazard_zones(address) — Returns latest flood/wind/earthquake zones
2. lookup_inspection_status(property_id) — Returns recent inspection findings
3. lookup_loss_history(property_id, years=3) — Returns claims in last 3 years
4. lookup_building_permits(address) — Returns recent building permit activity

PROPERTY: {property_name} at {property_address}
PROPERTY ID: {property_id}
PORTFOLIO DATA (may be stale): {portfolio_property_data}

Think about what data needs to be refreshed for an accurate renewal review.

Thought: The portfolio JSON contains property data, but some fields may be
outdated since the portfolio was compiled. I should check current hazard
zones first — FEMA updates flood maps periodically, and a zone change is
a material renewal factor.
Action: lookup_current_hazard_zones("{property_address}")
Observation: {{hazard_zone_result}}

Thought: I should check if any inspections were conducted since the last
renewal. Inspection findings could reveal new code violations or
improvements.
Action: lookup_inspection_status("{property_id}")
Observation: {{inspection_result}}

Thought: Loss history is critical for renewal — new claims since last year
directly affect the renewal decision. Let me pull the latest 3-year history.
Action: lookup_loss_history("{property_id}", years=3)
Observation: {{loss_history_result}}

Thought: Building permits can reveal renovations (positive) or unpermitted
modifications (negative). Let me check for recent activity.
Action: lookup_building_permits("{property_address}")
Observation: {{permit_result}}

Thought: I now have current hazard zones, inspection status, loss history,
and permit activity. Let me compile the refreshed data package.

Final Output: Compile all gathered data into a structured data package:
{{"refreshed_property_data": {{
    "property_id": "{property_id}",
    "hazard_zones": {{...}},
    "inspection_findings": {{...}},
    "loss_history_3yr": {{...}},
    "building_permits": {{...}},
    "data_refresh_timestamp": "current datetime",
    "stale_fields_updated": [list of fields that changed from portfolio data]
  }}
}}
```

**Example (Review Chat — Drill-Down Query with ReAct):**
```
You are the Portfolio Orchestrator in conversational review mode. The
underwriter is asking about specific portfolio results. Use tools to
retrieve the relevant data.

PORTFOLIO: {portfolio_name} ({property_count} properties)
BATCH STATUS: Complete
USER PROFILE: {experience_level}

You have the following tools:
1. query_results(portfolio_id, query_type, property_id) — Query stored results
2. get_property_detail(property_id) — Get full detail for one property
3. compare_properties(property_ids) — Compare multiple properties side-by-side
4. get_portfolio_summary(portfolio_id) — Get aggregated portfolio summary

Underwriter query: {user_query}

Thought: The underwriter is asking "{user_query}". I need to determine what
data to retrieve and how to present it based on their experience level
({experience_level}).
Action: [appropriate tool call based on query intent]
Observation: {{result}}

Thought: [Reason about the result and format the response appropriately
for a {experience_level} user. Junior users get explanations, senior users
get data tables.]

Response: [Formatted answer]
```

### 10.2 Prompt Type Selection Matrix

This table maps every significant task in the system to its prompt type, with reasoning:

| # | Task | Agent | Prompt Type | Why |
|---|------|-------|-------------|-----|
| 1 | Portfolio JSON validation | Orchestrator | Zero-Shot | Rule-based field checking, no examples needed |
| 2 | Batch progress reporting | Orchestrator | Zero-Shot | Simple status messages |
| 3 | User profiling (analyst classification) | Orchestrator | Few-Shot | Examples ensure consistent JUNIOR/SENIOR classification |
| 4 | Portfolio summary narration | Orchestrator | Zero-Shot | Straightforward summarization of computed aggregates |
| 5 | Review chat — general query | Orchestrator | Zero-Shot | Contextual answers from pre-training knowledge |
| 6 | Review chat — drill-down query | Orchestrator | ReAct | Requires tool calls to retrieve specific property data |
| 7 | Review chat — property comparison | Orchestrator | ReAct | Requires tool calls to retrieve and compare multiple properties |
| 8 | Field-by-field data comparison | Change Detection | Zero-Shot | Straightforward diff of two data objects |
| 9 | Change severity classification | Change Detection | Few-Shot | Examples ensure consistent material/notable/cosmetic classification |
| 10 | Fire risk re-evaluation (current vs. prior) | Property Re-evaluation | Chain-of-Thought | Complex temporal comparison requires step-by-step delta analysis |
| 11 | NatCat risk re-evaluation (current vs. prior) | Property Re-evaluation | Chain-of-Thought | Complex temporal comparison with zone reclassification reasoning |
| 12 | Occupancy risk re-evaluation (current vs. prior) | Property Re-evaluation | Chain-of-Thought | Tenant change impact analysis needs transparent reasoning |
| 13 | Building condition re-evaluation (current vs. prior) | Property Re-evaluation | Chain-of-Thought | Code violation and aging analysis needs step-by-step logic |
| 14 | Overall risk trend synthesis | Property Re-evaluation | Chain-of-Thought | Combining 4 deltas into portfolio-level trend needs justification |
| 15 | Renewal decision classification | Renewal Recommendation | Few-Shot | 4 examples (one per category) ensure consistent classification |
| 16 | Adjustment recommendation generation | Renewal Recommendation | Zero-Shot | Generating suggested premium/coverage changes from risk data |
| 17 | Updated property data collection | Orchestrator (tool) | ReAct | Requires tool calls to refresh stale property data from APIs |

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
├── manager.py                        # Core PromptManager class
└── templates/
    ├── __init__.py
    ├── orchestrator.py               # Portfolio Orchestrator prompts (system, profiling, batch, review)
    ├── re_evaluation.py              # Property Re-evaluation prompts (CoT for each risk category delta)
    ├── change_detection.py           # Change Detection prompts (field comparison, severity classification)
    └── renewal_recommendation.py     # Renewal Recommendation prompts (few-shot classification)
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
            agent: Agent name (e.g., "orchestrator", "re_evaluation")
            name: Template name (e.g., "fire_risk_delta_cot")
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
# prompt_manager/templates/renewal_recommendation.py

# Each template is a Python string with {variable} placeholders.
# Multiple versions allow comparison and iteration.

RENEWAL_CLASSIFICATION_FEWSHOT_V1 = """
Based on the detected changes and risk re-evaluation results, classify the
renewal recommendation into one of 4 categories: RENEW_AS_IS,
RENEW_WITH_ADJUSTMENTS, REFER_TO_SENIOR, or NON_RENEW.

Example 1 — RENEW AS-IS:
Property: 200 Lakeview Office Park, Chicago IL — Class A office building
Changes: 0 material, 1 notable (minor TIV increase +5%), 1 cosmetic
Risk deltas: Fire (stable), NatCat (stable), Occupancy (stable), Building (improved)
Overall trend: STABLE (prior: 3.2, current: 3.0, delta: -0.2)
Decision: RENEW_AS_IS
Confidence: HIGH
Rationale: No material changes. Risk stable with slight improvement from HVAC
upgrade. Recommend renewal at current terms.

Example 2 — RENEW WITH ADJUSTMENTS:
Property: 450 Gulf Coast Retail Plaza, Tampa FL — Strip retail center
Changes: 2 material (flood zone AE→VE, new restaurant tenant), 1 notable
Risk deltas: Fire (deteriorated), NatCat (deteriorated), Occupancy (stable), Building (stable)
Overall trend: DETERIORATED (prior: 4.5, current: 6.2, delta: +1.7)
Decision: RENEW_WITH_ADJUSTMENTS
Confidence: HIGH
Rationale: Flood zone upgrade and restaurant tenant elevate risk. Recommend
premium increase 15-20%, add flood sublimit, require fire suppression cert.

Example 3 — REFER TO SENIOR:
Property: 800 Industrial Ave, Newark NJ — Mixed-use warehouse/office
Changes: 3 material (code violations, occupancy shift, hazmat storage), 2 notable
Risk deltas: Fire (deteriorated), NatCat (stable), Occupancy (significantly deteriorated), Building (deteriorated)
Overall trend: DETERIORATED (prior: 5.0, current: 7.8, delta: +2.8)
Decision: REFER_TO_SENIOR
Confidence: MEDIUM
Rationale: Multiple material changes across 3 categories create a complex risk
picture requiring senior underwriter judgment.

Example 4 — NON-RENEW:
Property: 125 Riverside Warehouse, Biloxi MS — Vacant industrial warehouse
Changes: 4 material (condemned, full vacancy, sprinkler decommissioned, 3 fire losses)
Risk deltas: Fire (significantly deteriorated), NatCat (deteriorated), Occupancy (significantly deteriorated), Building (significantly deteriorated)
Overall trend: SIGNIFICANTLY DETERIORATED (prior: 5.5, current: 9.2, delta: +3.7)
Decision: NON_RENEW
Confidence: HIGH
Rationale: Property deteriorated beyond carrier appetite across all 4 categories.
Recommend non-renewal with appropriate statutory notice.

Now classify:
Property: {property_summary}
Changes: {change_summary}
Risk deltas: {risk_delta_summary}
Overall trend: {overall_risk_trend}
Decision:
"""

# Version 2: Tightened scoring thresholds (for A/B testing in Arize)
RENEWAL_CLASSIFICATION_FEWSHOT_V2 = """
... (refined prompt with stricter delta thresholds and additional edge-case examples) ...
"""


def register_all(manager):
    """Register all renewal recommendation templates."""
    manager.register("renewal_recommendation", "renewal_classification_fewshot", "v1", RENEWAL_CLASSIFICATION_FEWSHOT_V1)
    manager.register("renewal_recommendation", "renewal_classification_fewshot", "v2", RENEWAL_CLASSIFICATION_FEWSHOT_V2)
    # ... register adjustment_generation, rationale_generation, etc.
```

### 11.3.2 Usage in Agent Code

```python
# Inside the Renewal Recommendation Agent:
prompt = prompt_manager.get_prompt(
    agent="renewal_recommendation",
    name="renewal_classification_fewshot",
    variables={
        "property_summary": property_data["summary"],
        "change_summary": change_results["summary"],
        "risk_delta_summary": reevaluation_results["delta_summary"],
        "overall_risk_trend": reevaluation_results["overall_trend"],
    },
    version="v1"  # or "latest"
)

response = await model.generate_content(prompt)
```

### 11.4 Full CoT Template Example

The Fire Risk Re-evaluation CoT template demonstrates the Chain-of-Thought pattern unique to this project — reasoning about **changes over time** by comparing current-year vs. prior-year data:

```python
# prompt_manager/templates/re_evaluation.py

FIRE_RISK_DELTA_COT_V1 = """
You are an expert property fire risk re-evaluation specialist. Compare the
CURRENT YEAR and PRIOR YEAR fire risk data for this property and determine
how fire risk has changed.

PROPERTY: {property_name} — {property_address}

PRIOR YEAR FIRE DATA:
- Construction Type: {prior_construction_type}
- Sprinkler System: {prior_sprinkler_status}
- Fire Alarm: {prior_alarm_status}
- Fire Protection Class: {prior_protection_class}
- Cooking Operations: {prior_cooking_operations}
- Electrical System Age: {prior_electrical_age} years
- Distance to Fire Station: {prior_fire_station_distance} miles

CURRENT YEAR FIRE DATA:
- Construction Type: {current_construction_type}
- Sprinkler System: {current_sprinkler_status}
- Fire Alarm: {current_alarm_status}
- Fire Protection Class: {current_protection_class}
- Cooking Operations: {current_cooking_operations}
- Electrical System Age: {current_electrical_age} years
- Distance to Fire Station: {current_fire_station_distance} miles

DETECTED CHANGES (from Change Detection Agent):
{detected_fire_changes}

Think step by step to re-evaluate fire risk:

Step 1: Score the PRIOR YEAR fire risk (1-10).
  Evaluate each factor as it was last year.
  Consider construction type, protection systems, occupancy risk, response capability.

Step 2: Score the CURRENT YEAR fire risk (1-10).
  Evaluate each factor as it is now.
  Apply the same criteria used in Step 1 for consistency.

Step 3: Calculate the delta.
  Delta = Current Score - Prior Score
  If delta < 0: IMPROVED (risk decreased)
  If delta = 0: STABLE (no change)
  If delta > 0: DETERIORATED (risk increased)

Step 4: Assess the delta magnitude.
  |delta| < 1: Minor change (likely cosmetic or seasonal)
  |delta| 1-2: Moderate change (worth noting in review)
  |delta| > 2: Significant change (drives renewal decision)

Step 5: Identify the primary driver(s) of the change.
  Which specific changes between prior and current year caused the delta?
  Are these changes reversible or permanent?

Output JSON:
{{"category": "fire", "prior_score": <1-10>, "current_score": <1-10>,
  "delta": <number>, "direction": "<IMPROVED|STABLE|DETERIORATED>",
  "magnitude": "<MINOR|MODERATE|SIGNIFICANT>",
  "primary_drivers": [...], "reasoning": "<step-by-step>"}}
"""


def register_all(manager):
    manager.register("re_evaluation", "fire_risk_delta_cot", "v1", FIRE_RISK_DELTA_COT_V1)
    # ... register natcat_risk_delta_cot, occupancy_risk_delta_cot,
    #     building_condition_delta_cot, overall_trend_synthesis_cot
```

> **Tip for the intern:** Start by writing the fire risk delta CoT prompt, test it in Gemini with one property's prior + current year data, and iterate until the output is consistent. Then copy the pattern for NatCat, Occupancy, and Building Condition. Each category follows the same 5-step structure — only the domain-specific factors change.

---

## 12. User Profiling Design

### 12.1 What Is User Profiling?

The system builds a profile of each underwriter through conversation and adapts its behavior to match their expertise, preferences, and working style. A junior portfolio analyst who's new to renewal reviews gets a guided property-by-property walkthrough with explanations, while a senior underwriting manager gets an exception-only summary with drill-down on demand.

### 12.2 Profile Attributes

Stored as a simple JSON file at `data/profiles/{user_id}.json`:

```json
{
  "user_id": "analyst_001",
  "name": "Aisha Patel",
  "experience_level": "junior",
  "portfolio_segments": ["offices", "retail"],
  "preferred_detail_level": "verbose",
  "preferred_review_mode": "full-walkthrough",
  "session_count": 2,
  "last_interaction": "2026-04-18T10:30:00Z",
  "profiling_complete": true
}
```

> **Why JSON files instead of SQLite?** For a 7-day project, JSON files eliminate database setup time while still demonstrating persistence. Each user gets their own file. Simple to read, write, and debug.

### 12.3 Profiling Questions (Asked in First Session)

The Portfolio Orchestrator asks these questions **naturally in conversation** during the first review interaction (after batch processing completes):

| # | Question | Purpose | Maps To |
|---|----------|---------|---------|
| Q1 | "Before we review the results — how long have you been handling property portfolio renewals?" | Determine experience level | `experience_level` |
| Q2 | "Do you specialize in particular property segments — like offices, retail, industrial, or mixed-use?" | Identify portfolio segment expertise | `portfolio_segments` |
| Q3 | "When reviewing a portfolio, do you prefer a detailed walkthrough of every property, or would you rather see only the exceptions and flagged items?" | Set review mode preference | `preferred_review_mode` |
| Q4 | "For the portfolio summary, do you prefer a narrative write-up or data tables with key metrics?" | Set detail level and format | `preferred_detail_level` |

**Key Design Principle:** Questions are asked conversationally, not as a form. The agent weaves them into the natural flow after batch processing completes:

```
Agent: "Portfolio 'Midwest Office Portfolio' — 6 properties reviewed.
        Summary: 4 Renew As-Is, 1 Renew with Adjustments, 1 Refer to Senior.
        Batch completed in 12 seconds.

        Before we dive into the results — how long have you been handling
        property portfolio renewals?"

User:  "I just started doing renewals this quarter. Mostly been on new
        business submissions until now."

Agent: "Welcome to the renewal side! There's a lot in common, but the big
        difference is we're comparing this year vs. last year to spot changes.
        I'll walk you through each property step by step. Do you mainly
        handle certain property types — offices, retail, industrial?"

User:  "Mostly offices so far, but this portfolio has some retail too."

Agent: "Perfect — I'll flag any retail-specific nuances as we go.
        Let's start with the two properties that need attention..."
```

### 12.4 Adaptive Behavior Based on Profile

| Behavior | Junior Mode (Aisha) | Senior Mode (Robert) |
|----------|---------------------|----------------------|
| **Greeting** | Warm, explains the renewal review process | Brief, gets to exceptions immediately |
| **Portfolio review mode** | Guided property-by-property walkthrough starting with flagged items, then stable properties | Exceptions-only summary; stable properties skipped unless asked |
| **Change explanations** | Defines terms ("A flood zone reclassification from AE to VE means..."), explains why changes matter | Shows change data only — assumes familiarity with zone codes and risk impacts |
| **Recommendations** | Detailed rationale with step-by-step reasoning for each renewal decision | Brief decision + top driver only; drill-down available on request |
| **Report depth** | Narrative summaries with context for each property | Tabular view — property, decision, delta, top change, premium impact |
| **Drill-down** | Agent proactively suggests "Would you like me to explain the risk delta for this property?" | Agent waits for explicit drill-down request |

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
                "portfolio_segments": [],
                "preferred_detail_level": "verbose",
                "preferred_review_mode": "full-walkthrough",
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
| **Traces** | Every LLM call, tool invocation, agent-to-agent delegation; spans tagged with `property_id`, `portfolio_id` | Debug failures, understand per-property and batch-level latency |
| **Evaluations** | Change detection completeness, renewal classification accuracy, risk delta consistency across properties | Ensure output quality across the portfolio |
| **Annotations** | Human review of renewal decisions — was the AI classification correct? | Build golden dataset, improve few-shot examples and CoT prompts |
| **Metrics** | Per-property processing time, batch throughput (properties/minute), risk delta distribution, renewal decision breakdown, token usage per portfolio | Operational monitoring, batch performance tuning, cost control |

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
        model_id="property-portfolio-renewal",             # Project name in Arize
        model_version="1.0.0",
    )

    print("Arize tracing initialized successfully.")
```

#### Step 3: Instrument Agent Calls with Portfolio-Specific Spans

```python
# In each agent, wrap LLM calls with OpenTelemetry spans

from opentelemetry import trace

tracer = trace.get_tracer("portfolio-renewal-agents")

class PropertyReEvaluationAgent:
    async def reevaluate_risk(self, property_id: str, current_data: dict, prior_data: dict, changes: dict) -> dict:
        with tracer.start_as_current_span("property_reevaluation") as span:
            # Add portfolio-specific attributes for Arize
            span.set_attribute("property_id", property_id)
            span.set_attribute("portfolio_id", current_data.get("portfolio_id", ""))
            span.set_attribute("property_type", current_data.get("property_type", ""))
            span.set_attribute("material_changes_count", changes.get("material_count", 0))

            # Get prompt from Prompt Manager
            prompt = self.prompt_manager.get_prompt(
                agent="re_evaluation",
                name="fire_risk_delta_cot",
                variables=self._build_fire_variables(current_data, prior_data, changes)
            )

            # Track prompt version
            span.set_attribute("prompt_template", "fire_risk_delta_cot")
            span.set_attribute("prompt_version", "v1")

            # Call Gemini
            response = await self.model.generate_content(prompt)

            # Track portfolio-specific output attributes
            span.set_attribute("risk_delta", result["delta"])
            span.set_attribute("risk_direction", result["direction"])
            span.set_attribute("renewal_decision", result.get("renewal_decision", ""))
            span.set_attribute("change_severity", changes.get("overall_severity", ""))

            return result
```

#### Step 4: Set Up Annotation Queue

In the Arize dashboard:

1. **Create a new Space** → "Property Portfolio Renewal"
2. **Go to Annotations** → Create new annotation queue
3. **Name:** "Renewal Decision Quality Review"
4. **Filter:** traces where `renewal_decision` is set
5. **Annotation Schema:**
   - `decision_accurate` (boolean) — Is the AI renewal decision correct?
   - `change_detection_complete` (boolean) — Did the agent catch all material changes?
   - `risk_delta_reasonable` (boolean) — Is the risk delta score reasonable?
   - `reasoning_quality` (1-5) — How good is the CoT reasoning?
   - `corrected_decision` (string, optional) — What should the renewal decision be?
   - `notes` (text) — Reviewer comments

#### Step 5: Build Monitoring Dashboard

In Arize, create dashboard widgets for:

| Widget | Metric | Purpose |
|--------|--------|---------|
| Batch processing time | Total seconds per portfolio | End-to-end performance |
| Per-property latency | p50, p95 per-property agent response time | Identify slow properties |
| Risk delta distribution | Histogram of risk deltas across properties | Spot outlier risk shifts |
| Renewal decision breakdown | Pie chart: renew-as-is / adjust / refer / non-renew | Classification distribution monitoring |
| Material changes per property | Distribution of material change counts | Change detection thoroughness |
| Token usage per portfolio | Input + output tokens per batch run | Cost tracking |
| Error rate | % of failed per-property pipelines | Reliability |
| Annotation agreement | Human vs. AI renewal decision match rate | Quality tracking |
| Prompt version comparison | Performance by prompt version | A/B testing prompts |

### 13.3 Arize Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       PORTFOLIO BATCH PROCESSING                            │
│                                                                             │
│  ┌──────────────────┐                                                       │
│  │ Portfolio         │                                                       │
│  │ Orchestrator      │──── fan-out N properties ──────────────────────┐     │
│  │ [OTel Spans:      │                                                │     │
│  │  portfolio_id,    │     ┌────────────┐  ┌────────────┐  ┌────────┐│     │
│  │  batch_size,      │     │ Property 1 │  │ Property 2 │  │Prop. N ││     │
│  │  batch_duration]  │     │            │  │            │  │        ││     │
│  └──────────────────┘     │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────┐ ││     │
│                            │ │Change  │ │  │ │Change  │ │  │ │ CD │ ││     │
│                            │ │Detect. │ │  │ │Detect. │ │  │ │    │ ││     │
│                            │ │[Spans] │ │  │ │[Spans] │ │  │ │    │ ││     │
│                            │ └───┬────┘ │  │ └───┬────┘ │  │ └──┬─┘ ││     │
│                            │     ▼      │  │     ▼      │  │    ▼   ││     │
│                            │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────┐ ││     │
│                            │ │Re-eval │ │  │ │Re-eval │ │  │ │ RE │ ││     │
│                            │ │[Spans] │ │  │ │[Spans] │ │  │ │    │ ││     │
│                            │ └───┬────┘ │  │ └───┬────┘ │  │ └──┬─┘ ││     │
│                            │     ▼      │  │     ▼      │  │    ▼   ││     │
│                            │ ┌────────┐ │  │ ┌────────┐ │  │ ┌────┐ ││     │
│                            │ │Renewal │ │  │ │Renewal │ │  │ │ RR │ ││     │
│                            │ │Recomm. │ │  │ │Recomm. │ │  │ │    │ ││     │
│                            │ │[Spans] │ │  │ │[Spans] │ │  │ │    │ ││     │
│                            │ └────────┘ │  │ └────────┘ │  │ └────┘ ││     │
│                            └────────────┘  └────────────┘  └────────┘│     │
│                                                                       │     │
│  Each property pipeline creates parallel spans with property_id ──────┘     │
│                                                                             │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                          ┌────────▼─────────────────┐
                          │    OpenTelemetry          │
                          │    Collector              │
                          │                           │
                          │  Spans tagged with:       │
                          │  - portfolio_id           │
                          │  - property_id            │
                          │  - risk_delta             │
                          │  - renewal_decision       │
                          │  - change_severity        │
                          │  - prompt_version         │
                          └────────────┬──────────────┘
                                       │
                          ┌────────────▼──────────────┐
                          │                           │
                          │      Arize Cloud          │
                          │                           │
                          │  ┌─────────────────────┐  │
                          │  │ Traces (per-property │  │
                          │  │   + batch-level)     │  │
                          │  │ Dashboard            │  │
                          │  │ Annotations          │  │
                          │  │ Evals                │  │
                          │  └─────────────────────┘  │
                          │                           │
                          └───────────────────────────┘
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
    display_name="property-portfolio-renewal",
    description="Multi-agent system for property portfolio renewal and risk re-evaluation with batch processing and conversational review",
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
    query="Process the Midwest Office Portfolio for renewal. File: data/portfolios/midwest_office.json"
)
print(response)
```

> **Batch processing consideration:** When deployed to Agent Engine, the parallel fan-out (`asyncio.gather` for N properties) runs server-side. Ensure the Agent Engine timeout is set long enough for the largest portfolio — a 20-property portfolio with 3 agents per property may take 30-60 seconds total.

### 14.2 Option B: Local Deployment (Fallback)

If GCP access is not available:

```bash
# Run agents locally using ADK CLI
# Terminal 1: Change Detection Agent
adk run change_detection_agent --port 8002

# Terminal 2: Property Re-evaluation Agent
adk run property_reevaluation_agent --port 8003

# Terminal 3: Renewal Recommendation Agent
adk run renewal_recommendation_agent --port 8004

# Terminal 4: Portfolio Orchestrator Agent (main)
adk web portfolio_orchestrator --port 8001
```

The ADK web interface provides a chat UI for both batch triggering ("Process the Midwest Office Portfolio") and conversational review.

### 14.3 Environment Setup Guide

```bash
# 1. Clone the project
git clone <repo_url>
cd property-portfolio-renewal

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
adk web portfolio_orchestrator
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

> **Note:** No Streamlit, no Jinja2, no SQLite driver needed. We use ADK's built-in web UI, Python `str.format_map` for templates, and JSON files for storage. This keeps the dependency list lean and setup fast.

---

## 15. Data Models

### 15.1 Portfolio Input Schema

```json
{
  "portfolio_id": "PF-MIDWEST-001",
  "portfolio_name": "Midwest Office Portfolio",
  "account_name": "Heartland Commercial Properties LLC",
  "renewal_date": "2026-07-01",
  "broker": "Marsh McLennan — Chicago",
  "current_premium": 485000,
  "properties": [
    {
      "property_id": "PROP-MW-001",
      "property_name": "200 Lakeview Office Park",
      "address": "200 N Lakeview Ave, Chicago, IL 60601",
      "property_type": "office",
      "total_insured_value": 12500000,
      "current_year": { "...see Section 15.2..." },
      "prior_year": { "...see Section 15.2..." }
    },
    {
      "property_id": "PROP-MW-002",
      "property_name": "550 Wacker Drive Tower",
      "address": "550 W Wacker Dr, Chicago, IL 60606",
      "property_type": "office",
      "total_insured_value": 28000000,
      "current_year": { "..." },
      "prior_year": { "..." }
    }
  ]
}
```

### 15.2 Property Data Schema

Each property contains both `current_year` and `prior_year` snapshots with identical field structures, enabling field-by-field comparison by the Change Detection Agent:

```json
{
  "current_year": {
    "construction": {
      "construction_type": "fire_resistive",
      "construction_class": 6,
      "year_built": 1998,
      "stories": 12,
      "total_sqft": 185000,
      "roof_type": "built_up",
      "roof_age_years": 8,
      "last_renovation_year": 2023
    },
    "hazards": {
      "flood_zone": "X",
      "earthquake_zone": "low",
      "windstorm_zone": "moderate",
      "brush_fire_exposure": false,
      "hazardous_materials": false,
      "hazmat_details": null
    },
    "occupancy": {
      "occupancy_type": "professional_office",
      "primary_tenant": "Midwest Financial Group",
      "vacancy_rate_pct": 5,
      "cooking_operations": false,
      "public_assembly": false,
      "max_occupant_load": 1200
    },
    "protection": {
      "sprinkler_status": "full_wet",
      "alarm_status": "central_station",
      "fire_protection_class": 3,
      "security_system": true,
      "fire_extinguishers": true,
      "distance_to_fire_station_miles": 1.2
    },
    "loss": {
      "claims_last_3yr": 0,
      "total_incurred_last_3yr": 0,
      "largest_single_loss": 0,
      "loss_ratio_3yr": 0.0
    }
  },
  "prior_year": {
    "construction": {
      "construction_type": "fire_resistive",
      "construction_class": 6,
      "year_built": 1998,
      "stories": 12,
      "total_sqft": 185000,
      "roof_type": "built_up",
      "roof_age_years": 7,
      "last_renovation_year": 2021
    },
    "hazards": {
      "flood_zone": "X",
      "earthquake_zone": "low",
      "windstorm_zone": "moderate",
      "brush_fire_exposure": false,
      "hazardous_materials": false,
      "hazmat_details": null
    },
    "occupancy": {
      "occupancy_type": "professional_office",
      "primary_tenant": "Midwest Financial Group",
      "vacancy_rate_pct": 8,
      "cooking_operations": false,
      "public_assembly": false,
      "max_occupant_load": 1200
    },
    "protection": {
      "sprinkler_status": "full_wet",
      "alarm_status": "central_station",
      "fire_protection_class": 3,
      "security_system": true,
      "fire_extinguishers": true,
      "distance_to_fire_station_miles": 1.2
    },
    "loss": {
      "claims_last_3yr": 0,
      "total_incurred_last_3yr": 0,
      "largest_single_loss": 0,
      "loss_ratio_3yr": 0.0
    }
  }
}
```

### 15.3 Change Detection Output Schema

```json
{
  "property_id": "PROP-SE-003",
  "property_name": "450 Gulf Coast Retail Plaza",
  "detection_timestamp": "2026-04-18T14:35:00Z",
  "changes": [
    {
      "field": "hazards.flood_zone",
      "prior_value": "AE",
      "current_value": "VE",
      "change_severity": "material",
      "risk_impact": "NatCat risk increased — VE zone indicates coastal high-velocity flood hazard, significantly higher than AE riverine flood zone",
      "risk_category_affected": "natcat"
    },
    {
      "field": "occupancy.occupancy_type",
      "prior_value": "general_retail",
      "current_value": "restaurant",
      "change_severity": "material",
      "risk_impact": "Fire risk increased — restaurant cooking operations introduce grease fire and kitchen exhaust hazards",
      "risk_category_affected": "fire"
    },
    {
      "field": "occupancy.cooking_operations",
      "prior_value": false,
      "current_value": true,
      "change_severity": "material",
      "risk_impact": "Fire risk increased — cooking operations are a leading cause of commercial property fires",
      "risk_category_affected": "fire"
    },
    {
      "field": "construction.roof_age_years",
      "prior_value": 12,
      "current_value": 13,
      "change_severity": "cosmetic",
      "risk_impact": "Routine aging, no material risk change at this age",
      "risk_category_affected": "building"
    }
  ],
  "summary": {
    "total_changes": 4,
    "material_count": 3,
    "notable_count": 0,
    "cosmetic_count": 1,
    "overall_change_assessment": "Significant changes detected — multiple material changes affecting fire and NatCat risk categories"
  }
}
```

### 15.4 Renewal Assessment Output Schema

```json
{
  "property_id": "PROP-SE-003",
  "property_name": "450 Gulf Coast Retail Plaza",
  "assessment_timestamp": "2026-04-18T14:36:00Z",
  "risk_scores": {
    "fire": {
      "prior_score": 4.0,
      "current_score": 6.5,
      "delta": 2.5,
      "direction": "DETERIORATED",
      "magnitude": "SIGNIFICANT",
      "primary_drivers": ["New restaurant tenant with cooking operations", "Grease fire and kitchen exhaust hazards"]
    },
    "natcat": {
      "prior_score": 5.0,
      "current_score": 7.0,
      "delta": 2.0,
      "direction": "DETERIORATED",
      "magnitude": "MODERATE",
      "primary_drivers": ["Flood zone reclassified AE → VE (coastal high-velocity)"]
    },
    "occupancy": {
      "prior_score": 3.5,
      "current_score": 4.0,
      "delta": 0.5,
      "direction": "STABLE",
      "magnitude": "MINOR",
      "primary_drivers": ["Tenant type change — general retail to restaurant"]
    },
    "building": {
      "prior_score": 4.0,
      "current_score": 4.0,
      "delta": 0.0,
      "direction": "STABLE",
      "magnitude": "MINOR",
      "primary_drivers": []
    }
  },
  "overall_risk_trend": "DETERIORATED",
  "overall_prior_score": 4.1,
  "overall_current_score": 5.4,
  "overall_delta": 1.3,
  "renewal_decision": "RENEW_WITH_ADJUSTMENTS",
  "renewal_confidence": "HIGH",
  "premium_adjustment_pct": 18,
  "renewal_conditions": [
    "Premium increase of 15-20% to reflect increased fire and NatCat exposure",
    "Add flood sublimit specific to VE zone exposure",
    "Require restaurant tenant fire suppression hood certification",
    "Require annual kitchen exhaust cleaning documentation"
  ],
  "rationale": "Two material changes directly impact risk: flood zone upgrade from AE to VE increases NatCat exposure significantly, and new restaurant tenant with cooking operations elevates fire risk. Building condition and overall occupancy remain stable. Recommend renewal with premium adjustment and additional conditions.",
  "model_info": {
    "model": "gemini-2.5-pro",
    "prompt_versions": {
      "fire_risk_delta_cot": "v1",
      "natcat_risk_delta_cot": "v1",
      "occupancy_risk_delta_cot": "v1",
      "building_condition_delta_cot": "v1",
      "renewal_classification_fewshot": "v1"
    },
    "total_tokens": 12400,
    "latency_seconds": 7.8
  }
}
```

### 15.5 Portfolio Summary Schema

```json
{
  "portfolio_id": "PF-SOUTHEAST-001",
  "portfolio_name": "Southeast Retail Portfolio",
  "summary_timestamp": "2026-04-18T14:40:00Z",
  "total_properties": 5,
  "batch_duration_seconds": 18.5,
  "risk_distribution": {
    "prior_year_avg_score": 4.3,
    "current_year_avg_score": 5.1,
    "portfolio_delta": 0.8,
    "portfolio_trend": "DETERIORATED",
    "score_range": {"min": 3.0, "max": 7.8}
  },
  "renewal_decisions_summary": {
    "renew_as_is": {"count": 2, "pct": 40},
    "renew_with_adjustments": {"count": 2, "pct": 40},
    "refer_to_senior": {"count": 1, "pct": 20},
    "non_renew": {"count": 0, "pct": 0}
  },
  "total_premium_impact": {
    "current_portfolio_premium": 320000,
    "estimated_adjusted_premium": 358000,
    "net_change_pct": 11.9,
    "properties_with_increase": 2,
    "properties_with_decrease": 0,
    "properties_unchanged": 3
  },
  "flagged_properties": [
    {
      "property_id": "PROP-SE-003",
      "property_name": "450 Gulf Coast Retail Plaza",
      "decision": "RENEW_WITH_ADJUSTMENTS",
      "delta": 1.3,
      "material_changes": 3,
      "top_change": "Flood zone reclassified AE → VE"
    },
    {
      "property_id": "PROP-SE-005",
      "property_name": "800 Industrial Ave",
      "decision": "REFER_TO_SENIOR",
      "delta": 2.8,
      "material_changes": 3,
      "top_change": "Occupancy shift to light manufacturing with hazmat"
    }
  ],
  "material_changes_total": 8,
  "properties_with_material_changes": 3,
  "properties_stable": 2
}
```

### 15.6 Mock Data Strategy

Since this is a demo, all data is provided as **static JSON files** in the `data/portfolios/` directory. The intern creates 3 diverse portfolios that exercise different renewal scenarios:

| # | Portfolio | Properties | Risk Profile | Purpose |
|---|-----------|-----------|--------------|---------|
| 1 | **Midwest Office Portfolio** | 6 properties | Stable, low-risk — mostly Class A offices with minor changes | Demonstrates "Renew As-Is" majority path; 1-2 exceptions to keep it interesting |
| 2 | **Southeast Retail Portfolio** | 5 properties | Hurricane zone reclassifications, tenant turnover | Demonstrates NatCat-driven adjustments and flood zone material changes |
| 3 | **Northeast Mixed Portfolio** | 7 properties | Aging buildings, code violations, occupancy shifts | Demonstrates building condition deterioration, "Refer to Senior" and "Non-Renew" paths |

```python
# mock_apis/portfolio_loader.py

import json
import os

PORTFOLIO_DIR = "data/portfolios"

PORTFOLIO_INDEX = {
    "PF-MIDWEST-001": "midwest_office.json",
    "PF-SOUTHEAST-001": "southeast_retail.json",
    "PF-NORTHEAST-001": "northeast_mixed.json",
}


def load_portfolio(portfolio_id: str) -> dict:
    """Load a portfolio JSON file by ID."""
    filename = PORTFOLIO_INDEX.get(portfolio_id)
    if not filename:
        return {"error": f"Portfolio '{portfolio_id}' not found. Available: {list(PORTFOLIO_INDEX.keys())}"}

    path = os.path.join(PORTFOLIO_DIR, filename)
    if not os.path.exists(path):
        return {"error": f"Portfolio file not found: {path}"}

    with open(path) as f:
        portfolio = json.load(f)

    return portfolio


def list_portfolios() -> list[dict]:
    """List all available portfolios with basic metadata."""
    portfolios = []
    for pid, filename in PORTFOLIO_INDEX.items():
        path = os.path.join(PORTFOLIO_DIR, filename)
        if os.path.exists(path):
            with open(path) as f:
                data = json.load(f)
            portfolios.append({
                "portfolio_id": pid,
                "name": data.get("portfolio_name", ""),
                "property_count": len(data.get("properties", [])),
                "renewal_date": data.get("renewal_date", ""),
            })
    return portfolios
```

---

## 16. Tech Stack & Tools

### 16.1 Complete Tech Stack

| Category | Technology | Version | Purpose |
|----------|-----------|---------|---------|
| **Language** | Python | 3.11+ | Primary development language |
| **Agent Framework** | Google ADK | Latest | Agent development, orchestration, sub-agent parallel fan-out |
| **LLM** | Gemini 2.5 Pro | Latest | Portfolio orchestration, risk re-evaluation (complex CoT reasoning) |
| **LLM** | Gemini 2.5 Flash | Latest | Change detection, renewal classification (speed-critical batch tasks) |
| **Agent Communication** | A2A SDK | Latest | Inter-agent communication; Agent Cards for all 4 agents |
| **UI** | **ADK Web** (built-in) | Latest | Chat UI via `adk web` for batch trigger + conversational review — zero build needed |
| **Prompt Templates** | Python strings (`str.format_map`) | N/A | Simple, no extra dependency |
| **Storage** | JSON files | Built-in | Portfolios, per-property results, user profiles, portfolio summaries |
| **Monitoring** | Arize (Cloud) | Latest | Tracing, annotation, evaluation — per-property and batch-level |
| **Tracing** | OpenTelemetry | 1.20+ | Distributed tracing with parallel span support for batch processing |
| **Data Validation** | Pydantic | 2.0+ | Data model validation for portfolio and property schemas |
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
│ Setup  │───►│Change  │───►│Property│───►│ Orch + │───►│ A2A +  │───►│ Arize  │───►│ Demo   │───►│Stretch │
│ + Mock │    │Detect  │    │Re-eval │    │Renewal │    │Profile │    │+Deploy │    │ Prep   │    │ Goals  │
│Portfol.│    │ Agent  │    │ Agent  │    │+ Batch │    │+ Chat  │    │        │    │        │    │        │
└────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘
 Foundation    1st Agent      2nd Agent     Wire it up    A2A + UX     Observe      Ship it!      Polish
```

### Day 1: Foundation + Mock Portfolio Data
**Goal:** Project scaffolded, Gemini connected, 3 mock portfolios ready (each with current-year and prior-year data).

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 1.1 | Set up Python project | Create venv, install `google-adk`, `google-genai`, `python-dotenv` | `pip list` shows packages |
| 1.2 | Get Gemini API key | Create key at ai.google.dev, save in `.env` | Test call returns response |
| 1.3 | Hello World agent | Create a minimal ADK agent, run with `adk web`, chat with it | Agent responds in browser |
| 1.4 | Create project structure | Set up folders: `agents/`, `prompt_manager/`, `mock_apis/`, `user_profile/`, `data/portfolios/`, `telemetry/`, `a2a/` | Folders exist |
| 1.5 | Build 3 mock portfolios | Create JSON files with current-year + prior-year data for each property: **Midwest Office** (6 props), **Southeast Retail** (5 props), **Northeast Mixed** (7 props) | `load_portfolio("PF-MIDWEST-001")` returns 6 properties with current + prior year data |
| 1.6 | Build Prompt Manager | Implement the simple dict-based PromptManager class (Section 11) | `manager.get_prompt(...)` works |

**Key learning:** ADK basics, project structure, how Gemini API works, portfolio data modeling (current vs. prior year).

**Suggested project structure after Day 1:**
```
property-portfolio-renewal/
├── agents/
│   ├── __init__.py
│   ├── portfolio_orchestrator.py   # Skeleton only
│   ├── change_detection.py         # Skeleton only
│   ├── property_reevaluation.py    # Skeleton only
│   └── renewal_recommendation.py   # Skeleton only
├── prompt_manager/
│   ├── __init__.py
│   ├── manager.py                  # PromptManager class (working)
│   └── templates/
│       ├── __init__.py
│       ├── orchestrator.py         # Prompt strings
│       ├── change_detection.py
│       ├── property_reevaluation.py
│       └── renewal_recommendation.py
├── mock_apis/
│   ├── __init__.py
│   └── portfolio_loader.py         # Loads portfolio JSON by ID
├── user_profile/
│   ├── __init__.py
│   └── profile_store.py            # JSON file-based store
├── a2a/
│   └── agent_cards/                # Static agent card JSON files
├── data/
│   ├── profiles/                   # User profile JSON files
│   └── portfolios/
│       ├── midwest_office.json     # 6 properties, current + prior year
│       ├── southeast_retail.json   # 5 properties, current + prior year
│       └── northeast_mixed.json    # 7 properties, current + prior year
├── telemetry/
│   ├── __init__.py
│   └── arize_setup.py              # Skeleton only
├── main.py
├── requirements.txt
├── .env.example
└── README.md
```

---

### Day 2: Change Detection Agent (Zero-Shot + Few-Shot)
**Goal:** Agent that compares current-year vs. prior-year data for a property, identifies all changes, and classifies each by severity.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 2.1 | Register change detection prompts | Add zero-shot (field diff) and few-shot (change severity classification) prompt templates to Prompt Manager | Prompts retrievable |
| 2.2 | Implement comparison tool functions | Wire `compare_property_years(current, prior)` — returns structured diff of all changed fields with old/new values | Tool returns list of detected changes |
| 2.3 | Build Change Detection Agent | ADK agent with system instruction + tools. Uses zero-shot for field-level diff, few-shot for classifying changes as Material / Notable / Cosmetic | Agent returns structured change report with classified changes |
| 2.4 | Test with sample property pairs | Feed 2-3 property pairs from mock portfolios, verify change detection and classification accuracy | Changes detected, severity classifications reasonable |

**Key learning:** Data comparison logic, few-shot classification prompting, structured output from LLM.

**Prompt types practiced:**
- **Zero-shot** — Field-level diff: "Compare these two property records and list every field that changed"
- **Few-shot** — Change severity classification: "Given the following change, classify it as Material, Notable, or Cosmetic" (with 3-4 examples of each)

> **Tip:** Start with a simple field-by-field diff. Get that working reliably, then add the few-shot severity classifier on top. Don't try to do both at once.

---

### Day 3: Property Re-evaluation Agent (CoT + Parallel Categories)
**Goal:** Agent that re-evaluates risk across 4 categories using Chain-of-Thought, producing delta scores (current risk vs. prior risk).

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 3.1 | Write CoT re-evaluation prompts | Create Chain-of-Thought prompt for each risk category: **Fire**, **Natural Catastrophe**, **Occupancy**, **Building Condition** — each prompt compares current vs. prior year data and reasons through the delta | 4 prompt templates registered |
| 3.2 | Implement parallel assessment | Use `asyncio.gather()` to run all 4 category assessments simultaneously for a single property | All 4 run in parallel, results combined into single property risk profile |
| 3.3 | Build Property Re-evaluation Agent | ADK agent that receives current + prior year property data, runs 4 parallel CoT assessments, returns prior score, current score, and delta for each category | Agent returns 4 category scores with deltas + overall risk score |
| 3.4 | Test with sample properties | Feed mock properties with known changes, verify scores shift appropriately (e.g., flood zone reclassification raises NatCat score) | Scores 1-10 with step-by-step delta reasoning |

**Key learning:** Chain-of-Thought prompting with temporal comparison, parallel execution with asyncio, structured multi-category output.

**Prompt type practiced:** Chain-of-Thought (CoT) with temporal delta reasoning

> **Tip:** Start with the Fire Risk prompt. Get it producing good CoT reasoning with current vs. prior scores, then copy the pattern for NatCat, Occupancy, and Building Condition. Don't write all 4 from scratch — template from the first one.

---

### Day 4: Orchestrator + Renewal Recommendation + Parallel Batch
**Goal:** Wire the full parallel batch pipeline. Orchestrator processes N properties simultaneously through Change Detection → Re-evaluation → Recommendation.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 4.1 | Build Renewal Recommendation Agent | ADK agent that takes change report + risk scores → classifies into 4 renewal decisions: Renew As-Is, Renew with Adjustments, Refer to Senior, Non-Renew | Agent returns renewal decision + reasoning + premium impact |
| 4.2 | Write recommendation prompts | Few-shot prompt with 2-3 example renewal scenarios for each of the 4 decision categories | Consistent, well-reasoned recommendations |
| 4.3 | Build Portfolio Orchestrator (batch mode) | Main agent that: loads portfolio JSON → validates → fans out `asyncio.gather()` across all N properties → each property goes through Change Detection → Re-evaluation → Recommendation | All properties processed in parallel |
| 4.4 | Implement portfolio summary aggregation | Aggregation tool that combines per-property results into portfolio summary: renewal decision mix, risk trend, exception count, total premium impact | Summary matches schema (Section 15.5) |
| 4.5 | Test end-to-end batch with one portfolio | Run the Midwest Office portfolio (6 properties) through the full batch pipeline | All 6 properties processed, portfolio summary generated |

**Key learning:** Parallel fan-out pattern (N properties × 3 agents), portfolio-level aggregation, few-shot prompting for classification.

**Prompt types practiced:** Few-shot (renewal classification), Zero-shot (portfolio summary)

> **This is the most critical day.** By end of Day 4, you should have a working batch pipeline that processes an entire portfolio. Everything after this adds the conversational layer, A2A, and observability.

---

### Day 5: A2A + User Profiling + Review Chat
**Goal:** Add A2A protocol artifacts, user profiling, and the conversational review interface that lets underwriters query batch results.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 5.1 | Create Agent Cards | Write agent card JSON for all 4 agents (see Section 8) | 4 JSON files in `a2a/agent_cards/` |
| 5.2 | Implement A2A demo endpoint | Add A2A server to Change Detection Agent — serves agent card and accepts one task | `curl localhost:8002/.well-known/agent.json` works |
| 5.3 | A2A client call from Orchestrator | Orchestrator makes at least one call via A2A client (to Change Detection Agent) during batch processing | A2A task delegation works for change detection |
| 5.4 | Build review chat mode in Orchestrator | After batch completes, Orchestrator transitions to conversational mode — accepts natural language queries about portfolio results, specific properties, and recommendations | Underwriter can ask "Tell me about property X" and get relevant results |
| 5.5 | Build user profiling | Add profiling questions to Orchestrator's first conversational turn | Agent asks experience level, adapts style |
| 5.6 | Write profiling prompt (few-shot) | Few-shot prompt to classify user as junior analyst / senior manager from their responses | Classification works: "1.5 years" → junior, "15 years, show exceptions" → senior |
| 5.7 | Implement adaptive behavior | Orchestrator adjusts output based on profile: **Junior (Aisha)** — full property-by-property walkthrough, term explanations, guided review. **Senior (Robert)** — exception-only summary, concise tables, no explanations unless asked | Two demo scenarios show visibly different output styles |

**Key learning:** A2A protocol, dual-mode agent (batch → conversational), user profiling, adaptive prompt engineering.

**Prompt type practiced:** Few-shot (user classification)

---

### Day 6: Arize Integration + Deployment
**Goal:** Add observability and deploy.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 6.1 | Set up Arize account | Sign up at arize.com, create Space, get API key | Dashboard accessible |
| 6.2 | Initialize tracing | Add `arize-otel` setup code (Section 13), instrument main agent calls | Traces appear in Arize dashboard |
| 6.3 | Add span attributes | Tag traces with: `portfolio_id`, `property_id`, `risk_delta`, `renewal_decision`, `change_severity`, agent name, prompt version | Traces are filterable in Arize by portfolio and property |
| 6.4 | Set up annotation queue | Create an annotation queue for renewal decision accuracy review — annotators verify whether AI renewal recommendation matches expert judgment | Renewal decisions can be reviewed/annotated |
| 6.5 | Deploy or finalize local | **Option A:** Deploy to Vertex AI Agent Engine if GCP access available. **Option B:** Document `adk web` local deployment as the production setup. | Agent accessible for demo |
| 6.6 | Test deployed agent | Run 1 portfolio batch, verify traces flow to Arize with parallel property spans visible | Traces visible in Arize for all properties in the batch |

**Key learning:** OpenTelemetry tracing, LLM observability for parallel workloads, annotation workflows, cloud deployment.

---

### Day 7: Integration Testing + Demo Preparation
**Goal:** Polish, test all scenarios with both personas across all 3 portfolios, prepare for the demo.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 7.1 | Test Scenario 1: Junior Aisha | Run the guided Southeast Retail portfolio review scenario (Section 18.1) end-to-end — Aisha gets profiled, batch runs, full walkthrough with explanations | Guided, detailed flow works |
| 7.2 | Test Scenario 2: Senior Robert | Run the exceptions-only Northeast Mixed portfolio review scenario (Section 18.2) end-to-end — Robert recognized, batch runs, exception summary only | Concise, exceptions-only flow works |
| 7.3 | Test all 3 portfolios | Run batch processing on Midwest Office (6 props), Southeast Retail (5 props), Northeast Mixed (7 props) — verify all produce valid summaries | No crashes, scores and decisions make sense for all 18 properties |
| 7.4 | Verify Arize traces | Check Arize dashboard has traces for all test runs, annotation queue works, parallel property spans visible | Traces + annotations visible |
| 7.5 | Write demo script | Document the exact steps/inputs for the 2 demo scenarios | Demo script ready |
| 7.6 | Rehearse demo (dry run) | Run through both scenarios as if presenting — time it, fix any rough edges | Smooth, no errors, fits in allotted time |
| 7.7 | Update README | Document how to set up and run the project — env vars, portfolio data, running batch vs. chat | Someone else can run it |

---

### Day 8 (Stretch): Polish & Enhancements

If time allows, these additions make the demo even more impressive:

| # | Enhancement | Effort | Impact |
|---|------------|--------|--------|
| S1 | Add 2 more risk categories (Protection, Loss History) | 2-3 hours | Shows 6 parallel re-evaluation categories instead of 4 |
| S2 | Full A2A for all sub-agents | 2-3 hours | All 3 sub-agents use A2A protocol end-to-end |
| S3 | PDF export for portfolio renewal report | 1-2 hours | Production-ready renewal package output |
| S4 | Build Arize monitoring dashboard | 1-2 hours | Latency/cost/risk delta distribution widgets |
| S5 | Prompt v2 and A/B comparison in Arize | 1-2 hours | Demonstrates prompt iteration for renewal accuracy |
| S6 | Add more property data to existing portfolios | 1-2 hours | 10+ properties per portfolio for stress testing |

---

### Daily Checkpoint Validation

Use this checklist to verify each day's work is solid before moving on:

| Day | Checkpoint Question | Must Answer YES |
|-----|-------------------|----------------|
| 1 | Can I chat with a basic ADK agent in the browser? Are 3 portfolio JSON files loadable? | ✅ |
| 2 | Does the Change Detection Agent correctly identify and classify changes between current and prior year data? | ✅ |
| 3 | Does the Property Re-evaluation Agent run 4 category assessments in parallel and return delta scores? | ✅ |
| 4 | Can I run a full portfolio batch (6 properties) and get a portfolio summary with renewal decisions? | ✅ |
| 5 | Are Agent Cards created? Does the review chat work? Does user profiling change the output style? | ✅ |
| 6 | Do traces appear in Arize? Is the annotation queue set up? Are parallel property spans visible? | ✅ |
| 7 | Can I smoothly run both demo scenarios (junior Aisha + senior Robert) across different portfolios without errors? | ✅ |

---

## 18. Sample Interactions

### 18.1 Scenario: Junior Analyst Aisha — Guided Southeast Retail Portfolio Review

```
AGENT:  Welcome to the Property Portfolio Renewal system! I help
        underwriters review entire portfolios for renewal by analyzing
        changes, re-evaluating risk, and generating renewal recommendations.

        Before we get started, I'd like to understand your background
        so I can tailor the review experience. How long have you been
        working in commercial property underwriting?

AISHA:  Hi! I'm Aisha — I've been a portfolio analyst for about a year
        and a half. I was promoted from underwriting assistant recently.

AGENT:  Thanks, Aisha! Since you're relatively new to portfolio-level
        analysis, I'll make sure to walk you through each property in
        detail and explain what the changes mean for risk and renewal.

        What types of portfolios do you typically work on?

AISHA:  Mostly mid-market office portfolios in the Midwest. This is my
        first time reviewing a retail portfolio in a coastal area.

AGENT:  Got it — this will be a great learning experience. Coastal retail
        has some unique risk factors I'll highlight as we go, especially
        around natural catastrophe exposure.

        Do you have a portfolio you'd like to review for renewal today?

AISHA:  Yes — I need to review the Southeast Retail portfolio for renewal.
        The file is data/portfolios/southeast_retail.json.

AGENT:  Loading the Southeast Retail Portfolio now...

        📂 Portfolio loaded: **Southeast Retail Portfolio** (PF-SOUTHEAST-001)
        📊 Properties: 5
        📅 Renewal period: Current year vs. prior year comparison

        I'm going to process all 5 properties in parallel. For each one,
        I'll detect what changed since last year, re-evaluate risk across
        4 categories, and generate a renewal recommendation. This takes
        about 15-20 seconds for the full portfolio.

        Processing...

        ⏳ Property 1/5: 100 Bayshore Strip Mall .......... ✅
        ⏳ Property 2/5: 225 Palm Avenue Retail Center .... ✅
        ⏳ Property 3/5: 450 Gulf Coast Retail Plaza ...... ✅
        ⏳ Property 4/5: 610 Suncoast Shopping Village .... ✅
        ⏳ Property 5/5: 800 Coastal Industrial Ave ....... ✅

        ✅ **Batch complete!** All 5 properties processed in 18.5 seconds.

        Here's the portfolio summary:

        📊 **Southeast Retail Portfolio — Renewal Summary**

        | # | Property | Prior Score | Current Score | Delta | Changes | Decision |
        |---|----------|-------------|---------------|-------|---------|----------|
        | 1 | 100 Bayshore Strip Mall | 3.2 | 3.4 | +0.2 | 1 Notable | ✅ Renew As-Is |
        | 2 | 225 Palm Avenue Retail Center | 3.8 | 4.0 | +0.2 | 1 Notable | ✅ Renew As-Is |
        | 3 | 450 Gulf Coast Retail Plaza | 4.5 | 5.8 | +1.3 | 3 Material | ⚠️ Renew w/ Adjustments |
        | 4 | 610 Suncoast Shopping Village | 4.0 | 4.3 | +0.3 | 1 Notable | ✅ Renew As-Is |
        | 5 | 800 Coastal Industrial Ave | 4.8 | 7.6 | +2.8 | 3 Material | 🔴 Refer to Senior |

        **Portfolio Stats:**
        - Average risk delta: **+0.96** (portfolio trending worse)
        - Renewal mix: 3 Renew As-Is, 1 Adjust, 1 Refer
        - Flagged properties: **2** (Gulf Coast Plaza and Coastal Industrial)
        - Estimated premium impact: **+11.9%** ($320K → $358K)

        I've flagged 2 properties with material changes that need your
        attention. Would you like me to walk through each property in
        detail, or start with the flagged ones?

AISHA:  Let's start with the flagged ones. Tell me more about the
        Oceanfront — I mean, the Gulf Coast Retail Plaza changes.

AGENT:  Great choice — this one has 3 material changes worth understanding.

        🏢 **450 Gulf Coast Retail Plaza — Detailed Change Report**

        **Prior Year Risk Score:** 4.5 / 10 (MEDIUM)
        **Current Year Risk Score:** 5.8 / 10 (MEDIUM-HIGH)
        **Risk Delta:** +1.3 ⬆️

        **Material Changes Detected:**

        | # | Change | Prior Year | Current Year | Severity | Impact |
        |---|--------|-----------|--------------|----------|--------|
        | 1 | Flood zone reclassified | Zone X (minimal) | Zone AE (high risk) | 🔴 Material | NatCat score +2.1 |
        | 2 | Windstorm deductible triggered | Standard deductible | Named Storm deductible required | 🔴 Material | Premium adjustment needed |
        | 3 | Roof age crossed threshold | 12 years (acceptable) | 13 years (nearing replacement) | 🔴 Material | Building score +0.8 |

        **Category-by-Category Risk Re-evaluation:**

        | Category | Prior | Current | Delta | Key Driver |
        |----------|-------|---------|-------|------------|
        | 🔥 Fire | 3.5 | 3.8 | +0.3 | Roof age increases fire spread risk |
        | 🌊 NatCat | 4.2 | 6.3 | +2.1 | Flood zone X → AE is a major upgrade in exposure |
        | 🏭 Occupancy | 4.8 | 4.8 | 0.0 | No change — same retail tenants |
        | 🏗️ Building | 5.5 | 6.3 | +0.8 | Roof age + deferred maintenance concern |

        **Renewal Recommendation:** ⚠️ **Renew with Adjustments**
        - Increase premium by 15–20% to reflect new flood exposure
        - Require named storm deductible endorsement
        - Request roof inspection report before binding

        💡 The biggest factor here is the flood zone change. FEMA
        reclassified this area from Zone X to Zone AE, which significantly
        increases the property's natural catastrophe exposure.

        Want me to explain what FEMA Zone AE means?

AISHA:  Yes please — what does FEMA Zone AE mean? How is it different
        from Zone X?

AGENT:  Great question! This is one of the most important concepts in
        property underwriting for coastal areas.

        📘 **FEMA Flood Zone Definitions:**

        | Zone | Risk Level | Meaning |
        |------|-----------|---------|
        | **Zone X** | Minimal | Area of minimal flood hazard. Less than 0.2% annual chance of flooding (the "500-year" flood zone). Most properties are here — it's considered low risk. Flood insurance is optional. |
        | **Zone AE** | High | Area of **high flood hazard**. 1% or greater annual chance of flooding (the "100-year" flood zone). FEMA has determined the base flood elevation (BFE) for this area. Flood insurance is typically required by lenders. |

        **Why this matters for our property:**

        When FEMA moves a property from Zone X → Zone AE, it means:
        1. **Exposure dramatically increases** — from "unlikely to flood" to
           "statistically expected to flood within 100 years"
        2. **Insurance requirements change** — flood coverage may now be
           mandatory, and the cost increases significantly
        3. **Property value impact** — buyers and tenants factor flood risk
           into leasing decisions
        4. **Our underwriting response** — we need to increase the premium
           to reflect the higher expected loss, and may require specific
           flood deductibles or coverage sublimits

        This is classified as a "Material" change because it directly
        affects both the risk score and the renewal terms. A Zone X → AE
        reclassification is one of the highest-impact single changes you'll
        see in a property portfolio review.

        Would you like to look at the other flagged property (800 Coastal
        Industrial Ave), or do you have more questions about this one?

AISHA:  No, that makes sense now. Let's look at the 800 Coastal
        Industrial property.

AGENT:  🏢 **800 Coastal Industrial Ave — Detailed Change Report**

        **Prior Year Risk Score:** 4.8 / 10 (MEDIUM)
        **Current Year Risk Score:** 7.6 / 10 (HIGH)
        **Risk Delta:** +2.8 ⬆️⬆️ (largest in portfolio)

        **Material Changes Detected:**

        | # | Change | Prior Year | Current Year | Severity | Impact |
        |---|--------|-----------|--------------|----------|--------|
        | 1 | Occupancy shift | General retail | Light manufacturing + hazmat storage | 🔴 Material | Occupancy score +3.0 |
        | 2 | Fire protection downgraded | Class 3 (full sprinklers) | Class 5 (partial sprinklers only) | 🔴 Material | Fire score +1.5 |
        | 3 | Code violation open | None | Electrical code violation (unresolved) | 🔴 Material | Building score +1.2 |

        **Renewal Recommendation:** 🔴 **Refer to Senior Underwriter**

        This property has the highest risk delta in the portfolio (+2.8).
        The combination of a hazmat occupancy shift, downgraded fire
        protection, and an open code violation exceeds the threshold for
        a junior analyst to approve independently.

        💡 I'm recommending this goes to your senior manager (Robert)
        for review. The hazmat storage change alone would typically
        trigger a referral.

AISHA:  That makes sense — I wouldn't be comfortable approving that one
        on my own. Can you generate the portfolio renewal report so I
        can share it with Robert?

AGENT:  Of course! Generating the full portfolio renewal report...

        ✅ **Portfolio Renewal Report Generated**

        ---
        # PORTFOLIO RENEWAL REPORT
        ## Southeast Retail Portfolio (PF-SOUTHEAST-001)

        **Generated:** April 18, 2026
        **Analyst:** Aisha Patel
        **Properties Reviewed:** 5
        **Processing Time:** 18.5 seconds

        **Portfolio Risk Trend:** DETERIORATED (+0.96 avg delta)
        **Estimated Premium Impact:** +11.9% ($320K → $358K)

        | Decision | Count | Properties |
        |----------|-------|-----------|
        | Renew As-Is | 3 | Bayshore, Palm Avenue, Suncoast |
        | Renew w/ Adjustments | 1 | Gulf Coast Plaza |
        | Refer to Senior | 1 | Coastal Industrial |

        ... [Full formatted report continues for each property] ...
        ---

        The report is ready. You can share it with Robert for the
        Coastal Industrial referral review. Is there anything else
        you'd like to explore in this portfolio?
```

### 18.2 Scenario: Senior Manager Robert — Exceptions-Only Northeast Mixed Portfolio

```
AGENT:  Welcome back, Robert! I see you've reviewed 42 portfolios
        with us. What do you have today?

ROBERT: Northeast Mixed portfolio — show me exceptions only. I don't
        need to see the stable properties.

AGENT:  Running the Northeast Mixed Portfolio now...

        📂 Portfolio: **Northeast Mixed Portfolio** (PF-NORTHEAST-001)
        📊 Properties: 7

        Processing all 7 properties in parallel...

        ✅ **Batch complete!** 7 properties processed in 22.3 seconds.

        Filtering to exceptions only (material changes or non-standard
        renewal decisions)...

        📊 **Northeast Mixed — Exceptions Only** (3 of 7 flagged)

        | # | Property | Delta | Issue | Decision |
        |---|----------|-------|-------|----------|
        | 1 | 44 Elm Street Office | +1.8 | Open code violation — electrical panel non-compliant | ⚠️ Renew w/ Adjustments |
        | 2 | 150 Harbor View Mixed Use | +2.2 | Code violation (fire escape) + occupancy change (ground floor → restaurant) | ⚠️ Renew w/ Adjustments |
        | 3 | 320 Granite Industrial Park | +3.1 | Occupancy shift to chemical storage, building age 45+ years | 🔴 Refer to Senior |

        **Portfolio Stats (full):**
        - 7 properties total: 4 stable (Renew As-Is), 2 Adjust, 1 Refer
        - Average delta: +0.85
        - Premium impact: +8.2% ($485K → $525K)

        The other 4 properties are stable with minor or no changes.
        Want details on any of the flagged properties?

ROBERT: Override 44 Elm to renew-as-is. The renovation is already
        scheduled — electrical panel replacement starts next month.
        I've confirmed it with the building manager.

AGENT:  Acknowledged. Updating 44 Elm Street Office:

        ✏️ **Override Applied:**
        - Decision changed: Renew w/ Adjustments → **Renew As-Is** (manager override)
        - Override reason: Electrical panel renovation confirmed, starting next month
        - Overridden by: Robert Chen (Senior Underwriting Manager)
        - Note: Recommend conditional renewal — confirm renovation completion at mid-term

        Updated exception summary:

        | # | Property | Delta | Decision |
        |---|----------|-------|----------|
        | 1 | ~~44 Elm Street Office~~ | ~~+1.8~~ | ✅ Renew As-Is (override) |
        | 2 | 150 Harbor View Mixed Use | +2.2 | ⚠️ Renew w/ Adjustments |
        | 3 | 320 Granite Industrial Park | +3.1 | 🔴 Refer to Senior (remains) |

        The Granite Industrial referral stays flagged — that one needs
        senior review given the chemical storage change. Do you want
        to review it now or handle it separately?

ROBERT: I'll review Granite separately. What's the bottom line on
        Harbor View?

AGENT:  **150 Harbor View Mixed Use — Quick Summary:**
        - Ground floor converted from office → restaurant (occupancy risk ↑)
        - Fire escape code violation open (building risk ↑)
        - Delta: +2.2, driven by occupancy (+1.4) and building (+0.8)
        - Recommendation: Increase premium 12%, require fire escape
          remediation within 90 days as renewal condition

ROBERT: Fine. Generate the renewal package.

AGENT:  Generating portfolio renewal report with your overrides...

        ✅ **Portfolio Renewal Package Generated**

        ---
        # PORTFOLIO RENEWAL REPORT
        ## Northeast Mixed Portfolio (PF-NORTHEAST-001)

        **Generated:** April 18, 2026
        **Reviewer:** Robert Chen (Senior Underwriting Manager)
        **Properties Reviewed:** 7
        **Overrides Applied:** 1

        | Decision | Count | Properties |
        |----------|-------|-----------|
        | Renew As-Is | 5 | (4 stable + 44 Elm override) |
        | Renew w/ Adjustments | 1 | Harbor View |
        | Refer to Senior | 1 | Granite Industrial (pending) |

        **Premium Impact:** +8.2% ($485K → $525K)
        ---

        Report ready. Granite Industrial flagged for separate review.
        Anything else?

ROBERT: No, that's it. Good summary.

AGENT:  Thanks, Robert. Northeast Mixed renewal package filed.
        Granite Industrial remains in your review queue.
```

---

## 19. Acceptance Criteria

### 19.1 MVP — Must Demo (Days 1-7)

These are the **non-negotiable** deliverables for the demo:

| # | Criterion | Verification | Tech Topic Covered |
|---|-----------|-------------|-------------------|
| AC-01 | Parallel batch processing works for a portfolio of 5+ properties | Demo with Southeast Retail (5 props) or Northeast Mixed (7 props) | ADK |
| AC-02 | Change Detection Agent identifies material changes between current and prior year data | Structured change report with Material / Notable / Cosmetic classifications | ADK + Few-shot prompting |
| AC-03 | Risk re-evaluation runs 4 categories per property with CoT delta reasoning | Visible parallel execution, 4 scores with prior/current/delta returned | ADK + CoT prompting + Parallel |
| AC-04 | Re-evaluation includes step-by-step delta reasoning | Each score has CoT explanation comparing current vs. prior year factors | Prompt Engineering (CoT) |
| AC-05 | Renewal Recommendation classifies into 4 categories (Renew As-Is, Adjust, Refer, Non-Renew) | Correct classification with reasoning for each property | ADK + Few-shot prompting |
| AC-06 | User profiling detects junior vs. senior (Aisha gets guided walkthrough, Robert gets exceptions-only) | Two demo scenarios show visibly different output styles | User Profiling |
| AC-07 | Agent Cards (JSON) created for all 4 agents | Files exist with valid schema in `a2a/agent_cards/` | A2A |
| AC-08 | At least 1 A2A call works (Orchestrator → Change Detection Agent) | A2A task delegation demonstrated during batch processing | A2A |
| AC-09 | Prompt Manager serves versioned templates | Templates loaded via manager, version tracked, used by all agents | Prompt Manager |
| AC-10 | Arize traces visible in dashboard (including parallel property spans) | All LLM calls traced, portfolio and property IDs filterable | Arize |
| AC-11 | Arize annotation queue for renewal decision accuracy review set up | Renewal decisions can be reviewed/annotated by domain experts | Arize |
| AC-12 | Running locally via `adk web` **OR** deployed to Agent Engine | Accessible demo with batch + chat modes working | Agent Engine / Deployment |

**Count: 6/6 tech topics covered in MVP** ✅

### 19.2 Stretch — Nice to Show (Day 8)

| # | Criterion | Tech Topic |
|---|-----------|-----------|
| AC-13 | 6 parallel risk categories (add Protection, Loss History) | ADK + Parallel |
| AC-14 | All 3 sub-agents callable via full A2A protocol | A2A |
| AC-15 | PDF export for portfolio renewal report | ADK |
| AC-16 | Arize dashboard with latency, cost, and risk delta distribution widgets | Arize |
| AC-17 | Prompt v1 vs v2 comparison visible in Arize | Prompt Manager + Arize |
| AC-18 | 10+ properties per portfolio for stress testing | Mock Data |

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

### 20.6 Property Portfolio Renewal Domain
- ISO Construction Classes: 1 (Frame), 2 (Joisted Masonry), 3 (Non-Combustible), 4 (Masonry Non-Combustible), 5 (Modified Fire Resistive), 6 (Fire Resistive)
- ISO Fire Protection Classes: 1 (Best) to 10 (Unprotected)
- FEMA Flood Zone Definitions: A, AE (high risk — base flood elevation determined), V, VE (coastal high hazard), X (minimal risk — outside 500-year floodplain)
- Property portfolio renewal cycles: Annual renewal review process for commercial property insurance, comparing current-year exposure data against prior-year baseline to determine renewal terms
- Loss development factors and experience modification in portfolio-level underwriting decisions

---

*Document Version: 1.0*
*Created: April 18, 2026*
*Author: AI Planning Assistant*
*Status: Ready for Review*
*Scope: 7-day MVP + 1-day Stretch*
