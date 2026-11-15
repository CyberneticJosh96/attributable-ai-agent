# STRIDE Threat Model

## Scope
This is a lab system in my personal Azure tenant, built for learning and demonstration. It contains no employer data; the test files are synthetic.

The elements, data flows and trust boundaries come from [architecture.md](architecture.md). Every control links to evidence from Part 1 or Part 2. Where there's no evidence, the row is marked Partial or Gap.

## What STRIDE is
STRIDE is a threat classification model from Microsoft. Each letter is a type of threat and the security property it breaks:

| Letter | Threat | Breaks |
|---|---|---|
| S | Spoofing | Authentication |
| T | Tampering | Integrity |
| R | Repudiation | Non-repudiation |
| I | Information disclosure | Confidentiality |
| D | Denial of service | Availability |
| E | Elevation of privilege | Authorization |

**Where it stops.** STRIDE wasn't built for threats specific to agents, such as goal hijacking and tool misuse. Those are covered in `owasp-agentic.md` (planned).

## Method
1. **Per element.** Each element is checked against the categories that apply to its type: external entities for S and R, processes for all six, data stores for T, I and D, plus R for the log store because it holds the audit trail.
2. **Per interaction.** Each of the 9 flows that cross a trust boundary is checked again. A row is added only where the flow carries a threat the element rows don't already cover.

**Status:** Covered means a control is in place and evidenced. Partial means a control exists but doesn't fully address the threat, or hasn't been tested. Gap means there's no control.

## Per element

### External entities

| ID | Element / flow | STRIDE | Threat | Control | Evidence | Status |
|---|---|---|---|---|---|---|
| S‑01 | Analyst | S | Someone signs in as the analyst | MFA enforced across the tenant | [01](../evidence/01-security-defaults.png) | Covered |
| R‑01 | Analyst | R | Analyst denies asking a question | Each agent run is traced; whether the trace records the signed in user isn't verified | [21c](../evidence/21c-tool-call-trace.png) | Partial |
| S‑02 | Admin | S | Someone signs in as the admin | MFA enforced across the tenant | [01](../evidence/01-security-defaults.png) | Covered |
| R‑02 | Admin | R | Admin denies writing a file | Storage logs separate the collector's writes from human writes | [15](../evidence/15-attribution-kql.png) | Covered |
| S‑03 | Client app | S | A stolen private key lets someone act as the client app | Certificate credential, no secret; the key sits unencrypted on a laptop | [P2‑03](../images/part2/P2-03-appreg-cert-no-secret.png) | Partial |
| R‑03 | Client app | R | Client app denies making calls | Gateway logs record the verified app ID on every allowed or throttled call | [P2‑10](../images/part2/P2-10-kql-denials.png) | Covered |

### Processes

| ID | Element / flow | STRIDE | Threat | Control | Evidence | Status |
|---|---|---|---|---|---|---|
| S‑04 | Collector | S | Something else writes snapshots as the collector | System assigned managed identity with no secret; only the collector holds write access | [10](../evidence/10-collector-identity.png), [16](../evidence/16-temp-role-removed.png) | Covered |
| T‑01 | Collector | T | Someone edits the workflow to write false data | No change control or alert on workflow edits | None | Gap |
| R‑04 | Collector | R | A write can't be
