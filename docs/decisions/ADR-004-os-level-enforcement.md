# ADR-004: OS-Level Enforcement (AppArmor + nftables) as Primary Security Boundary

Date: 2026-02-11
Status: Accepted

## Context
The governance architecture has multiple layers: CISO analysis, schema validation, risk scoring, Arbiter rules, AppArmor profiles, nftables firewall rules. Which layers are the actual security boundaries vs. quality improvements?

## Decision
OS-level enforcement (AppArmor + nftables) is the primary security boundary. Application-level governance (CISO, schema validation, risk scoring) improves decision quality but is not a security control.

## Rationale
AppArmor and nftables are enforced by the kernel. They cannot be bypassed by application-level exploits, prompt injection, or LLM manipulation. If every other layer fails, AppArmor still prevents the Operator from accessing files outside its profile, and nftables still blocks unauthorized network access.

Application-level controls can be evaded by a sufficiently motivated attacker (CISO patterns are regex-based, schema validation trusts the Operator's tier classification). They catch accidental risk, not intentional attacks.

## Evidence
- **GPT-I:** "Your real boundaries are: OS sandbox, deterministic Arbiter, file permission hygiene. Not: CISO LLM analysis, schema validation, risk scoring."
- **Opus:** "Defense in depth is genuine, not theater. Five independent layers where any single layer can fail."
- **Opus (concern):** AppArmor profiles are untested in enforcement mode. "An untested AppArmor profile is a false sense of security."
- **Opus (concern):** Node.js runtime bypasses exec restrictions — AppArmor allows `/usr/bin/node ix`, and Node.js can do `child_process.exec()`, `fs.writeFile()`, `net.createConnection()`.

## Consequences
- AppArmor profiles MUST be tested in enforcement mode (not just written)
- nftables rules must be verified with integration tests
- Node.js capabilities within AppArmor constraints need audit
- Cannot rely on CISO patterns or schema validation for security-critical decisions
- eBPF should be evaluated for finer-grained syscall filtering in production (Phase 4)
