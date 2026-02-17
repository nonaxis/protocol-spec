# Nonaxis — Quick Start

## What Is This?

Nonaxis is a governance system for AI agents with system access. Three separate agent instances (Operator, CISO, Arbiter) challenge each other before actions execute. The Operator has tools but must request approval. The CISO reviews code for security risks. The Arbiter applies deterministic policy rules and issues time-limited execution tokens. Enforcement works at any layer — kernel (AppArmor + nftables), container (seccomp + eBPF), or application (process isolation + hooks). Same protocol everywhere. Patent-pending multi-instance architecture designed for AI agents with shell/file/network access.

## The Problem It Solves

AI agent frameworks give agents powerful tools (shell, file writes, API calls, package installs) but don't govern what they do at runtime. Prompt-level guardrails are bypassable. Single-process supervisors share a trust boundary with the agent. Human-in-the-loop means clicking "approve" on actions you can't fully evaluate. Nonaxis solves this with separation of duties: the agent that proposes actions can't approve them, the agent that reviews code can't execute it, and the decision engine is deterministic (no LLM in the approval path). Every action is audited, every token is time-limited and scope-pinned, and the audit trail is tamper-evident via chain hashing.

## Run the Demo (Under 2 Minutes)

```bash
# Clone and set up
git clone https://github.com/philg0rman/nonaxis.git
cd nonaxis
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

# Run the operator showcase demo (no services required)
GOVERNANCE_BASE_DIR=/tmp/demo python3 demo/operator_showcase.py
```

That's it. The demo runs standalone and simulates the full pipeline.

## What You'll See

The demo walks through 7 phases in ~30 seconds:

1. **Session Boot** — Fetch canonical state from the Arbiter (or gracefully handle offline mode)
2. **Action Submission** — Operator proposes a Tier 2 network request (GitHub API call)
3. **CISO Review** — Security analysis for injection patterns, credential access, network scope
4. **Arbiter Decision** — Deterministic policy engine applies rules and issues verdict
5. **Token Retrieval** — Time-limited execution token with content hash and scope constraints
6. **Token Validation** — Verify signature, expiry, content hash, and allowed network hosts
7. **Session Handoff** — Transfer state, assumptions, task queue to next session

Output includes:
- Request IDs with chain hashes (tamper-evident)
- Decision reasoning and policy rules applied
- Token details (content hash, expiry, allowed hosts)
- File paths for all artifacts (`/tmp/demo/pending-review/`, `/decisions/`, `/tokens/`)

The demo shows the Operator client API, the governance workflow, and token-based execution control. No LLM is used in the decision path — the CISO provides a recommendation, but the Arbiter decides via deterministic rules.

## Where to Learn More

- **Blog:** [Your AI Agent Has Root Access. Now What?](TBD) — Why governance matters for AI agents
- **Architecture:** [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — Full design with data flow diagrams
- **Problem Statement:** [`docs/PROBLEM-STATEMENT.md`](docs/PROBLEM-STATEMENT.md) — Threat landscape and motivation
- **Patent:** US Provisional 63/980,205 — Multi-instance governance architecture
- **NIST RFI Submission:** [Docket NIST-2025-0035](https://www.regulations.gov) — Governance recommendations for AI agents
- **GitHub:** https://github.com/philg0rman/nonaxis
- **Tests:** 136 passing — run `pytest` to verify all components
- **Full Demo:** `python3 demo/run_demo.py` (requires services: `bash demo/start_services.sh`)

## Next Steps

After running the demo:

```bash
# Run the full test suite
pytest

# Inspect artifacts created by the demo
cat /tmp/demo/pending-review/*.json | jq
cat /tmp/demo/decisions/*.json | jq
cat /tmp/demo/tokens/*.json | jq

# Read the architecture docs
cat docs/ARCHITECTURE.md

# Try the full pipeline with live services
bash demo/start_services.sh    # Terminal 1
python3 demo/run_demo.py       # Terminal 2
```

## Status

**Phase 2 complete** (February 2026):
- ✅ Operator client with 8 methods (boot, submit, poll, token ops, handoff)
- ✅ CISO adversarial code review
- ✅ Arbiter deterministic rule engine
- ✅ Filesystem bus with chain-hashed audit trail
- ✅ Session state management with handoff protocol
- ✅ 136 tests passing
- ✅ Patent filed (US Provisional 63/980,205)
- ✅ NIST RFI submitted (Docket NIST-2025-0035)

**Not production-ready.** This is a research prototype. The architecture has been adversarially reviewed by GPT-5.2 and Gemini 2.0 Flash Thinking, but it has never governed a real agent in a real deployment.

---

**License:** Apache 2.0  
**Copyright:** 2026 Nonaxis LLC
