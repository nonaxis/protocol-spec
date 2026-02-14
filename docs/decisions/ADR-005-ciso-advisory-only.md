# ADR-005: CISO as Advisory, Not a Security Gate

Date: 2026-02-11
Status: Accepted

## Context
The CISO instance analyzes proposed actions using static pattern matching and LLM-based risk analysis. Should it be a hard gate (block on CISO rejection) or an advisory signal (inform the Arbiter's decision)?

## Decision
The CISO is a "risk oracle" — it provides risk assessments that feed into the Arbiter's decision. It is not a security gate (it cannot unilaterally approve/deny). A CISO failure does not mean governance fails; it means the Arbiter has less information (and should fail-closed per ADR-008).

## Rationale
The CISO's static patterns are trivially evadable (~30 regexes, shell-centric). Its LLM analysis is susceptible to prompt injection. Making the CISO a hard security gate would mean the governance system is only as strong as a regex scanner + an LLM — both of which are unreliable against motivated attackers.

The CISO's real value is catching accidental risk in legitimate skills, not stopping attacks. Framing it as advisory prevents over-reliance on a weak signal.

## Evidence
- **GPT-I:** Reframed CISO as "risk oracle" — the insight that defined this decision.
- **Opus:** "Pattern library has ~30 regexes. Any competent attacker bypasses them in minutes." But: "This matters less than it seems because the CISO is explicitly documented as advisory."
- **Opus:** CISO patterns are shell-centric; Node.js payloads (`https.request()`, `child_process.execSync()`) evade every pattern.

## Consequences
- CISO downtime degrades decision quality, not security
- Arbiter must be able to make decisions without CISO input (with degraded confidence)
- CISO false-positive rate should be tuned for usefulness, not completeness
- Semgrep should replace custom patterns for static analysis (Phase 2-3)
