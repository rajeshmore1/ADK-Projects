# ADK-Projects

# Intern Projects Overview — Insurance AI Agent Platform

## Program Summary

| Attribute | Detail |
|-----------|--------|
| **Domain** | Insurance (Auto, Property, Commercial) |
| **Enterprise Tier** | Middle & Large Enterprise |
| **Interns** | 6 (CS background, medium Python expertise) |
| **Model** | 1 project per intern |
| **LLM** | Google Gemini 2.5 Flash / Pro |
| **Deployment** | GCP (own environment) or Local |
| **Monitoring** | Arize (Cloud) |
| **Deliverable** | Working practical demo |

---

## Tech Stack Integration (Every Project)

Each project must integrate **all** of the following:

| # | Tech Area | What Intern Builds |
|---|-----------|-------------------|
| 1 | **Prompt Engineering & Prompt Manager** | Implement multiple prompt types (zero-shot, few-shot, CoT, ReAct). Build a prompt manager for versioning/serving templates. |
| 2 | **Agent Development (ADK)** | Build multi-agent system using Google Agent Development Kit with Gemini. |
| 3 | **A2A Protocol** | Wire 2-3 agents within the project to communicate via Agent-to-Agent protocol (Agent Cards, Tasks, Messages). |
| 4 | **User Profiling & Conversational Chat** | Profile the user through chat interactions and adapt agent behavior accordingly. |
| 5 | **Arize Debugging & Annotation** | Instrument agents with Arize tracing, set up annotation queues, build monitoring dashboard. |
| 6 | **Agent Engine Deployment** | Deploy agents to Vertex AI Agent Engine (or local ADK server as fallback). |

---

## 6 Projects — Diversity Matrix

| # | Project | Insurance Line | Enterprise | Conversational? | Workflow Type | Primary Focus |
|---|---------|---------------|------------|-----------------|---------------|---------------|
| 1 | Commercial Property Underwriting Risk Assessment | Commercial + Property | Large | ✅ Conversational | Sequential + Parallel | Underwriting |
| 2 | Auto Fleet Claims FNOL & Triage | Auto | Large | ✅ Conversational | Sequential | Claims |
| 3 | Property Portfolio Renewal & Risk Re-evaluation | Property | Mid-market | ⚡ Batch + Review Chat | Parallel | Renewal |
| 4 | Commercial Auto Fraud Detection & Investigation | Auto + Commercial | Large | ❌ Non-conversational | Parallel | Fraud |
| 5 | Commercial Liability Policy Comparison & Recommendation | Commercial | Mid-Large | ✅ Conversational | Sequential + Parallel | Sales/Broking |
| 6 | Multi-Line Submission Intake & Document Processing | Auto + Property + Commercial | Large | ⚡ Pipeline + Status Chat | Parallel + Sequential | Operations |

---

## Project Summaries

### Project 1 — Commercial Property Underwriting Risk Assessment Agent
**Intern builds:** A multi-agent system that assists underwriters in evaluating commercial property risks for large enterprises (warehouses, office complexes, manufacturing plants). The orchestrator agent converses with the underwriter, delegates to a data collection agent, a risk scoring agent (runs parallel risk factor assessments), and a report generator agent.

### Project 2 — Auto Fleet Claims FNOL & Triage Agent
**Intern builds:** A conversational agent system for First Notice of Loss (FNOL) intake and claims triage for large commercial auto fleets (logistics, transport companies). The system guides adjusters through incident reporting, auto-classifies severity, estimates reserves, and routes to the right handler — all in a sequential workflow.

### Project 3 — Property Portfolio Renewal & Risk Re-evaluation Agent
**Intern builds:** A batch-processing agent system that reviews an entire property portfolio at renewal time. It simultaneously re-evaluates risk for multiple properties (parallel), flags changes (new hazards, code violations), and generates renewal recommendations. Includes a conversational review interface for the underwriter to query results.

### Project 4 — Commercial Auto Fraud Detection & Investigation Agent
**Intern builds:** A non-conversational pipeline that ingests commercial auto claims in bulk, runs parallel fraud pattern analysis (duplicate claims, suspicious timing, inflated amounts, provider networks), generates fraud scores, and produces investigation reports with evidence summaries.

### Project 5 — Commercial Liability Policy Comparison & Recommendation Agent
**Intern builds:** A conversational agent that helps insurance brokers compare commercial liability policies for mid-to-large businesses. It gathers client requirements through chat, searches policy databases in parallel, scores matches, and presents side-by-side comparisons with recommendations.

### Project 6 — Multi-Line Submission Intake & Document Processing Agent
**Intern builds:** A document processing pipeline agent that handles incoming insurance submissions across multiple lines (auto, property, commercial). It parses documents in parallel, extracts structured data, validates completeness, and routes to the appropriate underwriting queue. Includes a chatbot for submission status queries.

---

## Workflow Coverage

```
Sequential Workflows:  Project 1, 2, 5, 6
Parallel Workflows:    Project 1, 3, 4, 5, 6
Conversational:        Project 1, 2, 5
Non-Conversational:    Project 3, 4, 6
Mixed (Both):          Project 3, 6
```

---

## Documentation Structure

Each project will have its own folder under `docs/` with:
```
docs/
├── 00-intern-projects-overview.md          ← This file
├── project-1-commercial-property-underwriting/
│   └── PROJECT.md                           ← Full BA + Technical doc
├── project-2-auto-fleet-claims-fnol/
│   └── PROJECT.md
├── project-3-property-portfolio-renewal/
│   └── PROJECT.md
├── project-4-commercial-auto-fraud/
│   └── PROJECT.md
├── project-5-commercial-liability-comparison/
│   └── PROJECT.md
└── project-6-multiline-submission-intake/
    └── PROJECT.md
```
