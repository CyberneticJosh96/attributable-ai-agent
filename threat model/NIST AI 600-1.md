# NIST AI 600-1: GenAI Risk Triage

## Scope
This is a lab system in my personal Azure tenant, built for learning and demonstration. It is NOT production and contains synthetic data.

The system has two paths:

- **Agent path (Part 1).** A read only Foundry agent with its own Entra agent identity answers questions over immutable security metric snapshots. A Logic App collects the snapshots on a schedule under a managed identity, with no stored secrets. The agent's one tool reads those snapshots from storage.
- **Gateway path (Part 2).** Azure API Management sits in front of the Foundry model (a GPT-5 mini deployment). Client apps authenticate with a certificate. Every request passes `validate-jwt` (audience, issuer, and the `Gateway.invoke` role) and per caller rate and token limits. The gateway reaches Foundry with its own managed identity, and key authentication is disabled.

This file triages all 12 risks in NIST AI 600-1 (the Generative AI Profile) against this system: which risks apply, which control covers each one, and where the gaps are.
