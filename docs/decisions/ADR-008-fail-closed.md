# ADR-008: Fail-Closed on All Component Failures

Date: 2026-02-11
Status: Accepted

## Context
When a governance component fails (CISO crashes, Arbiter unreachable, bus down, token expired), should the system fail-open (allow the action) or fail-closed (block the action)?

## Decision
Fail-closed everywhere. CISO down → block. Arbiter down → block. Bus down → block. Token expired → block. TOKEN_SECRET is default value → refuse to start.

## Rationale
Fail-open in a security system means an attacker can bypass governance by causing a denial-of-service on any governance component. Crashing the CISO shouldn't grant the Operator unrestricted access.

The cost is reduced availability — the Operator can't act when governance is down. This is the correct tradeoff for a security system: safety over liveness.

## Evidence
- **Opus:** "Fail-closed everywhere... This is the correct default."
- **Opus:** `TOKEN_SECRET = "CHANGE-ME-IN-PRODUCTION"` default should fail-closed (refuse to start), not silently run with a guessable secret.
- All reviewers implicitly assumed fail-closed behavior.

## Consequences
- Governance availability directly impacts Operator availability
- Must monitor governance component health (detect silent failures)
- Must have clear recovery procedures for each failure mode
- Escalation timeouts must also fail-closed (timeout → deny, not timeout → allow)
