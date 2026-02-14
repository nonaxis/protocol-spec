# ADR-003: Filesystem Bus for PoC, HTTP Bus for Production

Date: 2026-02-11
Status: Accepted

## Context
The three instances need to communicate. The PoC uses a filesystem bus: JSON files in shared directories, polled by an orchestrator. All four reviewers flagged this as not production-ready.

## Decision
- **PoC (current):** Filesystem bus with JSON files. Acceptable for single-agent, low-throughput testing.
- **Production:** HTTP sidecars per instance, communicating via a central bus (Caddy or message queue).
- **Optional intermediate:** SQLite WAL if HTTP timeline slips.

## Rationale
The filesystem bus has fundamental limitations: TOCTOU vulnerabilities, no atomic writes, symlink/path traversal risks, O(n) glob scans, no backpressure, 2-second polling latency. These are inherent to the approach, not implementation bugs.

**Dissenting opinion on migration path:**
- Gemini-I recommended SQLite WAL as a quick interim step (atomic writes, locking, queries).
- Opus recommended skipping SQLite and going directly to HTTP sidecars, arguing SQLite adds engineering cost without solving the polling problem.
- Decision: HTTP is the target. SQLite is acceptable if HTTP is delayed, but not a required step.

## Evidence
- **All four reviewers** flagged filesystem bus race conditions.
- **GPT-E:** Found specific vulnerabilities — symlink attacks, path traversal, TOCTOU on partial writes.
- **Opus:** Scaling analysis shows filesystem bus breaks at ~10 concurrent agents.
- **Opus:** "`shutil.move` is not atomic on cross-device moves" — affects Docker shared mounts.

## Consequences
- PoC bus is acceptable for development and single-agent testing
- Must not deploy filesystem bus with real data or multiple agents
- HTTP migration is a prerequisite for production (Phase 3-4)
- Bus hardening (O_NOFOLLOW, path sanitization, atomic writes) is still needed for PoC reliability
