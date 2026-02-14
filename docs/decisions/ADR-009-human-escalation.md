# ADR-009: Human Escalation over LLM Fallback for Edge Cases

Date: 2026-02-11
Status: Accepted

## Context
When the Arbiter's deterministic rules don't match a request, something must happen. The original design used LLM fallback. After removing that (ADR-002), the question is: what happens for unmatched requests?

## Decision
Escalate to a human. The Arbiter produces an escalation decision with a timeout. A human reviews and approves/denies via Telegram or similar channel. If the timeout expires, the request is denied (fail-closed per ADR-008).

## Rationale
An LLM fallback is fast but untrustworthy in the enforcement layer. A human is slow but makes the system's trust boundary explicit: "we trust a human to make judgment calls, not an LLM."

**Critical risk (Gemini-I):** If escalation rate is too high, the human will start auto-approving everything. The security of the entire system depends on keeping escalation rate low enough that the human stays engaged. This means rules must be comprehensive and CISO false-positive rate must be tuned.

## Evidence
- **GPT-I:** Proposed human escalation as replacement for LLM fallback.
- **Gemini-I:** "User fatigue / auto-approve risk" — the #1 operational threat.
- **Opus:** "Escalation to human has no implementation. The Arbiter produces escalation decisions with `timeout_minutes`, but there's no actual human notification system." **This is currently unimplemented.**

## Consequences
- Must implement actual notification (Telegram/Discord integration) — currently missing
- Must implement escalation timeout enforcement — currently missing
- Must track escalation rate and alert if too high
- Rule coverage must be comprehensive to minimize escalations
- Escalation UX must be fast and low-friction for the human
