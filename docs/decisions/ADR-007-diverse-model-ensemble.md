# ADR-007: Diverse Model Ensemble (Different LLMs for Operator vs CISO)

Date: 2026-02-11
Status: Accepted

## Context
If the Operator and CISO use the same LLM provider/model, a prompt injection vulnerability affecting that model compromises both the governed entity and its reviewer simultaneously. Correlated failures defeat the purpose of separation.

## Decision
Use different LLM providers for the Operator and CISO. E.g., Operator on Claude, CISO on GPT (or vice versa). The Arbiter uses no LLM (see ADR-002).

## Rationale
Different model families have different vulnerability profiles. A prompt injection that works on Claude may not work on GPT, and vice versa. Using diverse models means an attacker must find exploits for multiple model families simultaneously.

## Evidence
- **Gemini-E:** Explicitly recommended diverse model ensemble for CISO vs Operator.
- **Opus:** The adversarial review process itself demonstrated that different models find different things — validating that model diversity improves coverage.

## Consequences
- Higher operational complexity (multiple API keys, different prompt formats)
- Higher cost (can't use volume discounts from a single provider)
- Must maintain compatibility with multiple LLM APIs
- Provides defense against correlated model vulnerabilities
