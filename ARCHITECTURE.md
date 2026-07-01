# grepsemfindings — Architecture

## Overview

grepsemfindings is a multi-phase, AI-orchestrated SAST triage pipeline designed to reduce alert fatigue from high-volume Semgrep findings. It combines a 3-script pipeline with an agent orchestration layer to automate finding enrichment, codebase investigation, and report generation — with an optional batch-marking step to close the loop in the findings platform.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        AI Platform (Orchestrator)               │
│               OpenCode Desktop or TUI on analyst workstation    │
└────────────────────────────┬────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
   Phase 1 Script      Preflight Script     Phase 2 Script
   (Data Acquisition)  (Permission Gate)    (Investigation Support)
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                    Orchestrator Layer
                 (Agent Dispatch & Validation)
                    [Explore Agents × N]
                             │
                    Phase 3 Script
                    (Report Generation)
```

---

## Pipeline Phases

### Phase 1 — Data Acquisition Script

Handles all input ingestion and finding enrichment before any AI investigation begins.

**Steps:**

1. **Initialization** — AI familiarizes itself with project context, workflow, scripts, and constraints via configuration files. This initial orientation prompt compensates for the high volume of context across config files.
2. **Environment check** — Script checks for `.env` and prompts the analyst to populate it if missing. Required values:
   - Ticketing platform API token
   - Internal SAST findings platform API token
   - Path to locally cloned project repo
3. **Ticket input** — Analyst provides a ticket number. Script retrieves the associated `.csv` from the ticket, containing Internal SAST findings platform finding links and flagged Semgrep rules.
4. **Internal SAST platform findings enrichment** — For each entry in the `.csv`, the script calls the SecHub API to retrieve full finding details:
   - File path
   - Flagged code snippet
   - Line number(s)
   - Repository name
   - State (`pending`, `fp`, `tp`)
5. **Filtering** — Findings are evaluated before inclusion:
   - Non-`pending` findings → skipped
   - Non-main-project repos → skipped
   - Skips are logged with reason and surfaced in the final report
6. **JSONL formation** — Non-skipped findings are written to a `.jsonl` file. Format chosen for efficient sequential AI parsing in this pipeline context.

**Output:** Enriched `.jsonl` findings file, skip log

---

### Preflight Script — Permission Gate

A two-pass CLI script that establishes filesystem access before the orchestrator and explore agents begin working.

**Pass 1 (CLI arg: `--request-access`)**
- Reads `.env` for the repo path
- Requests top-level read permission on the project repo directory from the AI platform

**Pass 2 (CLI arg: `--confirm-access`)**
- Confirms top-level read was granted
- Unlocks orchestrator to begin agent dispatch

**Why this matters:** Explore agents inherit this top-level read grant rather than each requesting access independently. Ask once, carry it through — avoids repeated permission prompts across potentially 20+ parallel agents.

---

### Orchestrator Layer — Agent Dispatch & Validation

Not a script — this is the AI platform's own orchestration capability, operating above the script layer.

**Explore Agent Design:**
- Agent type: deep codebase comprehension (optimized for reading and understanding code, not fast shallow parsing)
- Dispatch: parallel batches of 5–20 agents, configurable
- Each agent receives dynamically constructed instructions including:
  - Full file path (`.env` repo path + appended Semgrep-reported path)
  - Flagged rule
  - Finding details from `.jsonl`
- Each agent traces source-to-sink and returns structured findings

**Validation Chain:**

To account for the error surface of multi-agent parallel execution, a 4-stage validation chain ensures data integrity:

```
Explore Agent output
        ↓
  Orchestrator (validates agent output structure and completeness)
        ↓
  Temp file (validates write consistency)
        ↓
  Working file (validates no duplicates, proper appending, all fields present)
```

Each stage checks that finding details are consistent across every occurrence before promotion to the next stage.

---

### Phase 2 — Investigation Support Script

Handles the mechanical side of the orchestrator's work: managing appends to the working file and maintaining working file integrity throughout the agent dispatch cycle.

---

### Phase 3 — Report Generation Script

Consumes the completed working file and produces a structured markdown report in three stages.

**Report categories:**

| Category | Description |
|---|---|
| True Positives | Confirmed vulnerable findings with source-to-sink analysis |
| Standards Violations | Not exploitable but violates coding standards |
| False Positives | Confirmed safe with explanation of why |
| Needs Investigation | Ambiguous — too many matched lines, file not found, or determination not possible |

Each entry includes:
- Source-to-sink trace
- Assessment rationale
- Direct link to the Internal SAST findings platform finding page for one-click marking

**Optional: Batch Marking**

After report generation, the analyst is prompted to optionally batch-mark FPs or TPs directly in Internal SAST findings platform.

**Implementation note:** The Internal SAST findings platform does not support `PUT` via API. Batch marking therefore requires:
1. Analyst obtains a valid session refresh token and bearer token
2. Tokens are added to `.env` before running the batch mark step
3. Script executes the marking operations using those tokens
4. Tokens are cleared from `.env` at end of script execution

---

## Folder Structure

```
grepsemfindings/
├── .backups/               # Auto-generated backups
├── archive/
│   ├── docs-backup/        # Previous doc versions
│   └── one-off-scripts/    # Experimental/retired scripts
├── docs/
│   ├── .backups/           # Doc-level backups
│   ├── implementation/     # Implementation notes and decisions
│   └── investigations/     # Notes from past triage sessions
├── examples/               # Sample inputs/outputs for reference
├── input/                  # Ticket CSVs placed here for processing
├── output/
│   ├── analyzed/           # Completed reports
│   ├── raw/                # Intermediate working files
│   └── temp/               # Temp files during agent validation chain
├── scripts/
│   └── __pycache__/        # Python cache
└── templates/              # Prompt/output templates
```

---

## Key Design Decisions

**JSONL over JSON or CSV** — Sequential line-by-line format reduces memory overhead and is better suited for AI parsing in a streaming pipeline context. Suggested by AI during development and validated in practice.

**Explore agents over shallow parsers** — Deep codebase comprehension agents were chosen deliberately over faster, partial-parsing alternatives. The accuracy tradeoff is worth it for triage decisions that carry real security weight.

**Batch size configurability (5–20)** — Allows tuning based on workstation capability and finding volume. Larger batches are faster; smaller batches reduce risk of validation errors compounding.

**Two-pass preflight** — Separates the permission request from the confirmation, ensuring access is actually granted before the orchestrator begins dispatching agents. Prevents silent failures from missing file access mid-pipeline.

**Token hygiene on batch marking** — Bearer tokens are cleared from `.env` at script end to avoid credential persistence. Compensates for the platform's lack of `PUT` API support without leaving credentials at rest.

**4-stage validation chain** — Multiple agents working in parallel create significant surface area for data integrity issues (duplicates, partial writes, field mismatches). Validation at each promotion step catches errors before they propagate to the report.

---

## Data Flow Summary

```
Ticket # (analyst input)
    → CSV (finding links + rules)
    → Internal SAST findings platform API enrichment
    → Filter (skip non-pending / non-main-repo)
    → .jsonl (enriched findings)
    → Preflight (permission gate)
    → Explore agents × N (parallel, batched)
    → Validation chain (agent → orchestrator → temp → working file)
    → 3-stage report (.md)
    → [Optional] Batch mark FPs/TPs in SecHub
```
