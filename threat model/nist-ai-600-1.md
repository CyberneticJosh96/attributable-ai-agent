# NIST AI 600-1: GenAI Risk Triage

## Scope
This is a lab system in my personal Azure tenant, built for learning and demonstration. It contains no employer data; the test files are synthetic.

The system has two paths:

- **Agent path (Part 1).** A read only Foundry agent with its own Entra agent identity answers questions over immutable security metric snapshots. A Logic App collects the snapshots on a schedule under a managed identity, with no stored secrets. The agent's one tool reads those snapshots from storage.
- **Gateway path (Part 2).** Azure API Management sits in front of the Foundry model (a GPT-5 mini deployment). Client apps authenticate with a certificate. Every request passes `validate-jwt` (audience, issuer, and the `Gateway.invoke` role) and per caller rate and token limits. The gateway reaches Foundry with its own managed identity, and key authentication is disabled.

This file triages all 12 risks in NIST AI 600-1 (the Generative AI Profile) against this system: which risks apply, which control covers each one, and where the gaps are.

## What NIST AI 600-1 is
NIST AI 600-1 is a profile of the NIST AI Risk Management Framework (AI RMF 1.0). A profile applies the framework's functions (Govern, Map, Measure, Manage) to a specific setting or technology. 600-1 does this for generative AI across all sectors. It doesn't replace the AI RMF but it does add a GenAI layer on top of an existing AI risk program.

It names 12 risks that generative AI creates or makes worse, and more than 200 suggested actions, each tagged with the risks it addresses. This triage uses the July 2024 version, which is still current.

**Why use it here, and where it stops.** The system runs on a generative model, so 600-1's risks apply to it. But 600-1 doesn't extend to agentic AI. Risks specific to agents, like tool misuse and identity abuse, are covered separately in `owasp-agentic.md` (planned).
## Triage

### In scope

| ID | Risk | Why it applies | Control | Evidence | Status |
|---|---|---|---|---|---|
| NIST-02 | Confabulation | The agent could report a wrong metric with confidence | Instructions require a cited snapshot file for every answer; out of scope questions get "not in the data" instead of a guess | [Out of scope](../evidence/23b-out-of-scope.png), [Cited answer](../evidence/21a-grounded-answer.png) | Partial |
| NIST-04 | Data Privacy | What the agent can see and repeat | Read only tool (GET operations only) scoped to the snapshots; reads denied until access was granted | [Read denied (403)](../evidence/19-agent-denied-403.png), [Tool spec](../agent/snapshot-reader.openapi.json) | Covered |
| NIST-09 | Information Security | Prompt injection, unauthorized callers, stolen credentials, resource abuse | Guardrails blocked a planted prompt injection; agent identity with no stored secrets; key auth disabled; certificate credential, validate-jwt and per-caller limits at the gateway | [Injection blocked](../evidence/23c-injection-blocked.png), [Guardrails](../evidence/05a-guardrail-controls.png), [No secrets](../evidence/22b-blueprint-federated-no-secrets.png), [Key auth off](../evidence/04-foundry-key-auth-disabled.png), [No token: 401](../images/part2/P2-08-no-token-401.png), [Wrong audience: 401](../images/part2/P2-08-wrong-aud-401.png), [Token limit: 429](../images/part2/P2-09-token-limit-429.png) | Covered || NIST-12 | Value Chain and Component Integration | Third-party model and Azure services I don't control | Model deployment named in Part 2; no record of version, source or vendor review | None yet | Gap |

### Out of scope

| ID | Risk | Why it doesn't apply |
|---|---|---|
| NIST-01 | CBRN Information or Capabilities | The agent only answers over security metric snapshots |
| NIST-03 | Dangerous, Violent, or Hateful Content | Narrow internal use case with no open ended generation |
| NIST-05 | Environmental Impacts | A training scale concern; this system uses a managed model |
| NIST-06 | Harmful Bias or Homogenization | The agent makes no decisions about people |
| NIST-08 | Information Integrity | No public output; this risk is about misinformation at scale |
| NIST-10 | Intellectual Property | Outputs summarize my own lab data |
| NIST-11 | Obscene, Degrading, and/or Abusive Content | Narrow internal use case with no open ended generation |
