# grepsemfindings 🔍

> AI-assisted SAST triage at scale — so your analysts can focus on what matters.

---

## The Problem

SAST programs generate findings faster than analysts can review them. At scale, weekly finding volumes can reach into the hundreds — and without a way to prioritize, filter, and investigate efficiently, the backlog grows, alert fatigue sets in, and real vulnerabilities get buried under noise.

grepsemfindings was built to change that ratio. Instead of an analyst manually opening each finding, navigating to the affected file, tracing the code path, and making a judgment call — an AI-assisted pipeline does the investigative legwork, traces source-to-sink for each finding, and delivers a structured report with findings pre-categorized as True Positives, False Positives, Standards Violations, or Needs Investigation. The analyst reviews the report and acts on it, rather than starting from scratch on every finding.

---

## How It Works

grepsemfindings is a three-phase scripted pipeline with an orchestrator-managed agent layer handling the core investigation work.

### Phase 1 — Data Acquisition

**Initialization script:**
- Checks for `.env` configuration
- Prompts for missing credentials: ticketing platform API token, Internal SAST findings platform API token, locally cloned project repo path
- Validates setup before proceeding

**CSV acquisition script:**
- Accepts a ticket number as input
- Retrieves the findings table from the ticket as a `.csv`
- Each row contains a Internal SAST findings platform finding link and its flagged Semgrep rule — the minimum needed to begin enrichment

**Internal SAST findings platform enrichment script:**
- Calls the Internal SAST findings platform API for each finding in the `.csv`
- Retrieves full finding details: filepath, flagged code snippet, line number(s), repository, and state (pending / FP / TP)
- Skips non-pending findings and findings from non-main project repos — each skip is logged with a reason and surfaced in the final report
- Writes enriched findings to a `.jsonl` file — chosen for its suitability as a structured, line-by-line format for AI parsing in batch pipelines

---

### Preflight — Permission Gate

Before the orchestrator dispatches any agents, a dedicated preflight script runs a two-pass permission check:

**Pass 1:** Requests top-level read access on the project repo path defined in `.env`

**Pass 2:** Confirms read access was granted

This gate ensures that all explore agents inherit file system access from a single approved permission rather than each agent requesting access independently. Ask once, carry it through.

Once Pass 2 confirms access, control is handed to the orchestrator.

---

### Orchestrator — Agent Layer

The orchestrator manages the investigation layer directly — this is not scripted, but handled by the AI platform's native agent dispatching capabilities.

**Explore agent dispatch:**
- Dynamically generates per-finding investigation instructions, including the full filesystem path to the finding (constructed from `.env` repo path + appended Semgrep-reported path)
- Dispatches explore agents in configurable batches of 5–20, running in parallel
- Explore agents are specifically selected for deep codebase comprehension — optimized for reading and understanding code structure rather than fast shallow parsing
- Each agent traces the finding from source to sink within the codebase and returns structured investigation details

**Validation chain:**
Due to the complexity of parallel agent output, a multi-stage validation layer runs between each handoff:

```
Explore agent output
       ↓
Orchestrator validation (completeness, structure)
       ↓
Temp file validation (no duplicates, correct format)
       ↓
Working file append (consistency confirmed)
```

Each stage checks that finding details are consistent across every occurrence, properly formatted, and free of duplicates before the next stage proceeds.

---

### Phase 2 — Investigation Support

The Phase 2 script supports the orchestrator's working file management — handling appends and ensuring the working file remains clean and consistent as batches complete.

---

### Phase 3 — Reporting

**Report generation script:**
Processes the completed working file through a 3-stage report generation pipeline, producing a structured markdown report organized into four categories:

| Category | Description |
|---|---|
| **True Positives** | Confirmed vulnerabilities with source-to-sink trace and Internal SAST findings platform link |
| **Standards Violations** | Code that violates security standards but may not be directly exploitable |
| **False Positives** | Confirmed safe findings with reasoning |
| **Needs Investigation** | Findings the pipeline couldn't fully resolve — too many matched line numbers, file not found, or determination unclear |

Each entry includes source-to-sink details, reasoning, and a direct link to the finding's Internal SAST findings platform page for one-click marking.

**Batch marking (optional):**
After report generation, the user is prompted to optionally batch-mark FPs or TPs directly in Internal SAST findings platform. Due to platform constraints (PUT not supported via standard API), this requires a valid session token and bearer token added to `.env` before running. The script handles the marking and clears tokens from `.env` on completion.

---

## Project Status

| Component | Status |
|---|---|
| Initialization + env check | ✅ Complete |
| CSV acquisition | ✅ Complete |
| Internal SAST findings platform enrichment + JSONL formation | ✅ Complete |
| Preflight permission gate | ✅ Complete |
| Orchestrator + explore agent dispatch | ✅ Complete |
| Validation chain | ✅ Complete |
| 3-stage report generation | ✅ Complete |
| Batch FP/TP marking | ✅ Complete |

---

## Stack

- **Python** — Phase 1 and Phase 3 scripts, preflight script
- **AI platform** (OpenCode Desktop / TUI) — orchestrator and explore agent layer
- **Semgrep / Internal SAST findings platform** — finding source and marking platform
- **JSONL** — intermediate finding storage format
- **Markdown** — report output format

---

## Configuration

All credentials and paths are managed via `.env`:

```
TICKETING_API_TOKEN=
SAST_PLATFORM_API_TOKEN=
REPO_PATH=
# Added before batch marking, cleared after:
SESSION_TOKEN=
BEARER_TOKEN=
```

---

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for a detailed breakdown of the pipeline, orchestrator design, validation chain, and agent selection rationale.

---

## Roadmap

See [ROADMAP.md](./ROADMAP.md) for planned features and improvements.

---

## Background

grepsemfindings grew out of a real scaling problem — weekly SAST finding volumes that outpaced analyst capacity, and the recognition that most of the investigative work (navigate to file, read the code, trace the path, make a call) was mechanical enough that an AI could do it faster and more consistently than a human doing it cold on every finding. The analyst's judgment is still in the loop — it's applied to a pre-investigated, pre-categorized report rather than to raw findings one at a time.

---

*grepsemfindings is a portfolio reconstruction project. No proprietary code, data, or organizational specifics from prior employment are included.*
