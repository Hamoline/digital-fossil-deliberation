# Safe Start 3-Axis Risk Intake Rubric & Governance Gate

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24348597.svg)](https://doi.org/10.5281/zenodo.24348597)

* **Framework:** The Digital Fossil of Deliberation (Reference Architecture v1.1)
* **Author:** Heath Hamoline, Thompson Rivers University ([ORCID: 0009-0002-7436-1899](https://orcid.org/0009-0002-7436-1899))
* **Whitepaper DOI:** `10.5281/zenodo.24348598` (Concept DOI: `10.5281/zenodo.24348597`)
* **License:** MIT License

---

## 1. Purpose

Enforcing mandatory human review gates on low-risk personal productivity prompts induces alert fatigue and degrades institutional velocity. Conversely, allowing straight-through AI execution on workflows affecting individuals' rights, finances, or personal data creates statutory privacy and administrative fairness liabilities.

The **Safe Start Intake Rubric** provides a repeatable, quantitative scoring mechanism to classify proposed AI and automation workflows prior to production deployment.

---

## 2. Three-Axis Scoring Matrix (0–9 Points)

Evaluate every proposed workflow across three independent risk axes (0 to 3 points per axis; maximum composite score of 9 points):

| Score | Axis 1: PII & Data Sensitivity | Axis 2: Decision & Legal Impact | Axis 3: Exposure & Blast Radius |
| :--- | :--- | :--- | :--- |
| **0 Points** | Public catalog, marketing, or open data | Personal productivity drafting or brainstorming | Single author private workspace |
| **1 Point** | Internal operational or procedural documentation | Internal IT/administrative ticket triage or routing | Internal departmental team channel |
| **2 Points** | Directory, contact, or general enrollment metadata | Resource, facility, or classroom scheduling | Cross-unit internal institutional broadcast |
| **3 Points** | Student grades/appeals, HR, medical, or financial PII | Academic standing, employment, contracts, safety alerts | External transmission to students, vendors, or public |

---

## 3. Tier Classification & Architectural Gating Rules

| Composite Score | Risk Tier | Governance & Architectural Requirement |
| :--- | :--- | :--- |
| **0 – 2 Points** | **Tier 1: Low Risk** | Cleared for standard enterprise AI assistant usage; monitored passively via Data Security Posture Management (DSPM) and DLP policies. |
| **3 – 5 Points** | **Tier 2: Moderate / Internal PII** | **Mandatory Digital Fossil Gate.** Must route through the Two-Stage Validation Gate and log provenance metadata and hashed payloads before execution. |
| **6 – 9 Points** | **Tier 3: High / Legal & External** | **Mandatory Digital Fossil Gate + Privacy Review.** Requires Two-Stage Validation Gate, Privacy Impact Assessment (PIA) check, and formal UAT sign-off prior to production pipeline promotion. |

---

## 4. Production Promotion UAT Checklist (Tier 2 & Tier 3)

Before promoting any Tier 2 or Tier 3 workflow from development/sandbox into production, governance reviewers must verify five technical controls:

- [ ] **Run-History Redaction:** Workflow execution logs display `outputs hidden` on all triggers, messaging cards, and payload write actions containing draft text.
- [ ] **Two-Stage Timing Isolation:** Stage 1 (*Unlock & Begin Review*) stamps `ReviewOpenedUTC` ($t_1$) separately from `CardSentUTC` ($t_0$), isolating inbox queue wait from active cognitive dwell time.
- [ ] **Client-Side Attestation Enforcement:** Attempting to click *Approve & Commit State B* without checking the statutory attestation box is blocked client-side, while *Reject / Escalate* succeeds immediately without requiring attestation.
- [ ] **Atomic Split-Storage Commit:** Parent `DeliberationFossil` (with SHA-256 digests and normalized edit delta) and child `DeliberationPayloads` commit atomically, and downstream actions strictly consume *State B*.
- [ ] **Payload Overflow & Timeout Handlers:** Drafts exceeding messaging card payload limits (e.g., $>3,500$ characters) route cleanly to a long-form review workbench, and unanswered cards trigger a 3-day reminder and 7-day timeout escalation.

---

## 5. Non-Punitive Coaching Governance

Telemetry captured in `DeliberationFossil` is aggregated strictly at the **workflow and organizational-unit level**—never as an individual employee surveillance scorecard.

* **Compliance Theater Threshold:** If a workflow exhibits `ActiveDwellSeconds < 5` AND `EditDeltaPercentage = 0%` across more than $85\%$ of runs, governance leads treat the signal as a **systemic design indicator**—prompting system-prompt refinement, UI adjustments, or intake tier recalibration rather than punitive review.