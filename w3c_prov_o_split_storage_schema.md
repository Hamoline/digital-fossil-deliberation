# W3C PROV-O Split-Storage Provenance Schema

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24348597.svg)](https://doi.org/10.5281/zenodo.24348597)

* **Framework:** The Digital Fossil of Deliberation (Reference Architecture v1.1)
* **Author:** Heath Hamoline, Thompson Rivers University ([ORCID: 0009-0002-7436-1899](https://orcid.org/0009-0002-7436-1899))
* **Whitepaper DOI:** `10.5281/zenodo.24348598` (Concept DOI: `10.5281/zenodo.24348597`)
* **License:** MIT License (Code & Schemas) / CC-BY 4.0 (Conceptual Architecture)

---

## 1. Architectural Overview

To reconcile administrative fairness accountability with statutory data minimization and retention schedules (such as British Columbia FOIPPA Sections 28, 30, 30.1, and 31), the **Digital Fossil of Deliberation** partitions provenance telemetry from sensitive personal information across two relational tables:

1. **Parent Table (`DeliberationFossil`):** Stores zero Personally Identifiable Information (PII). Contains W3C PROV-O metadata, three-point UTC timestamps, normalized edit deltas, and SHA-256 cryptographic digests of both the raw AI draft (*State A*) and the human-approved text (*State B*). Table-level auditing is **ON**, and records are retained permanently.
2. **Child Table (`DeliberationPayloads`):** Stores the raw multiline text payloads under strict column-level security. Linked to the parent table via a `Referential, Restrict Delete` lookup. Table-level auditing is **OFF** (preventing database log bloat), and rows are automatically purged via scheduled bulk deletion once the statutory retention period expires.

---

## 2. Relational Schema & W3C PROV-O Mapping

| Table & Column Name | Data Type | W3C PROV-O Class / Predicate | Specification & Security Control |
| :--- | :--- | :--- | :--- |
| **`DeliberationFossil`** *(Parent)* | Table | `prov:Bundle` | Table Auditing: **ON**; Zero PII permitted. |
| `FossilID` | Autonumber (PK) | `prov:Bundle` Identifier | Format: `FOSSIL-{SEQNUM:6}` paired with pre-generated GUID. |
| `WorkflowName` | String (150) | `prov:Activity` Descriptor | Canonical name of the governed automation workflow. |
| `RiskTier` | Choice | `prov:Activity` Attribute | `Tier 1 - Low`, `Tier 2 - Internal PII`, `Tier 3 - High/Legal`. |
| `StateA_SHA256` | String (64) | `prov:Entity` (Source Draft) | Hex SHA-256 digest of UTF-8/LF-canonicalized AI output. |
| `StateB_SHA256` | String (64) | `prov:Entity` (`prov:wasRevisionOf` State A) | Hex SHA-256 digest of UTF-8/LF-canonicalized human output. |
| `AIModelID` | String (100) | `prov:Agent` (Software Agent) | Identifier of originating model/agent (e.g., Copilot Studio). |
| `ReviewerUPN` | String (150) | `prov:Agent` (`prov:wasAttributedTo`) | Authenticated Entra ID UPN of the actual card responder. |
| `CardSentUTC` ($t_0$) | DateTime (UTC) | `prov:Activity` Notification Time | Time-zone-independent timestamp when Stage 1 card was sent. |
| `ReviewOpenedUTC` ($t_1$) | DateTime (UTC) | `prov:startedAtTime` | Timestamp when human clicked *Unlock & Begin Review*. |
| `ApprovedUTC` ($t_2$) | DateTime (UTC) | `prov:endedAtTime` | Timestamp when human committed Stage 2 decision. |
| `QueueLatencySeconds` | Integer | Derived Temporal Metric | Inbox wait latency: $t_1 - t_0$ (in seconds). |
| `ActiveDwellSeconds` | Integer | Derived Cognitive Metric | Active human review duration: $t_2 - t_1$ (in seconds). |
| `StateACharCount` | Integer | `prov:Entity` Metric | Character length of canonicalized State A text. |
| `StateBCharCount` | Integer | `prov:Entity` Metric | Character length of canonicalized State B text. |
| `EditDeltaPercentage` | Decimal (5,2) | `prov:qualifiedRevision` Metric | Normalized Levenshtein edit distance ($\Delta_{\text{edit}}$). |
| `DeliberationOutcome` | Choice | `prov:Activity` Disposition | `Approved Unchanged`, `Minor Edit`, `Substantial Rewrite`, `Rejected / Escalated`, `Expired / Timed Out`. |
| **`DeliberationPayloads`** *(Child)* | Table | `prov:Entity` Value Store | Table Auditing: **OFF**; Column-Level Security **ON**. |
| `PayloadID` | Autonumber (PK) | `prov:Entity` Identifier | Format: `PAY-{SEQNUM:6}`. |
| `FossilLookup` | Lookup (FK) | `prov:specializationOf` | Points to `DeliberationFossil` (`Restrict Delete`). |
| `StateA_AIDraft` | Multiline Text | `prov:value` (State A) | Raw unedited AI output (purged on retention schedule). |
| `StateB_HumanFinal` | Multiline Text | `prov:value` (State B) | Final human-approved output (purged on retention schedule). |
| `ReviewerRationale` | Multiline Text | `rdfs:comment` | Optional justification note entered by the reviewer. |

---

## 3. Canonicalization, Hashing & Edit Delta Formulas

To prevent cross-platform line-ending differences (`\r\n` vs. `\n`) from causing false hash mismatches, normalize both strings in-memory prior to computing hashes or edit distances:

1. **String Canonicalization:** Apply Unicode NFC normalization and convert all CRLF (`\r\n`) sequences to LF (`\n`) using UTF-8 byte encoding.
2. **Cryptographic Digest:** Compute `SHA-256` over the canonicalized byte arrays of State A and State B.
3. **Normalized Levenshtein Edit Distance ($\Delta_{\text{edit}}$):**

$$\Delta_{\text{edit}} = \left[ \frac{d_{\text{Levenshtein}}(\text{State A}, \text{State B})}{\max(\text{length}(\text{State A}), \text{length}(\text{State B}))} \right] \times 100\%$$

4. **Post-Purge Non-Repudiation:** After `DeliberationPayloads` rows are purged at the end of the statutory retention period, re-hashing the official executed artifact on file (e.g., the final decision letter or signed contract) and matching it against `StateB_SHA256`—while verifying `StateA_SHA256` $\neq$ `StateB_SHA256` alongside `ActiveDwellSeconds`—proves human modification without retaining duplicate PII archives.

---

## 4. Orchestration Implementation Notes

* **JSON Escaping for Adaptive Cards:** Escape backslashes, double quotes, carriage returns, and line feeds before injecting State A into `adaptive-card-gate.json`:
  ```text
  @{replace(replace(replace(replace(triggerBody()['stateA_text'], '\', '\\'), '"', '\"'), decodeUriComponent('%0D'), ''), decodeUriComponent('%0A'), '\n')}
  ```
* **Atomic Changeset Binding:** Pre-generate the parent GUID via `guid()` before opening a database changeset scope, then bind the child `DeliberationPayloads` lookup using `/deliberationfossils(@{outputs('Compose_FossilGUID')})`.
* **Execution Log Redaction:** Enable **Secure Inputs** and **Secure Outputs** on all workflow triggers, messaging card actions, and payload write steps so PII never appears in workflow run histories.