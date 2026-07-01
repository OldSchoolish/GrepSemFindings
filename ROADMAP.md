# grepsemfindings — Roadmap

## Current Status

grepsemfindings is operational and in active use. The core pipeline — data acquisition, agent dispatch, validation chain, and report generation — is stable. The batch marking step (using session-authenticated curl PUT requests to the finding endpoint, compensating for the absence of a platform PUT API) is functional and intentional by design.

The tool is not feature-incomplete. Known friction is behavioral rather than structural.

---

## Known Friction

### AI Workflow Drift
The orchestrating AI can stray from the documented workflow — skipping steps, reordering phases, or making assumptions that diverge from the established pipeline. This is the primary reliability concern in day-to-day use.

**Mitigations in place:**
- Initial orientation prompt at session start (familiarize with project context, workflow, scripts, constraints)
- Configuration files lock in key behaviors
- Scripts are discrete and phase-gated to reduce surface area for improvisation

**Ongoing:** Prompt and config refinement as new drift patterns are observed.

### Ad-hoc Script Generation
The AI occasionally writes inline or one-off scripts for tasks that existing pipeline scripts already handle — and handle better. This creates inconsistency and can introduce untested code paths into a workflow that depends on validated, known-good scripts.

**Mitigations in place:**
- Whitelisted script list in configuration explicitly designates approved scripts for each task
- Documented anti-patterns call out known improvisation behaviors
- Orientation prompt reinforces script usage expectations at session start

**Ongoing:** Config and anti-pattern documentation updated as new instances surface.

---

## Potential Future Improvements

These are not committed — they represent directions worth revisiting if the tool sees broader use or if friction points grow. Reporting metrics summaries (TP/FP/NI/skip counts) are already implemented in the current report output.

### Batch Marking Token Handling
The current flow requires the analyst to manually obtain a session refresh token and bearer token and insert them into `.env` before each batch marking run. The hesitation around automating this step stems from uncertainty around what token retrieval methods might generate Windows event logs or trip browser-level session controls — a tradeoff that warrants further investigation before changing the approach.

### Finding Volume Scaling
The current batch size ceiling is set just below the point where visible system slowdown occurs on analyst workstations (observed around 25–30 agents). This is a hardware-informed limit, not an arbitrary one, and is consistent across the team. No changes planned unless hardware changes.

---

## What's Not on the Roadmap

- **Replacing the curl-based batch marking** — The approach is deliberate and working. It compensates correctly for the platform's lack of a PUT API and is not a candidate for replacement without a platform-side change.
- **Major pipeline restructuring** — The 3-script + orchestrator layer architecture is stable and intentional. Structural changes are not planned.
