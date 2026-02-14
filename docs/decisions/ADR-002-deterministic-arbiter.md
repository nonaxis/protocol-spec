# ADR-002: Deterministic Arbiter with No LLM Fallback

Date: 2026-02-11
Status: Accepted

## Context
The original Arbiter design had a YAML rule engine as the primary decision path with an LLM fallback for cases where no rule matched. This meant the enforcement layer — the single component that MUST be trustworthy — was susceptible to prompt injection, model drift, and non-deterministic behavior.

## Decision
The Arbiter is a pure deterministic rule engine. If no rule matches a request, the Arbiter escalates to a human. There is no LLM fallback. Ever.

## Rationale
The Arbiter is the security boundary. If an attacker can prompt-inject the Arbiter's LLM fallback, they can approve arbitrary actions. A deterministic rule engine can be audited, tested, and formally verified. An LLM cannot.

The cost of this decision is more human escalations for edge cases. The cost of the alternative is an untrustworthy enforcement layer.

## Evidence
- **GPT-I (first to identify):** "LLM fallback introduces prompt injection risk, non-determinism in enforcement, and model drift risk in the one component that must be trustworthy."
- **Gemini-I:** Independently confirmed. "Arbiter must be 100% deterministic."
- **Opus:** "Removing LLM fallback from the decision path was the single most important change."
- **GPT-E:** Found that the `src/governance/` package **still has LLM fallback** — the fix was only applied to the root-level prototype. This must be unified.

All four reviewers agreed. This is the highest-confidence finding across all reviews.

## Consequences
- More human escalations for novel/unmatched requests
- Requires comprehensive rule coverage to minimize escalation rate
- Human fatigue risk if escalation rate is too high (see ADR-009)
- Rules must be maintained and tested like code
- OPA/Rego should be evaluated as a replacement for the custom YAML evaluator (it's battle-tested for exactly this use case)
