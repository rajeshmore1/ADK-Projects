# Project 4: Commercial Auto Fraud Detection & Investigation Agent

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
A non-conversational multi-agent AI pipeline that ingests commercial auto insurance claims in bulk, enriches each claim with claimant history and provider data, runs four parallel fraud pattern analyses per claim (duplicate/similar claims, suspicious timing, inflated amounts, provider network anomalies), generates composite fraud scores with confidence levels, and produces detailed investigation reports with evidence summaries for Special Investigations Unit (SIU) analysts. A lightweight status and query chat interface lets SIU investigators check pipeline progress, query results, and drill into specific claims without interrupting the batch pipeline.

### Why
Insurance fraud is a $40+ billion annual problem in the United States, and commercial auto is one of the most fraud-prone lines. Fleet operations, high claim volumes, multi-vehicle incidents, and complex provider networks create fertile ground for organized fraud rings, staged accidents, and billing inflation. Today, SIU teams rely on manual red-flag checklists, spreadsheet-based claim reviews, and individual investigator intuition to identify suspicious claims — a process that is slow, inconsistent, and scales poorly against rising claim volumes. Common failure modes:
- **Pattern blindness** — Individual investigators review claims one at a time and miss cross-claim patterns (e.g., a provider billing inflated repairs on 12 different fleet claims, or the same claimant filing eerily similar claims 90 days apart)
- **Inconsistent scoring** — Junior analysts flag different indicators than senior investigators, producing wildly different referral rates depending on who reviews the claim
- **Overwhelming volume** — A large carrier may process 500–2,000 commercial auto claims per week; manual review of every claim is impossible, so SIU samples only 5–10% of total volume
- **Slow turnaround** — Manual fraud investigation takes 2–4 weeks per claim, during which fraudulent claims may already be paid out
- **False positive fatigue** — Overly broad rule-based systems generate 60–70% false positives, causing investigators to ignore alerts entirely

An AI agent system can ingest an entire batch of claims simultaneously, run parallel pattern analysis across multiple fraud dimensions, detect cross-claim patterns that no individual reviewer could spot, and produce ranked, evidence-backed investigation reports — reducing triage time from weeks to minutes and dramatically improving detection accuracy.

### How
Four specialized agents communicate via Google's A2A protocol within a non-conversational pipeline:
1. **Fraud Pipeline Orchestrator** — Accepts a batch of claims JSON, validates input, fans out N claims in parallel for enrichment and analysis, collects results, ranks by fraud score, and produces a final batch report. Also serves a lightweight status/query chat for SIU investigators to check pipeline progress and drill into results.
2. **Claim Data Enrichment Agent** — For each claim, enriches raw claim data with claimant history (prior claims, prior flags), provider lookup (billing patterns, license status), vehicle valuation (market value vs. claimed damage), and policy details (coverage gaps, recent changes). Task-based, non-conversational.
3. **Fraud Pattern Analyzer** — The analytical core. For each enriched claim, runs **4 fraud patterns in parallel**: Duplicate/Similar Claims, Suspicious Timing, Inflated Amounts, and Provider Network Anomalies. Each pattern produces a sub-score and evidence list. The agent then computes a composite fraud score. Task-based, non-conversational.
4. **Investigation Report Generator** — For each claim flagged above a configurable threshold, generates a structured investigation report including evidence summary, recommended investigative actions, priority classification, and a narrative suitable for SIU case files. Task-based, non-conversational.

Built with Google ADK + Gemini 2.5 (Pro for orchestration and fraud reasoning, Flash for enrichment and report generation), monitored via Arize Cloud and OpenTelemetry, deployed to Vertex AI Agent Engine.

### Timeline & Scope

| Attribute | Detail |
|-----------|--------|
| **Duration** | 7–8 working days |
| **Intern Level** | CS background, medium Python expertise |
| **Delivery** | Working demo with 2 claim batches ("Weekly Auto Claims" — 8 claims with 5 clean + 3 fraud; "Flagged Commercial Claims" — 6 claims), 4 fraud patterns (MVP) + 2 stretch patterns, and 2 SIU personas (junior analyst Kevin + senior lead Diana) |

The project is organized into **MVP (Must-ship in 7 days)** and **Stretch (Day 8 if time permits)** scopes. The MVP alone covers all 6 tech topics and produces a compelling fraud detection demo. Stretch goals add two additional fraud patterns (Staged Accident Indicators, Policy Manipulation) and enhanced reporting.

### Key Tech Stack Coverage

| Tech Topic | How It's Used in This Project | Scope |
|------------|------------------------------|-------|
| Prompt Engineering | Zero-shot (status/query chat responses, data validation), Few-shot (fraud pattern classification with labeled examples of real fraud vs. clean claims), CoT (fraud reasoning — step-by-step analysis of why a claim is suspicious across multiple dimensions), ReAct (claim data enrichment — tool calls to gather claimant history, provider data, vehicle valuation) | **MVP** |
| Prompt Manager | Version and serve all prompt templates across 4 agents using a lightweight dict-based Python module with `str.format_map` | **MVP** |
| ADK | All 4 agents built with Google Agent Development Kit; non-conversational batch pipeline orchestration + status/query chat via `adk web` | **MVP** |
| A2A | Agents publish Agent Cards; Orchestrator delegates tasks to sub-agents via A2A protocol; enrichment, analysis, and report generation tasks dispatched via A2A | **MVP** |
| User Profiling | Profile the SIU investigator querying results (junior analyst Kevin vs. senior lead Diana) and adapt query responses — detailed walkthroughs with term explanations vs. concise high-confidence flags with statistical summaries | **MVP** |
| Arize | Trace all agent interactions with OpenTelemetry; monitor fraud scoring accuracy, pattern detection rates, and false positive ratios; set up annotation queue for fraud classification quality review | **MVP** |
| Agent Engine | Deploy agents to Vertex AI Agent Engine (or local `adk web` fallback) | **MVP** |

### What Makes This an Excellent Learning Project

1. **Non-conversational pipeline architecture** — The intern builds a true batch processing system where claims flow through a pipeline without human interaction, fundamentally different from the conversational agents in Projects 1 and 2 — this is how real enterprise AI fraud detection systems operate
2. **Fraud and adversarial domain** — Fraud detection is inherently adversarial: the system must reason about intent, deception, and pattern manipulation. This teaches the intern to build AI systems that analyze evidence, weigh conflicting signals, and produce defensible conclusions — skills that transfer to any domain requiring analytical rigor
3. **Dual-level parallelism** — The architecture features two layers of concurrent execution: N claims processed in parallel at the batch level, AND 4 fraud patterns analyzed in parallel per claim. This nested `asyncio.gather` pattern is a production-grade concurrency design the intern will encounter in real-world systems
4. **Evidence-based scoring** — Unlike binary accept/reject systems, the fraud analyzer produces composite scores backed by specific evidence from each pattern. The intern learns to build AI systems that explain their reasoning, not just output a number — critical for regulated industries where investigators must justify referrals
5. **Cross-line insurance domain** — Commercial auto fraud spans auto insurance (vehicle damage, bodily injury) and commercial insurance (fleet operations, business liability), teaching the intern to work at the intersection of multiple insurance lines — a common enterprise scenario
6. **SIU query interface** — The status/query chat gives SIU investigators the ability to check pipeline progress, drill into specific claims, and ask questions about fraud patterns — demonstrating how a non-conversational system can still provide human-accessible interaction without being conversational-first
7. **Real fraud patterns with industry relevance** — The four MVP patterns (duplicate claims, suspicious timing, inflated amounts, provider networks) are drawn directly from actual SIU playbooks, giving the intern exposure to genuine insurance fraud investigation methodology

---

## 2. Business Context

### 2.1 Industry Background

**Insurance fraud** is the second-largest economic crime in the United States after tax evasion. The Coalition Against Insurance Fraud estimates that fraud costs the U.S. insurance industry over **$40 billion annually** across all lines, translating to $400–$700 per year in increased premiums for the average American family. Within this landscape, **commercial auto insurance** is among the most fraud-susceptible lines due to the inherent complexity of fleet operations, the volume of claims, and the difficulty of verifying incidents involving commercial vehicles.

**Commercial auto fraud** differs from personal auto fraud in several critical ways:
- **Fleet scale** — A single commercial policyholder may operate 50–500+ vehicles, generating dozens of claims per year. This volume makes it easy to bury fraudulent claims among legitimate ones.
- **Higher claim values** — Commercial vehicle repairs, cargo damage, and business interruption claims are typically 3–5x higher than personal auto claims, making fraud more lucrative.
- **Complex provider networks** — Fleet operations involve specialized repair shops, body shops, medical providers, towing companies, and rental agencies. Organized fraud rings establish colluding provider networks that systematically inflate charges.
- **Multi-party incidents** — Commercial vehicle accidents often involve multiple claimants (passengers, pedestrians, other vehicles), creating opportunities for staged accidents and phantom injuries.
- **Regulatory complexity** — Commercial auto insurance spans multiple jurisdictions, coverage types (liability, physical damage, cargo, hired/non-owned auto), and reporting requirements, creating gaps that fraudsters exploit.

**Types of commercial auto fraud** relevant to this project:

| Fraud Type | Description | Typical Indicators |
|-----------|-------------|-------------------|
| **Duplicate/Similar Claims** | Same or nearly identical claims filed multiple times, sometimes across different carriers or policies | Matching vehicle VINs, similar loss descriptions, overlapping dates, same claimant |
| **Suspicious Timing** | Claims filed shortly after policy inception, just before policy expiration, or clustered around premium payment dates | Claims within 30 days of policy start, claims right before cancellation, Friday-night losses reported Monday |
| **Inflated Amounts** | Legitimate incidents with grossly inflated repair costs, medical bills, or lost-income claims | Repair estimates 2x+ above market value, medical treatment far exceeding injury severity, excessive rental duration |
| **Provider Network Anomalies** | Colluding providers (body shops, medical clinics, attorneys) who refer claimants to each other and systematically overbill | Same provider appearing on 20%+ of claims, provider billing 3x market rate, circular referral patterns |
| **Staged Accidents** | Manufactured collisions involving recruited participants, often targeting commercial vehicles for higher payouts | Multiple witnesses with same address, claimant history of similar accidents, vehicle damage inconsistent with described impact |
| **Policy Manipulation** | Coverage added retroactively, vehicles added to fleet policy after loss occurs, or policy limits increased just before a claim | Policy changes within 72 hours of loss, vehicle added to policy same day as accident, coverage upgrade preceding a large claim |

**Special Investigations Unit (SIU)** is the department within an insurance carrier responsible for detecting, investigating, and resolving suspected fraud. SIU operations typically follow a referral-based model:
1. Claims adjusters or automated rules flag suspicious claims
2. SIU analysts triage flagged claims and prioritize investigations
3. Investigators gather evidence (recorded statements, surveillance, data analysis)
4. Cases are either closed (no fraud found), resolved internally (claim denied or adjusted), or referred to law enforcement
5. SIU reports metrics to state insurance departments as required by anti-fraud regulations

The challenge: SIU teams are chronically understaffed. A typical large carrier's SIU has 15–30 investigators handling 3,000–5,000 referrals per year. Each investigation takes 2–4 weeks. The result is that only 5–10% of potentially fraudulent claims are ever investigated, and by the time an investigation concludes, the claim has often already been paid.

### 2.2 Current Process (Pain Points)

```
┌──────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌──────────────┐
│ Claim Filed  │───►│ Adjuster      │───►│ Rules Engine  │───►│ SIU Analyst   │───►│ Manual       │
│ (Commercial  │    │ Initial       │    │ / Red-Flag    │    │ Triage &      │    │ Investigation│
│  Auto)       │    │ Review        │    │ Checklist     │    │ Prioritization│    │ (2-4 weeks)  │
│              │    │ (1-2 days)    │    │ (Automated,   │    │ (Spreadsheet  │    │              │
│              │    │               │    │  high false    │    │  review,      │    │              │
│              │    │               │    │  positive)     │    │  1-3 hrs/     │    │              │
│              │    │               │    │               │    │  claim)        │    │              │
└──────────────┘    └───────────────┘    └───────────────┘    └───────────────┘    └──────────────┘
                                                                                          │
                                                                                          ▼
                                                                                   ┌──────────────┐
                                                                                   │ Outcome:     │
                                                                                   │ • Deny claim │
                                                                                   │ • Adjust $   │
                                                                                   │ • Refer to   │
                                                                                   │   law enforce│
                                                                                   │ • Close (no  │
                                                                                   │   fraud)     │
                                                                                   └──────────────┘

  Total elapsed: 2–6 weeks per suspicious claim
  Detection rate: ~5-10% of fraudulent claims caught
  False positive rate: 60-70% of rule-based flags
```

**Pain Points:**
- **Slow detection** — Rule-based systems flag claims 1–2 days after filing, but SIU triage adds another 1–3 days. By the time an analyst reviews a suspicious claim, legitimate processing is already underway and fraudulent payments may already be authorized.
- **High false positive rate** — Traditional rules engines (e.g., "flag any claim over $25,000" or "flag any claim within 60 days of policy inception") generate 60–70% false positives, overwhelming SIU teams and causing alert fatigue. Investigators begin ignoring flags, defeating the system's purpose.
- **No cross-claim analysis** — Each claim is evaluated in isolation. An investigator reviewing Claim #A has no visibility into Claim #B filed by the same claimant last month, or Claim #C involving the same provider. Pattern detection across claims requires manual spreadsheet correlation that rarely happens at scale.
- **Inconsistent triage** — Junior SIU analysts lack the experience to weight fraud indicators correctly. A junior analyst might escalate a $50,000 repair estimate (which is normal for a commercial truck) while missing a $15,000 claim from a provider who appears on 30 other claims — a much stronger fraud signal.
- **Volume overwhelm** — A large commercial auto carrier processes 500–2,000 claims per week. Even with rules-based pre-filtering, SIU receives 50–200 referrals per week and can only deeply investigate 20–30. The rest are either auto-closed or never reviewed.
- **Evidence assembly burden** — When an SIU investigator does identify fraud, assembling the evidence package (prior claims, provider history, policy timeline, damage photos, medical records) takes 4–8 hours per claim. Most of this is data retrieval, not analysis.
- **Provider ring detection failure** — The most costly fraud is organized (rings of colluding providers, recruiters, and claimants), but detecting ring structures requires network analysis across hundreds of claims — impossible to do manually.

### 2.3 Proposed Solution

```
┌──────────────┐    ┌───────────────────────────────────────────────────────────────────┐    ┌──────────────┐
│ SIU Analyst  │───►│              AI Fraud Detection Pipeline (This Project)            │───►│ Investigation│
│ Submits      │    │                                                                    │    │ Reports      │
│ Claim Batch  │    │  STEP 1: Batch Intake & Validation                                │    │ (Ranked by   │
│ JSON         │    │    ► Validate N claims, queue for parallel processing              │    │  fraud score)│
│              │    │                                                                    │    │              │
│ ("Weekly     │    │  STEP 2: Parallel Claim Processing (N claims simultaneously)       │    │ Per-Claim:   │
│  Auto Claims"│    │                                                                    │    │ • Fraud Score│
│  — 8 claims) │    │    Claim 1 ──┬─► Enrich ─► Analyze (4 patterns ║) ─► Report      │    │ • Evidence   │
│              │    │    Claim 2 ──┤                                                     │    │ • Priority   │
│              │    │    Claim 3 ──┤   ┌──────────────────────────────┐                  │    │ • Actions    │
│              │    │    ...       │   │ Per-Claim Parallel Patterns: │                  │    │              │
│              │    │    Claim N ──┘   │  ► Duplicate/Similar Claims  │                  │    │ Batch:       │
│              │    │                  │  ► Suspicious Timing         │                  │    │ • Summary    │
│              │    │                  │  ► Inflated Amounts          │                  │    │ • Rankings   │
│              │    │                  │  ► Provider Network Anomaly  │                  │    │ • Statistics │
│              │    │                  └──────────────────────────────┘                  │    │              │
│              │    │                                                                    │    │              │
│              │    │  STEP 3: Score, Rank & Report                                     │    │              │
│              │    │    ► Composite fraud scores, rank by risk, generate reports        │    │              │
│              │    │                                                                    │    │              │
│              │    │  ═══════════════════════════════════════════════════               │    │              │
│              │    │  STATUS/QUERY CHAT (parallel to pipeline):                         │    │              │
│              │    │    ► "How many claims processed?" / "Show me Claim #5"             │    │              │
│              │    │    ► "Which claims scored highest?" / "Explain the timing flag"    │    │              │
│              │    │                                                                    │    │              │
└──────────────┘    └───────────────────────────────────────────────────────────────────┘    └──────────────┘

  Dual-Level Parallelism:
    Level 1: N claims processed simultaneously (batch-level)
    Level 2: 4 fraud patterns analyzed simultaneously per claim (pattern-level)

  Target: Batch of 8 claims fully analyzed in < 5 minutes (vs. 2-4 weeks manual)
```

**Target outcomes:**
- Reduce claim fraud triage from 2–6 weeks → under 5 minutes for a batch of 8 claims
- Replace manual red-flag checklists with 4-dimensional parallel fraud pattern analysis that catches cross-claim patterns
- Produce investigation-ready reports with evidence summaries, eliminating 4–8 hours of manual evidence assembly per claim
- Reduce false positive rate from 60–70% (rules engines) to under 30% through multi-pattern weighted scoring
- Enable SIU teams to review 100% of incoming claims (not just the 5–10% they can manually triage)

### 2.4 Target Enterprise Segment

| Attribute | Detail |
|-----------|--------|
| **Company Size** | Large commercial auto carriers, national fleet insurers, large SIU departments (15+ investigators) |
| **Claim Volume** | 500–2,000 commercial auto claims per week; batch sizes of 5–50 claims per pipeline run |
| **Fleet Policyholder Types** | Trucking companies, delivery fleets, construction vehicle operators, taxi/rideshare commercial divisions, municipal vehicle fleets |
| **Typical Claim Values** | $5,000–$500,000 per claim (vehicle damage, bodily injury, cargo, business interruption) |
| **Geographic Focus** | United States (state-specific fraud statutes, NICB integration patterns) |
| **Lines of Business** | Commercial Auto Liability, Commercial Auto Physical Damage, Hired & Non-Owned Auto, Motor Truck Cargo — cross-line with Commercial General Liability for multi-party incidents |
| **SIU Team Size** | 15–30 investigators, 3–5 supervisors, 1–2 data analysts |
| **Current Fraud Detection** | Rules-based flagging (NICB ISO ClaimSearch, internal red-flag rules), manual triage by SIU analysts |
| **Regulatory Environment** | State insurance fraud bureaus, NICB (National Insurance Crime Bureau) reporting, anti-fraud plan requirements per state |

---

## 3. Personas & User Stories

### 3.1 Personas

#### Persona 1: Junior SIU Analyst — "Kevin"

| Attribute | Detail |
|-----------|--------|
| **Name** | Kevin Torres |
| **Experience** | 2 years in SIU, joined directly from a criminal justice degree program. Previously interned at a county fraud bureau before moving to the private sector. Still developing his pattern recognition intuition. |
| **Daily Volume** | Reviews 15–25 flagged claims per day for initial triage; conducts deep investigation on 2–3 claims per week |
| **Pain Points** | Not confident distinguishing genuine red flags from coincidental patterns. Spends too much time on false positives from the rules engine. Doesn't know what "normal" looks like for commercial fleet claims (is a $45,000 truck repair suspicious or standard?). Struggles to assemble evidence packages efficiently — spends 4–6 hours pulling claimant history, provider records, and policy timelines for each investigation. Often misses cross-claim patterns because he reviews claims individually. |
| **Needs** | Clear explanations of why each fraud indicator was triggered and what makes it significant. Contextual benchmarks (e.g., "this repair cost is 2.3x the market average for this vehicle type"). Step-by-step evidence summaries he can use directly in investigation reports. Guidance on which claims to prioritize and which to close. Help building his investigative instincts by learning from the system's reasoning. |
| **Tech Comfort** | High — comfortable with JSON data, chat interfaces, and web tools. Prefers digital workflows over paper. |
| **Typical Caseload** | Routine commercial auto claims: fender benders, parking lot incidents, minor theft. Occasionally assigned a medium-complexity case (inflated repair estimate, suspicious medical billing). |

#### Persona 2: Senior SIU Lead — "Diana"

| Attribute | Detail |
|-----------|--------|
| **Name** | Diana Okafor |
| **Experience** | 14 years in insurance fraud investigation. Former law enforcement (5 years as a detective in an insurance fraud task force) before moving to the carrier side. Certified Fraud Examiner (CFE). Has personally investigated over 400 cases and testified as an expert witness in 30+ fraud prosecutions. Supervises a team of 6 SIU analysts including Kevin. |
| **Daily Volume** | Reviews 40–60 claim summaries per day at triage level; personally investigates 5–8 complex cases per month (organized rings, staged accidents, large-loss fraud). Approves all SIU referrals to law enforcement. |
| **Pain Points** | Drowning in low-quality referrals from the rules engine — 60%+ are false positives that waste her team's time. Needs to quickly identify the 5–10 claims out of 200 weekly referrals that are genuinely suspicious. Frustrated that the existing system flags individual claims but never connects the dots across claims (a provider appearing on 15 claims in 3 months is a ring indicator, but the system treats each claim as independent). Spends too much management time re-triaging claims her junior analysts should have handled. |
| **Needs** | High-confidence fraud flags — she only wants to see claims with composite scores above 70, backed by multi-pattern evidence. Network analysis showing provider/claimant relationships across claims. Concise, data-driven summaries (not lengthy narratives). Statistical context: "This provider's average billing is 3.1x the regional mean, appearing on 23 claims in 6 months." Exportable investigation packages she can hand directly to law enforcement or submit to the state fraud bureau. Ability to drill into specific patterns and evidence without reading an entire report. |
| **Tech Comfort** | Medium — uses specialized SIU software (ISO ClaimSearch, NICB databases) daily but prefers structured data and tables over conversational interfaces. Values precision and reliability over novelty. |
| **Typical Caseload** | Complex organized fraud: provider rings billing $500K+ across dozens of claims, staged accident operations with recruited participants, fleet policy manipulation schemes. Also handles escalations from junior analysts who encounter claims beyond their experience level. |

### 3.2 User Stories

| ID | As a... | I want to... | So that... | Scope |
|----|---------|-------------|-----------|-------|
| US-01 | Junior SIU Analyst (Kevin) | submit a batch of commercial auto claims and have the system automatically analyze all of them for fraud patterns | I don't have to manually review each claim against a red-flag checklist, which takes 1–3 hours per claim | **MVP** |
| US-02 | Junior SIU Analyst (Kevin) | see a clear explanation of why each claim was flagged, with specific evidence for each fraud pattern | I can understand what makes a claim suspicious and learn to recognize these patterns myself, instead of just seeing a score with no explanation | **MVP** |
| US-03 | Junior SIU Analyst (Kevin) | get investigation reports with recommended next steps for each flagged claim | I can start my investigation immediately with an evidence-based action plan instead of spending 4–6 hours assembling background information | **MVP** |
| US-04 | Senior SIU Lead (Diana) | see a ranked list of claims sorted by composite fraud score with confidence levels | I can focus my team's limited investigation resources on the highest-probability fraud cases, not waste time on likely false positives | **MVP** |
| US-05 | Senior SIU Lead (Diana) | query pipeline results to drill into specific claims, patterns, or providers | I can quickly assess whether a flagged claim merits full investigation or can be closed, without reading the entire batch report | **MVP** |
| US-06 | Senior SIU Lead (Diana) | detect provider network anomalies that span multiple claims in the batch | I can identify organized fraud rings that no individual claim review would catch — this is where the biggest losses are | **MVP** |
| US-07 | Any SIU Investigator | check the status of a running pipeline ("How many claims processed so far?") | I know when the batch analysis will be complete and can plan my work accordingly | **MVP** |
| US-08 | Any SIU Investigator | have the system adapt its response detail to my experience level when I query results | Kevin gets detailed explanations with term definitions, and I (Diana) get concise statistical summaries — without either of us changing settings | **MVP** |
| US-09 | Senior SIU Lead (Diana) | see additional fraud patterns for staged accidents and policy manipulation on flagged claims | my team can detect sophisticated fraud schemes that go beyond the basic four patterns | **Stretch** |

---

## 4. Functional Requirements

### 4.1 Core Features

#### F1: Batch Claim Intake & Validation

- The Fraud Pipeline Orchestrator accepts a JSON file containing a batch of commercial auto claims (5–50 claims per batch)
- Each claim in the batch includes: claim ID, claimant info, loss date, report date, loss description, claimed amount, vehicle info (VIN, year, make, model), provider info (repair shop, medical provider, attorney), policy number, and coverage type
- Validates each claim in the batch:
  - Required fields present (claim ID, claimant, loss date, claimed amount, vehicle VIN, policy number)
  - Data type validation (dates parseable, amounts numeric, VIN format correct)
  - Cross-field consistency (report date ≥ loss date, claimed amount > 0, vehicle year within reasonable range)
- Returns validation summary: total claims, valid claims, invalid claims with specific errors
- Invalid claims are logged but do not block processing of valid claims in the batch
- **Scope:** MVP

#### F2: Claim Data Enrichment

The Claim Data Enrichment Agent augments each raw claim with contextual data from mock data sources (simulating real SIU databases). For each claim, the agent calls four enrichment tools:

| Enrichment Tool | Data Retrieved | Fraud Relevance |
|----------------|---------------|-----------------|
| **Claimant History Lookup** | Prior claims filed by this claimant (past 5 years), prior fraud flags, claim frequency, total prior payouts | Identifies serial claimants, prior fraud involvement, unusually high claim frequency |
| **Provider Lookup** | Provider billing history (average charge, claim count, license status), referral network (which providers refer to each other), geographic service area | Detects inflated billing, unlicensed providers, suspicious referral rings |
| **Vehicle Valuation** | Market value for the vehicle (year/make/model/mileage), regional repair cost benchmarks, salvage value | Identifies claims where repair costs exceed vehicle value, inflated damage estimates |
| **Policy Detail Retrieval** | Policy inception date, recent policy changes (additions, coverage upgrades), premium payment history, coverage limits | Detects claims filed shortly after policy inception, suspicious policy changes before a loss |

- All four enrichment tools are called via ReAct pattern (the agent reasons about what data to gather, calls tools, and incorporates results)
- Enriched data is attached to the claim record for downstream fraud pattern analysis
- **Scope:** MVP

#### F3: Parallel Fraud Pattern Analysis

The Fraud Pattern Analyzer is the analytical core of the pipeline. For each enriched claim, it runs **4 fraud patterns in parallel** (MVP), expandable to 6 (Stretch). Each pattern is an independent analysis that produces a sub-score and evidence list.

**MVP Fraud Patterns (4):**

| # | Pattern | Description | Red Flags Analyzed | Score Range | Scope |
|---|---------|-------------|-------------------|-------------|-------|
| 1 | **Duplicate / Similar Claims** | Detects claims that are identical or suspiciously similar to other claims in the batch or in claimant history — same vehicle, same loss type, overlapping dates, matching descriptions | Same VIN appearing on multiple claims; loss descriptions with >80% textual similarity; overlapping loss dates within 30 days; same claimant filing similar claims across different policies or carriers; identical damage amounts on different claims | 0–100 | **MVP** |
| 2 | **Suspicious Timing** | Analyzes the temporal relationship between the claim and key policy/behavioral events — claims filed at statistically improbable times relative to policy lifecycle or claimant behavior | Claim within 30 days of policy inception ("new policy fraud"); claim within 60 days of policy expiration or cancellation; loss occurs on Friday night, reported Monday (weekend staging); claim filed within 72 hours of coverage upgrade or vehicle addition; seasonal anomaly (e.g., ice-related claim in July); loss date on a holiday or long weekend | 0–100 | **MVP** |
| 3 | **Inflated Amounts** | Compares claimed amounts against market benchmarks and evaluates whether the financial magnitude of the claim is proportionate to the described incident | Repair estimate exceeds vehicle market value (total loss threshold); repair cost >2x regional benchmark for described damage; medical billing disproportionate to injury description (e.g., $50K medical for a low-speed parking lot collision); rental car duration >3x expected repair time; lost income claims exceeding documented earnings; parts pricing above manufacturer MSRP | 0–100 | **MVP** |
| 4 | **Provider Network Anomalies** | Evaluates the providers involved in the claim (repair shops, medical facilities, attorneys, towing companies) for patterns consistent with organized fraud networks | Provider appears on >15% of claims in the batch or in recent history; provider billing average >2x regional mean; circular referral pattern (body shop → clinic → attorney → same body shop); provider operating outside normal geographic service area; provider license expired, suspended, or under investigation; multiple claimants in the same batch using the same unusual provider combination | 0–100 | **MVP** |

**Stretch Fraud Patterns (2 additional):**

| # | Pattern | Description | Red Flags Analyzed | Score Range | Scope |
|---|---------|-------------|-------------------|-------------|-------|
| 5 | **Staged Accident Indicators** | Detects patterns consistent with manufactured or intentionally caused collisions — a hallmark of organized auto fraud rings | Multiple passengers in vehicle with no prior injury history; all claimants share common address or phone number; witnesses with connections to claimants; damage inconsistent with described collision mechanics (e.g., rear-end damage on a "head-on collision"); vehicle has history of similar incidents; accident location is a known staging area (high-fraud zip code) | 0–100 | **Stretch** |
| 6 | **Policy Manipulation** | Identifies suspicious policy activity preceding or surrounding the claim — indicating the policyholder may have engineered coverage to enable a fraudulent claim | Vehicle added to fleet policy within 72 hours of reported loss; coverage limits increased within 30 days of a large claim; policy reinstated after lapse, then immediate claim; premium payment made same day as claim filing (catching up to activate lapsed coverage); multiple policy changes in 90-day window preceding claim; fleet size suddenly increased before a multi-vehicle claim | 0–100 | **Stretch** |

**Composite Fraud Score Calculation:**
- Each pattern produces a sub-score (0–100) and an evidence list
- The composite fraud score is a weighted average of all pattern sub-scores:
  - Duplicate/Similar Claims: 30% weight
  - Suspicious Timing: 20% weight
  - Inflated Amounts: 25% weight
  - Provider Network Anomalies: 25% weight
- Confidence level based on evidence density: **High** (3+ patterns flagged with evidence), **Medium** (2 patterns flagged), **Low** (1 pattern flagged or weak evidence)
- **Scope:** MVP (4 patterns) / Stretch (6 patterns, re-weighted)

#### F4: Fraud Scoring & Ranking

- After all claims in the batch are analyzed, the Orchestrator collects results and produces a ranked list:
  - Claims sorted descending by composite fraud score
  - Each claim tagged with a **priority tier**:

| Tier | Fraud Score Range | Action | Color Code |
|------|------------------|--------|------------|
| **Critical** | 80–100 | Immediate SIU investigation. Full evidence package generated. Likely organized or large-loss fraud. | Red |
| **High** | 60–79 | SIU investigation recommended. Multiple fraud indicators present. Investigation report generated. | Orange |
| **Medium** | 40–59 | Enhanced monitoring. Some indicators present but may be coincidental. Summary report generated. | Yellow |
| **Low** | 0–39 | No immediate action. Claim appears clean or has only minor anomalies. Logged for historical pattern analysis. | Green |

- Batch-level statistics: total claims analyzed, claims per tier, average fraud score, top patterns triggered, provider frequency analysis
- **Scope:** MVP

#### F5: Investigation Report Generation

The Investigation Report Generator produces a structured investigation report for every claim scoring **Medium or above** (≥40). Reports are tailored for SIU case files.

**Report Structure:**

| Section | Content | Purpose |
|---------|---------|---------|
| **Case Header** | Claim ID, claimant name, loss date, claimed amount, policy number, composite fraud score, priority tier, confidence level | Quick identification and triage |
| **Executive Summary** | 2–3 sentence overview of why this claim is suspicious and which patterns triggered | Senior reviewer quick-read |
| **Pattern Analysis Detail** | For each triggered pattern: sub-score, specific evidence found, comparison to benchmarks, severity assessment | Investigator deep-dive |
| **Evidence Summary** | Consolidated list of all evidence items across patterns, with source attribution (e.g., "claimant history shows 4 prior claims in 18 months — source: claimant_history_lookup") | Documentation for case file |
| **Recommended Actions** | Specific investigative steps (e.g., "Request itemized repair invoice from provider X," "Pull recorded statement from claimant," "Cross-reference provider Y billing across all 2024 claims") | Guide investigation |
| **Risk Factors** | Key risk indicators ranked by severity | Quick reference for SIU lead |
| **Claimant & Provider Profile** | Summary of enrichment data — claimant history snapshot, provider billing profile, vehicle valuation comparison | Background context |

- Reports output as structured JSON (for programmatic access) and rendered as Markdown (for human reading in the query chat)
- **Scope:** MVP

#### F6: Status / Query Interface

The Fraud Pipeline Orchestrator provides a lightweight status and query chat for SIU investigators, available via ADK Web. This is NOT the primary interaction mode — the pipeline runs non-conversationally — but investigators can use the chat to:

| Query Type | Example Queries | Response |
|-----------|----------------|----------|
| **Pipeline Status** | "How many claims have been processed?" / "Is the batch done?" / "Status" | Progress update: "Processing batch 'Weekly Auto Claims': 6/8 claims complete. Estimated time remaining: ~45 seconds." |
| **Batch Summary** | "Show me the results" / "Give me the summary" / "How many fraud flags?" | Batch summary: claim count per tier, top fraud scores, key statistics |
| **Claim Drill-Down** | "Tell me about Claim CLM-2024-0847" / "Why was Claim 5 flagged?" / "Show evidence for the highest-scoring claim" | Full detail for the specified claim: fraud score, pattern breakdown, evidence list, recommended actions |
| **Pattern Query** | "Which claims had provider network issues?" / "Show me all timing-flagged claims" / "What's the most common pattern in this batch?" | Filtered list of claims matching the specified pattern |
| **Provider Query** | "Which providers appear multiple times?" / "Tell me about Elite Auto Body" | Provider frequency analysis, billing comparison to benchmarks |
| **Explanation** | "What does the duplicate claim score mean?" / "How is the composite score calculated?" | Educational explanation adapted to user profile (detailed for Kevin, brief for Diana) |

- Adapts response style based on user profile (Kevin gets step-by-step explanations and term definitions; Diana gets concise tables and statistics)
- Status queries work during pipeline execution; result queries work after pipeline completes
- **Scope:** MVP

### 4.2 Feature Interaction Map

```
                    PIPELINE FLOW (Non-Conversational)
                    ══════════════════════════════════

Claim Batch JSON ──► [F1: Batch Intake & Validate]
                          │
                          │  (N claims in parallel — Level 1 parallelism)
                          │
                          ├──► Claim 1 ──► [F2: Enrich] ──► [F3: Analyze 4 Patterns ║] ──► Score
                          ├──► Claim 2 ──► [F2: Enrich] ──► [F3: Analyze 4 Patterns ║] ──► Score
                          ├──► Claim 3 ──► [F2: Enrich] ──► [F3: Analyze 4 Patterns ║] ──► Score
                          ├──► ...
                          └──► Claim N ──► [F2: Enrich] ──► [F3: Analyze 4 Patterns ║] ──► Score
                                                                    │
                         ┌──────────────────────────────────────────┘
                         │  Level 2 parallelism (per claim):
                         │    ├── Duplicate/Similar Claims ──► sub-score + evidence
                         │    ├── Suspicious Timing ──────────► sub-score + evidence
                         │    ├── Inflated Amounts ───────────► sub-score + evidence
                         │    └── Provider Network Anomalies ─► sub-score + evidence
                         │                │
                         │                ▼
                         │         Composite Fraud Score
                         ▼
                   [F4: Score, Rank & Tier] ──► [F5: Generate Investigation Reports]
                          │
                          ▼
                   Batch Results (JSON) + Reports (Markdown)


                    STATUS / QUERY CHAT (Available in parallel)
                    ═══════════════════════════════════════════

                   [F6: Status/Query Interface via ADK Web]
                          │
                          ├── Pipeline status (during execution)
                          ├── Batch summary (after completion)
                          ├── Claim drill-down
                          ├── Pattern queries
                          ├── Provider queries
                          └── [User Profiling] adapts all responses
```

---

## 5. Non-Functional Requirements

| # | Requirement | Target | Notes |
|---|------------|--------|-------|
| NFR-01 | Per-claim fraud analysis time (enrichment + 4 patterns + scoring) | < 30 seconds | Includes enrichment tool calls + 4 parallel pattern analyses + score computation |
| NFR-02 | Full batch processing time (8 claims) | < 5 minutes | With dual-level parallelism; 8 claims × 4 patterns each processed concurrently |
| NFR-03 | Batch-level parallelism | 5–50 claims concurrently | Limited by Gemini API rate limits; use `asyncio.gather` for claim-level fan-out |
| NFR-04 | Pattern-level parallelism | 4 patterns per claim concurrently | Nested `asyncio.gather` within each claim pipeline |
| NFR-05 | Status/query chat response time | < 10 seconds | For status checks during pipeline and result queries after completion |
| NFR-06 | Investigation report generation time | < 15 seconds per flagged claim | Flash model generates reports from structured fraud analysis output |
| NFR-07 | Tracing coverage | 100% of LLM calls | All calls instrumented via OpenTelemetry → Arize Cloud |
| NFR-08 | Prompt template versioning | All prompts versioned | Via dict-based Prompt Manager module; every template versioned and retrievable |
| NFR-09 | Fraud score consistency | Same inputs → same composite score | Deterministic scoring given identical claim data and enrichment results |
| NFR-10 | Graceful degradation (batch-level) | Fallback to sequential claim processing | If parallel execution fails (rate limits), process claims one at a time |
| NFR-11 | Graceful degradation (pattern-level) | Fallback to sequential pattern analysis | If nested parallelism fails, run 4 patterns sequentially per claim |
| NFR-12 | Data persistence | JSON file-based | Claim batches, enrichment results, fraud scores, reports, and user profiles persist as JSON files. **No SQLite.** |
| NFR-13 | False positive reduction | < 30% false positive rate | Multi-pattern weighted scoring should significantly outperform rules-based 60–70% false positive rate |
| NFR-14 | Evidence traceability | Every fraud flag linked to specific evidence | No "black box" scores — every sub-score backed by enumerated evidence items with source attribution |

---

## 6. System Architecture

### 6.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              PRESENTATION LAYER                                   │
│                                                                                   │
│   ┌────────────────────────────────────────────────────────────────────────┐     │
│   │                    ADK Web UI (built-in `adk web`)                      │     │
│   │                                                                        │     │
│   │   PRIMARY: Non-conversational pipeline trigger                         │     │
│   │     - Submit claim batch JSON path to start pipeline                   │     │
│   │     - Pipeline runs autonomously (no human interaction needed)         │     │
│   │                                                                        │     │
│   │   SECONDARY: Status / Query chat for SIU investigators                 │     │
│   │     - "Status?" → pipeline progress                                    │     │
│   │     - "Show results" → batch summary + ranked claims                   │     │
│   │     - "Tell me about Claim X" → drill-down with evidence              │     │
│   │     - "Which providers appear multiple times?" → cross-claim query     │     │
│   │     - Response style adapts to investigator profile (Kevin vs Diana)   │     │
│   │                                                                        │     │
│   └──────────────────────────────────┬─────────────────────────────────────┘     │
│                                      │                                           │
└──────────────────────────────────────┼───────────────────────────────────────────┘
                                       │
                                       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                             AGENT LAYER (ADK + A2A)                               │
│                                                                                   │
│   ┌────────────────────────────────────────────────────────────────────────┐     │
│   │      FRAUD PIPELINE ORCHESTRATOR (Gemini 2.5 Pro, Main Agent)          │     │
│   │                                                                        │     │
│   │   PIPELINE MODE (Non-Conversational):                                  │     │
│   │     1. Accept batch JSON → validate N claims                           │     │
│   │     2. Fan out N claims in parallel (Level 1 parallelism)              │     │
│   │        Per claim: Enrich → Analyze (4 patterns ║) → Score              │     │
│   │     3. Collect results → rank by fraud score → assign tiers            │     │
│   │     4. Trigger report generation for flagged claims (≥40 score)        │     │
│   │                                                                        │     │
│   │   QUERY MODE (Conversational, for SIU):                                │     │
│   │     - Status checks, result queries, claim drill-downs                 │     │
│   │     - User profiling (Kevin=detailed, Diana=concise)                   │     │
│   │                                                                        │     │
│   └──┬────────────────────┬────────────────────┬───────────────────────────┘     │
│      │ A2A                │ A2A                │ A2A                              │
│      │ (per claim,        │ (per claim,        │ (per flagged                     │
│      │  N in parallel)    │  N in parallel,    │  claim)                          │
│      │                    │  4 patterns each)  │                                  │
│      ▼                    ▼                    ▼                                  │
│  ┌──────────────┐   ┌──────────────────┐   ┌──────────────────┐                 │
│  │ CLAIM DATA   │   │ FRAUD PATTERN    │   │ INVESTIGATION    │                 │
│  │ ENRICHMENT   │   │ ANALYZER         │   │ REPORT           │                 │
│  │ AGENT        │   │                  │   │ GENERATOR        │                 │
│  │              │   │ (Gemini 2.5 Pro) │   │                  │                 │
│  │ (Gemini 2.5  │   │                  │   │ (Gemini 2.5      │                 │
│  │  Flash)      │   │ Per claim, runs  │   │  Flash)          │                 │
│  │              │   │ 4 patterns in ║: │   │                  │                 │
│  │ Task-based:  │   │                  │   │ Task-based:      │                 │
│  │ • Claimant   │   │ ┌──────────────┐ │   │ • Case header    │                 │
│  │   history    │   │ │ Duplicate/   │ │   │ • Executive      │                 │
│  │ • Provider   │   │ │ Similar      │ │   │   summary        │                 │
│  │   lookup     │   │ ├──────────────┤ │   │ • Pattern detail │                 │
│  │ • Vehicle    │   │ │ Suspicious   │ │   │ • Evidence list  │                 │
│  │   valuation  │   │ │ Timing       │ │   │ • Recommended    │                 │
│  │ • Policy     │   │ ├──────────────┤ │   │   actions        │                 │
│  │   details    │   │ │ Inflated     │ │   │ • Risk factors   │                 │
│  │              │   │ │ Amounts      │ │   │ • Profiles       │                 │
│  │ (ReAct:      │   │ ├──────────────┤ │   │                  │                 │
│  │  reason +    │   │ │ Provider     │ │   │ (Structured      │                 │
│  │  tool calls) │   │ │ Network      │ │   │  JSON + Markdown │                 │
│  │              │   │ │ Anomalies    │ │   │  output)         │                 │
│  │              │   │ └──────────────┘ │   │                  │                 │
│  │              │   │                  │   │                  │                 │
│  │              │   │ → Composite      │   │                  │                 │
│  │              │   │   fraud score    │   │                  │                 │
│  │              │   │   + evidence     │   │                  │                 │
│  └──────┬───────┘   └────────┬─────────┘   └────────┬─────────┘                 │
│         │                    │                       │                            │
└─────────┼────────────────────┼───────────────────────┼────────────────────────────┘
          │                    │                       │
          ▼                    ▼                       ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                              SERVICES LAYER                                       │
│                                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │ Prompt       │  │ User Profile │  │ Mock Fraud   │  │ Arize Cloud      │    │
│  │ Manager      │  │ Store        │  │ Data APIs    │  │ + OTel           │    │
│  │              │  │              │  │              │  │                  │    │
│  │ - Templates  │  │ - JSON files │  │ - Claimant   │  │ - Trace all LLM  │    │
│  │   per agent  │  │ - Kevin      │  │   history    │  │   calls          │    │
│  │ - Versions   │  │   (junior)   │  │ - Provider   │  │ - Fraud score    │    │
│  │ - format_map │  │ - Diana      │  │   lookup     │  │   distributions  │    │
│  │ - Fraud-     │  │   (senior)   │  │ - Vehicle    │  │ - Pattern        │    │
│  │   specific   │  │ - Experience │  │   valuation  │  │   trigger rates  │    │
│  │   templates  │  │   level,     │  │ - Policy     │  │ - Annotation     │    │
│  │              │  │   preferred  │  │   details    │  │   queue for      │    │
│  │              │  │   detail     │  │              │  │   fraud review   │    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘    │
│                                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────┐    │
│  │ Fraud Data Store (JSON Files)                                             │    │
│  │                                                                           │    │
│  │ - claim_batches/        → input claim batch JSON files                    │    │
│  │ - enriched_claims/      → per-claim enrichment results                    │    │
│  │ - fraud_analysis/       → per-claim pattern scores + evidence             │    │
│  │ - investigation_reports/→ generated investigation reports (JSON + MD)      │    │
│  │ - batch_results/        → batch summaries, rankings, statistics           │    │
│  │ - user_profiles/        → SIU investigator profiles (Kevin, Diana)        │    │
│  │ - mock_data/            → claimant histories, provider records,           │    │
│  │                           vehicle valuations, policy records               │    │
│  └──────────────────────────────────────────────────────────────────────────┘    │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                            DEPLOYMENT LAYER                                       │
│                                                                                   │
│  ┌───────────────────────────────────┐  ┌──────────────────────────────────────┐│
│  │ Vertex AI Agent Engine            │  │ Local ADK Server (Fallback)          ││
│  │ - Serverless scaling for batch    │  │ - adk web / adk run                 ││
│  │   workloads                       │  │ - Local development & testing        ││
│  │ - Cloud Logging + Monitoring      │  │ - Full pipeline execution locally    ││
│  │ - Production deployment target    │  │ - Demo mode                         ││
│  └───────────────────────────────────┘  └──────────────────────────────────────┘│
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Dual-Level Parallelism Detail

The architecture features nested parallelism that is central to the system's throughput:

```
LEVEL 1 — Batch Parallelism (N claims simultaneously)
═══════════════════════════════════════════════════════

  asyncio.gather(
      process_claim(claim_1),     ─────►  Enrich → Analyze → Score → Report
      process_claim(claim_2),     ─────►  Enrich → Analyze → Score → Report
      process_claim(claim_3),     ─────►  Enrich → Analyze → Score → Report
      ...
      process_claim(claim_N),     ─────►  Enrich → Analyze → Score → Report
  )

LEVEL 2 — Pattern Parallelism (4 patterns per claim, within each claim pipeline)
═════════════════════════════════════════════════════════════════════════════════

  Inside each process_claim():

      enriched_data = await enrich_claim(claim)       # Sequential: must complete first

      pattern_results = await asyncio.gather(          # Parallel: 4 patterns at once
          analyze_duplicate_similar(enriched_data),
          analyze_suspicious_timing(enriched_data),
          analyze_inflated_amounts(enriched_data),
          analyze_provider_network(enriched_data),
      )

      composite_score = compute_fraud_score(pattern_results)   # Sequential: needs all patterns
      report = await generate_report(claim, composite_score)   # Sequential: needs score

RESULT: 8 claims × 4 patterns = 32 analyses, but effective parallelism reduces wall-clock
        time to approximately max(per-claim time) rather than sum(all analyses).
```

### 6.3 Tech Stack Summary

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Agent Framework | Google ADK (Python) | Required tech stack; native Gemini integration, sub-agent support, pipeline orchestration primitives |
| LLM (reasoning) | Gemini 2.5 Pro | Fraud Pipeline Orchestrator (complex batch coordination, query response generation) and Fraud Pattern Analyzer (multi-dimensional fraud reasoning requiring CoT analysis) |
| LLM (speed) | Gemini 2.5 Flash | Claim Data Enrichment Agent (structured tool calling, fast enrichment) and Investigation Report Generator (high-throughput report generation from structured data) — speed-critical in batch pipeline |
| Agent Communication | A2A Protocol | Required tech stack; Orchestrator delegates enrichment, analysis, and report generation tasks to sub-agents via A2A; each agent publishes an Agent Card |
| UI | **ADK Web** (built-in `adk web`) | Zero build time; ADK provides chat UI for pipeline trigger and SIU query interface. **No Streamlit.** |
| Storage | JSON files | Simplest possible for demo; claim batches, enrichment results, fraud scores, reports, and user profiles as JSON files. **No SQLite.** |
| Prompt Management | Custom Python module (dict-based, `str.format_map`) | Lightweight, demonstrates concept clearly; versioned templates for fraud-specific prompts. **No Jinja2.** |
| Monitoring | Arize Cloud + OpenTelemetry | Required tech stack; cloud-hosted observability for all pipeline traces, fraud score distributions, pattern trigger rates, and annotation queues |
| Deployment | Vertex AI Agent Engine / Local ADK | GCP Agent Engine if available, local `adk web` fallback for development and demo |

---

## 7. Agent Specifications (ADK)

### 7.1 Fraud Pipeline Orchestrator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `fraud_pipeline_orchestrator` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Ingests claim batches, orchestrates dual-level parallel processing (N claims × 4 fraud patterns), aggregates results, and provides a lightweight status/query chat for SIU investigators |
| **Mode** | Dual: Non-conversational (batch pipeline processing) + Conversational (status queries and result drill-down) |

**Responsibilities:**
1. Accept a batch JSON file containing N commercial auto claims and validate all claims (required fields, data types, cross-field consistency)
2. Fan out parallel processing for each valid claim at the batch level (Level 1 parallelism — N claims simultaneously)
3. For each claim, orchestrate the sequential per-claim pipeline: Enrichment → Parallel Fraud Analysis (4 patterns) → Composite Scoring → Investigation Report
4. Within each claim's fraud analysis step, coordinate Level 2 parallelism — 4 fraud pattern sub-tasks running simultaneously via the Fraud Pattern Analyzer
5. Aggregate all per-claim results into a batch summary: claims ranked by composite fraud score, tier distribution (Critical/High/Medium/Low), top triggered patterns, provider frequency analysis
6. After pipeline completion, enter status/query chat mode for SIU investigators to check progress, query results, drill into specific claims, and ask about fraud patterns
7. Handle drill-down queries with user-profile-adaptive responses (detailed explanations for Kevin, concise statistical summaries for Diana)

**ADK Tools (Functions):**

```python
# Tool 1: Ingest and validate a batch of commercial auto claims
def ingest_claim_batch(batch_json: str) -> dict:
    """
    Parses a JSON string or file path containing a batch of commercial auto claims.
    Validates each claim for required fields (claim_id, claimant, loss_date,
    claimed_amount, vehicle_vin, policy_number), correct data types (dates parseable,
    amounts numeric, VIN format), and cross-field consistency (report_date >= loss_date,
    claimed_amount > 0).
    Returns: {
        'batch_id': str,
        'batch_name': str,
        'total_claims': int,
        'valid_claims': list[dict],
        'invalid_claims': list[{'claim_id': str, 'errors': list[str]}],
        'validation_summary': str
    }
    """
    pass

# Tool 2: Delegate enrichment to the Claim Data Enrichment Agent
def enrich_claim(claim_id: str) -> dict:
    """
    Sends a single validated claim to the Claim Data Enrichment Agent for
    contextual data augmentation. The Enrichment Agent calls 4 mock APIs
    (claimant history, provider network, vehicle valuation, policy details)
    and returns a structured enriched data package.
    Returns: {
        'claim_id': str,
        'claimant_history': dict,
        'provider_network': dict,
        'vehicle_valuation': dict,
        'policy_details': dict,
        'enrichment_status': 'complete' | 'partial' | 'failed',
        'enrichment_timestamp': str
    }
    """
    pass

# Tool 3: Delegate fraud analysis to the Fraud Pattern Analyzer Agent
def analyze_fraud_patterns(claim_id: str, enriched_data: dict) -> dict:
    """
    Sends an enriched claim to the Fraud Pattern Analyzer Agent, which runs
    4 fraud detection patterns in parallel (Duplicate/Similar Claims,
    Suspicious Timing, Inflated Amounts, Provider Network Anomalies).
    Returns: {
        'claim_id': str,
        'pattern_results': {
            'duplicate_similar': {'score': float, 'evidence': list, 'reasoning': str},
            'suspicious_timing': {'score': float, 'evidence': list, 'reasoning': str},
            'inflated_amounts': {'score': float, 'evidence': list, 'reasoning': str},
            'provider_network': {'score': float, 'evidence': list, 'reasoning': str}
        },
        'composite_score': float,
        'confidence': 'high' | 'medium' | 'low',
        'priority_tier': 'critical' | 'high' | 'medium' | 'low'
    }
    """
    pass

# Tool 4: Delegate report generation to the Investigation Report Generator Agent
def generate_investigation_report(claim_id: str, fraud_analysis: dict) -> str:
    """
    Sends fraud analysis results to the Investigation Report Generator Agent,
    which produces a formatted investigation report with case header,
    executive summary, pattern analysis detail, evidence summary,
    recommended actions, and risk factors.
    Returns: Markdown-formatted investigation report string.
    """
    pass

# Tool 5: Search and filter batch results for SIU query chat
def query_results(query: str, batch_id: str) -> str:
    """
    Searches stored batch results to answer SIU investigator queries.
    Supports query types: pipeline status, batch summary, claim drill-down,
    pattern filtering, provider analysis, and explanation requests.
    Adapts response format based on the active user profile (detailed for
    junior analysts, concise for senior leads).
    Returns: formatted response string tailored to the query and user profile.
    """
    pass
```

**System Instruction (Summary):**
```
You are the Fraud Pipeline Orchestrator for a commercial auto insurance fraud
detection system. You operate in two modes:

MODE 1 — BATCH PIPELINE (Non-Conversational):
When the user provides a claim batch JSON file or data, you:
1. Ingest and validate all claims using ingest_claim_batch
2. For each valid claim, execute the per-claim pipeline:
   a. Enrich the claim (enrich_claim → Claim Data Enrichment Agent)
   b. Analyze 4 fraud patterns in parallel (analyze_fraud_patterns → Fraud Pattern Analyzer)
   c. Generate investigation report for claims scoring ≥40 (generate_investigation_report → Report Generator)
3. Aggregate all results: rank claims by composite fraud score, compute tier distribution,
   identify top triggered patterns, flag providers appearing across multiple claims
4. Present the batch summary and transition to query mode

MODE 2 — STATUS/QUERY CHAT (Conversational):
During or after batch processing, SIU investigators can ask:
- Pipeline status: "How many claims processed?" → Progress update
- Batch summary: "Show results" → Tier distribution, top scores, key statistics
- Claim drill-down: "Tell me about CLM-2024-0847" → Full detail with pattern breakdown
- Pattern query: "Which claims had provider issues?" → Filtered list
- Provider query: "Which providers appear multiple times?" → Frequency analysis
- Explanation: "How is the composite score calculated?" → Educational explanation

Adapt responses based on user profile:
- Junior analyst (Kevin): Detailed explanations, define SIU terminology, step-by-step evidence walkthroughs
- Senior lead (Diana): Concise tables, statistical summaries, confidence-flagged highlights only

NEVER fabricate evidence or scores. All fraud assessments must be backed by
specific data from the enrichment and analysis pipeline.
```

**Why Gemini Pro:** The Orchestrator must coordinate two layers of parallel execution (N claims × 4 patterns per claim), manage the sequential dependency chain (enrich → analyze → score → report) within each parallel branch, aggregate complex multi-dimensional results into coherent summaries, and then switch to conversational mode for nuanced SIU queries that require reasoning about evidence, comparing claims, and adapting explanations to different user expertise levels. Pro's superior instruction-following for complex multi-step orchestration and its contextual reasoning for query interpretation make it essential for this role.

### 7.2 Claim Data Enrichment Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `claim_data_enrichment` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Enriches each raw claim with contextual data from mock external sources — claimant history, provider network information, vehicle valuation, and policy details |
| **Mode** | Non-conversational (task-based); runs in parallel — one instance per claim in the batch |

**Responsibilities:**
1. Receive a single raw claim from the Orchestrator
2. Determine which enrichment data sources are relevant for this claim (all 4 are called for MVP)
3. Call mock APIs via ReAct pattern: reason about what data to gather, call tools sequentially (claimant history → provider network → vehicle valuation → policy details), and incorporate results at each step
4. Structure the enriched data package with all contextual information attached to the claim record, ready for downstream fraud pattern analysis

**ADK Tools (Functions):**

```python
# Tool 1: Look up claimant's claim history and associated records
def lookup_claimant_history(claimant_id: str) -> dict:
    """
    Retrieves the claimant's history from the mock claims database.
    Returns: {
        'claimant_id': str,
        'total_prior_claims': int,
        'prior_claims': list[{
            'claim_id': str, 'loss_date': str, 'loss_type': str,
            'claimed_amount': float, 'outcome': str, 'fraud_flag': bool
        }],
        'known_addresses': list[str],
        'associated_vehicles': list[{'vin': str, 'year': int, 'make': str, 'model': str}],
        'associated_policies': list[str],
        'prior_fraud_flags': int,
        'claim_frequency_per_year': float,
        'total_prior_payouts': float
    }
    """
    pass

# Tool 2: Look up provider billing history and network connections
def lookup_provider_network(provider_id: str, provider_type: str) -> dict:
    """
    Retrieves provider information from the mock provider database.
    provider_type: 'repair_shop', 'medical', 'attorney', 'towing'
    Returns: {
        'provider_id': str,
        'provider_name': str,
        'provider_type': str,
        'license_status': 'active' | 'expired' | 'suspended' | 'under_investigation',
        'total_claims_served': int,
        'average_billing_amount': float,
        'regional_average_billing': float,
        'billing_ratio': float,  # provider avg / regional avg
        'referral_connections': list[{'provider_id': str, 'provider_name': str, 'shared_claims': int}],
        'geographic_service_area': str,
        'claims_outside_service_area': int,
        'flagged_claims_count': int
    }
    """
    pass

# Tool 3: Look up vehicle market value and repair cost benchmarks
def lookup_vehicle_valuation(vin: str, year: int, make: str, model: str) -> dict:
    """
    Retrieves vehicle valuation data from the mock vehicle database.
    Returns: {
        'vin': str,
        'year': int,
        'make': str,
        'model': str,
        'market_value': float,
        'salvage_value': float,
        'total_loss_threshold': float,  # typically 70-75% of market value
        'regional_repair_benchmarks': {
            'minor_collision': {'low': float, 'median': float, 'high': float},
            'major_collision': {'low': float, 'median': float, 'high': float},
            'comprehensive': {'low': float, 'median': float, 'high': float}
        },
        'prior_claims_on_vin': list[{'claim_id': str, 'date': str, 'amount': float}],
        'title_history': list[str]  # e.g., 'clean', 'salvage', 'rebuilt'
    }
    """
    pass

# Tool 4: Look up policy details, coverage, and recent changes
def lookup_policy_details(policy_number: str) -> dict:
    """
    Retrieves policy information from the mock policy administration system.
    Returns: {
        'policy_number': str,
        'inception_date': str,
        'expiration_date': str,
        'days_since_inception': int,
        'days_until_expiration': int,
        'coverage_type': str,
        'coverage_limits': dict,
        'premium_amount': float,
        'premium_payment_history': list[{'date': str, 'amount': float, 'status': str}],
        'recent_policy_changes': list[{
            'change_date': str, 'change_type': str, 'details': str
        }],
        'vehicles_on_policy': list[{'vin': str, 'date_added': str}],
        'policy_status': 'active' | 'lapsed' | 'cancelled' | 'reinstated',
        'lapse_history': list[{'lapse_date': str, 'reinstate_date': str}]
    }
    """
    pass
```

**System Instruction (Summary):**
```
You are a Claim Data Enrichment specialist for commercial auto insurance fraud
detection. Your job is to augment raw claim data with contextual information
from external data sources.

For each claim you receive, you must:
1. Look up claimant history — prior claims, fraud flags, addresses, associated vehicles
2. Look up provider information — billing patterns, license status, referral network
3. Look up vehicle valuation — market value, salvage value, repair benchmarks
4. Look up policy details — inception date, recent changes, coverage, premium history

Use a ReAct approach:
- THOUGHT: "I need to gather claimant history for claimant C-4821 to check for prior fraud flags"
- ACTION: Call lookup_claimant_history("C-4821")
- OBSERVATION: Claimant has 4 prior claims in 18 months, 1 prior fraud flag
- THOUGHT: "High claim frequency and a prior fraud flag — important context. Now I need provider info."
- ACTION: Call lookup_provider_network("PRV-0093", "repair_shop")
- ... and so on for all 4 tools.

Package all enrichment data into a single structured response. Do NOT analyze
for fraud — that is the Fraud Pattern Analyzer's job. Your role is data
retrieval and structuring only.
```

**Why Gemini Flash:** Claim data enrichment is primarily a structured data retrieval task — calling 4 mock APIs, parsing responses, and packaging the results. It does not require complex reasoning or nuanced judgment. Flash's speed advantage is critical because each claim in the batch triggers an independent enrichment call; in a batch of 8 claims running in parallel, 8 enrichment instances execute simultaneously, and latency directly impacts total pipeline throughput.

### 7.3 Fraud Pattern Analyzer Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `fraud_pattern_analyzer` |
| **Model** | Gemini 2.5 Pro |
| **Role** | Runs 4 independent fraud detection patterns in parallel per claim (Level 2 parallelism), produces evidence-based fraud flags with sub-scores, and computes a composite fraud score with confidence level |
| **Mode** | Non-conversational (task-based); runs in parallel — one instance per claim, each instance internally parallelizes 4 pattern sub-tasks |

**Responsibilities:**
1. Receive enriched claim data from the Orchestrator (raw claim + claimant history + provider network + vehicle valuation + policy details)
2. Spawn 4 parallel fraud pattern sub-tasks using `asyncio.gather` (Duplicate/Similar Claims, Suspicious Timing, Inflated Amounts, Provider Network Anomalies)
3. Each pattern sub-task uses Chain-of-Thought reasoning to evaluate evidence, weigh red flags, and produce a sub-score (0–100) with a cited evidence list
4. After all 4 patterns complete, combine sub-scores into a weighted composite fraud score: Duplicate/Similar 30%, Suspicious Timing 20%, Inflated Amounts 25%, Provider Network 25%
5. Assign confidence level based on evidence density: High (3+ patterns flagged with evidence), Medium (2 patterns flagged), Low (1 pattern or weak evidence)
6. Return composite results with full pattern breakdown to the Orchestrator

**Parallel Pattern Sub-Tasks:**

```
Enriched Claim Data
    │
    ├──► [Duplicate/Similar Claims]    ──► sub-score + evidence ──┐
    │      (CoT: check VINs, loss                                 │
    │       descriptions, dates,                                  │
    │       claimant overlap)                                     │
    │                                                             │
    ├──► [Suspicious Timing]           ──► sub-score + evidence ──┤
    │      (CoT: policy inception                                 │
    │       proximity, day-of-week,                               ├──► Combine
    │       pre-cancellation)                                     │    Weighted Avg
    │                                                             │    → Composite
    ├──► [Inflated Amounts]            ──► sub-score + evidence ──┤      Score
    │      (CoT: repair vs market                                 │    → Confidence
    │       value, medical vs injury,                             │    → Priority Tier
    │       rental duration)                                      │
    │                                                             │
    └──► [Provider Network Anomalies]  ──► sub-score + evidence ──┘
           (CoT: provider frequency,
            billing ratios, referral
            rings, geographic anomaly)

    (Stretch: + Staged Accident Indicators, Policy Manipulation)
```

**Chain-of-Thought Prompt Structure for Each Pattern:**

Each of the 4 pattern sub-tasks uses the same CoT prompt structure, customized with pattern-specific evaluation criteria:

```
PATTERN: [Pattern Name]
CLAIM: {claim_id} — {claimant_name} — {loss_date} — ${claimed_amount}

ENRICHED DATA PROVIDED:
{relevant enrichment data for this pattern}

EVALUATE this claim for [pattern name] fraud indicators using step-by-step reasoning:

Step 1: Identify all relevant data points for this pattern
  - List every field from the enriched data that relates to [pattern]
  - Note both suspicious and exculpatory (non-suspicious) indicators

Step 2: Assess each indicator
  - For each data point, determine if it is:
    • A red flag (suspicious, with severity: minor/moderate/major)
    • Neutral (within normal parameters)
    • Exculpatory (actively suggests legitimacy)
  - Cite specific numbers and thresholds

Step 3: Weigh the evidence
  - Count red flags by severity
  - Consider whether multiple minor flags combine into a significant concern
  - Account for exculpatory evidence that may explain apparent red flags
  - Assess the coherence of the fraud narrative (do the flags tell a
    consistent story of fraud, or are they unrelated anomalies?)

Step 4: Score the pattern (0–100)
  - 0–20: Clean — no meaningful indicators
  - 21–40: Minor anomalies — likely coincidental
  - 41–60: Notable concerns — warrants monitoring
  - 61–80: Significant fraud indicators — investigation recommended
  - 81–100: Strong fraud evidence — critical priority

Step 5: Compile evidence list
  - List each piece of evidence that contributed to the score
  - Include specific data values (e.g., "repair estimate $42,000 vs. vehicle market value $28,500 = 1.47x")
  - Attribute each evidence item to its data source

OUTPUT FORMAT:
{
  "pattern": "[pattern_name]",
  "score": <0-100>,
  "severity": "clean|minor|notable|significant|critical",
  "evidence": [
    {"indicator": "...", "value": "...", "benchmark": "...", "severity": "...", "source": "..."},
    ...
  ],
  "reasoning": "Step-by-step reasoning summary...",
  "exculpatory_factors": ["...", "..."]
}
```

**Parallel Execution Code Pattern:**

```python
import asyncio
from google.adk import Agent

# The Fraud Pattern Analyzer's internal parallel execution
async def analyze_claim_fraud_patterns(enriched_claim: dict) -> dict:
    """
    Runs 4 fraud pattern analyses in parallel for a single enriched claim.
    This is Level 2 parallelism — within each claim.
    """

    async def run_pattern(pattern_name: str, pattern_prompt: str, claim_data: dict) -> dict:
        """Execute a single fraud pattern analysis using CoT."""
        # Each pattern runs as an independent LLM call with CoT prompt
        prompt = pattern_prompt.format_map({
            "claim_id": claim_data["claim_id"],
            "claimant_name": claim_data["claimant"]["name"],
            "loss_date": claim_data["loss_date"],
            "claimed_amount": claim_data["claimed_amount"],
            "enriched_data": format_pattern_data(pattern_name, claim_data),
        })
        result = await llm_call(prompt)
        return parse_pattern_result(result)

    # Run all 4 patterns simultaneously
    duplicate_result, timing_result, inflated_result, provider_result = await asyncio.gather(
        run_pattern("duplicate_similar", DUPLICATE_SIMILAR_COT_PROMPT, enriched_claim),
        run_pattern("suspicious_timing", SUSPICIOUS_TIMING_COT_PROMPT, enriched_claim),
        run_pattern("inflated_amounts", INFLATED_AMOUNTS_COT_PROMPT, enriched_claim),
        run_pattern("provider_network", PROVIDER_NETWORK_COT_PROMPT, enriched_claim),
    )

    # Compute weighted composite score
    composite_score = (
        duplicate_result["score"] * 0.30 +
        timing_result["score"] * 0.20 +
        inflated_result["score"] * 0.25 +
        provider_result["score"] * 0.25
    )

    # Determine confidence based on how many patterns flagged (score > 40)
    flagged_count = sum(
        1 for r in [duplicate_result, timing_result, inflated_result, provider_result]
        if r["score"] > 40
    )
    confidence = "high" if flagged_count >= 3 else "medium" if flagged_count == 2 else "low"

    # Assign priority tier
    if composite_score >= 80:
        tier = "critical"
    elif composite_score >= 60:
        tier = "high"
    elif composite_score >= 40:
        tier = "medium"
    else:
        tier = "low"

    return {
        "claim_id": enriched_claim["claim_id"],
        "pattern_results": {
            "duplicate_similar": duplicate_result,
            "suspicious_timing": timing_result,
            "inflated_amounts": inflated_result,
            "provider_network": provider_result,
        },
        "composite_score": round(composite_score, 1),
        "confidence": confidence,
        "priority_tier": tier,
        "patterns_flagged": flagged_count,
    }
```

**System Instruction (Summary):**
```
You are a Fraud Pattern Analyzer for commercial auto insurance claims. You
receive enriched claim data and run 4 independent fraud detection analyses
in parallel:

1. DUPLICATE/SIMILAR CLAIMS: Check for matching VINs across claims, similar
   loss descriptions (>80% textual similarity), overlapping loss dates within
   30 days, same claimant filing across policies, identical damage amounts.

2. SUSPICIOUS TIMING: Check claim filing relative to policy lifecycle — within
   30 days of inception, within 60 days of expiration/cancellation, Friday
   night loss reported Monday, within 72 hours of coverage change, seasonal
   anomalies.

3. INFLATED AMOUNTS: Compare claimed amounts to market benchmarks — repair
   estimate vs. vehicle market value, repair cost vs. regional benchmark,
   medical billing vs. injury severity, rental duration vs. expected repair time.

4. PROVIDER NETWORK ANOMALIES: Evaluate providers for fraud ring indicators —
   provider appearing on >15% of batch claims, billing >2x regional average,
   circular referral patterns, operating outside service area, license issues.

For EACH pattern, use Chain-of-Thought reasoning:
- Identify all relevant data points
- Assess each as red flag, neutral, or exculpatory
- Weigh the totality of evidence
- Score 0–100 with cited evidence

Then compute the composite score (weighted average) and confidence level.

CRITICAL: Never fabricate evidence. If the enriched data does not contain
information relevant to a pattern, score that pattern 0 and note "insufficient
data" — do not invent red flags.
```

**Why Gemini Pro:** Fraud detection is the most analytically demanding task in the pipeline. Each pattern sub-task requires the model to weigh circumstantial evidence, assess whether multiple minor anomalies combine into a significant fraud indicator, distinguish genuine red flags from coincidental data patterns, and produce defensible reasoning that SIU investigators can act on. Pro's superior Chain-of-Thought reasoning, its ability to handle nuanced multi-factor assessment, and its lower hallucination rate when evaluating evidence make it the right choice. Using Flash here would risk false positives from superficial pattern matching and false negatives from missing subtle cross-reference signals.

### 7.4 Investigation Report Generator Agent

| Attribute | Value |
|-----------|-------|
| **Name** | `investigation_report_generator` |
| **Model** | Gemini 2.5 Flash |
| **Role** | Produces formatted investigation reports with evidence summaries, recommended actions, and priority classification for each claim scoring Medium or above (≥40 composite score) |
| **Mode** | Non-conversational (task-based); runs in parallel — one instance per claim that qualifies for a report |

**Responsibilities:**
1. Receive fraud analysis results (composite score, pattern breakdown, evidence lists) and enriched claim data
2. Generate a structured investigation report in Markdown format suitable for SIU case files
3. Compile a consolidated evidence summary table across all triggered patterns
4. Calculate investigation priority based on fraud score and claim amount (high-value + high-score = highest priority)
5. Generate recommended investigative actions specific to the triggered patterns and evidence found

**ADK Tools (Functions):**

```python
# Tool 1: Format the full investigation report
def format_investigation_report(data: dict) -> str:
    """
    Generates a complete Markdown-formatted investigation report from
    fraud analysis results and enriched claim data.
    Report sections: Case Header, Executive Summary, Pattern Analysis Detail
    (per triggered pattern), Evidence Summary Table, Recommended Actions,
    Risk Factors, Claimant & Provider Profile.
    Returns: Markdown string of the full investigation report.
    """
    pass

# Tool 2: Format the evidence summary as a structured table
def format_evidence_summary(flags: list) -> str:
    """
    Compiles all evidence items across triggered fraud patterns into a
    consolidated evidence summary table with columns: Evidence Item, Source,
    Pattern, Severity, Specific Value, Benchmark Comparison.
    Deduplicates evidence that appears in multiple patterns.
    Returns: Markdown table string.
    """
    pass

# Tool 3: Calculate investigation priority score
def calculate_priority_score(fraud_score: float, claim_amount: float) -> dict:
    """
    Calculates an investigation priority score that combines the composite
    fraud score with the financial magnitude of the claim. A $200K claim
    with a 75 fraud score is higher priority than a $5K claim with the
    same fraud score.
    Returns: {
        'priority_score': float,
        'priority_rank': 'P1-Immediate' | 'P2-Urgent' | 'P3-Standard' | 'P4-Monitor',
        'reasoning': str,
        'estimated_exposure': float  # claim_amount * (fraud_score / 100)
    }
    """
    pass
```

**System Instruction (Summary):**
```
You are an Investigation Report Generator for commercial auto insurance fraud
detection. You produce structured investigation reports suitable for SIU
case files.

For each claim, generate a report with these sections:

1. CASE HEADER
   Claim ID | Claimant | Loss Date | Claimed Amount | Policy Number
   Composite Fraud Score | Priority Tier | Confidence Level

2. EXECUTIVE SUMMARY (2-3 sentences)
   What makes this claim suspicious and which patterns triggered.

3. PATTERN ANALYSIS DETAIL (one subsection per triggered pattern)
   Pattern name | Sub-score | Severity | Key evidence points | CoT reasoning summary

4. EVIDENCE SUMMARY TABLE
   All evidence items across all patterns, consolidated and deduplicated:
   | # | Evidence Item | Value | Benchmark | Pattern | Severity | Source |

5. RECOMMENDED ACTIONS
   Specific investigative steps based on the evidence found:
   - "Request itemized repair invoice from [provider name]"
   - "Pull recorded statement from [claimant] regarding [specific inconsistency]"
   - "Cross-reference [provider] billing across all 2024 claims"

6. RISK FACTORS
   Top risk indicators ranked by severity (bullet list)

7. CLAIMANT & PROVIDER PROFILE
   Summary of enrichment data context

Format as clean Markdown. Be specific — cite claim IDs, dollar amounts,
dates, provider names, and data sources. Never use vague language like
"suspicious activity detected" without specifying exactly what was detected.
```

**Why Gemini Flash:** Report generation is a formatting and compilation task, not an analytical one. The evidence, scores, and reasoning have already been produced by the Fraud Pattern Analyzer (Pro). The Report Generator's job is to structure that information into a readable, well-organized report — a task where Flash's speed advantage matters significantly for batch processing (generating 5–8 reports in parallel per batch) and where Pro's reasoning superiority provides no meaningful benefit.

---

## 8. A2A Communication Design

### 8.1 What is A2A?

Agent-to-Agent (A2A) is Google's open protocol for agents to discover each other's capabilities and exchange tasks. Key concepts:

- **Agent Card** — A JSON metadata file describing an agent's capabilities, URL, and supported skills. Published at `/.well-known/agent.json`
- **Task** — A unit of work one agent requests from another, with lifecycle states (submitted → working → completed/failed)
- **Message** — Communication payloads within a task containing text, structured data, or file references
- **Artifact** — Output produced by a completed task (reports, data files, analysis results)

### 8.1.1 Scope for 7-Day MVP

| A2A Feature | MVP (Days 1–7) | Stretch (Day 8) |
|-------------|---------------|-----------------|
| Agent Cards | ✅ Static JSON files for all 4 agents describing fraud-specific skills | Auto-served via A2A server `/.well-known/agent.json` endpoint |
| Agent discovery | ✅ Hardcoded agent URLs in the Orchestrator's configuration | Dynamic discovery via registry or well-known endpoint scanning |
| Task delegation | ✅ ADK's built-in sub-agent pattern — Orchestrator calls Enrichment, Analyzer, and Report Generator as sub-agents directly | Full A2A SDK client/server with TaskHandler for all agents |
| A2A protocol demo | ✅ One real A2A client→server endpoint on the Claim Data Enrichment Agent to demonstrate the protocol concept | All 3 sub-agents (Enrichment, Analyzer, Report Generator) served via A2A endpoints |
| Parallel fan-out | ✅ `asyncio.gather` with ADK sub-agent calls — N claims in parallel at batch level, 4 patterns in parallel per claim | Full A2A parallel tasking with streaming status updates per task |
| Streaming | ❌ Not in MVP | SSE streaming for real-time pipeline progress (claim-by-claim completion events) |

> **Why this approach?** ADK natively supports sub-agents, so the Orchestrator can call the Enrichment, Analyzer, and Report Generator agents as sub-agents with zero A2A protocol overhead. For the MVP, the intern uses this built-in mechanism for the actual batch fraud pipeline, then adds one real A2A server endpoint (on the Claim Data Enrichment Agent) to demonstrate the A2A protocol. Agent Cards are created as static JSON files for all 4 agents — these are valuable artifacts that show the intern understands A2A discovery and capability advertising. The dual-level parallel fan-out pattern (N claims × 4 patterns) makes this project's A2A design notably more complex than sequential A2A call chains — it demonstrates how A2A tasks can be composed in parallel, not just sequentially.

### 8.2 Agent Cards

Each agent publishes an Agent Card at `/.well-known/agent.json`:

**Fraud Pipeline Orchestrator Agent Card:**
```json
{
  "name": "fraud_pipeline_orchestrator",
  "description": "Commercial auto insurance fraud detection pipeline orchestrator. Ingests claim batches, orchestrates dual-level parallel processing (N claims × 4 fraud patterns), aggregates results ranked by composite fraud score, and provides a status/query chat for SIU investigators.",
  "url": "http://localhost:8001",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "batch-fraud-pipeline",
      "name": "Batch Fraud Detection Pipeline",
      "description": "Processes a batch of 5-50 commercial auto claims through parallel enrichment, 4-pattern fraud analysis, composite scoring, and investigation report generation. Returns claims ranked by fraud score with tier classification (Critical/High/Medium/Low).",
      "inputModes": ["text"],
      "outputModes": ["text"]
    },
    {
      "id": "siu-query-chat",
      "name": "SIU Investigator Query Chat",
      "description": "Lightweight chat interface for SIU investigators to check pipeline status, query batch results, drill into specific claims, filter by fraud pattern, and analyze provider frequency — with responses adapted to investigator experience level.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Claim Data Enrichment Agent Card:**
```json
{
  "name": "claim_data_enrichment",
  "description": "Enriches raw commercial auto insurance claims with contextual data: claimant history, provider network information, vehicle valuation, and policy details. Task-based, non-conversational.",
  "url": "http://localhost:8002",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "claim-enrichment",
      "name": "Claim Data Enrichment",
      "description": "Augments a single commercial auto claim with claimant history (prior claims, fraud flags, associated vehicles), provider network info (billing patterns, referral connections, license status), vehicle valuation (market value, repair benchmarks), and policy details (inception date, recent changes, premium history). Returns a structured enriched data package.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Fraud Pattern Analyzer Agent Card:**
```json
{
  "name": "fraud_pattern_analyzer",
  "description": "Runs 4 parallel fraud detection patterns per claim (Duplicate/Similar Claims, Suspicious Timing, Inflated Amounts, Provider Network Anomalies) using Chain-of-Thought reasoning. Produces evidence-based fraud flags, sub-scores, and a composite fraud score. Task-based, non-conversational.",
  "url": "http://localhost:8003",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "fraud-pattern-analysis",
      "name": "Parallel Fraud Pattern Analysis",
      "description": "Analyzes a single enriched commercial auto claim across 4 fraud dimensions simultaneously: duplicate/similar claims (VIN matching, description similarity, date overlap), suspicious timing (policy lifecycle proximity, day-of-week anomalies), inflated amounts (repair vs. market value, medical vs. injury severity), and provider network anomalies (billing ratios, referral rings, geographic outliers). Returns sub-scores per pattern, composite weighted score, confidence level, and evidence lists.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

**Investigation Report Generator Agent Card:**
```json
{
  "name": "investigation_report_generator",
  "description": "Generates formatted investigation reports with evidence summaries, recommended actions, and priority classification for commercial auto insurance fraud claims. Task-based, non-conversational.",
  "url": "http://localhost:8004",
  "version": "1.0.0",
  "capabilities": {
    "streaming": false,
    "pushNotifications": false
  },
  "skills": [
    {
      "id": "investigation-report",
      "name": "Fraud Investigation Report Generation",
      "description": "Produces a structured investigation report for a single claim: case header, executive summary, per-pattern analysis detail, consolidated evidence table, recommended investigative actions, risk factor ranking, and claimant/provider profile. Output in Markdown format suitable for SIU case files.",
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

### 8.3 A2A Task Flow — Dual-Level Parallel Fan-Out

This project's A2A flow features **dual-level parallelism**: the Orchestrator fans out N claims in parallel at the batch level (Level 1), and within each claim, the Fraud Pattern Analyzer runs 4 pattern sub-tasks in parallel (Level 2). The per-claim pipeline is sequential (Enrich → Analyze → Report), but each claim's pipeline runs concurrently with every other claim's pipeline.

```
Fraud Pipeline          Claim Data          Fraud Pattern         Investigation
Orchestrator            Enrichment          Analyzer              Report Generator
     │                       │                    │                      │
     │  [Ingest batch JSON:  │                    │                      │
     │   "Weekly Auto Claims"│                    │                      │
     │   — 8 claims]         │                    │                      │
     │                       │                    │                      │
     │  [Validate: 8 valid,  │                    │                      │
     │   0 invalid]          │                    │                      │
     │                       │                    │                      │
     │══ LEVEL 1 PARALLEL FAN-OUT (N claims simultaneously) ════════════│
     │                       │                    │                      │
     │─ CLM-001: Enrich ───►│                    │                      │
     │─ CLM-002: Enrich ───►│                    │                      │
     │─ CLM-003: Enrich ───►│                    │                      │
     │─ CLM-004: Enrich ───►│   (A2A endpoint    │                      │
     │─ CLM-005: Enrich ───►│    — MVP demo)     │                      │
     │─ CLM-006: Enrich ───►│                    │                      │
     │─ CLM-007: Enrich ───►│                    │                      │
     │─ CLM-008: Enrich ───►│                    │                      │
     │   "Enrich claim X:    │                    │                      │
     │    claimant history,   │                    │                      │
     │    provider, vehicle,  │                    │                      │
     │    policy"             │                    │                      │
     │                       │                    │                      │
     │◄─ CLM-001: enriched ──│                    │                      │
     │◄─ CLM-002: enriched ──│                    │                      │
     │◄─ ...                 │                    │                      │
     │◄─ CLM-008: enriched ──│                    │                      │
     │                       │                    │                      │
     │─ CLM-001: Analyze ────────────────────►│                      │
     │─ CLM-002: Analyze ────────────────────►│                      │
     │─ CLM-003: Analyze ────────────────────►│                      │
     │─ CLM-004: Analyze ────────────────────►│   ┌──────────────────┐ │
     │─ CLM-005: Analyze ────────────────────►│   │ LEVEL 2 PARALLEL │ │
     │─ CLM-006: Analyze ────────────────────►│   │ (per claim):     │ │
     │─ CLM-007: Analyze ────────────────────►│   │ ├─ Duplicate     │ │
     │─ CLM-008: Analyze ────────────────────►│   │ ├─ Timing        │ │
     │   "Analyze enriched    │                │   │ ├─ Inflated      │ │
     │    claim X across 4    │                │   │ └─ Provider      │ │
     │    fraud patterns"     │                │   │ → Composite Score│ │
     │                       │                │   └──────────────────┘ │
     │                       │                    │                      │
     │◄─ CLM-001: scored ────────────────────│                      │
     │◄─ CLM-002: scored ────────────────────│                      │
     │◄─ ...                 │                    │                      │
     │◄─ CLM-008: scored ────────────────────│                      │
     │                       │                    │                      │
     │  [Filter: claims with │                    │                      │
     │   score ≥ 40 get      │                    │                      │
     │   investigation       │                    │                      │
     │   reports]            │                    │                      │
     │                       │                    │                      │
     │─ CLM-003: Report ──────────────────────────────────────────►│
     │─ CLM-005: Report ──────────────────────────────────────────►│
     │─ CLM-007: Report ──────────────────────────────────────────►│
     │   "Generate investigation│                  │                      │
     │    report for claim X"  │                   │                      │
     │                       │                    │                      │
     │◄─ CLM-003: report ─────────────────────────────────────────│
     │◄─ CLM-005: report ─────────────────────────────────────────│
     │◄─ CLM-007: report ─────────────────────────────────────────│
     │                       │                    │                      │
     │══ AGGREGATE RESULTS ══════════════════════════════════════════│
     │                       │                    │                      │
     │  [Rank all 8 claims   │                    │                      │
     │   by composite score, │                    │                      │
     │   compute batch       │                    │                      │
     │   statistics]         │                    │                      │
     │                       │                    │                      │
     ▼ Enter SIU Query Mode  │                    │                      │
     ▼ "Batch complete. 8    │                    │                      │
       claims analyzed.      │                    │                      │
       3 flagged (≥40).      │                    │                      │
       Ask me anything."     │                    │                      │
```

### 8.4 Implementation Guidance

**How to implement A2A in this project:**

1. **Install the A2A SDK:**
   ```bash
   pip install a2a-sdk
   ```

2. **Each agent runs as a separate A2A-compatible server** (separate port):
   ```
   fraud_pipeline_orchestrator      → http://localhost:8001
   claim_data_enrichment            → http://localhost:8002  (A2A endpoint for MVP demo)
   fraud_pattern_analyzer           → http://localhost:8003
   investigation_report_generator   → http://localhost:8004
   ```

3. **The MVP demo uses a real A2A client→server call for the Enrichment Agent:**
   ```python
   from a2a.client import A2AClient
   import asyncio
   import json

   # Discover the enrichment agent via A2A
   enrichment_client = A2AClient(url="http://localhost:8002")
   card = await enrichment_client.get_agent_card()
   print(f"Discovered: {card['name']} — skills: {[s['id'] for s in card['skills']]}")

   # Send an enrichment task via A2A protocol
   async def enrich_claim_via_a2a(claim_data: dict) -> dict:
       """Send a single claim to the Enrichment Agent via real A2A."""
       task = await enrichment_client.send_task({
           "id": f"enrich-{claim_data['claim_id']}",
           "skill_id": "claim-enrichment",
           "message": {
               "role": "user",
               "parts": [{
                   "text": json.dumps({
                       "claim_id": claim_data["claim_id"],
                       "claimant": claim_data["claimant"],
                       "vehicle": claim_data["vehicle"],
                       "provider": claim_data["provider"],
                       "policy_number": claim_data["policy_number"],
                       "loss_date": claim_data["loss_date"],
                       "claimed_amount": claim_data["claimed_amount"],
                       "loss_description": claim_data["loss_description"]
                   })
               }]
           }
       })
       return json.loads(task.artifacts[0].parts[0].text)

   # Fan-out: enrich ALL claims in parallel via A2A
   batch = load_and_validate_batch("weekly_auto_claims.json")
   enriched_claims = await asyncio.gather(*[
       enrich_claim_via_a2a(claim)
       for claim in batch["valid_claims"]
   ])
   ```

4. **The Enrichment Agent implements a TaskHandler that processes one claim:**
   ```python
   from a2a.server import A2AServer, TaskHandler
   import json

   class EnrichmentTaskHandler(TaskHandler):
       async def handle_task(self, task):
           """Enrich a single commercial auto claim with contextual data."""
           claim = json.loads(task.message.parts[0].text)

           # Gather enrichment data from mock APIs
           claimant_history = await self.lookup_claimant_history(claim["claimant"]["id"])
           provider_network = await self.lookup_provider_network(
               claim["provider"]["id"], claim["provider"]["type"]
           )
           vehicle_valuation = await self.lookup_vehicle_valuation(
               claim["vehicle"]["vin"],
               claim["vehicle"]["year"],
               claim["vehicle"]["make"],
               claim["vehicle"]["model"]
           )
           policy_details = await self.lookup_policy_details(claim["policy_number"])

           enriched = {
               "claim_id": claim["claim_id"],
               "raw_claim": claim,
               "claimant_history": claimant_history,
               "provider_network": provider_network,
               "vehicle_valuation": vehicle_valuation,
               "policy_details": policy_details,
               "enrichment_status": "complete",
           }

           return TaskResult(
               artifacts=[Artifact(parts=[TextPart(text=json.dumps(enriched))])]
           )

   # Run the A2A server
   server = A2AServer(
       agent_card=load_agent_card("claim_data_enrichment"),
       task_handler=EnrichmentTaskHandler(),
   )
   server.start(port=8002)
   ```

5. **MVP workflow — ADK sub-agents with dual-level `asyncio.gather` for the full pipeline:**
   ```python
   from google.adk import Agent
   import asyncio

   # Define all 3 sub-agents
   claim_data_enrichment = Agent(
       name="claim_data_enrichment",
       model="gemini-2.5-flash",
       instruction=ENRICHMENT_PROMPT,
   )
   fraud_pattern_analyzer = Agent(
       name="fraud_pattern_analyzer",
       model="gemini-2.5-pro",
       instruction=ANALYZER_PROMPT,
   )
   investigation_report_generator = Agent(
       name="investigation_report_generator",
       model="gemini-2.5-flash",
       instruction=REPORT_PROMPT,
   )

   # Orchestrator with sub-agents
   fraud_pipeline_orchestrator = Agent(
       name="fraud_pipeline_orchestrator",
       model="gemini-2.5-pro",
       instruction=ORCHESTRATOR_SYSTEM_PROMPT,
       sub_agents=[
           claim_data_enrichment,
           fraud_pattern_analyzer,
           investigation_report_generator,
       ],
   )

   # Dual-level parallel processing tool function:
   async def process_claim_batch(batch_json: str) -> dict:
       """Process all claims with Level 1 (batch) and Level 2 (pattern) parallelism."""
       batch = validate_batch(batch_json)

       async def process_single_claim(claim):
           """Sequential pipeline per claim, called in parallel across N claims."""
           # Step 1: Enrich (calls Enrichment Agent)
           enriched = await claim_data_enrichment.run(claim)

           # Step 2: Analyze — Fraud Pattern Analyzer internally runs 4 patterns in parallel
           fraud_analysis = await fraud_pattern_analyzer.run(enriched)

           # Step 3: Generate report (only for claims scoring ≥ 40)
           report = None
           if fraud_analysis["composite_score"] >= 40:
               report = await investigation_report_generator.run(fraud_analysis)

           return {
               "claim_id": claim["claim_id"],
               "enriched": enriched,
               "fraud_analysis": fraud_analysis,
               "report": report,
           }

       # LEVEL 1 PARALLELISM: All N claims processed simultaneously
       results = await asyncio.gather(*[
           process_single_claim(claim)
           for claim in batch["valid_claims"]
       ])

       # Aggregate and rank by fraud score
       results.sort(key=lambda r: r["fraud_analysis"]["composite_score"], reverse=True)
       return {
           "batch_id": batch["batch_id"],
           "batch_name": batch["batch_name"],
           "total_claims": len(results),
           "results": results,
           "tier_distribution": compute_tier_distribution(results),
           "top_patterns": compute_top_patterns(results),
           "provider_frequency": compute_provider_frequency(results),
       }
   ```

---

## 9. Workflow Design

### 9.1 Dual-Level Parallel Pipeline — Batch Fraud Processing

This is the **PRIMARY workflow** for this project. Unlike conversational agents in Projects 1 and 2 or the single-level parallelism in Project 3, this project features **dual-level parallelism**: N claims processed in parallel at the batch level (Level 1), with 4 fraud patterns analyzed in parallel per claim (Level 2). This nested `asyncio.gather` pattern mirrors real enterprise fraud detection pipelines.

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                        BATCH FRAUD PROCESSING PIPELINE                                  │
│                                                                                         │
│  ┌─────────────────────┐                                                                │
│  │ INPUT: Batch JSON   │                                                                │
│  │ "Weekly Auto Claims"│                                                                │
│  │ — 8 claims          │                                                                │
│  └─────────┬───────────┘                                                                │
│            │                                                                            │
│            ▼                                                                            │
│  ┌─────────────────────┐                                                                │
│  │ STEP 1: Validate    │  Orchestrator validates all 8 claims                           │
│  │ (Sequential)        │  → 8 valid, 0 invalid                                         │
│  └─────────┬───────────┘                                                                │
│            │                                                                            │
│  ══════════╪══════ LEVEL 1 PARALLELISM: N claims simultaneously ══════════════════      │
│            │                                                                            │
│    ┌───────┼───────┬───────────┬───────────┬──────┬───────────┬───────────┬────────┐    │
│    │       │       │           │           │      │           │           │        │    │
│    ▼       ▼       ▼           ▼           ▼      ▼           ▼           ▼        │    │
│  CLM-001 CLM-002 CLM-003   CLM-004     CLM-005 CLM-006   CLM-007     CLM-008     │    │
│    │       │       │           │           │      │           │           │        │    │
│    │       │       │           │           │      │           │           │        │    │
│    │  Each claim runs the SAME per-claim sequential pipeline:                     │    │
│    │                                                                              │    │
│    │  ┌──────────────┐   ┌────────────────────┐   ┌───────────────┐              │    │
│    │  │ STEP 2:      │   │ STEP 3:            │   │ STEP 4:       │              │    │
│    ├─►│ Enrich       │──►│ Analyze 4 Patterns │──►│ Score & Report│              │    │
│    │  │              │   │ (Level 2 Parallel)  │   │ (if ≥40)      │              │    │
│    │  │ • Claimant   │   │ ┌───────────────┐  │   │               │              │    │
│    │  │   history    │   │ │Duplicate/Sim  │  │   │ • Composite   │              │    │
│    │  │ • Provider   │   │ │Susp. Timing   │  │   │   score       │              │    │
│    │  │   network    │   │ │Inflated Amts  │  │   │ • Tier        │              │    │
│    │  │ • Vehicle    │   │ │Provider Net   │  │   │ • Report      │              │    │
│    │  │   valuation  │   │ └───────────────┘  │   │   (Markdown)  │              │    │
│    │  │ • Policy     │   │ → Composite score  │   │               │              │    │
│    │  │   details    │   │ → Evidence list     │   │               │              │    │
│    │  └──────────────┘   └────────────────────┘   └───────────────┘              │    │
│    │                                                                              │    │
│    └──────────────────────────────────────────────────────────────────────────────┘    │
│            │                                                                            │
│  ══════════╪══════════════════════════════════════════════════════════════════════      │
│            │                                                                            │
│            ▼                                                                            │
│  ┌────────────────────────┐                                                             │
│  │ STEP 5: Aggregate      │  Collect all 8 results, rank by fraud score                │
│  │ (Sequential)           │  Tier distribution: 1 Critical, 2 High, 2 Medium, 3 Low   │
│  │                        │  Top patterns triggered, provider frequency analysis        │
│  └─────────┬──────────────┘                                                             │
│            │                                                                            │
│            ▼                                                                            │
│  ┌────────────────────────┐                                                             │
│  │ STEP 6: Query Mode     │  SIU investigator can now query results via chat           │
│  │ (Conversational)       │  "Show me the batch summary" / "Details on CLM-005"        │
│  └────────────────────────┘                                                             │
│                                                                                         │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

**Step Details:**

| Step | Name | Agent | Parallelism | Input | Output | Prompt Type |
|------|------|-------|-------------|-------|--------|-------------|
| 1 | Batch Intake & Validation | Fraud Pipeline Orchestrator | Sequential (one batch) | Raw batch JSON file | Validated claims list, validation errors | Zero-shot (field validation) |
| 2 | Claim Data Enrichment | Claim Data Enrichment | Level 1: N claims in parallel | Single raw claim | Enriched claim (+ claimant history, provider info, vehicle value, policy details) | ReAct (reason + tool calls) |
| 3 | Fraud Pattern Analysis | Fraud Pattern Analyzer | Level 1 × Level 2: N claims in parallel, each running 4 patterns in parallel | Single enriched claim | 4 pattern sub-scores + evidence + composite score | CoT (per pattern) |
| 4 | Score & Report | Investigation Report Generator | Level 1: in parallel for claims scoring ≥40 | Fraud analysis results + enriched data | Formatted investigation report (Markdown) | Zero-shot (report formatting) |
| 5 | Aggregate & Rank | Fraud Pipeline Orchestrator | Sequential (one aggregation) | All N per-claim results | Ranked batch summary, tier distribution, statistics | Zero-shot (aggregation) |
| 6 | SIU Query Chat | Fraud Pipeline Orchestrator | On-demand (conversational) | Investigator query | Adapted response (detailed for Kevin, concise for Diana) | Few-shot (query classification) + Zero-shot (response) |

### 9.2 Per-Claim Sequential Pipeline

Each of the N claims runs through the same sequential pipeline independently. The pipeline within a single claim is strictly sequential because each step depends on the output of the previous step — you cannot analyze fraud patterns without enriched data, and you cannot generate a report without fraud scores.

```
Single Claim: CLM-2024-0847 ("Suspicious Fleet Truck Repair")
═══════════════════════════════════════════════════════════════

Raw Claim Data ──► [ENRICH] ──► [ANALYZE (4 parallel)] ──► [SCORE] ──► [REPORT]
                       │                  │                     │            │
                       │                  │                     │            │
                       ▼                  ▼                     ▼            ▼
                 Enriched Data:     Pattern Results:      Composite:    Investigation
                 • 4 prior claims   • Duplicate: 35       Score: 72.5   Report:
                 • Provider bills   • Timing: 78          Tier: High    • Case Header
                   2.8x regional    • Inflated: 82        Confidence:   • Exec Summary
                 • Vehicle worth    • Provider: 71        High          • Pattern Detail
                   $28.5K                                 (3/4 flagged) • Evidence Table
                 • Policy 45 days                                       • Actions
                   old                                                  • Risk Factors
```

**Per-Claim Pipeline Step Details:**

| Step | Agent | Prompt Type | Duration (est.) | Dependencies |
|------|-------|-------------|-----------------|-------------|
| Enrich | Claim Data Enrichment (Flash) | ReAct — reason about which data to gather, call 4 tools sequentially: `lookup_claimant_history` → `lookup_provider_network` → `lookup_vehicle_valuation` → `lookup_policy_details` | ~3–5 sec | Raw claim data |
| Analyze (4 patterns) | Fraud Pattern Analyzer (Pro) | CoT × 4 in parallel — each pattern uses step-by-step reasoning to evaluate evidence and produce a sub-score | ~5–8 sec (all 4 in parallel) | Enriched claim data |
| Score | Fraud Pattern Analyzer (Pro) | Zero-shot — weighted average computation, confidence assignment, tier classification | ~1 sec (part of Analyze) | 4 pattern sub-scores |
| Report | Investigation Report Generator (Flash) | Zero-shot — format evidence, generate narrative, calculate priority | ~3–5 sec | Fraud analysis results + enriched data |

**Total per-claim pipeline: ~12–18 seconds** (Steps 2–4 sequential within each claim, but all N claims run in parallel at the batch level, so total batch time ≈ single claim time + aggregation overhead)

### 9.3 Fraud Pattern Parallel Analysis Detail

This section describes what each of the 4 fraud patterns checks, the specific red flags it evaluates, and the data sources it uses from the enrichment step.

#### Pattern 1: Duplicate / Similar Claims

**What it checks:** Whether this claim is identical or suspiciously similar to other claims — either in the current batch or in the claimant's history. Duplicate and "near-duplicate" claims are among the most common fraud types, ranging from simple double-dipping (filing the same claim twice) to sophisticated schemes where a claimant files materially similar claims across different policies, vehicles, or time periods.

**Red Flags Evaluated:**

| Red Flag | Data Source | Threshold / Check | Severity |
|----------|------------|-------------------|----------|
| Same VIN appears on another claim in the batch | Batch data cross-reference | Exact VIN match across 2+ claims in the batch | Major |
| Same VIN appears in claimant's prior claim history | `lookup_claimant_history` → `associated_vehicles` | Exact VIN match with a prior claim within 12 months | Major |
| Same claimant filed a similar claim within 12 months | `lookup_claimant_history` → `prior_claims` | Same claimant + same loss type + within 365 days | Major |
| Loss descriptions show >80% textual similarity to another claim | Batch data + claimant history | Cosine similarity or keyword overlap > 0.80 between loss descriptions | Moderate |
| Overlapping loss dates within 30 days of another claim by same claimant | `lookup_claimant_history` → `prior_claims` | |loss_date_1 - loss_date_2| ≤ 30 days, same claimant | Moderate |
| Identical claimed amounts on different claims | Batch data + claimant history | Exact dollar match (± $50) across 2+ claims | Minor |
| Claimant's claim frequency exceeds 2 claims/year | `lookup_claimant_history` → `claim_frequency_per_year` | Frequency > 2.0 per year | Minor |

**Scoring Logic:**
- 0–20: No matches found; claim appears unique
- 21–40: Minor overlaps (similar amounts or near-annual frequency) that may be coincidental
- 41–60: One significant match (same VIN within 12 months or moderately similar descriptions)
- 61–80: Multiple matches or a very strong single match (same VIN + similar loss description + proximate dates)
- 81–100: Near-certain duplicate — same VIN, same claimant, nearly identical loss description, overlapping dates

#### Pattern 2: Suspicious Timing

**What it checks:** Whether the temporal relationship between the claim filing and key policy or behavioral events follows a pattern consistent with pre-meditated fraud. Timing analysis examines when the loss allegedly occurred relative to the policy lifecycle (inception, expiration, changes) and whether the day-of-week and season are consistent with the claimed loss type.

**Red Flags Evaluated:**

| Red Flag | Data Source | Threshold / Check | Severity |
|----------|------------|-------------------|----------|
| Claim filed within 30 days of policy inception | `lookup_policy_details` → `days_since_inception` | inception_date – loss_date ≤ 30 days ("new policy fraud") | Major |
| Claim filed within 60 days of policy cancellation or expiration | `lookup_policy_details` → `days_until_expiration` | loss_date within 60 days of expiration/cancellation | Moderate |
| Loss occurred Friday evening/night, reported Monday morning | Raw claim data → `loss_date`, `report_date` | Loss on Fri 5PM–Sun 11:59PM, report on Monday (classic staging window) | Moderate |
| Claim filed within 72 hours of a coverage upgrade or vehicle addition | `lookup_policy_details` → `recent_policy_changes` | Any coverage change within 72 hours before loss_date | Major |
| Policy recently reinstated after lapse, then immediate claim | `lookup_policy_details` → `lapse_history` | Reinstatement within 30 days before loss_date | Major |
| Seasonal anomaly — claim type inconsistent with season | Raw claim data → `loss_description`, `loss_date` | Ice/snow claim in July; flooding claim during drought period | Minor |
| Claim filed on a holiday or long weekend | Raw claim data → `loss_date` | Loss date falls on a federal holiday or holiday weekend | Minor |

**Scoring Logic:**
- 0–20: Timing appears normal; claim filed in a routine window
- 21–40: One minor timing anomaly (holiday filing, moderate proximity to expiration)
- 41–60: One major timing flag (within 30 days of inception OR 72 hours of coverage change)
- 61–80: Multiple timing flags (new policy + coverage change + weekend staging)
- 81–100: Extreme timing pattern (reinstated after lapse + coverage upgrade + immediate large claim)

#### Pattern 3: Inflated Amounts

**What it checks:** Whether the financial amounts claimed are proportionate to the described incident, the vehicle involved, and regional benchmarks. Inflated amounts can indicate either opportunistic fraud (legitimate incident with padded bills) or provider collusion (systematic overbilling).

**Red Flags Evaluated:**

| Red Flag | Data Source | Threshold / Check | Severity |
|----------|------------|-------------------|----------|
| Repair estimate exceeds vehicle market value | `lookup_vehicle_valuation` → `market_value` vs. `claimed_amount` | claimed_amount > market_value (exceeds total loss threshold) | Major |
| Repair cost >2x regional benchmark for described damage | `lookup_vehicle_valuation` → `regional_repair_benchmarks` | claimed_amount > 2.0 × regional median for damage type | Major |
| Medical billing disproportionate to injury severity | Raw claim data → `loss_description` + `claimed_amount` | $50K+ medical for low-speed collision; $30K+ for parking lot fender bender | Major |
| Repair cost 1.5–2x regional benchmark | `lookup_vehicle_valuation` → `regional_repair_benchmarks` | 1.5 × median ≤ claimed_amount ≤ 2.0 × median | Moderate |
| Rental car duration >3x expected repair time | Raw claim data → rental details (if present) | Rental days > 3 × expected repair days for damage type | Moderate |
| Parts pricing above manufacturer MSRP | Raw claim data → repair estimate details (if itemized) | Individual parts priced above MSRP + reasonable markup | Minor |
| Lost income claim exceeds documented earnings pattern | Raw claim data → lost income claim details | Claimed lost income > historical earnings ratio | Minor |

**Scoring Logic:**
- 0–20: Amounts within normal range; no significant deviation from benchmarks
- 21–40: Slightly elevated amounts (1.2–1.5x benchmark) that may reflect legitimate high-cost repairs
- 41–60: One significant inflation flag (repair > 1.5x benchmark or approaching vehicle value)
- 61–80: Multiple inflation flags or one extreme flag (repair > vehicle value, medical wildly disproportionate)
- 81–100: Systematic inflation across multiple categories (repair + medical + rental all inflated)

#### Pattern 4: Provider Network Anomalies

**What it checks:** Whether the providers involved in this claim (repair shops, medical facilities, attorneys, towing companies) exhibit patterns consistent with organized fraud networks. Provider network analysis is the most powerful fraud detection dimension because organized rings — which account for the largest fraud losses — leave network fingerprints that individual claim review cannot detect.

**Red Flags Evaluated:**

| Red Flag | Data Source | Threshold / Check | Severity |
|----------|------------|-------------------|----------|
| Provider appears on >15% of claims in the batch | Batch cross-reference + `lookup_provider_network` → `total_claims_served` | provider_claim_count / batch_size > 0.15 | Major |
| Provider average billing >2x regional mean | `lookup_provider_network` → `billing_ratio` | billing_ratio > 2.0 | Major |
| Circular referral pattern detected | `lookup_provider_network` → `referral_connections` | Provider A refers to B, B refers to C, C refers back to A (or A→B→A) | Major |
| Provider operating outside normal geographic service area | `lookup_provider_network` → `geographic_service_area`, `claims_outside_service_area` | claims_outside_service_area > 3 for this provider | Moderate |
| Provider license expired, suspended, or under investigation | `lookup_provider_network` → `license_status` | license_status ≠ "active" | Major |
| Multiple claimants in the batch using the same unusual provider combination | Batch cross-reference | 2+ claimants using the same repair shop + same medical provider + same attorney | Major |
| Provider has prior flagged claims | `lookup_provider_network` → `flagged_claims_count` | flagged_claims_count > 2 | Moderate |
| Provider billing >1.5x regional mean | `lookup_provider_network` → `billing_ratio` | 1.5 < billing_ratio ≤ 2.0 | Minor |

**Scoring Logic:**
- 0–20: Provider profile clean; billing within norms, no referral anomalies
- 21–40: Minor anomalies (slightly elevated billing, a single shared provider across 2 claims)
- 41–60: Moderate concerns (billing 1.5–2x regional, provider on multiple flagged claims, geographic outlier)
- 61–80: Significant ring indicators (circular referrals, billing >2x, multiple claims from same provider combo)
- 81–100: Strong ring evidence (circular referral + inflated billing + expired license + high frequency)

### 9.4 Status / Query Interface Workflow

After the batch pipeline completes (or during execution for status checks), SIU investigators interact with the Orchestrator through a lightweight chat interface served via `adk web`. This is NOT the primary mode — the pipeline runs autonomously — but the chat provides human-accessible interaction for result review and drill-down.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                          SIU QUERY INTERFACE WORKFLOW                                    │
│                                                                                          │
│  ┌────────────────────┐                    ┌───────────────────────────────────────┐     │
│  │ SIU Investigator   │                    │ Fraud Pipeline Orchestrator            │     │
│  │ (Kevin or Diana)   │                    │ (Query Mode)                           │     │
│  └─────────┬──────────┘                    └───────────────────┬───────────────────┘     │
│            │                                                   │                         │
│            │  "How many claims have                            │                         │
│            │   been processed?"                                │                         │
│            │───────────────────────────────────────────────────►│                         │
│            │                                                   │  [Check pipeline state]  │
│            │                                                   │  [Return progress]       │
│            │◄──────────────────────────────────────────────────│                         │
│            │  "Processing batch 'Weekly Auto Claims':          │                         │
│            │   6/8 claims complete. ~45 sec remaining."        │                         │
│            │                                                   │                         │
│            │  ═══ (Pipeline completes) ═══                     │                         │
│            │                                                   │                         │
│            │  "Show me the batch summary"                      │                         │
│            │───────────────────────────────────────────────────►│                         │
│            │                                                   │  [Query batch results]   │
│            │                                                   │  [Profile user: Kevin]   │
│            │                                                   │  [Format: detailed]      │
│            │◄──────────────────────────────────────────────────│                         │
│            │  "Batch 'Weekly Auto Claims' — 8 claims analyzed: │                         │
│            │   🔴 Critical (1): CLM-005 — Score 87.3           │                         │
│            │   🟠 High (2): CLM-003 (72.5), CLM-007 (65.1)    │                         │
│            │   🟡 Medium (2): CLM-001 (48.2), CLM-008 (41.7)  │                         │
│            │   🟢 Low (3): CLM-002, CLM-004, CLM-006           │                         │
│            │   Top pattern: Inflated Amounts (triggered 4x)    │                         │
│            │   Provider alert: Elite Auto Body on 3 claims"    │                         │
│            │                                                   │                         │
│            │  "Tell me about CLM-005"                          │                         │
│            │───────────────────────────────────────────────────►│                         │
│            │                                                   │  [Drill into CLM-005]    │
│            │                                                   │  [Kevin: include         │
│            │                                                   │   explanations of        │
│            │                                                   │   what each pattern      │
│            │                                                   │   means]                 │
│            │◄──────────────────────────────────────────────────│                         │
│            │  "CLM-005 — Fleet truck rear-end collision:        │                         │
│            │   Composite Score: 87.3 (Critical)                │                         │
│            │   ├─ Duplicate/Similar: 45 — This means...        │                         │
│            │   ├─ Suspicious Timing: 91 — This means...        │                         │
│            │   ├─ Inflated Amounts: 88 — This means...         │                         │
│            │   └─ Provider Network: 85 — This means...         │                         │
│            │   Confidence: HIGH (3 of 4 patterns significant)  │                         │
│            │   Recommended: Full SIU investigation..."         │                         │
│            │                                                   │                         │
│            │  "Which claims had provider network issues?"       │                         │
│            │───────────────────────────────────────────────────►│                         │
│            │                                                   │  [Filter by pattern]     │
│            │◄──────────────────────────────────────────────────│                         │
│            │  "3 claims flagged for Provider Network Anomalies: │                         │
│            │   CLM-005 (score 85), CLM-003 (71), CLM-007 (63) │                         │
│            │   Common provider: Elite Auto Body (on all 3)     │                         │
│            │   Elite billing: 2.8x regional average, ...       │                         │
│            │   Possible referral ring detected: ..."            │                         │
│            │                                                   │                         │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

**Supported Query Types:**

| Query Type | Example Queries | What the Orchestrator Does |
|-----------|----------------|---------------------------|
| **Pipeline Status** | "How many claims processed?" / "Is the batch done?" / "Status" | Checks pipeline execution state; returns progress count and estimated time remaining |
| **Batch Summary** | "Show me the results" / "Give me the summary" / "How many fraud flags?" | Returns tier distribution, top fraud scores, key batch statistics, provider alerts |
| **Claim Drill-Down** | "Tell me about Claim CLM-005" / "Why was Claim 5 flagged?" / "Show evidence for the highest-scoring claim" | Returns full detail for the specified claim: all 4 pattern scores, evidence list, composite score reasoning, recommended actions |
| **Pattern Filter** | "Which claims had provider network issues?" / "Show me all timing-flagged claims" / "What's the most common pattern in this batch?" | Filters claims by specific fraud pattern; returns matching claims with pattern-specific details |
| **Provider Analysis** | "Which providers appear multiple times?" / "Tell me about Elite Auto Body" | Cross-claim provider frequency analysis, billing comparisons, referral network mapping |
| **Explanation** | "What does the duplicate claim score mean?" / "How is the composite score calculated?" / "What is a provider ring?" | Educational explanation adapted to user profile — Kevin gets detailed definitions and examples; Diana gets brief technical summaries |

---

## 10. Prompt Engineering Strategy

### 10.1 Prompt Types

This project uses four distinct prompt engineering techniques, each selected for specific tasks based on the reasoning complexity, available context, and reliability requirements of the task.

#### 10.1.1 Zero-Shot Prompts

**Used for:** Tasks where the instruction is self-contained and the model can produce correct output without examples — data validation, status reporting, batch aggregation, and straightforward query responses.

**Full Example — Status Query Response:**

```
SYSTEM: You are a fraud pipeline status reporter. Answer the investigator's
question about pipeline progress concisely.

PIPELINE STATE:
Batch: "Weekly Auto Claims"
Total claims: 8
Claims completed: 6
Claims in progress: 2 (CLM-007, CLM-008)
Start time: 2024-11-15T14:23:01Z
Current time: 2024-11-15T14:23:47Z
Average time per claim: 7.2 seconds

USER QUERY: "How many claims have been processed?"

Respond with current progress, claims remaining, and estimated completion time.
Do NOT include fraud scores or analysis details — this is a status query only.
```

**Full Example — Batch Validation:**

```
SYSTEM: You are a claim data validator. Validate the following claim against
these rules:

REQUIRED FIELDS: claim_id, claimant.name, claimant.id, loss_date, report_date,
claimed_amount, vehicle.vin, vehicle.year, vehicle.make, vehicle.model,
policy_number

DATA TYPE RULES:
- loss_date, report_date: ISO 8601 date format (YYYY-MM-DD)
- claimed_amount: positive numeric value
- vehicle.vin: exactly 17 alphanumeric characters
- vehicle.year: integer between 1990 and current year + 1
- policy_number: non-empty string

CROSS-FIELD RULES:
- report_date >= loss_date
- claimed_amount > 0

CLAIM DATA:
{claim_json}

Return a JSON object:
{
  "claim_id": "...",
  "valid": true/false,
  "errors": ["list of specific validation errors, empty if valid"]
}
```

#### 10.1.2 Few-Shot Prompts

**Used for:** Tasks that benefit from labeled examples to calibrate the model's classification — fraud pattern classification (showing what flagged vs. clean claims look like for each pattern), query type classification, and user profiling.

**Full Example — Duplicate/Similar Claims Classification (Few-Shot):**

```
SYSTEM: You are a fraud duplicate/similar claim detector. Analyze the claim
data for indicators of duplicate or near-duplicate claims. Use the following
labeled examples to calibrate your assessment.

=== EXAMPLE 1: FLAGGED (Score: 82) ===
Claim: CLM-2024-0312
Claimant: James Whitfield, ID: C-1293
Loss date: 2024-06-15
Vehicle VIN: 1HGBH41JXMN109186
Claimed amount: $18,750
Claimant history: Filed CLM-2024-0087 on 2024-03-22 for SAME VIN, same loss
type (rear-end collision), claimed $17,900. Description similarity: 87%.
Filed CLM-2023-0891 on 2023-11-05 for SAME VIN, side-swipe, $12,300.
Assessment: FLAGGED. Same VIN on 3 claims in 8 months. Two claims within 85
days with near-identical descriptions and amounts ($850 difference). Claim
frequency 4.5/year. Strong duplicate pattern.

=== EXAMPLE 2: CLEAN (Score: 12) ===
Claim: CLM-2024-0455
Claimant: Maria Santos, ID: C-3847
Loss date: 2024-08-20
Vehicle VIN: 5YJSA1E26MF123456
Claimed amount: $6,200
Claimant history: Filed CLM-2022-0671 on 2022-05-11 for a DIFFERENT VIN
(Toyota Camry), different loss type (theft), $22,000. No other claims.
Assessment: CLEAN. Different vehicle, different loss type, 27 months apart.
One prior claim in 2+ years is well within normal frequency. No duplicate
indicators.

=== EXAMPLE 3: FLAGGED (Score: 58) ===
Claim: CLM-2024-0519
Claimant: Robert Chen, ID: C-2105
Loss date: 2024-09-03
Vehicle VIN: 3C4PDCAB5JT500123
Claimed amount: $31,400
Claimant history: Filed CLM-2024-0401 on 2024-07-18 for SAME VIN, different
loss type (hail damage vs. current collision), $28,900. Description similarity:
34%. One prior claim 2 years ago for a different vehicle.
Assessment: FLAGGED (moderate). Same VIN twice in 47 days is notable, though
loss types differ. Amounts are close ($2,500 difference). Description
similarity is low (34%), reducing duplicate likelihood. Score reflects the
VIN recurrence without the matching descriptions that would indicate strong
duplication.

=== EXAMPLE 4: CLEAN (Score: 8) ===
Claim: CLM-2024-0602
Claimant: Priya Patel, ID: C-5521
Loss date: 2024-10-01
Vehicle VIN: WBAJB0C55JB084219
Claimed amount: $9,800
Claimant history: No prior claims. New policyholder (6 months). No associated
vehicles with prior claims.
Assessment: CLEAN. First-time claimant. No history to produce duplicates.
Score: 8 (baseline, not zero, because all claims carry minimal residual risk).

=== NOW ANALYZE THIS CLAIM ===
Claim: {claim_id}
Claimant: {claimant_name}, ID: {claimant_id}
Loss date: {loss_date}
Vehicle VIN: {vin}
Claimed amount: ${claimed_amount}
Loss description: {loss_description}
Claimant history: {claimant_history_formatted}
Batch cross-reference: {batch_vin_matches}

Analyze using Chain-of-Thought reasoning (Steps 1-5 as defined) and produce
the pattern result JSON.
```

**Full Example — User Profile Classification (Few-Shot):**

```
SYSTEM: Classify the SIU investigator's experience level based on their
query style to adapt response detail. Use these examples:

=== EXAMPLE 1: JUNIOR ANALYST ===
Query: "What does it mean when a claim has a high provider network score?
I see CLM-005 scored 85 on that pattern but I'm not sure what to look for."
Classification: JUNIOR
Reasoning: Asks what a score means, unsure how to interpret, seeks guidance.
Response style: Detailed explanation of provider network analysis, define terms
like "referral ring" and "billing ratio," provide step-by-step walkthrough.

=== EXAMPLE 2: SENIOR LEAD ===
Query: "Show me all claims where provider billing ratio exceeds 2x and the
provider appears on more than 2 claims in this batch."
Classification: SENIOR
Reasoning: Uses specific technical terms (billing ratio, 2x threshold),
requests precise data filtering, knows exactly what to look for.
Response style: Concise data table. No definitions. Statistical context only.

=== EXAMPLE 3: JUNIOR ANALYST ===
Query: "Can you explain why CLM-003 was flagged? The score seems high but
I looked at the claim and nothing jumped out at me."
Classification: JUNIOR
Reasoning: Cannot identify fraud indicators independently, needs system to
explain what "jumped out" in the data.
Response style: Walk through each pattern that triggered, explain the specific
evidence, provide benchmarks for context.

=== EXAMPLE 4: SENIOR LEAD ===
Query: "Pull the provider frequency analysis for this batch. Flag any
providers on 3+ claims."
Classification: SENIOR
Reasoning: Direct command, specific threshold, expects structured output.
Response style: Provider frequency table with claim counts and billing stats.

=== CLASSIFY THIS QUERY ===
Query: "{user_query}"
```

#### 10.1.3 Chain-of-Thought (CoT) Prompts

**Used for:** The project's most analytically demanding tasks — fraud score reasoning (weighing multiple red flags, resolving conflicting signals, calculating composite scores with justification) and evidence synthesis per pattern (systematically evaluating each data point and building a coherent fraud narrative).

**Full Example — Suspicious Timing Pattern CoT Analysis:**

```
PATTERN: Suspicious Timing
CLAIM: CLM-2024-0847 — Marcus Webb — 2024-11-01 — $42,000

ENRICHED DATA:
Policy inception: 2024-09-18 (45 days before loss)
Policy expiration: 2025-09-18
Recent policy changes:
  - 2024-10-28: Added vehicle VIN 1FTFW1E53NFA12345 to fleet policy (4 days before loss)
  - 2024-10-15: Increased liability coverage from $500K to $1M
Loss date: 2024-11-01 (Friday)
Report date: 2024-11-04 (Monday)
Claimant prior claim dates: 2023-08-15, 2024-02-10, 2024-06-22
Season/weather: November, region had no unusual weather events

EVALUATE this claim for Suspicious Timing fraud indicators:

Step 1: Identify all relevant timing data points
- Policy inception was 2024-09-18; loss date 2024-11-01 = 44 days after inception.
  This is outside the 30-day "new policy fraud" window but still relatively new.
- Vehicle (VIN 1FTFW1E53NFA12345) was added to the fleet policy on 2024-10-28;
  loss date 2024-11-01 = 4 days later. This is within the 72-hour critical window
  (actually 4 days = 96 hours, just outside strict 72-hour threshold but very close).
- Liability coverage was increased on 2024-10-15; loss date 2024-11-01 = 17 days
  later. Coverage increase preceding a $42,000 claim is notable.
- Loss occurred Friday (2024-11-01), reported Monday (2024-11-04). Classic
  "weekend staging" window — loss on Friday, reported on Monday gives time to
  arrange evidence.
- No seasonal anomaly — November in this region has normal driving conditions.
- Claimant has filed 3 prior claims at irregular intervals (every 4-5 months),
  and this claim comes 4.3 months after the last one — consistent with a pattern
  of frequent filing but not by itself suspicious.

Step 2: Assess each indicator
- Policy age 44 days: MINOR RED FLAG. Outside the strict 30-day threshold but
  the policy is still very new. Severity: Minor.
- Vehicle added 4 days before loss: MAJOR RED FLAG. Adding a vehicle to a fleet
  policy 4 days before it's involved in a $42,000 collision is highly suspicious.
  While technically 96 hours (just outside the 72-hour threshold), the operational
  significance is identical — the vehicle was insured for less than a week before
  the loss. Severity: Major.
- Coverage increase 17 days before loss: MODERATE RED FLAG. Increasing liability
  from $500K to $1M and then filing a $42,000 claim within 3 weeks is concerning,
  though the claim amount is well below both old and new limits. Severity: Moderate.
- Friday loss, Monday report: MODERATE RED FLAG. Classic staging pattern. However,
  many legitimate losses also follow this pattern (Friday is a high-traffic day,
  and claims offices are closed on weekends). Severity: Moderate.
- No seasonal anomaly: NEUTRAL. Nothing unusual about the season.
- Claim frequency (every ~4 months): MINOR RED FLAG. Elevated frequency but not
  extreme. Some fleet operators with 50+ vehicles legitimately file frequently.

Step 3: Weigh the evidence
- Major flags: 1 (vehicle added 4 days before loss)
- Moderate flags: 2 (coverage increase 17 days before, Friday/Monday pattern)
- Minor flags: 2 (new policy 44 days, elevated frequency)
- Neutral: 1 (no seasonal anomaly)
- Exculpatory: None
- Coherence assessment: The flags tell a coherent timing story — a new policy
  (44 days), a vehicle added just 4 days before the loss, a coverage increase
  2.5 weeks before the loss, and a weekend staging window. Each flag individually
  could be coincidental, but together they paint a picture of pre-meditated
  preparation followed by a conveniently timed loss.

Step 4: Score the pattern
Multiple converging timing flags, anchored by the vehicle-addition flag (4 days):
  1 major (40 pts) + 2 moderate (20 pts) + 2 minor (8 pts) + coherence bonus (10 pts)
SCORE: 78

Step 5: Compile evidence
{
  "pattern": "suspicious_timing",
  "score": 78,
  "severity": "significant",
  "evidence": [
    {"indicator": "Vehicle added to fleet policy 4 days before loss", "value": "VIN added 2024-10-28, loss 2024-11-01", "benchmark": "Critical threshold: 72 hours", "severity": "major", "source": "lookup_policy_details"},
    {"indicator": "Liability coverage increased 17 days before loss", "value": "$500K → $1M on 2024-10-15", "benchmark": "Suspicious if within 30 days of large claim", "severity": "moderate", "source": "lookup_policy_details"},
    {"indicator": "Friday loss, Monday report (weekend staging window)", "value": "Loss 2024-11-01 (Fri), Reported 2024-11-04 (Mon)", "benchmark": "Classic staging pattern", "severity": "moderate", "source": "raw_claim_data"},
    {"indicator": "Policy only 44 days old at time of loss", "value": "Inception 2024-09-18, Loss 2024-11-01", "benchmark": "New policy fraud threshold: 30 days", "severity": "minor", "source": "lookup_policy_details"},
    {"indicator": "Elevated claim frequency", "value": "4 claims in 15 months (~3.2/year)", "benchmark": "Normal: <2 claims/year for fleet of this size", "severity": "minor", "source": "lookup_claimant_history"}
  ],
  "reasoning": "Multiple converging timing anomalies centered on policy preparation — a new vehicle added 4 days before loss, coverage increased 17 days before, on a 44-day-old policy, with a Friday-to-Monday staging window. The temporal clustering of policy changes immediately preceding the loss is the strongest indicator.",
  "exculpatory_factors": ["Claim amount ($42K) is well below both old ($500K) and new ($1M) liability limits", "Fleet operators may legitimately add vehicles frequently"]
}
```

**Full Example — Composite Score Reasoning (CoT):**

```
CLAIM: CLM-2024-0847 — Marcus Webb — $42,000

PATTERN SUB-SCORES:
- Duplicate/Similar Claims: 35 (minor VIN recurrence, different loss types)
- Suspicious Timing: 78 (vehicle added 4 days before, coverage upgrade, weekend)
- Inflated Amounts: 82 (repair $42K vs. vehicle market value $28.5K = 1.47x)
- Provider Network Anomalies: 71 (Elite Auto Body on 3 batch claims, billing 2.8x)

COMPUTE COMPOSITE SCORE using step-by-step reasoning:

Step 1: Apply weights
- Duplicate/Similar: 35 × 0.30 = 10.5
- Suspicious Timing: 78 × 0.20 = 15.6
- Inflated Amounts: 82 × 0.25 = 20.5
- Provider Network: 71 × 0.25 = 17.75

Step 2: Sum weighted scores
Composite = 10.5 + 15.6 + 20.5 + 17.75 = 64.35

Step 3: Determine confidence
Patterns scoring above 40 (flagged): Timing (78), Inflated (82), Provider (71) = 3/4 flagged
Confidence: HIGH (3+ patterns flagged with evidence)

Step 4: Assign priority tier
64.35 falls in 60–79 range → HIGH priority tier

Step 5: Assess narrative coherence
The three flagged patterns tell a coherent story: a vehicle was added to a new
policy 4 days before a collision (Timing), the repair estimate significantly
exceeds market value (Inflated), and the repair shop involved appears on 3 claims
in this batch with billing 2.8x the regional average (Provider). This pattern is
consistent with a provider-assisted fraud scheme where a colluding body shop
inflates repair estimates for recently insured or added vehicles.

RESULT:
Composite Score: 64.4 (rounded)
Confidence: HIGH
Priority Tier: HIGH
Narrative: Provider-assisted inflation scheme — recent vehicle addition + inflated
repairs + high-frequency provider involvement suggests organized fraud.
```

#### 10.1.4 ReAct Prompts

**Used for:** Tasks requiring iterative reasoning and tool use — primarily the Claim Data Enrichment Agent's sequential data gathering process, where the agent must reason about what data to collect, call a tool, observe the result, and decide what to collect next. Also used for cross-reference analysis where the Orchestrator chains multiple result lookups to answer complex queries.

**Full Example — Claim Data Enrichment (ReAct):**

```
SYSTEM: You are a Claim Data Enrichment specialist. Enrich the following
claim by gathering contextual data from available tools. Use a ReAct approach:
think about what data you need, call a tool, observe the result, then decide
your next action.

CLAIM TO ENRICH:
Claim ID: CLM-2024-0847
Claimant: Marcus Webb, ID: C-4821
Loss date: 2024-11-01
Vehicle: 2022 Ford F-150, VIN: 1FTFW1E53NFA12345
Provider: Elite Auto Body, ID: PRV-0093 (repair_shop)
Policy: FLT-2024-00312
Claimed amount: $42,000
Loss description: "Rear-end collision on I-95 northbound. Fleet truck struck
from behind at approximately 45 mph. Significant damage to rear bumper, bed,
tailgate, rear axle, and frame. Driver reports whiplash and lower back pain."

THOUGHT: I need to gather comprehensive background data on this claim. I'll
start with claimant history since knowing prior claims and fraud flags will
set context for everything else.

ACTION: lookup_claimant_history("C-4821")

OBSERVATION: {
  "claimant_id": "C-4821",
  "total_prior_claims": 4,
  "prior_claims": [
    {"claim_id": "CLM-2023-0391", "loss_date": "2023-08-15", "loss_type": "collision",
     "claimed_amount": 28900, "outcome": "paid", "fraud_flag": false},
    {"claim_id": "CLM-2024-0112", "loss_date": "2024-02-10", "loss_type": "comprehensive",
     "claimed_amount": 15200, "outcome": "paid", "fraud_flag": false},
    {"claim_id": "CLM-2024-0398", "loss_date": "2024-06-22", "loss_type": "collision",
     "claimed_amount": 33400, "outcome": "paid", "fraud_flag": false}
  ],
  "known_addresses": ["742 Industrial Blvd, Newark NJ", "1205 Commerce Dr, Edison NJ"],
  "associated_vehicles": [
    {"vin": "1FTFW1E53NFA12345", "year": 2022, "make": "Ford", "model": "F-150"},
    {"vin": "3C4PDCAB5JT500789", "year": 2021, "make": "Dodge", "model": "Journey"}
  ],
  "prior_fraud_flags": 0,
  "claim_frequency_per_year": 3.2,
  "total_prior_payouts": 77500
}

THOUGHT: 4 prior claims with $77.5K in total payouts, claim frequency of 3.2/year.
No prior fraud flags, but the frequency is notable. The VIN 1FTFW1E53NFA12345
appears in prior history — this same truck was involved before. Now I need
provider information to assess the repair shop.

ACTION: lookup_provider_network("PRV-0093", "repair_shop")

OBSERVATION: {
  "provider_id": "PRV-0093",
  "provider_name": "Elite Auto Body",
  "provider_type": "repair_shop",
  "license_status": "active",
  "total_claims_served": 47,
  "average_billing_amount": 38200,
  "regional_average_billing": 13500,
  "billing_ratio": 2.83,
  "referral_connections": [
    {"provider_id": "PRV-0187", "provider_name": "Garden State Medical Center", "shared_claims": 12},
    {"provider_id": "PRV-0214", "provider_name": "Bergen Law Associates", "shared_claims": 8}
  ],
  "geographic_service_area": "Northern NJ",
  "claims_outside_service_area": 5,
  "flagged_claims_count": 3
}

THOUGHT: Significant findings. Elite Auto Body bills an average of $38,200 —
that's 2.83x the regional average of $13,500. They have referral connections
to a medical center (12 shared claims) and a law firm (8 shared claims).
This looks like a potential referral ring. Now I need the vehicle valuation
to compare the $42K claim against market value.

ACTION: lookup_vehicle_valuation("1FTFW1E53NFA12345", 2022, "Ford", "F-150")

OBSERVATION: {
  "vin": "1FTFW1E53NFA12345",
  "year": 2022,
  "make": "Ford",
  "model": "F-150",
  "market_value": 28500,
  "salvage_value": 8550,
  "total_loss_threshold": 19950,
  "regional_repair_benchmarks": {
    "minor_collision": {"low": 2500, "median": 5800, "high": 11000},
    "major_collision": {"low": 8000, "median": 18500, "high": 32000},
    "comprehensive": {"low": 1500, "median": 4200, "high": 9000}
  },
  "prior_claims_on_vin": [
    {"claim_id": "CLM-2023-0391", "date": "2023-08-15", "amount": 28900},
    {"claim_id": "CLM-2024-0398", "date": "2024-06-22", "amount": 33400}
  ],
  "title_history": ["clean"]
}

THOUGHT: The claimed amount ($42,000) exceeds the vehicle's market value ($28,500)
by 47%. It also exceeds the total loss threshold ($19,950) by over 2x. The regional
benchmark for a major collision maxes out at $32,000, so $42K is 1.31x above even
the high end. Additionally, this VIN has 2 prior claims totaling $62,300 — more
than double the vehicle's current market value. Now I need policy details.

ACTION: lookup_policy_details("FLT-2024-00312")

OBSERVATION: {
  "policy_number": "FLT-2024-00312",
  "inception_date": "2024-09-18",
  "expiration_date": "2025-09-18",
  "days_since_inception": 44,
  "days_until_expiration": 321,
  "coverage_type": "Commercial Auto Fleet",
  "coverage_limits": {"liability": 1000000, "physical_damage": 100000},
  "premium_amount": 28400,
  "premium_payment_history": [
    {"date": "2024-09-18", "amount": 28400, "status": "paid"}
  ],
  "recent_policy_changes": [
    {"change_date": "2024-10-15", "change_type": "coverage_upgrade",
     "details": "Liability increased from $500K to $1M"},
    {"change_date": "2024-10-28", "change_type": "vehicle_addition",
     "details": "Added VIN 1FTFW1E53NFA12345 to fleet"}
  ],
  "vehicles_on_policy": [
    {"vin": "1FTFW1E53NFA12345", "date_added": "2024-10-28"},
    {"vin": "3C4PDCAB5JT500789", "date_added": "2024-09-18"},
    {"vin": "5N1AT2MV9KC700456", "date_added": "2024-09-18"}
  ],
  "policy_status": "active",
  "lapse_history": []
}

THOUGHT: Major findings. The vehicle (VIN ...12345) was added to the fleet
policy only 4 days before the loss. The policy itself is only 44 days old.
Liability coverage was increased from $500K to $1M on October 15, then the
vehicle was added October 28, and the loss occurred November 1. This is a
clear temporal escalation pattern. I now have all 4 enrichment sources.
Packaging the complete enriched data.

FINAL ENRICHED DATA PACKAGE:
{
  "claim_id": "CLM-2024-0847",
  "enrichment_status": "complete",
  "claimant_history": { ... },
  "provider_network": { ... },
  "vehicle_valuation": { ... },
  "policy_details": { ... }
}
```

**Full Example — Cross-Reference Query (ReAct, Orchestrator):**

```
USER (Diana): "Show me all claims in this batch involving Elite Auto Body
and tell me if there's a pattern."

THOUGHT: Diana is asking for a provider-specific cross-reference. I need to
filter batch results for claims involving Elite Auto Body (PRV-0093), then
analyze the pattern across those claims.

ACTION: query_results("claims involving provider PRV-0093", batch_id="BATCH-2024-W45")

OBSERVATION: 3 claims involve Elite Auto Body:
- CLM-005: $42,000, Score 87.3, Critical
- CLM-003: $38,500, Score 72.5, High
- CLM-007: $35,200, Score 65.1, High

THOUGHT: Diana is a senior lead — she wants concise, statistical analysis.
Three claims all using the same repair shop, all scoring High or Critical,
all with similar repair amounts ($35K–$42K). I should present the pattern
analysis with network evidence.

RESPONSE: "3 claims involve Elite Auto Body (PRV-0093):

| Claim | Amount | Fraud Score | Tier |
|-------|--------|-------------|------|
| CLM-005 | $42,000 | 87.3 | Critical |
| CLM-003 | $38,500 | 72.5 | High |
| CLM-007 | $35,200 | 65.1 | High |

Pattern indicators:
- Elite billing avg: $38,200 (2.83× regional avg of $13,500)
- Referral connections: Garden State Medical Center (12 shared claims),
  Bergen Law Associates (8 shared claims)
- 3/8 batch claims (37.5%) routed to same shop
- All 3 claims have repair estimates exceeding vehicle market value
- Consistent with an organized referral ring pattern."
```

### 10.2 Prompt Type Selection Matrix

| # | Task | Agent | Prompt Type | Why This Type |
|---|------|-------|-------------|---------------|
| 1 | Batch JSON validation | Orchestrator | **Zero-shot** | Rule-based field validation with clear pass/fail criteria; no examples needed — the rules are explicit |
| 2 | Pipeline status response | Orchestrator | **Zero-shot** | Report on known state (counts, timestamps); no reasoning or classification required |
| 3 | Batch summary generation | Orchestrator | **Zero-shot** | Aggregation of computed results into a summary; all data is already structured and scored |
| 4 | Claim drill-down response | Orchestrator | **Zero-shot** + profile adaptation | Retrieve and format stored results for a specific claim; adapt detail level per user profile |
| 5 | User profile classification | Orchestrator | **Few-shot** | Classify investigator experience level from query style; labeled examples calibrate the boundary between junior/senior |
| 6 | Query type classification | Orchestrator | **Few-shot** | Route incoming query to the correct handler (status, summary, drill-down, pattern, provider, explanation); examples anchor each category |
| 7 | Claimant history retrieval | Enrichment | **ReAct** | Agent reasons about what data to gather → calls `lookup_claimant_history` → observes results → proceeds to next tool |
| 8 | Provider network retrieval | Enrichment | **ReAct** | Agent reasons about provider context → calls `lookup_provider_network` → integrates with prior observations |
| 9 | Vehicle valuation retrieval | Enrichment | **ReAct** | Agent reasons about repair benchmark needs → calls `lookup_vehicle_valuation` → connects to claimant history and claim amount |
| 10 | Policy details retrieval | Enrichment | **ReAct** | Agent reasons about policy lifecycle context → calls `lookup_policy_details` → completes the enrichment data package |
| 11 | Duplicate/Similar Claims analysis | Fraud Analyzer | **Few-shot** + **CoT** | Few-shot examples calibrate what constitutes a "duplicate" vs. coincidence; CoT ensures step-by-step evidence weighing |
| 12 | Suspicious Timing analysis | Fraud Analyzer | **CoT** | Multi-factor temporal assessment requires explicit reasoning about how timing indicators interact and escalate |
| 13 | Inflated Amounts analysis | Fraud Analyzer | **CoT** | Comparing claimed amounts against benchmarks requires explicit reasoning about proportionality, context, and severity |
| 14 | Provider Network analysis | Fraud Analyzer | **CoT** | Evaluating network anomalies (referral rings, billing patterns) requires step-by-step investigation that connects multiple data points |
| 15 | Composite fraud score computation | Fraud Analyzer | **CoT** | Narrative coherence assessment — determining whether individual pattern flags tell a consistent fraud story — requires explicit reasoning |
| 16 | Investigation report generation | Report Generator | **Zero-shot** | Formatting pre-analyzed data into a structured report; all evidence and reasoning already exists from the Analyzer — the generator compiles and formats |
| 17 | Evidence summary compilation | Report Generator | **Zero-shot** | Deduplication and tabulation of evidence items; structured transformation, not analytical reasoning |
| 18 | Investigation priority calculation | Report Generator | **Zero-shot** | Formulaic calculation (fraud score × claim amount) with rule-based tier assignment |
| 19 | Cross-reference query (provider analysis) | Orchestrator | **ReAct** | Investigator asks a complex question requiring multiple result lookups → reason → query → observe → synthesize |
| 20 | Fraud pattern explanation (educational) | Orchestrator | **Few-shot** | Explain what a pattern means, adapted to user profile; examples show appropriate detail level for junior vs. senior |

---

## 11. Prompt Manager Design

### 11.1 What Is the Prompt Manager?

A lightweight Python module that manages all prompt templates across the 4 fraud detection agents. It's deliberately simple — no heavy frameworks, no database — just enough to demonstrate the concept of centralized prompt management.

**Core capabilities:**
1. **Stores** prompt templates as Python string constants, organized by agent
2. **Versions** each template (v1, v2, etc.) so you can compare fraud scoring calibration across prompt iterations
3. **Renders** templates with variables using Python `str.format_map`
4. **Tracks** which prompt version was used per call (feeds into Arize tracing — critical for understanding why a claim was scored differently across prompt versions)

> **Design philosophy:** Keep it simple. A dict-based prompt manager is something an intern can build in 1–2 hours and understand completely. It demonstrates the *concept* of prompt management without getting lost in library complexity. For a fraud detection pipeline processing N claims × 4 patterns, consistent prompt versioning is essential for reproducibility.

### 11.2 Directory Structure

```
prompt_manager/
├── __init__.py
├── manager.py                        # Core PromptManager class
└── templates/
    ├── __init__.py
    ├── orchestrator.py               # Orchestrator prompts (batch intake, status/query, aggregation)
    ├── enrichment.py                 # Claim Data Enrichment prompts (ReAct enrichment, data packaging)
    ├── fraud_analyzer.py             # Fraud Pattern Analyzer prompts (4 CoT patterns, composite scoring)
    └── report_generator.py           # Investigation Report Generator prompts (report formatting, evidence summary)
```

### 11.3 Core Implementation

```python
# prompt_manager/manager.py

from datetime import datetime


class PromptManager:
    """
    Simple dict-based prompt manager.
    Stores versioned prompt templates for all 4 fraud detection agents.
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
            agent: Agent name (e.g., "fraud_analyzer", "enrichment")
            name: Template name (e.g., "duplicate_claims_cot")
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

The Duplicate/Similar Claims CoT template demonstrates the Chain-of-Thought pattern used by the Fraud Pattern Analyzer. This is the most complex pattern because it requires cross-claim reasoning — comparing the current claim against the claimant's history to detect re-filed, recycled, or suspiciously similar claims.

```python
# prompt_manager/templates/fraud_analyzer.py

# Each template is a Python string with {variable} placeholders.
# Multiple versions allow comparison and iteration via Arize.

DUPLICATE_CLAIMS_COT_V1 = """
PATTERN: Duplicate / Similar Claims Detection
CLAIM: {claim_id} — {claimant_name} — {loss_date} — ${claimed_amount}

ENRICHED DATA:
- Claimant ID: {claimant_id}
- Prior claims count: {prior_claims_count}
- Prior claims detail:
{prior_claims_detail}
- Associated VINs: {associated_vins}
- Current vehicle VIN: {current_vin}
- Loss description: {loss_description}
- Loss type: {loss_type}
- Repair shop: {repair_shop}

Evaluate this claim for DUPLICATE or SIMILAR claim fraud indicators using
step-by-step reasoning.

Step 1: VIN Overlap Analysis
  - Compare the current claim vehicle VIN ({current_vin}) against all VINs
    in the claimant's history ({associated_vins}).
  - Check if the same VIN has had prior claims. If yes, note dates, amounts,
    and loss types of those prior claims.
  - A VIN with 3+ claims in 24 months is a major red flag.
  - A VIN with 2 claims in 12 months is a moderate red flag.

Step 2: Claimant Overlap Analysis
  - Check the claimant's total prior claims count ({prior_claims_count}).
  - Compute claim frequency: claims per year. Above 2.0/year is a red flag.
  - Check if the claimant has prior fraud flags. Any prior flag elevates severity.
  - Check if the claimant has claims across multiple policies (policy shopping).

Step 3: Timing Proximity Analysis
  - Compare the current loss date ({loss_date}) against all prior claim loss dates.
  - Two claims within 30 days on the same or related VINs = major red flag.
  - Two claims within 90 days with the same loss type = moderate red flag.
  - Look for a regular cadence (e.g., claims every 6 months — possible staged pattern).

Step 4: Incident Similarity Analysis
  - Compare the current loss description ({loss_description}) against prior
    claim loss descriptions.
  - Look for textual similarity: same body parts damaged (rear-end vs. rear-end),
    same road conditions, same time of day, same repair shop.
  - Compare claimed amounts: are they suspiciously close to prior claims
    (within 10% of a prior claim amount)?
  - Same repair shop appearing across multiple claims = provider collusion indicator.

Step 5: Score the Pattern (0–100)
  - 0–20: Clean — no meaningful duplicate/similar indicators
  - 21–40: Minor overlap — likely coincidental (e.g., one prior claim, different circumstances)
  - 41–60: Notable similarity — repeated patterns warrant monitoring
  - 61–80: Significant duplication indicators — investigation recommended
  - 81–100: Strong duplicate/recycle evidence — critical priority

Step 6: Compile Evidence
  - List each piece of evidence that contributed to the score.
  - Cite specific values (e.g., "VIN 1HGCM82633A004352 has 3 prior claims in 18 months").
  - Note exculpatory factors that mitigate concern (e.g., "prior claims involved different
    vehicles and different loss types").

OUTPUT FORMAT:
{{
  "pattern": "duplicate_similar",
  "score": <0-100>,
  "severity": "clean|minor|notable|significant|critical",
  "evidence": [
    {{"indicator": "...", "value": "...", "benchmark": "...", "severity": "...", "source": "..."}},
    ...
  ],
  "reasoning": "Step-by-step reasoning summary...",
  "exculpatory_factors": ["...", "..."]
}}
"""


def register_all(manager):
    """Register all fraud analyzer templates."""
    manager.register("fraud_analyzer", "duplicate_claims_cot", "v1", DUPLICATE_CLAIMS_COT_V1)
    # ... register suspicious_timing_cot, inflated_amounts_cot, provider_network_cot
    # ... register composite_scoring_cot
```

### 11.3.2 Usage in Agent Code

```python
# Inside the Fraud Pattern Analyzer, when running the Duplicate/Similar pattern:

prompt = prompt_manager.get_prompt(
    agent="fraud_analyzer",
    name="duplicate_claims_cot",
    variables={
        "claim_id": enriched_claim["claim_id"],
        "claimant_name": enriched_claim["claimant"]["name"],
        "claimant_id": enriched_claim["claimant"]["claimant_id"],
        "loss_date": enriched_claim["loss_date"],
        "claimed_amount": enriched_claim["claimed_amount"],
        "prior_claims_count": enriched_claim["claimant_history"]["total_prior_claims"],
        "prior_claims_detail": format_prior_claims(enriched_claim["claimant_history"]["prior_claims"]),
        "associated_vins": ", ".join(
            v["vin"] for v in enriched_claim["claimant_history"]["associated_vehicles"]
        ),
        "current_vin": enriched_claim["vehicle"]["vin"],
        "loss_description": enriched_claim["loss_description"],
        "loss_type": enriched_claim["loss_type"],
        "repair_shop": enriched_claim.get("repair_shop", {}).get("name", "N/A"),
    },
    version="v1"  # or "latest"
)

response = await model.generate_content(prompt)
```

### 11.4 Full Few-Shot Template Example

The Fraud Classification Few-Shot template is used by the Fraud Pattern Analyzer after all 4 patterns complete. It classifies the overall claim as FLAGGED or CLEAN based on the composite score and pattern results. The few-shot examples calibrate the model's classification threshold and teach it how to weigh conflicting signals.

```python
# prompt_manager/templates/fraud_analyzer.py

FRAUD_CLASSIFICATION_FEWSHOT_V1 = """
Based on the composite fraud score and individual pattern results, classify
this claim as FLAGGED (requires SIU investigation) or CLEAN (no action needed).
Assign a priority tier and provide a brief classification rationale.

CLASSIFICATION RULES:
- Composite score ≥ 60 → FLAGGED (High or Critical)
- Composite score 40–59 → FLAGGED (Medium) — warrants monitoring or light investigation
- Composite score < 40 → CLEAN — no action unless a single pattern scored ≥ 70

EXAMPLE 1 — FLAGGED (Critical):
Claim: CLM-2024-0847 — "Fleet truck rear-end collision, $42K repair estimate"
Composite Score: 83.5
Pattern Results:
  - Duplicate/Similar: 88 (3 prior claims on same VIN in 14 months, same repair shop)
  - Suspicious Timing: 72 (claim filed 22 days after policy inception)
  - Inflated Amounts: 85 (repair estimate $42,000 vs. vehicle market value $28,500 = 1.47x)
  - Provider Network: 91 (repair shop appears on 6 of 8 batch claims, billing 2.3x regional avg)
Classification: FLAGGED
Priority: CRITICAL
Rationale: All 4 patterns triggered with high scores. Repair shop PRV-0093 is a
central node — appearing on 75% of batch claims with inflated billing. Combined
with VIN history and new-policy timing, this claim shows strong indicators of
organized fraud involving provider collusion.

EXAMPLE 2 — FLAGGED (Medium):
Claim: CLM-2024-0851 — "Delivery van side-swipe in parking lot, $8,200 repair"
Composite Score: 47.2
Pattern Results:
  - Duplicate/Similar: 35 (1 prior claim 11 months ago, different vehicle, different loss type)
  - Suspicious Timing: 62 (claim filed on Monday for Friday evening incident, 68 hours delay)
  - Inflated Amounts: 48 (repair estimate slightly above regional median but within high range)
  - Provider Network: 40 (repair shop has normal billing ratios, but 2 referral connections)
Classification: FLAGGED
Priority: MEDIUM
Rationale: Timing pattern is the primary concern — Friday-to-Monday reporting delay
is a known fraud indicator for staged incidents. Individually each pattern is
borderline, but the combination of delayed reporting + slightly inflated repair +
provider with referral connections warrants a light review.

EXAMPLE 3 — CLEAN:
Claim: CLM-2024-0853 — "Company sedan hit by another driver at intersection, $5,100 repair"
Composite Score: 18.4
Pattern Results:
  - Duplicate/Similar: 12 (no prior claims for this claimant or VIN)
  - Suspicious Timing: 22 (claim filed same day as incident, policy active 3 years)
  - Inflated Amounts: 15 (repair estimate $5,100 vs. regional median $4,800 — reasonable)
  - Provider Network: 24 (repair shop has normal volume, no referral anomalies)
Classification: CLEAN
Priority: LOW
Rationale: No pattern triggered meaningfully. First-time claimant, long-standing
policy, prompt reporting, repair costs within normal range, no provider concerns.
Standard legitimate claim.

EXAMPLE 4 — FLAGGED (High), Single-Pattern Override:
Claim: CLM-2024-0856 — "Box truck windshield replacement, $1,200"
Composite Score: 34.8
Pattern Results:
  - Duplicate/Similar: 15 (no prior claims)
  - Suspicious Timing: 18 (normal timing, mid-policy)
  - Inflated Amounts: 22 (windshield cost within normal range)
  - Provider Network: 78 (glass shop under active investigation for billing fraud, license suspended)
Classification: FLAGGED
Priority: HIGH
Rationale: Although the composite score is below the normal threshold, the Provider
Network pattern triggered at 78 because the glass shop (PRV-0241) is under active
investigation with a suspended license. Single-pattern override rule applies:
any individual pattern ≥ 70 triggers FLAGGED regardless of composite score.

NOW CLASSIFY:
Claim: {claim_id} — "{loss_description}"
Composite Score: {composite_score}
Pattern Results:
  - Duplicate/Similar: {duplicate_score} ({duplicate_summary})
  - Suspicious Timing: {timing_score} ({timing_summary})
  - Inflated Amounts: {inflated_score} ({inflated_summary})
  - Provider Network: {provider_score} ({provider_summary})
Classification:
Priority:
Rationale:
"""


def register_classification_templates(manager):
    """Register fraud classification templates."""
    manager.register("fraud_analyzer", "fraud_classification_fewshot", "v1", FRAUD_CLASSIFICATION_FEWSHOT_V1)
```

---

## 12. User Profiling Design

### 12.1 What Is User Profiling?

The system builds a profile of each SIU investigator who uses the status/query chat interface and adapts its responses to match their expertise, specialization, and working style. A junior fraud analyst who's new to SIU gets detailed explanations of fraud patterns with evidence walkthroughs, while a senior SIU lead gets concise statistical summaries with confidence-flagged highlights.

User profiling applies only to the **conversational query interface** (Mode 2 of the Orchestrator) — the batch pipeline itself runs identically regardless of who initiates it. This means the same claim batch produces the same fraud scores and reports no matter who triggers the pipeline, but the *way results are presented and explained* in follow-up queries adapts to the investigator's profile.

### 12.2 Profile Attributes

Stored as a simple JSON file at `data/profiles/{user_id}.json`:

```json
{
  "user_id": "investigator_kevin",
  "name": "Kevin Park",
  "experience_level": "junior",
  "specialization": "general",
  "preferred_detail_level": "verbose",
  "preferred_output_mode": "full_report",
  "session_count": 3,
  "last_interaction": "2026-04-18T14:22:00Z",
  "profiling_complete": true
}
```

**Attribute definitions:**

| Attribute | Type | Values | Purpose |
|-----------|------|--------|---------|
| `user_id` | string | Unique identifier | Map sessions to profiles |
| `name` | string | Full name | Personalized greetings |
| `experience_level` | string | `"junior"`, `"mid"`, `"senior"` | Controls explanation depth and terminology |
| `specialization` | string | `"staged_accidents"`, `"provider_fraud"`, `"billing_inflation"`, `"general"` | Emphasizes the investigator's area of expertise in result presentation |
| `preferred_detail_level` | string | `"verbose"`, `"concise"` | Controls how much narrative context accompanies data |
| `preferred_output_mode` | string | `"full_report"`, `"exceptions_only"`, `"evidence_summary"` | Controls which result format the investigator sees first |
| `session_count` | int | 0+ | Tracks usage; profiling questions asked on first session only |
| `last_interaction` | string | ISO timestamp | Session tracking |

> **Why JSON files instead of SQLite?** For a 7-day project, JSON files eliminate database setup time while still demonstrating persistence. Each investigator gets their own file. Simple to read, write, and debug.

### 12.3 Profiling Questions (Asked in First Session)

The Fraud Pipeline Orchestrator asks these questions **naturally in conversation** when an investigator first uses the query interface after a batch run completes:

| # | Question | Purpose | Maps To |
|---|----------|---------|---------|
| Q1 | "Before we dig into the results — how long have you been working in fraud investigation or SIU?" | Determine experience level | `experience_level` |
| Q2 | "Do you specialize in a particular fraud area — staged accidents, provider fraud, billing inflation — or do you handle the full range?" | Identify fraud specialization | `specialization` |
| Q3 | "When reviewing flagged claims, do you prefer the full investigation report with all evidence, or a quick exceptions-only summary you can drill into?" | Set output mode | `preferred_output_mode` |
| Q4 | "For explanations and reasoning — do you like detailed walkthroughs of how the fraud score was calculated, or just the bottom line with key evidence?" | Set detail level | `preferred_detail_level` |

**Key Design Principle:** Questions are asked conversationally, not as a form. The agent weaves them into the natural flow after batch processing completes:

```
Agent: "Batch 'Weekly Auto Claims' — 8 claims processed.
        Summary: 3 flagged (1 Critical, 1 High, 1 Medium), 5 clean.
        Pipeline completed in 18 seconds.

        Before we go through the results — how long have you been
        working in fraud investigation or SIU?"

User:  "About 2 years. I transferred from claims adjusting last year."

Agent: "Great background — adjusting experience gives you a good eye for
        repair estimates and loss descriptions. Do you specialize in a
        particular fraud area — staged accidents, provider fraud, billing
        inflation — or do you handle the full range?"

User:  "Mostly general. My team lead assigns me whatever comes in."

Agent: "Got it — I'll give you the full picture on every flagged claim
        then. When reviewing flagged claims, do you prefer the full
        investigation report with all the evidence laid out, or a quick
        summary you can drill into?"

User:  "Full report please. I'm still learning what to look for."

Agent: "Perfect. And for the fraud score explanations — detailed
        walkthrough of how we got to the score, or just the key evidence?"

User:  "Detailed. I want to understand the reasoning."

Agent: "Understood. I'll walk you through each flagged claim step by step.
        Let's start with the highest-priority finding — CLM-2024-0847,
        scored 83.5 Critical..."
```

### 12.4 Adaptive Behavior Based on Profile

| Behavior | Junior Mode (Kevin — 2yr SIU) | Senior Mode (Diana — 14yr SIU Lead) |
|----------|-------------------------------|--------------------------------------|
| **Greeting** | Warm, acknowledges batch completion, explains what the scores mean | Brief — "3 flagged, 1 critical. Here's your exceptions list." |
| **Result presentation** | Full investigation report for each flagged claim, presented one at a time with narrative walkthroughs | Exceptions-only table: Claim ID, Score, Tier, Top Pattern, Key Evidence — all flagged claims in one view |
| **Fraud pattern explanations** | Defines each pattern ("Duplicate/Similar means we're looking for claims that resemble prior claims from the same claimant — same vehicle, same type of damage, similar amounts...") | Pattern name + score only — Diana knows what "Provider Network: 91" means |
| **Evidence depth** | Full evidence list with explanations ("The repair estimate of $42,000 is 1.47 times the vehicle's market value of $28,500 — that ratio is a major red flag because repairs exceeding the vehicle's value suggest the damage is exaggerated or the vehicle was already compromised") | Evidence table with key values only — "Repair/Value ratio: 1.47x (threshold: 1.0x)" |
| **Recommended actions** | Step-by-step investigation checklist with explanation of why each action matters ("First, request the original repair estimate from the shop — we need to compare it against our independent assessment") | Prioritized action list — "1. Subpoena PRV-0093 billing records. 2. Cross-reference VIN history. 3. Interview claimant re: prior claims." |
| **Terminology** | Defines SIU terms on first use ("SIU — Special Investigations Unit — that's your team"), explains scoring thresholds, defines what "composite score" means | Uses SIU shorthand freely — "SIU referral", "NICB alert", "EUO recommendation", "ISO ClaimSearch hit" |

### 12.5 Implementation

```python
# user_profile/profile_store.py

import json
import os
from datetime import datetime


class UserProfileStore:
    """JSON file-backed user profile storage. One file per SIU investigator."""

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
                "name": "",
                "experience_level": "unknown",
                "specialization": "general",
                "preferred_detail_level": "verbose",
                "preferred_output_mode": "full_report",
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
| **Traces** | Every LLM call, tool invocation, agent-to-agent delegation; spans tagged with `claim_id`, `batch_id`, `fraud_score`, `fraud_tier`, `pattern_flags` | Debug per-claim pipeline failures, understand batch-level vs. per-claim latency, trace fraud score provenance |
| **Evaluations** | Fraud score accuracy (does the composite score align with known fraud labels?), false positive rate, pattern detection precision per pattern | Ensure scoring quality — a fraud detection system is only as good as its precision/recall |
| **Annotations** | Human SIU expert review of fraud classifications — was the AI FLAGGED/CLEAN decision correct? Was the priority tier appropriate? | Build a golden dataset of human-validated fraud labels to improve few-shot examples and CoT reasoning |
| **Metrics** | Batch throughput (claims/minute), per-claim processing time, fraud score distribution, pattern hit rates (% of claims triggering each pattern), false positive tracking, token usage per batch | Operational monitoring — detect prompt regressions, cost spikes, and scoring drift |

**Fraud-Specific Span Attributes:**

Every trace span in the fraud pipeline carries these attributes, enabling Arize filtering and dashboard construction:

| Attribute | Type | Example | Purpose |
|-----------|------|---------|---------|
| `batch_id` | string | `"BATCH-2026-04-18-001"` | Group all claims from the same batch run |
| `claim_id` | string | `"CLM-2024-0847"` | Isolate per-claim trace |
| `fraud_score` | float | `83.5` | The composite fraud score for dashboards and distribution analysis |
| `fraud_tier` | string | `"critical"` | Priority tier for filtering and alerting |
| `pattern_flags` | string | `"duplicate_similar,suspicious_timing,inflated_amounts,provider_network"` | Comma-separated list of triggered patterns (score > 40) |
| `enrichment_completeness` | string | `"complete"` | Whether all 4 enrichment sources returned data (`complete`, `partial`, `failed`) |
| `report_generated` | boolean | `true` | Whether an investigation report was produced for this claim |
| `prompt_template` | string | `"duplicate_claims_cot"` | Which prompt template was used |
| `prompt_version` | string | `"v1"` | Which version — for A/B testing across prompt iterations |

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
    """Initialize Arize OpenTelemetry tracing for the fraud detection pipeline."""

    os.environ["ARIZE_SPACE_ID"] = "your-space-id"       # From Arize dashboard
    os.environ["ARIZE_API_KEY"] = "your-api-key"          # From Arize settings

    register_otel(
        endpoints=Endpoints.ARIZE,
        space_id=os.environ["ARIZE_SPACE_ID"],
        api_key=os.environ["ARIZE_API_KEY"],
        model_id="commercial-auto-fraud-detection",        # Project name in Arize
        model_version="1.0.0",
    )

    print("Arize tracing initialized successfully.")
```

#### Step 3: Instrument Agent Calls with Fraud-Specific Span Attributes

```python
# In each agent, wrap LLM calls with OpenTelemetry spans

from opentelemetry import trace

tracer = trace.get_tracer("fraud-detection-agents")

class FraudPatternAnalyzerAgent:
    async def analyze_patterns(self, claim_id: str, enriched_data: dict) -> dict:
        with tracer.start_as_current_span("fraud_pattern_analysis") as span:
            # Add fraud-specific attributes for Arize
            span.set_attribute("claim_id", claim_id)
            span.set_attribute("batch_id", enriched_data.get("batch_id", ""))

            # Run 4 patterns in parallel — each gets its own child span
            results = await asyncio.gather(
                self._run_pattern_with_span("duplicate_similar", enriched_data),
                self._run_pattern_with_span("suspicious_timing", enriched_data),
                self._run_pattern_with_span("inflated_amounts", enriched_data),
                self._run_pattern_with_span("provider_network", enriched_data),
            )

            # Compute composite and set fraud-specific span attributes
            composite = self._compute_composite(results)
            span.set_attribute("fraud_score", composite["composite_score"])
            span.set_attribute("fraud_tier", composite["priority_tier"])
            span.set_attribute("pattern_flags", ",".join(
                r["pattern"] for r in results if r["score"] > 40
            ))
            span.set_attribute("enrichment_completeness",
                               enriched_data.get("enrichment_status", "unknown"))
            span.set_attribute("prompt_template", "duplicate_claims_cot")
            span.set_attribute("prompt_version", "v1")

            return composite

    async def _run_pattern_with_span(self, pattern_name: str, data: dict) -> dict:
        """Run a single pattern analysis with its own OTel span."""
        with tracer.start_as_current_span(f"pattern_{pattern_name}") as span:
            span.set_attribute("pattern_name", pattern_name)
            span.set_attribute("claim_id", data["claim_id"])

            prompt = self.prompt_manager.get_prompt(
                agent="fraud_analyzer",
                name=f"{pattern_name}_cot",
                variables=self._build_pattern_variables(pattern_name, data),
            )
            result = await self.model.generate_content(prompt)
            parsed = self._parse_pattern_result(result)

            span.set_attribute("pattern_score", parsed["score"])
            span.set_attribute("evidence_count", len(parsed.get("evidence", [])))
            return parsed
```

#### Step 4: Set Up Annotation Queue

In the Arize dashboard:

1. **Create a new Space** → "Commercial Auto Fraud Detection"
2. **Go to Annotations** → Create new annotation queue
3. **Name:** "Fraud Score Accuracy Review"
4. **Filter:** traces where `fraud_tier` is `"critical"` or `"high"` (prioritize high-stakes classifications for human review)
5. **Annotation Schema:**
   - `classification_correct` (boolean) — Is the FLAGGED/CLEAN decision correct?
   - `tier_appropriate` (boolean) — Is the priority tier (Critical/High/Medium/Low) appropriate?
   - `false_positive` (boolean) — Was this a false positive (flagged but actually clean)?
   - `missed_pattern` (string, optional) — A fraud pattern that was missed or under-scored
   - `corrected_score` (float, optional) — What should the composite score be?
   - `reasoning_quality` (1–5) — How good is the CoT reasoning across patterns?
   - `notes` (text) — SIU reviewer comments

#### Step 5: Build Monitoring Dashboard

In Arize, create dashboard widgets for:

| Widget | Metric | Purpose |
|--------|--------|---------|
| Batch processing time | Total seconds per batch (end-to-end) | Pipeline performance — detect slowdowns from prompt bloat or API throttling |
| Per-claim latency | p50, p95, p99 per-claim processing time | Identify claims that take unusually long (large enrichment data, complex CoT) |
| Fraud score distribution | Histogram of composite scores across all claims | Spot scoring drift — if the distribution shifts, prompts may need recalibration |
| Pattern hit rates | % of claims triggering each pattern (score > 40) | Detect pattern imbalance — if Provider Network triggers on 80% of claims, the threshold may be too loose |
| False positive tracking | % of FLAGGED claims annotated as false positives | The key quality metric — target < 30% false positive rate |
| Tier distribution | Bar chart: Critical / High / Medium / Low per batch | Classification distribution monitoring |
| Token usage per batch | Input + output tokens per batch run | Cost tracking and prompt efficiency |
| Prompt version comparison | Fraud score distribution by prompt version | A/B testing prompts — compare v1 vs. v2 scoring calibration |
| Error rate | % of failed per-claim pipelines (enrichment failure, parse error) | Reliability monitoring |

### 13.3 Arize Integration Architecture

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                     FRAUD DETECTION BATCH PIPELINE                            │
│                                                                               │
│  ┌──────────────────────┐                                                     │
│  │ Fraud Pipeline        │                                                     │
│  │ Orchestrator          │── fan-out N claims ──────────────────────────┐     │
│  │ [OTel Spans:          │                                              │     │
│  │  batch_id,            │    ┌────────────┐  ┌────────────┐  ┌──────┐ │     │
│  │  batch_size,          │    │ Claim 1    │  │ Claim 2    │  │Clm N │ │     │
│  │  batch_duration]      │    │            │  │            │  │      │ │     │
│  └──────────────────────┘    │ ┌────────┐ │  │ ┌────────┐ │  │┌────┐│ │     │
│                               │ │Enrich  │ │  │ │Enrich  │ │  ││ E  ││ │     │
│                               │ │[Spans: │ │  │ │[Spans: │ │  ││    ││ │     │
│                               │ │claim_id│ │  │ │claim_id│ │  ││    ││ │     │
│                               │ │enrich_ │ │  │ │enrich_ │ │  ││    ││ │     │
│                               │ │status] │ │  │ │status] │ │  ││    ││ │     │
│                               │ └───┬────┘ │  │ └───┬────┘ │  │└──┬─┘│ │     │
│                               │     ▼      │  │     ▼      │  │   ▼  │ │     │
│                               │ ┌────────┐ │  │ ┌────────┐ │  │┌────┐│ │     │
│                               │ │Analyze │ │  │ │Analyze │ │  ││ A  ││ │     │
│                               │ │4 Patt. │ │  │ │4 Patt. │ │  ││    ││ │     │
│                               │ │[Spans: │ │  │ │[Spans: │ │  ││    ││ │     │
│                               │ │fraud_  │ │  │ │fraud_  │ │  ││    ││ │     │
│                               │ │score,  │ │  │ │score,  │ │  ││    ││ │     │
│                               │ │tier,   │ │  │ │tier,   │ │  ││    ││ │     │
│                               │ │flags]  │ │  │ │flags]  │ │  ││    ││ │     │
│                               │ └───┬────┘ │  │ └───┬────┘ │  │└──┬─┘│ │     │
│                               │     ▼      │  │     ▼      │  │   ▼  │ │     │
│                               │ ┌────────┐ │  │ ┌────────┐ │  │┌────┐│ │     │
│                               │ │Report  │ │  │ │Report  │ │  ││ R  ││ │     │
│                               │ │Gen.    │ │  │ │Gen.    │ │  ││    ││ │     │
│                               │ │[Spans: │ │  │ │[Spans: │ │  ││    ││ │     │
│                               │ │report_ │ │  │ │report_ │ │  ││    ││ │     │
│                               │ │gen.]   │ │  │ │gen.]   │ │  ││    ││ │     │
│                               │ └────────┘ │  │ └────────┘ │  │└────┘│ │     │
│                               └────────────┘  └────────────┘  └──────┘ │     │
│                                                                         │     │
│  N parallel claim traces, each with Enrich → Analyze → Report spans ───┘     │
│                                                                               │
└──────────────────────────────────┬────────────────────────────────────────────┘
                                   │
                          ┌────────▼──────────────────┐
                          │    OpenTelemetry           │
                          │    Collector               │
                          │                            │
                          │  Spans tagged with:        │
                          │  - batch_id                │
                          │  - claim_id                │
                          │  - fraud_score             │
                          │  - fraud_tier              │
                          │  - pattern_flags           │
                          │  - enrichment_completeness │
                          │  - report_generated        │
                          │  - prompt_version          │
                          └────────────┬───────────────┘
                                       │
                          ┌────────────▼───────────────┐
                          │                            │
                          │       Arize Cloud          │
                          │                            │
                          │  ┌──────────────────────┐  │
                          │  │ Traces (per-claim     │  │
                          │  │   + batch-level)      │  │
                          │  │ Dashboard (score      │  │
                          │  │   distribution,       │  │
                          │  │   pattern hit rates,  │  │
                          │  │   false positive %)   │  │
                          │  │ Annotations (SIU      │  │
                          │  │   expert review)      │  │
                          │  │ Evals (precision /    │  │
                          │  │   recall tracking)    │  │
                          │  └──────────────────────┘  │
                          │                            │
                          └────────────────────────────┘
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
    display_name="commercial-auto-fraud-detection",
    description="Multi-agent fraud detection pipeline for commercial auto insurance — ingests claim batches, runs 4 parallel fraud pattern analyses per claim, generates investigation reports for SIU",
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
# Send a test batch to the deployed agent
response = agent_engine.query(
    query="Process batch: data/batches/weekly_auto_claims.json"
)
print(response)
```

> **Batch processing consideration:** When deployed to Agent Engine, the dual-level parallel fan-out (`asyncio.gather` for N claims, each with `asyncio.gather` for 4 patterns) runs server-side. Ensure the Agent Engine timeout is set long enough — a batch of 8 claims with 4 patterns each = 32 parallel analysis tasks plus enrichment and report generation. Budget 60–90 seconds for an 8-claim batch.

### 14.2 Option B: Local Deployment (Fallback)

If GCP access is not available:

```bash
# Run agents locally using ADK CLI
# Terminal 1: Claim Data Enrichment Agent
adk run claim_data_enrichment --port 8002

# Terminal 2: Fraud Pattern Analyzer Agent
adk run fraud_pattern_analyzer --port 8003

# Terminal 3: Investigation Report Generator Agent
adk run investigation_report_generator --port 8004

# Terminal 4: Fraud Pipeline Orchestrator (main — includes web UI)
adk web fraud_pipeline_orchestrator --port 8001
```

The ADK web interface provides a chat UI for both batch triggering ("Process batch: weekly_auto_claims.json") and SIU query chat ("Show me the batch summary", "Details on CLM-2024-0847").

### 14.3 Environment Setup Guide

```bash
# 1. Clone the project
git clone <repo_url>
cd commercial-auto-fraud

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
adk web fraud_pipeline_orchestrator
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

### 15.1 Claim Input Schema

Each claim in a batch follows this structure. The batch JSON file contains an array of these claims.

```json
{
  "claim_id": "CLM-2024-0847",
  "claim_date": "2026-04-15",
  "loss_date": "2026-04-12",
  "loss_type": "collision",
  "loss_description": "Fleet truck rear-ended at intersection during delivery route. Driver reports other vehicle ran red light. Significant rear body damage, tailgate inoperable.",
  "claimant": {
    "claimant_id": "C-4821",
    "name": "Marcus Rivera",
    "phone": "555-0147",
    "email": "m.rivera@example.com",
    "address": "4520 Industrial Pkwy, Houston TX 77032",
    "relationship_to_insured": "employee_driver"
  },
  "vehicle": {
    "vin": "1HGCM82633A004352",
    "year": 2021,
    "make": "Ford",
    "model": "F-350 Super Duty",
    "vehicle_type": "fleet_truck",
    "use": "commercial_delivery"
  },
  "incident": {
    "location": "Intersection of FM 1960 and Aldine Westfield, Houston TX",
    "police_report_filed": true,
    "police_report_number": "HPD-2026-041247",
    "witnesses": 1,
    "other_party_involved": true,
    "other_party_info": {
      "name": "Jennifer Walsh",
      "vehicle": "2023 Toyota Camry",
      "insurance_carrier": "StateFarm"
    }
  },
  "estimates": {
    "repair_estimate": 42000.00,
    "repair_shop": {
      "provider_id": "PRV-0093",
      "name": "AllStar Truck & Auto Repair",
      "address": "8801 Airline Dr, Houston TX 77037"
    },
    "medical_estimate": 0.00,
    "rental_estimate": 3200.00,
    "rental_duration_days": 21,
    "towing_cost": 450.00
  },
  "claimed_amount": 45650.00,
  "policy_number": "POL-COMM-9184",
  "reported_by": "Fleet Manager — Tony Nguyen",
  "report_date": "2026-04-14"
}
```

### 15.2 Enriched Claim Schema

After the Claim Data Enrichment Agent processes a claim, all original fields are preserved and the following enrichment data is attached:

```json
{
  "claim_id": "CLM-2024-0847",
  "original_claim": { "...all fields from 15.1..." },
  "enrichment_status": "complete",
  "enrichment_timestamp": "2026-04-18T14:22:15Z",
  "claimant_history": {
    "claimant_id": "C-4821",
    "total_prior_claims": 4,
    "prior_claims": [
      {
        "claim_id": "CLM-2023-0312",
        "loss_date": "2025-02-10",
        "loss_type": "collision",
        "loss_description": "Rear-end collision at traffic light. Bumper and tailgate damage.",
        "claimed_amount": 38500.00,
        "outcome": "paid",
        "fraud_flag": false,
        "repair_shop_id": "PRV-0093"
      },
      {
        "claim_id": "CLM-2023-0589",
        "loss_date": "2025-08-22",
        "loss_type": "collision",
        "loss_description": "Side-swipe on highway. Door panel and fender damage.",
        "claimed_amount": 28200.00,
        "outcome": "paid",
        "fraud_flag": false,
        "repair_shop_id": "PRV-0093"
      },
      {
        "claim_id": "CLM-2024-0104",
        "loss_date": "2025-11-05",
        "loss_type": "comprehensive",
        "loss_description": "Windshield cracked by road debris on I-45.",
        "claimed_amount": 1800.00,
        "outcome": "paid",
        "fraud_flag": false,
        "repair_shop_id": "PRV-0241"
      },
      {
        "claim_id": "CLM-2024-0601",
        "loss_date": "2026-01-18",
        "loss_type": "collision",
        "loss_description": "Rear-end collision at stop sign. Tailgate and bed damage.",
        "claimed_amount": 35750.00,
        "outcome": "under_review",
        "fraud_flag": true,
        "repair_shop_id": "PRV-0093"
      }
    ],
    "known_addresses": [
      "4520 Industrial Pkwy, Houston TX 77032",
      "1200 Commerce St, Houston TX 77002"
    ],
    "associated_vehicles": [
      {"vin": "1HGCM82633A004352", "year": 2021, "make": "Ford", "model": "F-350"},
      {"vin": "3FADP4BJ7BM112908", "year": 2019, "make": "Ford", "model": "E-350"}
    ],
    "associated_policies": ["POL-COMM-9184", "POL-COMM-7622"],
    "prior_fraud_flags": 1,
    "claim_frequency_per_year": 2.7,
    "total_prior_payouts": 104250.00
  },
  "provider_network": {
    "provider_id": "PRV-0093",
    "provider_name": "AllStar Truck & Auto Repair",
    "provider_type": "repair_shop",
    "license_status": "active",
    "total_claims_served": 47,
    "average_billing_amount": 36800.00,
    "regional_average_billing": 16200.00,
    "billing_ratio": 2.27,
    "referral_connections": [
      {"provider_id": "PRV-0108", "provider_name": "QuickTow Houston", "shared_claims": 12},
      {"provider_id": "PRV-0241", "provider_name": "ClearView Glass & Auto", "shared_claims": 8}
    ],
    "geographic_service_area": "Harris County, TX",
    "claims_outside_service_area": 3,
    "flagged_claims_count": 5
  },
  "vehicle_valuation": {
    "vin": "1HGCM82633A004352",
    "year": 2021,
    "make": "Ford",
    "model": "F-350 Super Duty",
    "market_value": 28500.00,
    "salvage_value": 8550.00,
    "total_loss_threshold": 21375.00,
    "regional_repair_benchmarks": {
      "minor_collision": {"low": 2500, "median": 5800, "high": 9500},
      "major_collision": {"low": 8000, "median": 16200, "high": 28000},
      "comprehensive": {"low": 500, "median": 1800, "high": 4200}
    },
    "prior_claims_on_vin": [
      {"claim_id": "CLM-2023-0312", "date": "2025-02-10", "amount": 38500.00},
      {"claim_id": "CLM-2024-0601", "date": "2026-01-18", "amount": 35750.00}
    ],
    "title_history": ["clean"]
  },
  "policy_details": {
    "policy_number": "POL-COMM-9184",
    "inception_date": "2026-03-25",
    "expiration_date": "2027-03-25",
    "days_since_inception": 21,
    "days_until_expiration": 341,
    "coverage_type": "commercial_auto_fleet",
    "coverage_limits": {
      "bodily_injury": 1000000,
      "property_damage": 500000,
      "comprehensive": 100000,
      "collision": 100000,
      "uninsured_motorist": 500000
    },
    "premium_amount": 28400.00,
    "premium_payment_history": [
      {"date": "2026-03-25", "amount": 28400.00, "status": "paid"}
    ],
    "recent_policy_changes": [
      {
        "change_date": "2026-03-20",
        "change_type": "new_policy",
        "details": "New commercial fleet policy — transferred from prior carrier"
      }
    ],
    "vehicles_on_policy": [
      {"vin": "1HGCM82633A004352", "date_added": "2026-03-25"},
      {"vin": "3FADP4BJ7BM112908", "date_added": "2026-03-25"}
    ],
    "policy_status": "active",
    "lapse_history": []
  }
}
```

### 15.3 Fraud Analysis Output Schema

The output of the Fraud Pattern Analyzer for a single claim — containing all 4 pattern sub-scores, evidence lists, composite score, and priority tier:

```json
{
  "claim_id": "CLM-2024-0847",
  "analysis_timestamp": "2026-04-18T14:22:42Z",
  "pattern_results": [
    {
      "pattern_name": "duplicate_similar",
      "score": 88,
      "severity": "critical",
      "flags": [
        "3 prior claims on same VIN in 14 months",
        "Same repair shop (PRV-0093) on 3 of 4 VIN claims",
        "Prior claim CLM-2024-0601 has active fraud flag",
        "Loss descriptions share rear-end/tailgate pattern"
      ],
      "evidence": [
        {
          "indicator": "VIN claim frequency",
          "value": "3 claims in 14 months",
          "benchmark": "< 1 claim per 24 months",
          "severity": "major",
          "source": "claimant_history"
        },
        {
          "indicator": "Same repair shop recurrence",
          "value": "PRV-0093 on 3 of 4 claims",
          "benchmark": "Random shop selection expected",
          "severity": "major",
          "source": "claimant_history + provider_network"
        },
        {
          "indicator": "Active fraud flag on prior claim",
          "value": "CLM-2024-0601 flagged",
          "benchmark": "0 prior flags",
          "severity": "major",
          "source": "claimant_history"
        }
      ],
      "reasoning": "VIN 1HGCM82633A004352 has accumulated 3 collision claims in 14 months, all involving rear-end or tailgate damage, and 3 of 4 were repaired at the same shop (PRV-0093). The most recent prior claim (CLM-2024-0601) already has an active fraud flag. This pattern is consistent with repeated staged rear-end collisions funneled to a colluding repair shop."
    },
    {
      "pattern_name": "suspicious_timing",
      "score": 72,
      "severity": "significant",
      "flags": [
        "Claim filed 21 days after policy inception",
        "Policy is a new transfer from prior carrier",
        "Loss on Friday, reported following Monday"
      ],
      "evidence": [
        {
          "indicator": "Days since policy inception",
          "value": "21 days",
          "benchmark": "> 90 days considered normal",
          "severity": "moderate",
          "source": "policy_details"
        },
        {
          "indicator": "Reporting delay",
          "value": "2 days (Friday loss → Monday report)",
          "benchmark": "Same day or next business day",
          "severity": "minor",
          "source": "claim_data"
        }
      ],
      "reasoning": "The policy was just established 21 days before the loss — this is within the high-risk new-policy window. Combined with the policy being a transfer from a prior carrier (possible carrier-shopping after scrutiny) and a Friday-to-Monday reporting delay, the timing indicators are concerning."
    },
    {
      "pattern_name": "inflated_amounts",
      "score": 85,
      "severity": "critical",
      "flags": [
        "Repair estimate $42,000 vs. vehicle market value $28,500 (1.47x)",
        "Repair exceeds total loss threshold by $20,625",
        "Repair estimate 2.59x regional major collision median"
      ],
      "evidence": [
        {
          "indicator": "Repair-to-value ratio",
          "value": "1.47x ($42,000 / $28,500)",
          "benchmark": "< 0.75x (total loss threshold)",
          "severity": "major",
          "source": "vehicle_valuation"
        },
        {
          "indicator": "Repair vs. regional benchmark",
          "value": "$42,000 vs. $16,200 median",
          "benchmark": "< $28,000 (regional high)",
          "severity": "major",
          "source": "vehicle_valuation"
        }
      ],
      "reasoning": "The repair estimate of $42,000 is 1.47 times the vehicle's market value of $28,500 and far exceeds the total loss threshold of $21,375. The estimate is also 2.59x the regional median for major collisions. This vehicle should likely be totaled, not repaired at 147% of its value."
    },
    {
      "pattern_name": "provider_network",
      "score": 91,
      "severity": "critical",
      "flags": [
        "PRV-0093 billing ratio 2.27x regional average",
        "PRV-0093 appears on 6 of 8 batch claims (75%)",
        "PRV-0093 has circular referral pattern with PRV-0108 and PRV-0241",
        "5 prior flagged claims associated with PRV-0093"
      ],
      "evidence": [
        {
          "indicator": "Provider billing ratio",
          "value": "2.27x regional average ($36,800 vs. $16,200)",
          "benchmark": "< 1.5x considered normal",
          "severity": "major",
          "source": "provider_network"
        },
        {
          "indicator": "Batch claim concentration",
          "value": "6 of 8 claims (75%)",
          "benchmark": "< 15% expected for any single provider",
          "severity": "major",
          "source": "provider_network + batch_analysis"
        },
        {
          "indicator": "Referral ring detected",
          "value": "PRV-0093 ↔ PRV-0108 (12 shared claims) ↔ PRV-0241 (8 shared claims)",
          "benchmark": "< 3 shared claims expected",
          "severity": "major",
          "source": "provider_network"
        }
      ],
      "reasoning": "PRV-0093 (AllStar Truck & Auto Repair) is a central node in what appears to be a fraud ring. The shop appears on 75% of batch claims, bills 2.27x the regional average, has circular referral connections with a tow company and a glass shop, and has 5 prior flagged claims. This is the strongest indicator of organized provider-driven fraud."
    }
  ],
  "composite_fraud_score": 83.5,
  "confidence": "high",
  "fraud_tier": "critical",
  "priority": "P1-Immediate",
  "patterns_flagged": 4,
  "classification": "FLAGGED"
}
```

### 15.4 Investigation Report Schema

The Investigation Report Generator produces a Markdown-formatted report for each claim scoring ≥40. The report follows this structure:

```markdown
# Investigation Report: CLM-2024-0847

---

## Case Header

| Field | Value |
|-------|-------|
| **Claim ID** | CLM-2024-0847 |
| **Claimant** | Marcus Rivera (C-4821) |
| **Loss Date** | 2026-04-12 |
| **Claimed Amount** | $45,650.00 |
| **Policy Number** | POL-COMM-9184 |
| **Vehicle** | 2021 Ford F-350 Super Duty (VIN: 1HGCM82633A004352) |
| **Composite Fraud Score** | 83.5 / 100 |
| **Priority Tier** | CRITICAL |
| **Confidence** | HIGH (4/4 patterns flagged) |
| **Investigation Priority** | P1 — Immediate |

---

## Executive Summary

CLM-2024-0847 is a $45,650 commercial fleet collision claim that triggered all
4 fraud detection patterns at high severity. The claim involves a repeat
claimant (4 prior claims in 18 months), a repair estimate that exceeds the
vehicle's market value by 47%, and a repair shop (PRV-0093) that appears on
75% of this batch's claims with billing 2.27x the regional average. The
combination of repeat claimant behavior, inflated estimates, and provider
network anomalies is consistent with an organized fraud ring involving
staged rear-end collisions funneled to a colluding repair shop.

---

## Pattern Analysis Detail

### Duplicate/Similar Claims — Score: 88 (Critical)
[Per-pattern breakdown with evidence and CoT reasoning summary]

### Suspicious Timing — Score: 72 (Significant)
[Per-pattern breakdown with evidence and CoT reasoning summary]

### Inflated Amounts — Score: 85 (Critical)
[Per-pattern breakdown with evidence and CoT reasoning summary]

### Provider Network Anomalies — Score: 91 (Critical)
[Per-pattern breakdown with evidence and CoT reasoning summary]

---

## Evidence Summary

| # | Evidence Item | Value | Benchmark | Pattern | Severity | Source |
|---|---------------|-------|-----------|---------|----------|--------|
| 1 | VIN claim frequency | 3 claims / 14 months | < 1 / 24 months | Duplicate | Major | claimant_history |
| 2 | Same repair shop | PRV-0093 on 3/4 claims | Random selection | Duplicate | Major | claimant_history |
| 3 | Prior fraud flag | CLM-2024-0601 flagged | 0 prior flags | Duplicate | Major | claimant_history |
| 4 | Days since inception | 21 days | > 90 days normal | Timing | Moderate | policy_details |
| 5 | Repair/value ratio | 1.47x | < 0.75x | Inflated | Major | vehicle_valuation |
| 6 | Repair vs. benchmark | $42K vs. $16.2K median | < $28K high | Inflated | Major | vehicle_valuation |
| 7 | Provider billing ratio | 2.27x regional avg | < 1.5x | Provider | Major | provider_network |
| 8 | Batch concentration | 75% of claims | < 15% expected | Provider | Major | provider_network |
| 9 | Referral ring | 3 providers, 20 shared claims | < 3 shared | Provider | Major | provider_network |

---

## Risk Assessment

| Factor | Assessment |
|--------|-----------|
| **Estimated Fraud Exposure** | $38,122 (claim amount × fraud score) |
| **Ring Indicators** | Yes — PRV-0093 appears across multiple claimants and batches |
| **Repeat Offender Risk** | High — claimant has 1 active fraud flag, 4 claims in 18 months |
| **Escalation Level** | SIU Lead review required |

---

## Recommended Actions

1. **Immediate:** Place claim on hold — do not issue payment pending investigation
2. **Provider investigation:** Subpoena PRV-0093 billing records for past 24 months; compare line-item charges against manufacturer repair guides
3. **Claimant examination:** Schedule Examination Under Oath (EUO) for Marcus Rivera regarding prior claim history and relationship with PRV-0093
4. **Cross-reference batch:** Investigate all 6 batch claims involving PRV-0093 as a coordinated ring investigation, not individual claims
5. **Vehicle inspection:** Commission independent appraisal of VIN 1HGCM82633A004352 to verify damage extent and compare against $42,000 estimate
6. **NICB referral:** Submit to National Insurance Crime Bureau for cross-carrier pattern matching
```

### 15.5 Batch Summary Schema

After all claims in a batch are processed, the Orchestrator aggregates results into a batch summary:

```json
{
  "batch_id": "BATCH-2026-04-18-001",
  "batch_name": "Weekly Auto Claims — Week of 2026-04-14",
  "processing_timestamp": "2026-04-18T14:23:05Z",
  "processing_duration_seconds": 18.4,
  "claims_processed": 8,
  "claims_valid": 8,
  "claims_invalid": 0,
  "fraud_distribution": {
    "critical": 1,
    "high": 1,
    "medium": 1,
    "low": 5
  },
  "flagged_count": 3,
  "clean_count": 5,
  "average_fraud_score": 34.2,
  "median_fraud_score": 22.8,
  "top_flagged_claims": [
    {
      "claim_id": "CLM-2024-0847",
      "claimant": "Marcus Rivera",
      "claimed_amount": 45650.00,
      "composite_score": 83.5,
      "tier": "critical",
      "top_pattern": "provider_network",
      "top_pattern_score": 91
    },
    {
      "claim_id": "CLM-2024-0849",
      "claimant": "David Chen",
      "claimed_amount": 67200.00,
      "composite_score": 68.3,
      "tier": "high",
      "top_pattern": "inflated_amounts",
      "top_pattern_score": 82
    },
    {
      "claim_id": "CLM-2024-0851",
      "claimant": "Lisa Thompson",
      "claimed_amount": 8200.00,
      "composite_score": 47.2,
      "tier": "medium",
      "top_pattern": "suspicious_timing",
      "top_pattern_score": 62
    }
  ],
  "pattern_statistics": {
    "duplicate_similar": {
      "average_score": 28.4,
      "max_score": 88,
      "claims_triggered": 2,
      "trigger_rate_pct": 25.0
    },
    "suspicious_timing": {
      "average_score": 31.6,
      "max_score": 72,
      "claims_triggered": 2,
      "trigger_rate_pct": 25.0
    },
    "inflated_amounts": {
      "average_score": 35.2,
      "max_score": 85,
      "claims_triggered": 3,
      "trigger_rate_pct": 37.5
    },
    "provider_network": {
      "average_score": 38.8,
      "max_score": 91,
      "claims_triggered": 2,
      "trigger_rate_pct": 25.0
    }
  },
  "provider_frequency": [
    {
      "provider_id": "PRV-0093",
      "provider_name": "AllStar Truck & Auto Repair",
      "claims_count": 6,
      "batch_percentage": 75.0,
      "average_fraud_score_of_claims": 54.2,
      "alert": "HIGH — dominates batch, investigate for ring activity"
    }
  ],
  "reports_generated": 3,
  "total_flagged_exposure": 121050.00
}
```

### 15.6 Mock Data Strategy

The project uses **2 mock claim batches** designed to exercise all 4 fraud patterns and produce a range of fraud scores from clean to critical. Each batch is a JSON file in `data/batches/`.

#### Batch 1: "Weekly Auto Claims" — 8 Claims (5 Clean, 3 Flagged)

This batch simulates a typical weekly intake of commercial auto claims. Most claims are legitimate, but three are suspicious — two connected through a common repair shop and one with timing anomalies.

| # | Claim ID | Claimant | Description | Expected Tier | Primary Pattern |
|---|----------|----------|-------------|---------------|-----------------|
| 1 | CLM-2024-0847 | Marcus Rivera | Fleet truck rear-end, $42K repair at PRV-0093 | **Critical** | Provider Network + Duplicate + Inflated |
| 2 | CLM-2024-0848 | Sarah Kim | Company car fender bender in parking garage, $4,200 | Low (Clean) | — |
| 3 | CLM-2024-0849 | David Chen | Cargo van rollover, $67K total loss claim at PRV-0093 | **High** | Inflated + Provider Network |
| 4 | CLM-2024-0850 | Ana Morales | Delivery sedan rear window broken by debris, $1,100 | Low (Clean) | — |
| 5 | CLM-2024-0851 | Lisa Thompson | Delivery van side-swipe, $8,200 — Friday to Monday report | **Medium** | Suspicious Timing |
| 6 | CLM-2024-0852 | James Wright | Fleet truck mirror clipped by passing vehicle, $950 | Low (Clean) | — |
| 7 | CLM-2024-0853 | Robert Kim | Company sedan intersection collision, $5,100 | Low (Clean) | — |
| 8 | CLM-2024-0854 | Michelle Davis | Company SUV rear-ended at stoplight, $7,800 | Low (Clean) | — |

#### Batch 2: "Flagged Commercial Claims" — 6 Claims (All Flagged)

This batch simulates a pre-filtered set of claims that were already flagged by a rules engine. All claims have some fraud indicators — the AI system must differentiate severity and prioritize. This exercises the full scoring range from Medium to Critical.

| # | Claim ID | Claimant | Description | Expected Tier | Primary Pattern |
|---|----------|----------|-------------|---------------|-----------------|
| 1 | CLM-2024-0901 | Carlos Mendez | 3 rear-end claims in 12 months, all at same shop, escalating amounts | **Critical** | Duplicate + Provider + Inflated |
| 2 | CLM-2024-0902 | Brian Foster | $95K repair on $52K vehicle, new policy (18 days), suspended-license shop | **Critical** | Inflated + Timing + Provider |
| 3 | CLM-2024-0903 | Patricia Wong | Claim filed 2 days before policy cancellation, 4 prior claims | **High** | Timing + Duplicate |
| 4 | CLM-2024-0904 | Derek Johnson | Fleet truck rear-end, $28K — shop has referral ring with tow company | **High** | Provider Network |
| 5 | CLM-2024-0905 | Amy Williams | Delivery van — repair estimate $18K vs. $14K median, Friday evening loss | **Medium** | Inflated (borderline) + Timing (minor) |
| 6 | CLM-2024-0906 | Kevin Park | Company car side-swipe, $6,800 — same shop as CLM-0904, but first claim ever | **Medium** | Provider (linked) |

#### Mock API Code Snippet

```python
# mock_apis/claim_data.py

import json
import os


def load_batch(batch_name: str) -> dict:
    """
    Load a claim batch from the mock data directory.

    Args:
        batch_name: "weekly_auto_claims" or "flagged_commercial_claims"

    Returns:
        Batch dict with metadata and claims array.
    """
    batch_path = os.path.join("data", "batches", f"{batch_name}.json")
    with open(batch_path) as f:
        return json.load(f)


def lookup_claimant_history(claimant_id: str) -> dict:
    """Mock claimant history lookup from pre-built data files."""
    history_path = os.path.join("data", "mock", "claimant_history", f"{claimant_id}.json")
    if os.path.exists(history_path):
        with open(history_path) as f:
            return json.load(f)
    return {
        "claimant_id": claimant_id,
        "total_prior_claims": 0,
        "prior_claims": [],
        "known_addresses": [],
        "associated_vehicles": [],
        "associated_policies": [],
        "prior_fraud_flags": 0,
        "claim_frequency_per_year": 0.0,
        "total_prior_payouts": 0.0,
    }


def lookup_provider_network(provider_id: str, provider_type: str) -> dict:
    """Mock provider network lookup from pre-built data files."""
    provider_path = os.path.join("data", "mock", "providers", f"{provider_id}.json")
    if os.path.exists(provider_path):
        with open(provider_path) as f:
            return json.load(f)
    return {
        "provider_id": provider_id,
        "provider_name": "Unknown Provider",
        "provider_type": provider_type,
        "license_status": "active",
        "total_claims_served": 0,
        "average_billing_amount": 0.0,
        "regional_average_billing": 0.0,
        "billing_ratio": 0.0,
        "referral_connections": [],
        "geographic_service_area": "Unknown",
        "claims_outside_service_area": 0,
        "flagged_claims_count": 0,
    }


def lookup_vehicle_valuation(vin: str, year: int, make: str, model: str) -> dict:
    """Mock vehicle valuation lookup from pre-built data files."""
    vehicle_path = os.path.join("data", "mock", "vehicles", f"{vin}.json")
    if os.path.exists(vehicle_path):
        with open(vehicle_path) as f:
            return json.load(f)
    return {
        "vin": vin,
        "year": year,
        "make": make,
        "model": model,
        "market_value": 0.0,
        "salvage_value": 0.0,
        "total_loss_threshold": 0.0,
        "regional_repair_benchmarks": {},
        "prior_claims_on_vin": [],
        "title_history": ["clean"],
    }


def lookup_policy_details(policy_number: str) -> dict:
    """Mock policy details lookup from pre-built data files."""
    policy_path = os.path.join("data", "mock", "policies", f"{policy_number}.json")
    if os.path.exists(policy_path):
        with open(policy_path) as f:
            return json.load(f)
    return {
        "policy_number": policy_number,
        "inception_date": "2025-01-01",
        "expiration_date": "2026-01-01",
        "days_since_inception": 365,
        "days_until_expiration": 0,
        "coverage_type": "commercial_auto",
        "coverage_limits": {},
        "premium_amount": 0.0,
        "premium_payment_history": [],
        "recent_policy_changes": [],
        "vehicles_on_policy": [],
        "policy_status": "active",
        "lapse_history": [],
    }
```

---

## 16. Tech Stack & Tools

### 16.1 Complete Tech Stack

| Category | Technology | Version | Purpose |
|----------|-----------|---------|---------|
| **Language** | Python | 3.11+ | Primary development language |
| **Agent Framework** | Google ADK | Latest | Agent development, orchestration, dual-level parallel fan-out (N claims × 4 patterns) |
| **LLM** | Gemini 2.5 Pro | Latest | Fraud Pipeline Orchestrator (complex multi-step orchestration, SIU query reasoning) and Fraud Pattern Analyzer (CoT fraud evidence reasoning across 4 patterns) |
| **LLM** | Gemini 2.5 Flash | Latest | Claim Data Enrichment (speed-critical parallel data retrieval, N instances) and Investigation Report Generator (structured report formatting) |
| **Agent Communication** | A2A SDK | Latest | Inter-agent communication; Agent Cards for all 4 agents |
| **UI** | **ADK Web** (built-in) | Latest | Chat UI via `adk web` for batch triggering + SIU investigator query chat — zero build needed |
| **Prompt Templates** | Python strings (`str.format_map`) | N/A | Simple dict-based Prompt Manager, no extra dependency |
| **Storage** | JSON files | Built-in | Claim batches, enriched claims, fraud results, batch summaries, user profiles |
| **Monitoring** | Arize (Cloud) | Latest | Tracing, annotation, evaluation — per-claim and batch-level fraud scoring quality |
| **Tracing** | OpenTelemetry | 1.20+ | Distributed tracing with nested parallel span support (N claims × 4 pattern spans) |
| **Data Validation** | Pydantic | 2.0+ | Data model validation for claim input, enriched data, and fraud analysis schemas |
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
│ Setup  │───►│Enrichmt│───►│ Fraud  │───►│ Orch + │───►│ A2A +  │───►│ Arize  │───►│ Demo   │───►│Stretch │
│ + Mock │    │ Agent  │    │Analyzer│    │ Report │    │Profile │    │+Deploy │    │ Prep   │    │ Goals  │
│ Claims │    │(ReAct) │    │(CoT×4) │    │+Pipeline│   │+Query  │    │        │    │        │    │        │
└────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘    └────────┘
 Foundation    1st Agent      2nd Agent     Wire it up    A2A + UX     Observe      Ship it!      Polish
```

### Day 1: Foundation + Mock Claim Data
**Goal:** Project scaffolded, Gemini connected, 2 mock claim batches ready (8 + 6 claims), Prompt Manager operational.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 1.1 | Set up Python project | Create venv, install `google-adk`, `google-genai`, `python-dotenv` | `pip list` shows packages |
| 1.2 | Get Gemini API key | Create key at ai.google.dev, save in `.env` | Test call returns response |
| 1.3 | Hello World agent | Create a minimal ADK agent, run with `adk web`, chat with it | Agent responds in browser |
| 1.4 | Create project structure | Set up folders: `agents/`, `prompt_manager/`, `mock_apis/`, `user_profile/`, `data/claims/`, `telemetry/`, `a2a/` | Folders exist |
| 1.5 | Build 2 mock claim batches | Create JSON files for **"Weekly Auto Claims"** (8 claims — 5 clean + 3 fraud) and **"Flagged Commercial Claims"** (6 claims — all flagged, varying severity). Include claimant info, loss details, vehicle VINs, provider IDs, policy numbers. See Section 15 for full schemas. | `load_batch("weekly_auto_claims")` returns 8 claims |
| 1.6 | Build Prompt Manager | Implement the simple dict-based PromptManager class (Section 11) | `manager.get_prompt(...)` works |

**Key learning:** ADK basics, project structure, how Gemini API works, fraud claim data modeling (batch structure, claim fields, enrichment targets).

**Suggested project structure after Day 1:**
```
commercial-auto-fraud/
├── agents/
│   ├── __init__.py
│   ├── fraud_pipeline_orchestrator.py    # Skeleton only
│   ├── claim_data_enrichment.py          # Skeleton only
│   ├── fraud_pattern_analyzer.py         # Skeleton only
│   └── investigation_report_generator.py # Skeleton only
├── prompt_manager/
│   ├── __init__.py
│   ├── manager.py                        # PromptManager class (working)
│   └── templates/
│       ├── __init__.py
│       ├── orchestrator.py               # Prompt strings
│       ├── enrichment.py
│       ├── fraud_patterns.py
│       └── report_generator.py
├── mock_apis/
│   ├── __init__.py
│   ├── claim_data.py                     # load_batch(), load_claim()
│   ├── claimant_history.py               # get_claimant_history(claimant_id)
│   ├── provider_network.py               # get_provider_info(provider_id)
│   ├── vehicle_valuation.py              # get_vehicle_valuation(vin)
│   └── policy_details.py                 # get_policy_details(policy_number)
├── user_profile/
│   ├── __init__.py
│   └── profile_store.py                  # JSON file-based store
├── a2a/
│   └── agent_cards/                      # Static agent card JSON files
├── data/
│   ├── profiles/                         # User profile JSON files
│   └── claims/
│       ├── weekly_auto_claims.json       # 8 claims (5 clean, 3 fraud)
│       └── flagged_commercial_claims.json # 6 claims (all flagged)
├── telemetry/
│   ├── __init__.py
│   └── arize_setup.py                    # Skeleton only
├── main.py
├── requirements.txt
├── .env.example
└── README.md
```

---

### Day 2: Claim Data Enrichment Agent (ReAct)
**Goal:** Agent that takes a raw claim and enriches it by calling 4 mock tool functions, returning a structured enriched data package.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 2.1 | Register enrichment prompts | Add ReAct prompt templates to Prompt Manager — system instruction tells the agent to gather claimant history, provider data, vehicle valuation, and policy details for each claim | Prompts retrievable |
| 2.2 | Implement 4 tool functions | Wire mock API functions: `get_claimant_history(claimant_id)` → prior claims, prior fraud flags, claim frequency; `get_provider_info(provider_id)` → billing patterns, license status, total claims served; `get_vehicle_valuation(vin)` → market value, condition, salvage value; `get_policy_details(policy_number)` → coverage type, inception date, recent changes, gaps | Each tool returns structured JSON from mock data |
| 2.3 | Build Claim Data Enrichment Agent | ADK agent (Gemini 2.5 Flash) with system instruction + 4 tools. Uses ReAct pattern: agent reasons about which data to gather, calls tools, assembles enriched package | Agent returns enriched claim with all 4 data sections populated |
| 2.4 | Test with sample claims | Feed 2-3 claims from the Weekly batch, verify enrichment quality and completeness | Enriched data matches expected mock values, all 4 sections present |

**Key learning:** ReAct prompting pattern (reason → act → observe → repeat), tool function design, structured output from LLM, ADK tool registration.

**Prompt type practiced:** ReAct (tool-calling with reasoning)

> **Tip:** Start with a single tool function (`get_claimant_history`). Get the ReAct loop working for that one data source, then add the other three. Don't try to wire all four tools simultaneously — debugging multi-tool ReAct is much harder than single-tool.

---

### Day 3: Fraud Pattern Analyzer Agent (CoT + Parallel)
**Goal:** Agent that takes an enriched claim and runs 4 fraud pattern analyses in parallel using Chain-of-Thought reasoning, producing sub-scores and a composite fraud score.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 3.1 | Write CoT prompts for 4 fraud patterns | Create Chain-of-Thought prompt for each pattern: **Duplicate/Similar Claims** (prior claim matching, VIN overlap, same providers), **Suspicious Timing** (loss-to-report gaps, policy inception proximity, weekend/holiday patterns), **Inflated Amounts** (claimed vs. market value, provider billing benchmarks, damage-to-cost ratios), **Provider Network Anomalies** (claim volume, billing deviation, referral patterns, license issues) | 4 prompt templates registered in Prompt Manager |
| 3.2 | Implement parallel analysis | Use `asyncio.gather()` to run all 4 fraud patterns simultaneously for a single enriched claim. Each pattern call sends the enriched data to the LLM with the appropriate CoT prompt and receives a sub-score (0.0–1.0) + evidence list + reasoning | All 4 patterns run in parallel, results combined into single fraud assessment |
| 3.3 | Build Fraud Pattern Analyzer Agent | ADK agent (Gemini 2.5 Pro) that receives enriched claim data, dispatches 4 parallel CoT analyses, collects sub-scores, computes composite fraud score (weighted average), classifies into tier (Critical ≥ 0.75 / High ≥ 0.50 / Medium ≥ 0.30 / Low < 0.30) | Agent returns composite score + 4 pattern results + tier classification |
| 3.4 | Test with sample claims | Feed enriched claims with known fraud indicators: a clean claim (should score Low), a suspicious claim (should score Medium/High), and a clearly fraudulent claim (should score Critical) | Scores align with expected tiers, CoT reasoning cites correct evidence |

**Key learning:** Chain-of-Thought prompting for multi-dimensional evidence analysis, parallel execution with `asyncio.gather`, composite scoring logic, fraud pattern domain knowledge.

**Prompt type practiced:** Chain-of-Thought (CoT) with evidence-based reasoning

> **Tip:** Start with the **Duplicate/Similar Claims** pattern — it has the most concrete signals (matching VINs, same provider, similar descriptions). Get the CoT prompt producing good reasoning with score and evidence, then template the same structure for the other three patterns. Don't write all 4 from scratch — 80% of the prompt structure is shared.

---

### Day 4: Orchestrator + Report Generator (Dual-Level Pipeline)
**Goal:** Wire the full dual-level parallel pipeline. Orchestrator ingests a batch, fans out N claims in parallel (Level 1), each claim runs enrichment → 4 parallel patterns (Level 2) → scoring → report. This is the most complex day.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 4.1 | Build Investigation Report Generator | ADK agent (Gemini 2.5 Flash) with few-shot prompt. Takes a claim's fraud analysis results (composite score, 4 pattern sub-scores, evidence lists) and generates a structured investigation report: executive summary, evidence table, pattern-by-pattern breakdown, recommended investigative actions, priority classification | Report generated with correct evidence and actionable recommendations |
| 4.2 | Write report generation prompts | Few-shot prompt with 2-3 example reports showing the expected output format: Markdown with headers, evidence tables, and action items. Include examples for different fraud tiers (Critical vs. Medium) to teach the model appropriate urgency and detail levels | Reports follow consistent formatting across different claim types |
| 4.3 | Build Fraud Pipeline Orchestrator (dual-level parallel) | Main agent (Gemini 2.5 Pro) that: loads batch JSON → validates all claims → fans out `asyncio.gather()` across N valid claims (Level 1). For each claim: Enrichment Agent → Fraud Pattern Analyzer (4 patterns in parallel, Level 2) → composite scoring → Investigation Report Generator (if score ≥ 0.40). Nested `asyncio.gather` pattern: outer gather for N claims, inner gather for 4 patterns per claim | All N claims processed in parallel, each with 4 parallel pattern analyses |
| 4.4 | Implement batch summary aggregation | Aggregation tool that combines per-claim results into batch summary: claims ranked by composite fraud score, tier distribution (Critical/High/Medium/Low/Clean counts), top triggered patterns across the batch, provider frequency analysis (which providers appear on multiple flagged claims), total flagged claim count and percentage | Summary matches schema (Section 15) |
| 4.5 | Test end-to-end with Weekly batch | Run the "Weekly Auto Claims" batch (8 claims — 5 clean + 3 fraud) through the full pipeline. Verify: all 8 claims processed, 3 flagged claims identified, scores in expected ranges, investigation reports generated for flagged claims, batch summary correct | 8 claims processed, 3 flagged correctly, reports generated, summary accurate |

**Key learning:** Dual-level parallel fan-out pattern (`asyncio.gather` nesting), pipeline orchestration, few-shot prompting for structured document generation, batch-level aggregation.

**Prompt types practiced:** Few-shot (report generation), Zero-shot (batch summary)

> 🚨 **This is the most critical day.** By end of Day 4, you should have a working end-to-end fraud detection pipeline that ingests a batch of claims and produces ranked fraud scores with investigation reports. Everything after this adds communication (A2A), human interaction (profiling + query chat), and observability (Arize). If Day 4 slips, prioritize getting the pipeline working over A2A or profiling — a working fraud detection pipeline is more impressive than a half-finished feature set.

---

### Day 5: A2A + User Profiling + Query Interface
**Goal:** Add A2A protocol artifacts, user profiling for SIU investigators, and the query chat interface that lets Kevin and Diana interact with pipeline results.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 5.1 | Create Agent Cards | Write agent card JSON for all 4 agents: Fraud Pipeline Orchestrator, Claim Data Enrichment, Fraud Pattern Analyzer, Investigation Report Generator (see Section 8 for full schemas) | 4 JSON files in `a2a/agent_cards/` with valid schemas |
| 5.2 | Implement A2A demo endpoint | Add A2A server to the Claim Data Enrichment Agent — serves agent card at `/.well-known/agent.json` and accepts enrichment tasks via A2A protocol | `curl localhost:8002/.well-known/agent.json` returns valid agent card |
| 5.3 | A2A client call from Orchestrator | Orchestrator makes at least one call via A2A client (to Claim Data Enrichment Agent) during batch processing to demonstrate the A2A protocol in action | A2A task delegation works for claim enrichment |
| 5.4 | Build query chat mode in Orchestrator | After batch pipeline completes, Orchestrator transitions to conversational mode — accepts natural language queries about batch results: pipeline status, batch summary, claim drill-down, pattern queries, provider queries, explanations (see Section 4.1, F6 for full query type table) | SIU investigator can ask "Tell me about CLM-2024-0847" and get detailed results |
| 5.5 | Build user profiling | Add profiling questions to Orchestrator's first conversational turn — detect SIU experience level and query preferences | Agent asks experience-related questions |
| 5.6 | Write profiling prompt (few-shot) | Few-shot prompt to classify user as junior analyst (Kevin-like) / senior SIU lead (Diana-like) from their responses. Include 3-4 examples: "2 years in SIU, still learning" → junior; "14 years, CFE, just show me the criticals" → senior | Classification works for varied input phrasing |
| 5.7 | Implement adaptive behavior | Orchestrator adjusts query response style based on profile: **Junior (Kevin)** — detailed explanations, term definitions ("provider network anomaly" explained), step-by-step evidence walkthroughs, guidance on next investigative steps. **Senior (Diana)** — concise tables, statistical summaries, no term definitions, exceptions-only when requested, data-driven evidence chains | Two demo scenarios show visibly different output styles for the same claim data |

**Key learning:** A2A protocol (agent cards, task delegation), dual-mode agent design (batch pipeline → conversational query), user profiling from natural language, adaptive prompt engineering.

**Prompt type practiced:** Few-shot (user classification)

---

### Day 6: Arize Integration + Deployment
**Goal:** Add observability for the fraud pipeline and deploy the system.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 6.1 | Set up Arize account | Sign up at arize.com, create Space, get API key | Dashboard accessible |
| 6.2 | Initialize tracing | Add `arize-otel` setup code (Section 13), instrument main agent calls with OpenTelemetry | Traces appear in Arize dashboard |
| 6.3 | Add span attributes | Tag traces with fraud-specific attributes: `claim_id`, `batch_id`, `fraud_score`, `fraud_tier` (Critical/High/Medium/Low), `pattern_flags` (comma-separated list of triggered patterns), `agent_name`, `prompt_version`. Ensure nested parallel spans are visible: batch span → N claim spans → 4 pattern spans per claim | Traces are filterable by claim ID, fraud tier, and pattern type in Arize |
| 6.4 | Set up annotation queue | Create an annotation queue for fraud score accuracy review — annotators (experienced SIU analysts) verify whether AI-assigned fraud tiers match their expert judgment. Include fields: correct_tier, notes, false_positive flag | Fraud scoring results can be reviewed/annotated by domain experts |
| 6.5 | Deploy or finalize local | **Option A:** Deploy to Vertex AI Agent Engine if GCP access available. **Option B:** Document `adk web` local deployment as the production setup. | Agent accessible for demo |
| 6.6 | Test deployed agent | Run the "Weekly Auto Claims" batch (8 claims) through the deployed system, verify traces flow to Arize with dual-level parallel spans visible (batch → claims → patterns) | Traces visible in Arize with batch, claim-level, and pattern-level spans |

**Key learning:** OpenTelemetry tracing for dual-level parallel workloads, LLM observability, annotation workflows for fraud scoring quality, cloud deployment.

---

### Day 7: Integration Testing + Demo Preparation
**Goal:** Polish, test all scenarios with both personas across both claim batches, prepare for the demo.

| # | Task | Details | Done When |
|---|------|---------|-----------|
| 7.1 | Test Scenario 1: Junior Kevin | Run the guided "Weekly Auto Claims" batch review scenario (Section 18.1) end-to-end — Kevin gets profiled as junior, batch of 8 claims runs, full summary with explanations, drill-down into flagged claims with term definitions | Guided, detailed flow works with junior-appropriate responses |
| 7.2 | Test Scenario 2: Senior Diana | Run the exceptions-only "Flagged Commercial Claims" batch review scenario (Section 18.2) end-to-end — Diana recognized as senior, 6-claim batch runs, only CRITICAL claims shown, concise evidence tables | Concise, exceptions-only flow works with senior-appropriate responses |
| 7.3 | Test both batches end-to-end | Run "Weekly Auto Claims" (8 claims — expect 3 flagged) and "Flagged Commercial Claims" (6 claims — expect 2 Critical, 2 High, 2 Medium) — verify fraud scores, tier assignments, investigation reports, and batch summaries for both | No crashes, scores and tiers align with expected results for all 14 claims |
| 7.4 | Verify Arize traces | Check Arize dashboard has traces for all test runs, annotation queue works, dual-level parallel spans visible (batch → N claims → 4 patterns per claim) | Traces + annotations visible with correct nesting |
| 7.5 | Write demo script | Document the exact steps/inputs for the 2 demo scenarios — which batch to load, what queries to ask, what to highlight | Demo script ready |
| 7.6 | Rehearse demo (dry run) | Run through both scenarios as if presenting — time it, fix any rough edges, verify the adaptive profiling visibly changes output | Smooth, no errors, fits in allotted time |
| 7.7 | Update README | Document how to set up and run the project — env vars, claim data, running batch pipeline, query chat, Arize setup | Someone else can run it |

---

### Day 8 (Stretch): Polish & Enhancements

If time allows, these additions make the demo even more impressive:

| # | Enhancement | Effort | Impact |
|---|------------|--------|--------|
| S1 | Add 2 more fraud patterns (Staged Accident Indicators + Policy Manipulation) | 2-3 hours | Shows 6 parallel fraud patterns instead of 4 — deeper fraud coverage |
| S2 | Full A2A for all sub-agents | 2-3 hours | All 3 sub-agents (Enrichment, Analyzer, Report Generator) use A2A protocol end-to-end |
| S3 | PDF export for investigation reports | 1-2 hours | Production-ready investigation packages for law enforcement referral |
| S4 | Build Arize monitoring dashboard | 1-2 hours | Fraud score distribution, false positive rate, pattern trigger frequency widgets |
| S5 | Prompt v2 and A/B comparison in Arize | 1-2 hours | Demonstrates prompt iteration for fraud scoring accuracy improvement |
| S6 | Add more mock claims to both batches | 1-2 hours | 15+ claims per batch for stress testing dual-level parallelism |

---

### Daily Checkpoint Validation

Use this checklist to verify each day's work is solid before moving on:

| Day | Checkpoint Question | Must Answer YES |
|-----|-------------------|----------------|
| 1 | Can I chat with a basic ADK agent in the browser? Are 2 claim batch JSON files loadable with correct schemas? | ✅ |
| 2 | Does the Claim Data Enrichment Agent call 4 tools via ReAct and return a fully enriched claim? | ✅ |
| 3 | Does the Fraud Pattern Analyzer run 4 CoT analyses in parallel and return a composite fraud score with evidence? | ✅ |
| 4 | Can I run the full "Weekly Auto Claims" batch (8 claims) and get a ranked batch summary with investigation reports for flagged claims? | ✅ |
| 5 | Are Agent Cards created? Does the SIU query chat work? Does user profiling change the response style for Kevin vs. Diana? | ✅ |
| 6 | Do traces appear in Arize with dual-level parallel spans? Is the annotation queue for fraud scoring set up? | ✅ |
| 7 | Can I demo both scenarios smoothly? Does the "Weekly Auto Claims" batch flag 3 claims and the "Flagged Commercial" batch rank all 6 by severity? | ✅ |

---

## 18. Sample Interactions

### 18.1 Scenario: Junior SIU Analyst Kevin — Guided Weekly Batch Review

Kevin is a 2-year SIU analyst who wants to process the weekly commercial auto claims batch. He needs detailed explanations and guidance on fraud indicators.

**Setting:** Kevin opens ADK Web, connects to the Fraud Pipeline Orchestrator.

---

**ORCHESTRATOR:** Welcome to the Commercial Auto Fraud Detection System. Before we begin, I'd like to understand your background to tailor my responses. Could you tell me about your role and experience in insurance fraud investigation?

**KEVIN:** I'm Kevin, SIU analyst. Been doing this about 2 years. Still getting the hang of commercial fleet claims — most of my experience is personal auto.

**ORCHESTRATOR:** Thanks, Kevin. I'll provide detailed explanations along the way, including context on commercial fleet-specific patterns. What would you like to do?

**KEVIN:** Run fraud analysis on this week's auto claims batch.

**ORCHESTRATOR:** Loading batch: **"Weekly Auto Claims — Week of 2026-04-14"**
— 8 claims detected. Validating...
— 8/8 claims valid. 0 rejected.

Starting dual-level parallel fraud analysis:
- **Level 1:** Processing all 8 claims simultaneously
- **Level 2:** Each claim analyzed across 4 fraud patterns in parallel

⏳ Enriching claims with claimant history, provider data, vehicle valuations, and policy details...

⏳ Running fraud pattern analysis (Duplicate/Similar, Suspicious Timing, Inflated Amounts, Provider Network) across all 8 claims...

⏳ Generating investigation reports for flagged claims...

✅ **Batch complete.** 8 claims processed in 47 seconds.

**Batch Summary:**

| # | Claim ID | Claimant | Fraud Score | Tier | Top Flag |
|---|----------|----------|-------------|------|----------|
| 1 | CLM-2024-0847 | Marcus Rivera | 0.87 | 🔴 **CRITICAL** | Duplicate VIN + Same Shop (PRV-0093) |
| 2 | CLM-2024-0849 | David Chen | 0.64 | 🟠 **HIGH** | Inflated Amount + Provider Network |
| 3 | CLM-2024-0851 | Lisa Thompson | 0.42 | 🟡 **MEDIUM** | Suspicious Timing (Fri→Mon report gap) |
| 4 | CLM-2024-0848 | Sarah Kim | 0.12 | ⚪ LOW | — |
| 5 | CLM-2024-0850 | Ana Morales | 0.08 | ⚪ LOW | — |
| 6 | CLM-2024-0852 | James Wright | 0.06 | ⚪ LOW | — |
| 7 | CLM-2024-0853 | Robert Kim | 0.09 | ⚪ LOW | — |
| 8 | CLM-2024-0854 | Michelle Davis | 0.07 | ⚪ LOW | — |

**3 claims flagged** (score ≥ 0.30): 1 Critical, 1 High, 1 Medium
**5 claims clean** (score < 0.30)

Investigation reports have been generated for all 3 flagged claims. Would you like to review any of them?

---

**KEVIN:** Tell me about CLM-2024-0847. Why is it critical?

**ORCHESTRATOR:** Here's the full breakdown for **CLM-2024-0847 — Marcus Rivera** (Composite Score: **0.87 CRITICAL**):

**Claim Details:**
- **Loss:** Fleet truck rear-end collision, $42,000 repair claim
- **Vehicle:** 2021 Ford F-350, VIN 1HGCM82633A004352
- **Repair Shop:** AllStar Truck & Auto Repair (PRV-0093)

**Pattern-by-Pattern Analysis:**

| Pattern | Sub-Score | Key Evidence |
|---------|----------|--------------|
| Duplicate/Similar Claims | 0.91 | VIN 1HGCM82633A004352 appears on a prior claim (CLM-2024-0601) filed 3 months ago — same vehicle, same type of damage (rear-end), same repair shop PRV-0093. The claimant also has 2.7 claims/year, well above the commercial fleet average of 0.8 claims/year. |
| Provider Network | 0.88 | Repair shop PRV-0093 (AllStar Truck & Auto) has served 47 claims with an average billing of $36,800 — that's **2.3x the regional median** for commercial truck repairs. This shop also appears on CLM-2024-0849 (David Chen) in the same batch. Two claims in one batch at the same high-billing shop is a strong ring indicator. |
| Inflated Amounts | 0.79 | $42,000 repair claim on a vehicle with market value of $38,500. The repair cost exceeds the vehicle's value by 9%, which typically triggers a total-loss designation. The damage described (rear-end collision) typically costs $18,000–$25,000 for this vehicle class. |
| Suspicious Timing | 0.34 | Moderate: claim filed 4 days after loss date (within normal range), but the prior claim on the same VIN was only 90 days ago. No weekend/holiday anomaly detected. |

**What does this mean?** This claim has strong indicators across three of four fraud patterns. The combination of a duplicate VIN at the same repair shop, with that shop billing 2.3x the regional average, and a repair cost exceeding the vehicle's market value, creates a multi-dimensional fraud signal. In commercial fleet fraud, this pattern often indicates a **provider-claimant collusion ring** — the repair shop inflates estimates while the claimant files repeated claims on the same vehicles.

**Recommended Investigative Actions:**
1. Pull the full claim history for VIN 1HGCM82633A004352 — look for additional claims beyond the 2 found
2. Request an independent vehicle inspection to verify damage severity
3. Subpoena repair invoices from AllStar Truck & Auto (PRV-0093) for the past 12 months
4. Cross-reference claimant Marcus Rivera's other policies (POL-COMM-9184, POL-COMM-7622)

---

**KEVIN:** What does "provider network anomaly" mean exactly? I've seen it flagged before but I'm not sure what makes a provider suspicious.

**ORCHESTRATOR:** Great question. A **provider network anomaly** is a fraud indicator based on the behavior of the service provider (repair shop, medical clinic, towing company, etc.) rather than the claimant.

Here's what the system looks at:

1. **Billing volume and deviation** — How many claims has this provider handled, and how does their average billing compare to other providers in the same region? A shop that handles 47 claims/year and bills 2.3x the regional average is statistically unusual. Most legitimate repair shops cluster within ±30% of the regional median.

2. **Claim concentration** — Does this provider appear on multiple claims in the same batch or time period? If PRV-0093 shows up on 2 of 8 claims in a single weekly batch, that's a **12.5x overrepresentation** vs. random chance (there are hundreds of repair shops in a metropolitan area).

3. **Referral patterns** — Does the provider have an unusually tight relationship with specific tow companies, attorneys, or claimants? Fraud rings often involve a chain: tow company steers damaged vehicles to a specific shop, which inflates estimates and kicks back a percentage.

4. **License and compliance issues** — Is the provider's license active? Have they been cited for fraud, overbilling, or operating without proper certification?

In the case of PRV-0093 (AllStar Truck & Auto), the flags are: high billing average ($36,800 vs. ~$16,000 regional median), appearing on 2 claims in one batch, and 47 total claims served (high volume). These are the kinds of signals that experienced SIU investigators like Diana look for when identifying organized fraud operations.

**Key takeaway for your investigations:** When you see a provider network flag, always check if the same provider appears in other recent claims — that's where you find rings.

---

**KEVIN:** Got it, that makes sense. Can you generate the investigation report for CLM-2024-0847?

**ORCHESTRATOR:** Investigation report for **CLM-2024-0847** has already been generated during the pipeline run. Here it is:

---

### INVESTIGATION REPORT — CLM-2024-0847

**Priority:** 🔴 CRITICAL | **Composite Fraud Score:** 0.87 | **Generated:** 2026-04-18

**Executive Summary:**
Commercial fleet claim by Marcus Rivera for $42,000 rear-end repair on a 2021 Ford F-350 at AllStar Truck & Auto Repair (PRV-0093) exhibits multi-dimensional fraud indicators. The same VIN appeared in a prior claim 90 days ago at the same shop. The repair shop's average billing is 2.3x the regional median and appears on a second claim in this batch (CLM-2024-0849). Repair cost exceeds vehicle market value. Recommend immediate deep investigation with focus on provider-claimant collusion.

**Evidence Summary:**

| Evidence Item | Detail | Source |
|--------------|--------|--------|
| Duplicate VIN | Same VIN on CLM-2024-0601 (3 months prior) | Claimant History |
| Same Repair Shop | PRV-0093 on both current and prior claim | Provider Network |
| Inflated Billing | $42,000 vs. $38,500 vehicle value (109%) | Vehicle Valuation |
| High Provider Volume | 47 claims, $36,800 avg (2.3x median) | Provider Network |
| Same-Batch Link | PRV-0093 also on CLM-2024-0849 | Batch Analysis |
| High Claim Frequency | 2.7 claims/year vs. 0.8 fleet average | Claimant History |

**Recommended Actions:**
1. Deep investigation — full VIN history pull and independent vehicle inspection
2. Provider audit — subpoena AllStar Truck & Auto invoices for past 12 months
3. Cross-claim analysis — link CLM-2024-0847 and CLM-2024-0849 as potentially related
4. Claimant interview — contact Marcus Rivera regarding repeated claims on same vehicle

---

The report is ready for your case file. Would you like to review the other flagged claims (CLM-2024-0849 or CLM-2024-0851)?

---

### 18.2 Scenario: Senior SIU Lead Diana — Exception-Only Flagged Batch

Diana is a 14-year SIU veteran and team lead. She wants to quickly triage the pre-flagged commercial claims batch, focusing only on critical cases.

**Setting:** Diana opens ADK Web, connects to the Fraud Pipeline Orchestrator.

---

**ORCHESTRATOR:** Welcome to the Commercial Auto Fraud Detection System. Could you tell me about your role and experience so I can tailor my responses?

**DIANA:** Diana Okafor, SIU lead, 14 years. CFE certified. Show me CRITICAL only — I don't need hand-holding.

**ORCHESTRATOR:** Understood, Diana. CRITICAL-only mode, concise output. What would you like to run?

**DIANA:** Run the flagged commercial batch, CRITICAL only.

**ORCHESTRATOR:** Processing **"Flagged Commercial Claims"** — 6 claims. All pre-flagged.

✅ **Complete.** 6 claims analyzed. Showing **CRITICAL only** (2 claims):

| Claim ID | Claimant | Score | Primary Pattern | Recommended Action |
|----------|----------|-------|-----------------|--------------------|
| CLM-2024-0901 | Carlos Mendez | 0.93 | Duplicate + Provider + Inflated | Deep investigation — possible ring |
| CLM-2024-0902 | Brian Foster | 0.89 | Inflated + Timing + Provider | Deep investigation — new policy fraud |

4 additional claims scored below CRITICAL: 2 High, 2 Medium. Say "show all" to see full results.

---

**DIANA:** CLM-2024-0901 — pull the evidence chain.

**ORCHESTRATOR:**

**CLM-2024-0901 — Carlos Mendez** | Score: 0.93 CRITICAL

| Pattern | Score | Evidence |
|---------|-------|----------|
| Duplicate/Similar | 0.95 | 3 rear-end claims in 12 months, all same VIN, escalating amounts ($18K → $27K → $41K) |
| Provider Network | 0.92 | All 3 claims at same shop — shop has 52 claims, avg billing 2.8x regional median |
| Inflated Amounts | 0.88 | Latest claim $41K on $32K vehicle (128%), damage description inconsistent with estimate |
| Suspicious Timing | 0.38 | Claims spaced ~90 days apart (regular cadence, but within normal filing windows) |

**Cross-claim links:** Same provider as CLM-2024-0904 (Derek Johnson) — shared referral tow company (Metro Towing LLC).

---

**DIANA:** Flag for law enforcement referral. Add notes: possible fraud ring involving provider and tow company. Link to CLM-2024-0904.

**ORCHESTRATOR:** Updated CLM-2024-0901:
- ✅ Status → **Law Enforcement Referral**
- ✅ Notes added: "Possible fraud ring involving repair shop and Metro Towing LLC. Link to CLM-2024-0904 (shared provider + tow referral pattern). —D. Okafor, 2026-04-18"
- ✅ Cross-linked with CLM-2024-0904

Investigation report updated with referral status. Anything else on this batch?

---

## 19. Acceptance Criteria

### 19.1 MVP — Must Demo (Days 1-7)

These are the **non-negotiable** deliverables for the demo:

| # | Criterion | Verification | Tech Topic Covered |
|---|-----------|-------------|-------------------|
| AC-01 | Batch intake processes 5+ claims in parallel (both batches: 8-claim Weekly and 6-claim Flagged) | Demo with "Weekly Auto Claims" (8 claims) — all processed simultaneously via `asyncio.gather` | ADK |
| AC-02 | Claim Data Enrichment Agent returns structured enriched claim data with 4 data sections (claimant history, provider network, vehicle valuation, policy details) | Enriched claim JSON has all 4 sections populated with data from mock API tool calls | ADK + ReAct prompting |
| AC-03 | Fraud Pattern Analyzer runs 4 fraud patterns in parallel per claim using `asyncio.gather` | Visible parallel execution — 4 sub-scores returned simultaneously per claim | ADK + CoT prompting + Parallel |
| AC-04 | Each fraud pattern includes Chain-of-Thought evidence-based reasoning (not just a score) | Each sub-score accompanied by step-by-step reasoning citing specific evidence from enriched data | Prompt Engineering (CoT) |
| AC-05 | Investigation Report Generator produces structured reports with evidence summary, pattern breakdown, and recommended actions | Reports generated for all flagged claims (score ≥ 0.40), following few-shot example format | ADK + Few-shot prompting |
| AC-06 | User profiling detects junior SIU analyst (Kevin) vs. senior SIU lead (Diana) and adapts response style accordingly | Two demo scenarios show visibly different output: Kevin gets detailed explanations + term definitions; Diana gets concise tables + exceptions-only | User Profiling |
| AC-07 | Agent Cards (JSON) created for all 4 agents with valid schemas | Files exist in `a2a/agent_cards/` — Orchestrator, Enrichment, Analyzer, Report Generator | A2A |
| AC-08 | At least 1 A2A call works (Orchestrator → Claim Data Enrichment Agent) | A2A task delegation demonstrated: Orchestrator sends enrichment task via A2A client, Enrichment Agent processes via A2A server | A2A |
| AC-09 | Prompt Manager serves versioned templates for all 4 agents (orchestrator, enrichment, fraud patterns, report generation) | Templates loaded via `manager.get_prompt(...)`, version tracked, used by all agents | Prompt Manager |
| AC-10 | Arize traces visible in dashboard with dual-level parallel spans (batch → N claims → 4 patterns per claim) | All LLM calls traced, filterable by `claim_id`, `fraud_score`, `fraud_tier`, `pattern_flags` | Arize |
| AC-11 | Arize annotation queue for fraud score accuracy review set up | Fraud tier assignments can be reviewed/annotated by SIU domain experts | Arize |
| AC-12 | Running locally via `adk web` **OR** deployed to Vertex AI Agent Engine | Accessible demo with batch pipeline + SIU query chat both working | Deployment |

**Count: 6/6 tech topics covered in MVP** ✅

### 19.2 Stretch — Nice to Show (Day 8)

| # | Criterion | Tech Topic |
|---|-----------|-----------|
| AC-13 | 6 parallel fraud patterns per claim (add Staged Accident Indicators + Policy Manipulation) | ADK + Parallel |
| AC-14 | All 3 sub-agents (Enrichment, Analyzer, Report Generator) callable via full A2A protocol | A2A |
| AC-15 | PDF export for investigation reports (suitable for law enforcement referral packages) | ADK |
| AC-16 | Arize dashboard with fraud score distribution, false positive rate, and pattern trigger frequency widgets | Arize |
| AC-17 | Prompt v1 vs v2 comparison visible in Arize (demonstrates prompt iteration for fraud scoring accuracy) | Prompt Manager + Arize |
| AC-18 | 15+ claims per batch for stress testing dual-level parallelism | Mock Data |

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

### 20.6 Fraud Detection Domain
- NICB (National Insurance Crime Bureau) — commercial auto fraud patterns and red flag indicators: https://www.nicb.org/
- Coalition Against Insurance Fraud — industry statistics and fraud type taxonomy: https://insurancefraud.org/
- SIU best practices for commercial fleet investigations — staged accident identification, provider ring detection, billing inflation benchmarks
- ISO ClaimSearch — industry standard cross-company claims database used by SIU teams for claimant and provider history lookups
- Composite fraud scoring methodologies — weighted multi-dimensional scoring with evidence-backed reasoning for defensible SIU referrals

---

*Document Version: 1.0*
*Created: April 18, 2026*
*Author: AI Planning Assistant*
*Status: Ready for Review*
*Scope: 7-day MVP + 1-day Stretch*
