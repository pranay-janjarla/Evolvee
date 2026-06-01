# EvolveForm v0.1 — System Design Document

> A self-evolving form framework powered by a 3-agent Claude pipeline.  
> This document covers architecture rationale, data flows, agent mechanics, and design decisions in depth.

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Architecture Overview](#2-architecture-overview)
3. [Layer 1 — Presentation](#3-layer-1--presentation)
4. [Layer 2 — Scheduling](#4-layer-2--scheduling)
5. [Layer 3 — Orchestration](#5-layer-3--orchestration)
6. [Layer 4 — Intelligence (3-Agent Pipeline)](#6-layer-4--intelligence-3-agent-pipeline)
7. [Layer 5 — Persistence](#7-layer-5--persistence)
8. [Workspace Context System](#8-workspace-context-system)
9. [Question-Level Telemetry](#9-question-level-telemetry)
10. [Confidence Scoring Algorithm](#10-confidence-scoring-algorithm)
11. [Full Data Flow — One Analysis Cycle](#11-full-data-flow--one-analysis-cycle)
12. [Database Schema & Rationale](#12-database-schema--rationale)
13. [Key Design Decisions](#13-key-design-decisions)
14. [Failure Modes & Mitigations](#14-failure-modes--mitigations)
15. [Evolution from v0 → v0.1](#15-evolution-from-v0--v01)

---

## 1. Problem Statement

Most forms are static. A product team creates questions, deploys the form, and then periodically reviews results manually to decide if the form needs updating. This creates several structural problems:

- **Survey fatigue from irrelevant questions**: When 95% of respondents pick the same answer, the question provides zero discriminating information. But it stays there because nobody noticed.
- **Blind spots from skipped questions**: A question may be displayed 200 times but answered only 80 times (60% skip rate). The team only sees the 80 answers — not the 120 silent signals of confusion or discomfort.
- **Delayed signal detection**: A safety signal (e.g., a spike in adverse reactions) takes days to surface because someone has to manually download and review the data.
- **Domain lock-in**: Form analysis tools are often built for specific domains (product feedback, NPS, HR surveys) and cannot transfer their detection logic to a new context.

**EvolveForm's answer:** A background AI pipeline that continuously monitors form responses, detects patterns the team hasn't noticed, proposes surgical changes to the form itself, and builds an institutional memory of everything it has learned — without any manual intervention from the form owner unless they want it.

---

## 2. Architecture Overview

EvolveForm is organized into five layers. Each layer has a single, clearly bounded responsibility. Data flows downward during a submission cycle and upward during an analysis cycle.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 1 — PRESENTATION                                                          │
│                                                                                  │
│  Login → Onboarding → Admin Panel (Builder, Dashboard, Pending, Analytics)       │
│                      ← Public Form (/form/<token>)                               │
│                                                                                  │
│  Technology: Flask + Jinja2 + Alpine.js + SortableJS                             │
└──────────────────────────────────┬──────────────────────────────────────────────┘
                                   │  form loads / submissions arrive
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 2 — SCHEDULING                                                            │
│                                                                                  │
│  APScheduler (configurable) | Event Trigger (threshold) | Urgency Recheck (30m) │
│                                                                                  │
│  Technology: APScheduler (in-process)                                            │
└──────────────────────────────────┬──────────────────────────────────────────────┘
                                   │  pipeline fire signal
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 3 — ORCHESTRATION                                                         │
│                                                                                  │
│  Fetches data → calls pipeline → scores confidence → routes changes              │
│  Auto-applies (≥ 0.85) | Queues (0.5–0.85) | Discards (< 0.5)                  │
│                                                                                  │
│  Technology: Python (orchestrator.py)                                            │
└──────────────────────────────────┬──────────────────────────────────────────────┘
                                   │  structured inputs/outputs (JSON via tool_use)
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 4 — INTELLIGENCE                                                          │
│                                                                                  │
│  Analyst Agent → Evolution Agent → Scribe Agent                                  │
│  (3 sequential Claude API calls, each with a narrow tool schema)                 │
│  All 3 share prompt caching on workspace_context.md                              │
│                                                                                  │
│  Technology: Anthropic Claude Sonnet 4.6 + Python SDK                           │
└──────────────────────────────────┬──────────────────────────────────────────────┘
                                   │  reads & writes
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  LAYER 5 — PERSISTENCE                                                           │
│                                                                                  │
│  workspaces · workspace_context · form_versions · submissions                    │
│  question_telemetry · pending_changes · knowledge_entries                        │
│  analysis_runs · loop_runs                                                       │
│                                                                                  │
│  Technology: SQLite (sqlite3, no ORM)                                            │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.1 System Architecture — Node View

The same five layers, expanded to show the key components in each and the data that flows between them. Submissions flow **downward**; findings, alerts, and context patches flow **upward**.

```
┌────────── PRESENTATION  —  Flask · Jinja2 · Alpine.js · SortableJS ──────────┐
│ Login → Onboarding                                                           │
│ Admin Panel:  Dashboard · Form Builder · Pending Changes · Analytics         │
│ Public Form:  /form/<token>   (respondents, no auth)                         │
└──────────────────────────────────────────────────────────────────────────────┘
                                        │  form load / submission  ▲ alerts surfaced to admin
                                        ▼
┌────────────────── SCHEDULING  —  APScheduler (in-process) ───────────────────┐
│ APScheduler (configurable interval)  ·  Event Trigger (count ≥ thresh)       │
│ Urgency Recheck  (fires 30 min after any CRITICAL finding)                   │
└──────────────────────────────────────────────────────────────────────────────┘
                                        │  pipeline fire signal
                                        ▼
┌───────── ORCHESTRATION  —  orchestrator.py (deterministic, no LLM) ──────────┐
│ Fetch data → run pipeline → score confidence → route change:                 │
│ ≥ 0.85 auto-apply   ·   0.5–0.85 queue   ·   < 0.5 discard                   │
│ marks submissions analyzed = 1   ·   logs analysis_run                       │
└──────────────────────────────────────────────────────────────────────────────┘
                                        │  structured I/O — JSON via tool_use
                                        ▼
┌─────────── INTELLIGENCE  —  3-Agent Pipeline (Claude Sonnet 4.6) ────────────┐
│ Analyst Agent ──findings──▶ Evolution Agent ──changes──▶ Scribe Agent        │
│ 'what patterns?'           'should the form change?'    'write memory'       │
│ forced tool_use  ·  shared cached system prompt (workspace_context) × 3      │
└──────────────────────────────────────────────────────────────────────────────┘
                                        │  reads submissions + telemetry   ▲ writes findings / context patch
                                        ▼
┌────────────────── PERSISTENCE  —  SQLite (sqlite3, no ORM) ──────────────────┐
│ workspaces · workspace_context · form_versions · submissions                 │
│ question_telemetry · pending_changes · knowledge_entries                     │
│ analysis_runs · loop_runs                                                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Data flows**
- **Submission → Event Trigger**: each POST checks the unanalyzed count; crossing the threshold fires the pipeline without waiting for the schedule.
- **Scheduled / Urgency trigger → Orchestrator**: APScheduler (or a 30-min CRITICAL recheck) kicks off a run when unanalyzed submissions exist.
- **Analyst → Evolution → Scribe**: structured JSON hand-off between agents — data, not prose.
- **Scribe → workspace_context patch**: derived baselines flow back into the cached system prompt, making the next run sharper.

The architecture is intentionally **thin at the top, rich at the bottom**. The Presentation and Scheduling layers do minimal work. The Orchestration layer does deterministic math (no LLM). The Intelligence layer is where all probabilistic reasoning happens. The Persistence layer is the source of truth that outlives any single run.

---

## 3. Layer 1 — Presentation

### 3.1 Public Form (`/form/<token>`)

The public form is the only surface that unauthenticated users (respondents) see. It is completely decoupled from the admin panel.

**On every GET `/form/<token>`:**
1. Flask reads the latest `form_versions` row for this workspace from SQLite (no in-memory cache — changes appear immediately)
2. Questions are rendered from `questions_json` via Jinja2
3. For every question in the current form version, `question_telemetry.times_shown` is incremented
4. The current `form_version` integer is embedded as a hidden field

**On every POST `/form/<token>/submit`:**
1. Submission saved to `submissions` table with `analyzed = 0`
2. For each question field present in the POST body, `times_answered` incremented
3. `skip_rate` recomputed: `1.0 - (times_answered / times_shown)`
4. `response_distribution_json` updated: `{option: count}` map
5. Unanalyzed count is checked — if it crosses `workspace.event_trigger_threshold`, an event trigger fires asynchronously

**Why no cache on form_version?** Form changes should be visible to the next respondent immediately after the admin approves them. Caching introduces a gap between the database state and what respondents see, which is dangerous when a change was driven by a safety signal.

### 3.2 Admin Panel

The admin panel has four sections, each a separate Flask route and Jinja2 template:

| Route | Purpose |
|---|---|
| `/admin` | Dashboard: submission count, last run timestamp, recent alerts, current form version |
| `/admin/builder` | Tally-like form builder — Alpine.js cards, SortableJS drag-to-reorder, live preview |
| `/admin/pending` | Pending Changes — confidence bars, approve/reject per proposal |
| `/admin/analytics` | Analysis run history, findings log, form version changelog |

**Form Builder interaction model:**
The builder does not edit questions in-place. Every save (POST `/admin/builder/save`) creates a new `form_versions` row with `change_source = 'manual'`. This means the version history is append-only — you can always roll back by reading an older row.

### 3.3 Onboarding

Onboarding happens once, immediately after first login. It collects:
- Company name, industry, country
- Applicable policies (multi-select: GDPR, HIPAA, CCPA, none)
- Form purpose (free text: "monthly employee engagement pulse check")
- Preferred loop interval

On submission, a **4th Claude call** generates the `workspace_context` document. This is the most expensive call in the entire system (it has no historical caching) but it only happens once per workspace. The generated document becomes the stable system prompt for every subsequent agent call, amortizing this cost across hundreds of pipeline runs.

---

## 4. Layer 2 — Scheduling

EvolveForm uses a **hybrid trigger model**: three overlapping mechanisms that coexist and complement each other.

### 4.1 APScheduler (time-based)

Runs as an in-process background thread via APScheduler. The interval is configurable per workspace:

```
5 minutes (development/testing)
60 minutes (hourly — high-traffic deployments)
1440 minutes (daily — default)
10080 minutes (weekly — low-volume forms)
```

The scheduler only calls the orchestrator if `SELECT COUNT(*) FROM submissions WHERE workspace_id=? AND analyzed=0` returns > 0. An empty queue is a no-op; it logs `outcome = 'skipped-no-data'` and returns.

**Adaptive backoff:** If the last 3 consecutive analysis runs all returned NORMAL severity (no findings above LOW), the scheduler doubles the current interval temporarily. This prevents wasted compute during stable periods. The interval resets to its configured value the moment any run returns a finding above LOW.

### 4.2 Event Trigger (count-based)

On every form submission, Flask checks the unanalyzed count. If it crosses `workspace.event_trigger_threshold` (default: 10), the orchestrator fires immediately in a background thread — without waiting for the next scheduled interval.

This means a sudden surge (e.g., 50 submissions in an hour) triggers analysis after the 10th, 20th, 30th, etc. — catching problems during the surge, not the next day.

**Why 10 as default?** Ten submissions is enough to produce statistically meaningful skip rates and response distributions for most questions. Five is too noisy; 50 is too slow. The threshold is tunable per workspace because a large-scale enterprise survey has different signal properties than a boutique product feedback form.

### 4.3 Urgency Recheck (severity-based)

After any analysis run that produces a `CRITICAL` severity finding, the orchestrator schedules a follow-up run exactly 30 minutes later — regardless of the current interval.

This is the most important trigger for safety use cases. In the original PROJECT_BRIEF, a single `post_feel = "burning"` response is a CRITICAL override. EvolveForm generalizes this: any CRITICAL finding triggers a recheck to verify whether the signal is persistent or a one-off. If the second run also finds CRITICAL, the confidence on the related proposed change receives a significant boost.

---

## 5. Layer 3 — Orchestration

The orchestrator (`orchestrator.py`) is the **only layer that applies business logic deterministically**. It does not call Claude. Its job is to:

1. Fetch all unanalyzed submissions + current telemetry for the workspace
2. Call the 3-agent pipeline in sequence (Analyst → Evolution → Scribe)
3. Receive proposed changes from Evolution Agent
4. Compute a final confidence score for each proposed change
5. Route each change: auto-apply | queue | discard
6. Mark processed submissions as `analyzed = 1`
7. Log the full run to `analysis_runs`
8. Schedule urgency recheck if CRITICAL found

**Why keep confidence scoring out of Claude?**

If Claude computed confidence, the scores would vary between runs on the same data (temperature > 0), making the auto-apply threshold unreliable. The Evolution Agent provides a `confidence_basis` string (a natural language rationale) and a raw `confidence_basis_score` (its raw estimate). The orchestrator then applies deterministic boosts on top of that estimate using hard data — batch size, historical proposal count, telemetry — producing a final score that is reproducible and auditable.

---

## 6. Layer 4 — Intelligence (3-Agent Pipeline)

This is the core innovation of EvolveForm over a naive single-agent approach.

### 6.1 Why 3 Agents Instead of 1

The original architecture used a single Claude call with a large, overloaded tool schema:
- It detected patterns
- It wrote alerts
- It mutated the form
- It updated the knowledge base

This monolithic design has four problems:

| Problem | Impact |
|---|---|
| Large context per call | Higher token cost and latency on every run |
| Mixed responsibilities in one prompt | Claude conflates detection confidence with change confidence |
| Cannot partially re-run | If form mutation fails, knowledge base update is lost too |
| One schema covers everything | Hard to tune prompts independently per task |

The 3-agent design solves each problem: smaller context, focused prompts, independent re-runability, and independently tunable tool schemas.

**The pipeline is sequential by design.** Evolution Agent reads Analyst Agent's output as structured input. This is not chain-of-thought — it is structured data hand-off. Evolution Agent cannot hallucinate about what the data said because the findings are presented as JSON, not prose.

### 6.2 Shared Prompt Caching

All 3 agents share the same `workspace_context` document as their system prompt:

```python
cached_system = [{
    "type": "text",
    "text": workspace_context_content,
    "cache_control": {"type": "ephemeral"}
}]
```

Anthropic's prompt caching stores the system prompt after the first call and reuses it for subsequent calls within the 5-minute TTL. Since all 3 agents fire within seconds of each other, all 3 benefit from the cache hit on the first fire of a session.

**Token economics:**
- `workspace_context` document: ~1,500–3,000 tokens
- Without caching: paid 3× per pipeline run
- With caching: paid once, then cache_read tokens for calls 2 and 3 (typically ~10× cheaper)
- At 1,440 pipeline runs per day (1-minute interval): the savings compound significantly

### 6.3 Analyst Agent

**Responsibility:** "What patterns exist in this data?"

**Input (from orchestrator):**
```json
{
  "submissions": [
    {
      "id": "uuid",
      "submitted_at": "ISO8601",
      "form_version": 3,
      "data": { "satisfaction": 2, "recommend": "no", "open_text": "..." }
    }
  ],
  "question_telemetry": [
    {
      "question_id": "uuid",
      "label": "How satisfied are you?",
      "times_shown": 150,
      "times_answered": 90,
      "skip_rate": 0.40,
      "response_distribution": {"1": 5, "2": 12, "3": 8, "4": 30, "5": 35}
    }
  ],
  "batch_size": 47,
  "workspace_summary": "Employee engagement survey — 150-person tech company — Netherlands"
}
```

**Tool schema (forced output):**
```json
{
  "name": "analyze_patterns",
  "input_schema": {
    "type": "object",
    "properties": {
      "findings": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "category":           { "type": "string" },
            "severity":           { "type": "string", "enum": ["low","medium","high","critical"] },
            "description":        { "type": "string" },
            "metric_name":        { "type": "string" },
            "metric_value":       { "type": "number" },
            "threshold":          { "type": "number" },
            "affected_question_id": { "type": ["string","null"] }
          },
          "required": ["category","severity","metric_name","metric_value","threshold"]
        }
      },
      "alerts":           { "type": "array", "items": { "type": "string" } },
      "analysis_summary": { "type": "string" }
    },
    "required": ["findings","alerts","analysis_summary"]
  }
}
```

**What the system prompt tells it:**
- The workspace context (company, domain, policies, established baselines)
- Instructions to compare metric_value against threshold from the context document
- Instructions on severity classification

**Output passed to:** Evolution Agent (as `findings`) + Orchestrator (for alert routing)

### 6.4 Evolution Agent

**Responsibility:** "Should the form change, and how?"

**Input (from orchestrator, built from Analyst output):**
```json
{
  "findings": [ /* Analyst's structured findings array */ ],
  "current_form": [
    {
      "id": "uuid",
      "type": "scale",
      "label": "How satisfied are you with your role?",
      "scale_min": 1,
      "scale_max": 5,
      "required": true,
      "source": "manual",
      "order": 0
    }
  ],
  "question_telemetry": { /* same as Analyst input */ },
  "policy_guardrails": [
    "GDPR: do not add questions that collect PII",
    "Max questions: 12",
    "Questions must be answerable anonymously"
  ]
}
```

**Tool schema:**
```json
{
  "name": "propose_form_changes",
  "input_schema": {
    "type": "object",
    "properties": {
      "proposed_changes": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "action":           { "type": "string", "enum": ["add","modify","remove"] },
            "question_id":      { "type": "string" },
            "question_spec": {
              "type": "object",
              "properties": {
                "type":    { "type": "string" },
                "label":   { "type": "string" },
                "options": { "type": "array", "items": { "type": "string" } },
                "required": { "type": "boolean" }
              }
            },
            "reason":             { "type": "string" },
            "confidence_basis":   { "type": "string" },
            "confidence_basis_score": { "type": "number", "minimum": 0, "maximum": 1 }
          },
          "required": ["action","question_id","reason","confidence_basis","confidence_basis_score"]
        }
      },
      "no_change_reason": { "type": ["string","null"] }
    },
    "required": ["proposed_changes","no_change_reason"]
  }
}
```

**Key constraint:** The policy guardrails in the system prompt (derived from the workspace_context) act as hard stops. If a company has GDPR active, the context document will explicitly say "do not propose questions collecting email, name, age, or location." The Evolution Agent sees this before it reasons, so it self-censors proposals that would violate policy.

**Output passed to:** Orchestrator (for confidence scoring and routing)

### 6.5 Scribe Agent

**Responsibility:** "Write the findings to long-term memory."

**Input (from orchestrator):**
```json
{
  "findings":        [ /* Analyst findings */ ],
  "applied_changes": [ /* changes that were auto-applied this run */ ],
  "queued_changes":  [ /* changes queued for human review */ ],
  "run_timestamp":   "2026-05-31T14:22:00Z",
  "trigger_type":    "event"
}
```

**Tool schema:**
```json
{
  "name": "update_knowledge",
  "input_schema": {
    "type": "object",
    "properties": {
      "entries": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "entry_type": { "type": "string", "enum": ["finding","form_change","baseline"] },
            "severity":   { "type": "string" },
            "content":    { "type": "string" }
          },
          "required": ["entry_type","content"]
        }
      },
      "workspace_context_patch": { "type": ["string","null"] }
    },
    "required": ["entries"]
  }
}
```

**`workspace_context_patch`** is the most architecturally interesting field. After the first real analysis run, the Scribe Agent can update the `## Baseline Metrics` section of the workspace_context document with derived values:

> "satisfaction_avg baseline established at 3.8 (n=150). Threshold: < 3.0 → MEDIUM. Updated: 2026-05-31."

This patch is applied to the stored `workspace_context` row. On the next pipeline run, the freshly updated context becomes the cached system prompt — meaning the Analyst Agent now has real baselines to compare against instead of Claude-estimated ones.

**This is the key mechanism by which EvolveForm becomes domain-adaptive over time without any manual configuration.**

---

## 7. Layer 5 — Persistence

All state lives in SQLite, accessed via raw `sqlite3` (no ORM). The schema is append-friendly — most tables only grow, never shrink.

### Key table relationships:

```
workspaces (1)
    │
    ├── workspace_context (1:1)
    │
    ├── form_versions (1:many) ← append-only
    │
    ├── submissions (1:many)
    │       │
    │       └── feeds into ──► analysis_runs (1:many)
    │
    ├── question_telemetry (1:many, keyed by question_id)
    │
    ├── pending_changes (1:many)
    │       │
    │       └── resolved by ──► admin approve/reject
    │                                    │
    │                            creates new form_version
    │
    ├── knowledge_entries (1:many) ← append-only
    │
    └── loop_runs (1:many)
```

---

## 8. Workspace Context System

The workspace context document is EvolveForm's most important design innovation. It solves the **domain generalization problem**: how does a single AI pipeline produce appropriate analysis thresholds for a retail NPS survey, an employee engagement form, an EU medical device feedback form, and a B2B SaaS onboarding survey — all without hardcoded rules?

### 8.1 Structure

```markdown
# Workspace Context — [Company Name]
Generated: 2026-05-01T10:00:00Z  |  Last Updated: 2026-05-31T14:22:00Z

## Organization Profile
[Company name, industry, country, target respondent audience]

## Form Purpose
[Verbatim onboarding input + Claude's expansion of implied intent]

## Policy Constraints
- GDPR active: do not propose questions collecting name, email, age, or precise location
- Questions must be answerable anonymously
- Form may not exceed 12 questions (completion rate degrades beyond this)

## Alert Routing Rules
- CRITICAL: immediate console alert + flag for DPO review (GDPR jurisdiction)
- HIGH: console alert + alerts.log entry
- MEDIUM: alerts.log entry only

## Active Alert Thresholds
satisfaction_avg: < 3.0 → MEDIUM, < 2.5 → HIGH
response_completion_rate: < 60% → LOW, < 40% → MEDIUM
recommend_rate: < 50% → MEDIUM, < 30% → HIGH

## Baseline Metrics
[Initially: "Establishing baselines — first analysis run pending"]
[After first run: populated with real computed values by Scribe Agent]

## Historical Findings Log
[Append-only. Scribe Agent writes timestamped entries here]

## Form Evolution Log
[Append-only. Scribe Agent writes change records here]
```

### 8.2 Generation

The workspace_context is generated during onboarding by a dedicated Claude call with its own tool schema (`generate_workspace_context`). Claude reads the onboarding data and produces:
- Derived policy constraints (e.g., GDPR → no PII questions)
- Appropriate alert thresholds for the domain (employee engagement gets different thresholds than a product return survey)
- Suggested question-count limits
- Alert routing rules calibrated to the country/compliance context

### 8.3 Evolution

The document is versioned in the `workspace_context` table. The Scribe Agent's `workspace_context_patch` field allows targeted updates (only the Baseline Metrics section, for example). The version integer increments on each patch, and the full patch history is auditable via `knowledge_entries`.

This is a **living document**, not a static config file. It accumulates knowledge over the lifetime of the deployment.

---

## 9. Question-Level Telemetry

Standard form platforms track answers. EvolveForm tracks *engagement*.

### 9.1 The Skip Rate Problem

Consider a form with 8 questions. A respondent loads the form (8 × `times_shown` increment) and submits answers for 6 of them (6 × `times_answered` increment). Two questions were skipped.

Across 200 respondents, if the same question is skipped 120 times, its `skip_rate` is `1 - (80/200) = 0.60`. A 60% skip rate is a loud signal that the question is confusing, irrelevant, or uncomfortable — but you would never see this signal if you only look at the 80 submitted answers.

### 9.2 Response Distribution Saturation

The `response_distribution_json` field tracks the count per option:

```json
{"strongly_agree": 142, "agree": 45, "neutral": 8, "disagree": 3, "strongly_disagree": 2}
```

When one option captures > 95% of responses ("strongly_agree" = 142/200 = 71% here), the question is no longer discriminating. The Analyst Agent flags this as a `low_variance` finding. The Evolution Agent then proposes either removing the question or restructuring it (finer-grained options, different framing).

### 9.3 How Telemetry Feeds the Pipeline

Both signals are passed as structured input to both the Analyst Agent and the Evolution Agent:

- **Analyst Agent** uses them to produce `skip_rate_anomaly` and `low_variance` findings
- **Evolution Agent** uses `skip_rate` directly to validate removal proposals (a "remove" proposal backed by 60% skip rate gets a higher `confidence_basis_score`)
- **Orchestrator** uses `skip_rate` as a deterministic boost in the confidence formula

---

## 10. Confidence Scoring Algorithm

Confidence scoring is the mechanism that prevents the AI from recklessly mutating the form on noisy data.

### 10.1 Formula

```python
def compute_confidence(change, findings, telemetry, workspace_id, db):
    base = change.confidence_basis_score   # Claude's estimate: 0.0–1.0

    # Boost 1: Persistence — same proposal seen in prior runs
    prior_count = db.count_prior_proposals(workspace_id, change.question_id, change.action)
    persistence_boost = min(0.20 * prior_count, 0.25)

    # Boost 2: Severity — how far the driving metric is outside the threshold
    finding = find_related_finding(change, findings)
    severity_boost = {"critical": 0.15, "high": 0.10, "medium": 0.05, "low": 0.0}.get(
        finding.severity if finding else "low", 0.0
    )

    # Boost 3: Supporting telemetry (for remove proposals)
    telemetry_boost = 0.0
    if change.action == "remove":
        skip_rate = telemetry.get(change.question_id, {}).get("skip_rate", 0.0)
        telemetry_boost = min(skip_rate * 0.5, 0.15)

    final = min(base + persistence_boost + severity_boost + telemetry_boost, 1.0)
    return final
```

### 10.2 Routing Decision Tree

```
Final confidence computed
          │
     ≥ 0.85?
     ┌──── YES ────┐
     │              │
   auto-apply    threshold
   immediately   configurable
   as new        per workspace
   form_version
   (source='ai-auto')
     │
     NO → between 0.5 and 0.85?
          ┌──── YES ────┐
          │              │
       queue to       pending_changes
       admin panel    (status='pending')
       for review     times_proposed=1
          │
          NO (< 0.5)
          │
       discard
       log to
       analysis_run
       with reason
```

### 10.3 Confidence Building Over Time

Suppose a change has initial confidence 0.63 (queued, not auto-applied). The admin leaves it pending. Next analysis run produces the same proposal: `times_proposed` increments to 2, `persistence_boost` is now 0.40 (capped at 0.25). If no other boosts apply:

```
0.63 (base) + 0.25 (persistence cap) = 0.88 → crosses auto-apply threshold
```

The form changes after the second run without any admin action. The admin sees it in the builder with an "AI" badge and the reason string.

**This is the system learning that its proposal is correct — not through more data, but through consistency of the signal across multiple independent analysis runs.**

### 10.4 Confidence Ceiling

The formula is capped at 1.0. No single boost can push a weak proposal to auto-apply. A `base = 0.20` with all boosts maxed reaches `0.20 + 0.25 + 0.15 + 0.15 = 0.75` — still below the default auto-apply threshold.

---

## 11. Full Data Flow — One Analysis Cycle

Below is the complete sequence for a single event-triggered analysis cycle.

```
RESPONDENT
    │
    │ POST /form/abc123/submit
    ▼
FLASK (server.py)
    │
    ├─ INSERT INTO submissions (id, workspace_id, data_json, analyzed=0)
    │
    ├─ UPDATE question_telemetry (times_answered, skip_rate, response_distribution)
    │
    └─ SELECT COUNT(*) FROM submissions WHERE analyzed=0
           │
        count ≥ threshold?
           │
          YES
           │
           ▼ (background thread)
ORCHESTRATOR (orchestrator.py)
    │
    ├─ SELECT * FROM submissions WHERE analyzed=0
    ├─ SELECT * FROM question_telemetry WHERE workspace_id=?
    ├─ SELECT content FROM workspace_context WHERE workspace_id=?
    │
    │ ── ANALYST AGENT CALL ──────────────────────────────────────────
    ├─ POST /messages (Claude claude-sonnet-4-6)
    │    system: [workspace_context, cache_control: ephemeral]
    │    tool: analyze_patterns (forced)
    │    input: {submissions, question_telemetry, batch_size}
    │    output: {findings[], alerts[], analysis_summary}
    │
    ├─ Write alerts to console / alerts.log (based on severity)
    │
    │ ── EVOLUTION AGENT CALL ─────────────────────────────────────────
    ├─ POST /messages (Claude claude-sonnet-4-6)
    │    system: [workspace_context, cache_control: ephemeral]  ← cache HIT
    │    tool: propose_form_changes (forced)
    │    input: {findings, current_form_questions, question_telemetry, policy_guardrails}
    │    output: {proposed_changes[], no_change_reason}
    │
    ├─ For each proposed change:
    │     final_confidence = compute_confidence(...)
    │     if final_confidence >= 0.85:
    │         INSERT INTO form_versions (source='ai-auto', ...)
    │     elif final_confidence >= 0.50:
    │         INSERT INTO pending_changes (status='pending', ...)
    │     else:
    │         log discard in analysis_run
    │
    │ ── SCRIBE AGENT CALL ────────────────────────────────────────────
    ├─ POST /messages (Claude claude-sonnet-4-6)
    │    system: [workspace_context, cache_control: ephemeral]  ← cache HIT
    │    tool: update_knowledge (forced)
    │    input: {findings, applied_changes, queued_changes, run_timestamp}
    │    output: {entries[], workspace_context_patch}
    │
    ├─ INSERT INTO knowledge_entries (all entries from Scribe)
    │
    ├─ If workspace_context_patch is not null:
    │     UPDATE workspace_context SET content=patched_content, version=version+1
    │
    ├─ UPDATE submissions SET analyzed=1 WHERE id IN (batch_ids)
    │
    ├─ INSERT INTO analysis_runs (trigger_type, submissions_analyzed, findings_json, ...)
    │
    └─ If any finding.severity == 'critical':
           schedule urgency recheck in 30 minutes

ADMIN PANEL
    │
    └─ /admin/pending shows new pending_change with confidence bar
       Admin clicks [Approve]
           │
           ▼
       INSERT INTO form_versions (source='ai-approved')
       UPDATE pending_changes SET status='approved', resolved_at=now()
```

---

## 12. Database Schema & Rationale

### Design Principles

- **Append-only for auditable tables**: `form_versions`, `knowledge_entries`, `analysis_runs`, `loop_runs` are never updated in place. History is always preserved.
- **No cascade deletes**: Deleting a workspace marks it inactive; orphan records are kept for audit.
- **JSON columns for flexible schemas**: `questions_json`, `data_json`, `findings_json` store structured data that evolves without schema migrations.
- **Raw sqlite3, no ORM**: Keeps the dependency surface minimal; the queries are simple enough that an ORM would add more complexity than it removes.

### Critical Table: `pending_changes`

```sql
CREATE TABLE pending_changes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    workspace_id INTEGER REFERENCES workspaces(id),
    proposed_at TEXT DEFAULT (datetime('now')),
    proposed_by_run INTEGER REFERENCES analysis_runs(id),
    times_proposed INTEGER DEFAULT 1,     -- increments each run
    action TEXT,                           -- add | modify | remove
    question_id TEXT,
    question_spec_json TEXT,
    reason TEXT,
    confidence REAL,                       -- final computed score
    status TEXT DEFAULT 'pending',         -- pending | auto-applied | approved | rejected
    resolved_at TEXT
);
```

The `times_proposed` field is what enables confidence to build over time without the admin doing anything. Each run, the orchestrator checks for existing pending rows matching `(workspace_id, question_id, action)`. If found, it increments `times_proposed` and recomputes confidence.

### Critical Table: `question_telemetry`

```sql
CREATE TABLE question_telemetry (
    question_id TEXT NOT NULL,
    workspace_id INTEGER REFERENCES workspaces(id),
    times_shown INTEGER DEFAULT 0,
    times_answered INTEGER DEFAULT 0,
    skip_rate REAL DEFAULT 0.0,
    response_distribution_json TEXT DEFAULT '{}',
    last_updated TEXT DEFAULT (datetime('now')),
    PRIMARY KEY (question_id, workspace_id)
);
```

The `PRIMARY KEY (question_id, workspace_id)` means one row per question per workspace — an `INSERT OR REPLACE` pattern on every form load/submit. The `question_id` is the UUID of the question as defined in `form_versions.questions_json`.

**Orphan telemetry:** When a question is removed from the form (new `form_versions` row without it), its telemetry row persists. This is intentional — if the question is re-added later, it inherits its historical skip rate as a prior.

---

## 13. Key Design Decisions

### Decision 1: Tool use with forced `tool_choice`

All Claude calls use `tool_choice: {"type": "tool", "name": "<tool_name>"}`. This forces Claude to call the specified tool and return its input as the response — guaranteeing structured JSON output even when the model might "prefer" to respond in prose.

Without forced tool_choice, a high-temperature run might produce a findings narrative without the required `metric_value` field, breaking the orchestrator's parsing. With forced tool_choice, the response is always a valid JSON object matching the schema.

### Decision 2: SQLite over PostgreSQL

For a v0.1 open-source framework, SQLite means zero infrastructure. Any developer can `git clone` and `python server.py` — no database setup, no connection strings, no migration tooling. The data volume that necessitates PostgreSQL (concurrent writes at scale, > 100GB, multi-node) is far beyond what a single-workspace EvolveForm deployment will see.

When EvolveForm needs to support multi-tenant cloud deployment, the schema is already clean enough to migrate. The JSON columns (`questions_json`, `data_json`) are the only friction points.

### Decision 3: 3-Agent Split (not 2, not 4)

Two agents (Analysis + Write) would still mix form mutation and knowledge archiving in the second call. Four agents would split detection from alerting — but alerting is a deterministic function of severity, not something Claude needs to decide. Three is the minimal split that gives each agent a single, testable responsibility.

### Decision 4: Human-in-the-loop via confidence band, not a review gate

Many agentic systems require human approval for every AI action. EvolveForm inverts this: the default is autonomous (high-confidence changes apply immediately), with human review reserved for uncertain changes. This is appropriate for form management because:
- Form changes are low-stakes and reversible (you can always revert to a prior `form_versions` row)
- Requiring approval for every change defeats the "self-evolving" purpose
- The confidence mechanism provides a principled basis for the autonomy decision

### Decision 5: No streaming responses

All three Claude calls use standard (non-streaming) API calls. Streaming would complicate the `tool_use` parsing without providing meaningful UX benefit — the admin never watches analysis run in real time; they see results after the fact in the analytics tab.

---

## 14. Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Claude API call fails (network / rate limit) | Pipeline run aborted mid-cycle | Orchestrator wraps each agent call in try/except; failed run logged with `outcome='error'`; submissions remain `analyzed=0` for next run |
| Evolution Agent proposes a GDPR-violating question | PII question added to form | Policy guardrails in workspace_context explicitly list prohibited question types; Evolution Agent self-censors; admin can also reject from Pending tab |
| Spam submissions spike confidence | Junk change auto-applied | Event trigger threshold and minimum batch size for auto-apply can be raised per workspace; admin can roll back via form_versions history |
| Urgency recheck loop (CRITICAL persists) | Re-fires every 30 minutes indefinitely | After 3 consecutive CRITICAL runs, urgency recheck interval doubles to 60 min, then 120 min; admin is alerted to take manual action |
| workspace_context becomes stale | Agent misapplies old thresholds | Scribe Agent patches baseline metrics section after every run; version integer makes staleness auditable |
| Orchestrator crashes mid-run | Some submissions marked analyzed, others not | Submissions are only marked `analyzed=1` after all three agent calls succeed; a crash leaves them `analyzed=0` for the next run to pick up |

---

## 15. Evolution from v0 → v0.1

The original PROJECT_BRIEF described a single-agent, body-lotion-specific form with file-based storage. The 5 architectural improvements in v0.1 are:

| # | v0 | v0.1 | Why |
|---|---|---|---|
| 1 | Single monolithic agent | 3-agent pipeline | Smaller context, focused prompts, structured data hand-off |
| 2 | Immediate form mutation | Pending changes queue + confidence scoring | Prevents form thrashing from noisy data |
| 3 | `PRODUCT_KNOWLEDGE.md` (static, domain-specific) | `workspace_context` (Claude-generated, per-workspace, evolving) | Domain generalization without hardcoded rules |
| 4 | Answer-only analysis | Question-level telemetry (skip_rate, response_distribution) | Visibility into engagement, not just answers |
| 5 | Fixed 5-minute interval | Hybrid trigger: schedule + event threshold + urgency recheck | Responsive to activity level, not clock |

Each improvement is **additive**: v0.1 is fully backwards-compatible with the v0 design. A deployment running v0 logic inside the v0.1 architecture (single agent, no telemetry, fixed interval) would produce identical form behavior with better infrastructure.

---

## Appendix: Agent Call Sequence with Timing

For a typical analysis run on a workspace with 25 unanalyzed submissions:

```
t=0ms    Orchestrator fetches submissions + telemetry (SQLite read)
t=12ms   Analyst Agent call starts
t=~1800ms Analyst Agent returns (1.8s — ~400 tokens out)
t=1812ms Evolution Agent call starts (findings passed as input)
t=~2400ms Evolution Agent returns (2.4s — ~600 tokens out, cache HIT)
t=2415ms Scribe Agent call starts
t=~1600ms Scribe Agent returns (1.6s — ~400 tokens out, cache HIT)
t=4020ms Orchestrator computes confidence, routes changes, writes DB
t=4080ms Full pipeline complete — analysis_run row inserted
```

Total wall-clock time: ~4 seconds per analysis cycle for a 25-submission batch.
Prompt cache reduces calls 2 and 3 by ~10× on the system prompt tokens.
```

---

*EvolveForm v0.1 — MIT License — Open source. Fork it. Deploy it. Evolve it.*
