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

**Where it stops.** STRIDE wasn't built for agent specific threats such as goal hijacking and tool misuse. Those are covered in `owasp-agentic.md` (planned).

## Method
1. **Per element.** Each element is checked against the categories that apply to its type: external entities for S and R, processes for all six, data stores for T, I and D, plus R for the log store because it holds the audit trail.
2. **Per interaction.** Each of the 9 flows that cross a trust boundary is checked again. A row is added only where the flow carries a threat the element rows don't already cover.

**Status:** Covered means a control is in place and evidenced. Partial means a control exists but doesn't fully address the threat, or hasn't been tested. Gap means there's no control.

## Per element

### External entities

| ID | Element / flow | STRIDE | Threat | Control | Evidence | Status |
|---|---|---|---|---|---|---|
| S‑01 | Analyst | S | Someone signs in as the analyst | MFA enforced tenant wide | [01](../evidence/01-security-defaults.png) | Covered |
| R‑01 | Analyst | R | Analyst denies asking a question | Each agent run is traced; whether the trace records the signed-in user isn't verified | [21c](../evidence/21c-tool-call-trace.png) | Partial |
| S‑02 | Admin | S | Someone signs in as the admin | MFA enforced tenant wide | [01](../evidence/01-security-defaults.png) | Covered |
| R‑02 | Admin | R | Admin denies writing a file | Storage logs separate the collector's writes from human writes | [15](../evidence/15-attribution-kql.png) | Covered |
| S‑03 | Client app | S | A stolen private key lets someone act as the client app | Certificate credential, no secret; the key sits unencrypted on a laptop | [P2‑03](../images/part2/P2-03-appreg-cert-no-secret.png) | Partial |
| R‑03 | Client app | R | Client app denies making calls | Gateway logs record the verified app ID on every allowed or throttled call | [P2‑10](../images/part2/P2-10-kql-denials.png) | Covered |

### Processes

| ID | Element / flow | STRIDE | Threat | Control | Evidence | Status |
|---|---|---|---|---|---|---|
| S‑04 | Collector | S | Something else writes snapshots as the collector | System assigned managed identity with no secret; only the collector holds write access | [10](../evidence/10-collector-identity.png), [16](../evidence/16-temp-role-removed.png) | Covered |
| T‑01 | Collector | T | Someone edits the workflow to write false data | No change control or alert on workflow edits | None | Gap |
| R‑04 | Collector | R | A write can't be tied to the collector | Storage logs show the collector's identity on each write | [15](../evidence/15-attribution-kql.png) | Covered |
| I‑01 | Collector | I | The collector's identity is used to read security data | Three scoped Graph read permissions; Security Reader is subscription-wide by design | [12](../evidence/12-graph-permissions.png), [11c](../evidence/11c-security-reader-scope-fixed.png) | Partial |
| D‑01 | Collector | D | The collector stops running and snapshots go stale | No alert on failed runs | None | Gap |
| E‑01 | Collector | E | The collector's identity is used beyond its job | Write access scoped to the snapshots container; Security Reader moved to its intended scope after landing on the wrong resource | [11a](../evidence/11a-collector-storage-role.png), [11b](../evidence/11b-security-reader-wrong-scope.png), [11c](../evidence/11c-security-reader-scope-fixed.png) | Covered |
| S‑05 | Agent | S | Another identity acts as the agent | Published agent identity with a federated credential, no secrets | [22a](../evidence/22a-agent-identity-created.png), [22b](../evidence/22b-blueprint-federated-no-secrets.png) | Covered |
| T‑02 | Agent | T | Someone changes the agent's instructions or tools | No change control or alert on agent edits | None | Gap |
| R‑05 | Agent | R | The agent's actions can't be reconstructed | Traces record each tool call and its output | [21c](../evidence/21c-tool-call-trace.png), [24b](../evidence/24b-trace-tool-output.png) | Covered |
| I‑02 | Agent | I | The agent reveals more than it should | Read-only scope; answers cite their source and flag synthetic data; the sensitive data guardrail annotates but doesn't block | [21b](../evidence/21b-synthetic-disclosed.png), [05b](../evidence/05b-guardrail-sensitive-data.png) | Partial |
| D‑02 | Agent | D | Heavy use exhausts model quota or budget | Budget alerts only; no per-user limit on the agent path | [02](../evidence/02-budget-alerts.png) | Partial |
| E‑02 | Agent | E | The agent is talked into writing or deleting data | GET only tool; write requests refused; jailbreak guardrail set to block | [18](../evidence/18-tool-spec.png), [23a](../evidence/23a-write-refused.png), [05a](../evidence/05a-guardrail-controls.png) | Covered |
| S‑06 | Storage tool | S | The tool reaches storage as the shared Foundry identity, not the agent's | Logged as an exception; the production fix is one agent per resource or a tool that uses the agent identity | [18](../evidence/18-tool-spec.png), [Exception register](../docs/Exception-register.md) | Partial |
| T‑03 | Storage tool | T | The tool spec is changed to add write operations | The identity holds read access only; immutability refuses overwrite and delete | [18](../evidence/18-tool-spec.png), [08](../evidence/08-immutable-overwrite-blocked.png), [09](../evidence/09-immutable-delete-blocked.png) | Covered |
| R‑06 | Storage tool | R | Tool calls go unrecorded | Each tool call and its output appear in the trace | [21c](../evidence/21c-tool-call-trace.png) | Covered |
| I‑03 | Storage tool | I | The tool reads beyond the snapshots | Read access scoped to one container; denied before access was granted | [19](../evidence/19-agent-denied-403.png) | Covered |
| E‑03 | Storage tool | E | Any agent in the same Foundry resource inherits the tool's read access | Logged as an exception | [Exception register](../docs/Exception-register.md) | Partial |
| S‑07 | Model | S | A caller skips the gateway and calls the model directly | Key auth disabled; anyone with an Entra role on the Foundry resource can still call it directly | [04](../evidence/04-foundry-key-auth-disabled.png), [Part 2 finding 2](../docs/part2-ai-gateway.md#findings) | Partial |
| T‑04 | Model | T | Instructions planted in snapshot data steer the model | Indirect prompt injection guardrail set to block; the planted injection was blocked on the tool response | [05a](../evidence/05a-guardrail-controls.png), [23c](../evidence/23c-injection-blocked.png), [23d](../evidence/23d-injection-file-read-log.png) | Covered |
| I‑04 | Model | I | Prompts or outputs leave the expected boundary | Global Standard deployment may process prompts outside the region; the sensitive data guardrail annotates only | [05b](../evidence/05b-guardrail-sensitive-data.png), [Exception register](../docs/Exception-register.md) | Partial |
| D‑03 | Model | D | Model quota is exhausted | Per caller token limits on the gateway path; none on the agent path | [P2‑09 429](../images/part2/P2-09-token-limit-429.png) | Partial |
| E‑04 | Model | E | A jailbreak makes the model ignore its instructions | Jailbreak guardrail set to block | [05a](../evidence/05a-guardrail-controls.png) | Covered |
| S‑08 | APIM | S | A caller with no token gets through | validate-jwt checks signature, expiry, audience, issuer and role | [P2‑06](../images/part2/P2-06-validate-jwt-policy.png), [P2‑08 no token](../images/part2/P2-08-no-token-401.png) | Covered |
| T‑05 | APIM | T | Someone edits the policy and removes validate-jwt | Policy kept in the repo; no drift detection or alert | [Policy file](../policies/foundry-model-inbound.xml) | Gap |
| R‑07 | APIM | R | Gateway decisions can't be traced to a caller | Every request logged with its outcome, reason and verified app ID | [P2‑10](../images/part2/P2-10-kql-denials.png) | Covered |
| I‑05 | APIM | I | Error responses tell an attacker why a call failed | The same 401 for every rejection; the reason is recorded only in the log | [P2‑08 no token](../images/part2/P2-08-no-token-401.png), [P2‑08 wrong audience](../images/part2/P2-08-wrong-aud-401.png), [P2‑10](../images/part2/P2-10-kql-denials.png) | Covered |
| D‑04 | APIM | D | The gateway is flooded | Per-caller rate and token limits; unauthenticated traffic still reaches the public gateway, and the Developer tier has no SLA | [P2‑09 policy](../images/part2/P2-09-limits-policy.png), [P2‑09 429](../images/part2/P2-09-token-limit-429.png) | Partial |
| E‑05 | APIM | E | A caller reaches the model without being granted access | The token must carry the Gateway.invoke role | [P2‑02](../images/part2/P2-02-gateway-api-app-role.png), [P2‑04](../images/part2/P2-04-token-claims-cert.png), [P2‑06](../images/part2/P2-06-validate-jwt-policy.png) | Covered |

### Data stores

| ID | Element / flow | STRIDE | Threat | Control | Evidence | Status |
|---|---|---|---|---|---|---|
| T‑06 | Snapshot store | T | Existing snapshots are changed or deleted | Time-based immutability refuses overwrite and delete; the policy is 1 day and unlocked | [07c](../evidence/07c-immutability-policy.png), [08](../evidence/08-immutable-overwrite-blocked.png), [09](../evidence/09-immutable-delete-blocked.png) | Partial |
| T‑07 | Snapshot store | T | A false or malicious file is added | Only the collector holds write access; the temporary user role was removed | [11a](../evidence/11a-collector-storage-role.png), [16](../evidence/16-temp-role-removed.png) | Covered |
| I‑06 | Snapshot store | I | Snapshots are read without authorization | Shared key and anonymous access off; reads need an Entra role; public network access is still on | [07b](../evidence/07b-storage-security-settings.png), [19](../evidence/19-agent-denied-403.png) | Partial |
| D‑05 | Snapshot store | D | Snapshots are deleted or unreachable | Delete refused by policy; retention is 1 day and unlocked | [09](../evidence/09-immutable-delete-blocked.png) | Partial |
| T‑08 | Log store | T | Logs are deleted or purged | No export to immutable storage; an Owner can delete the workspace | None | Gap |
| R‑08 | Log store | R | The attribution trail is lost | Every other repudiation row (R-01 to R-07, R-09, R-10) depends on this one workspace, with no separate copy | None | Gap |
| I‑07 | Log store | I | Traces hold a second copy of snapshot contents | Access isn't restricted to match the data | [Exception register](../docs/Exception-register.md) | Gap |
| D‑06 | Log store | D | A daily ingestion cap or an outage stops logging and nobody notices | No alert on missing logs | None | Gap |

**Not rated:**
- **Entra ID, Microsoft Graph and Defender for Cloud.** Microsoft runs them, so they're platform trust.
- **Model, R.** The model takes no action on its own. Its calls are attributed at the agent (R-05) and the gateway (R-07).
- **Storage tool, D.** Its availability is Azure Storage's.

## Per interaction

These rows cover the boundary-crossing flows where the flow adds a threat of its own. The other crossing flows are covered by element rows: F-08's spoofing, tampering and disclosure by S-06, T-03 and I-06, and F-11 by S-03. F-01, F-02 and F-11 reach Microsoft endpoints over HTTPS with Entra authentication, so threats in transit on them are platform trust.

| ID | Element / flow | STRIDE | Threat | Control | Evidence | Status |
|---|---|---|---|---|---|---|
| T‑09 | F‑04 Admin → Snapshot store | T | The admin path bypasses the collector | Write role removed after the build; an Owner could grant it again, with no alert on role grants | [16](../evidence/16-temp-role-removed.png) | Partial |
| T‑10 | F‑05 Analyst ↔ Agent | T | A user types a jailbreak or injection into a question | Jailbreak guardrail set to block; not tested on this path | [05a](../evidence/05a-guardrail-controls.png) | Partial |
| R‑09 | F‑08 Storage tool → Snapshot store | R | Storage logs can't tell which agent read a file | Match the Foundry trace to the storage log on file and time | [24a](../evidence/24a-storage-read-log.png), [24b](../evidence/24b-trace-tool-output.png) | Partial |
| T‑11 | F‑10 Agent → Log store | T | Fake traces are sent with a leaked connection string | Ingestion runs through the project managed identity; whether Application Insights still accepts connection-string ingestion isn't verified | [06a](../evidence/06a-tracing-managed-identity.png), [06c](../evidence/06c-tracing-role.png) | Partial |
| S‑09 | F‑12 Client app ↔ APIM | S | A token stolen from another service is replayed against the gateway | The audience check rejects tokens minted for other APIs | [P2‑08 wrong audience](../images/part2/P2-08-wrong-aud-401.png) | Covered |
| D‑07 | F‑12 Client app ↔ APIM | D | One caller floods the gateway | Per-caller limits keyed on the token's app ID | [P2‑09 policy](../images/part2/P2-09-limits-policy.png), [P2‑09 429](../images/part2/P2-09-token-limit-429.png) | Covered |
| R‑10 | F‑13 APIM ↔ Model | R | Foundry sees APIM's identity, not the caller's | Attribution happens at the gateway, which logs the verified app ID | [P2‑05](../images/part2/P2-05-foundry-role-apim-mi.png), [P2‑10](../images/part2/P2-10-kql-denials.png) | Covered |

## Findings

**49 threats rated: 23 Covered, 18 Partial, 8 Gap.**

| STRIDE | Covered | Partial | Gap |
|---|---|---|---|
| S | 6 | 3 | 0 |
| T | 3 | 4 | 4 |
| R | 7 | 2 | 1 |
| I | 2 | 4 | 1 |
| D | 1 | 4 | 2 |
| E | 4 | 1 | 0 |

1. **Per-interaction analysis found the attribution gap that per-element analysis missed.** Per element, the shared identity shows up only as a spoofing issue on the tool (S-06), and every repudiation row looks Covered because each element records its own actions (R-04 to R-07). Only the F-08 flow shows the consequence: storage can't tell which agent read a file (R-09). That's the gap Part 1 found during testing, reached here through the method.
2. **Every repudiation control rests on one workspace.** Storage logs, agent traces and gateway logs all land in the same Log Analytics workspace, with no protected copy (R-08, T-08). Whoever can delete that workspace can erase the trail that the rest of this model depends on.
3. **The configuration that enforces the controls has no change control.** The collector workflow, the agent's instructions and tools, and the APIM policy can all be edited with no alert (T-01, T-02, T-05). A control holds only as long as nobody quietly removes it.
4. **The gateway can be bypassed.** Anyone with an Entra role on the Foundry resource can call the model directly and skip validate-jwt and the limits (S-07).
5. **Telemetry is a second copy of the data, and its integrity isn't verified.** Traces hold snapshot contents (I-07), and whether a leaked connection string could inject fake traces hasn't been checked (T-11).

Every Partial and Gap row moves to `gaps.md` (planned), with the control or test that would close it.
