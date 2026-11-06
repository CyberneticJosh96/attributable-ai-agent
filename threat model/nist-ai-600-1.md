# NIST AI 600-1: GenAI Risk Triage

## Scope
This is a lab system in my personal Azure tenant, built for learning and demonstration. It is NOT production and contains synthetic data.

The system has two paths:

- **Agent path (Part 1).** A read only Foundry agent with its own Entra agent identity answers questions over immutable security metric snapshots. A Logic App collects the snapshots on a schedule under a managed identity, with no stored secrets. The agent's one tool reads those snapshots from storage.
- **Gateway path (Part 2).** Azure API Management sits in front of the Foundry model (a GPT-5 mini deployment). Client apps authenticate with a certificate. Every request passes `validate-jwt` (audience, issuer, and the `Gateway.invoke` role) and per caller rate and token limits. The gateway reaches Foundry with its own managed identity, and key authentication is disabled.

This file triages all 12 risks in NIST AI 600-1 (the Generative AI Profile) against this system: which risks apply, which control covers each one, and where the gaps are.

## What NIST AI 600-1 is
NIST AI 600-1 is a profile of the NIST AI Risk Management Framework (AI RMF 1.0). A profile applies the framework's functions (Govern, Map, Measure, Manage) to a specific setting or technology. 600-1 does this for generative AI across all sectors. It doesn't replace the AI RMF but it does add a GenAI layer on top of an existing AI risk program.

It names 12 risks that generative AI creates or makes worse, and more than 200 suggested actions, each tagged with the risks it addresses. This triage uses the July 2024 version, which is still current.

**Why use it here, and where it stops.** The system runs on a generative model, so 600-1's risks apply to it. But 600-1 doesn't extend to agentic AI. Risks specific to agents, like tool misuse and identity abuse, are covered separately in `owasp-agentic.md` (planned).
