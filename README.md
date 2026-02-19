# Nonaxis

> **Institutional design for artificial cognition**

AI agents now have shell access, file system access, and the ability to install arbitrary code. They're autonomous actors with real-world consequences. Yet every major agent framework treats them like sole proprietors: they propose actions, approve actions, execute actions, and audit themselves.

We wouldn't design a human organization this way. Why do we design AI agents this way?

**Nonaxis** is a standardized governance architecture that applies centuries of institutional design — separation of duties, independent oversight, deterministic enforcement, fail-safe defaults — to AI agent execution. Three isolated instances with different capabilities challenge and constrain each other. A deterministic policy engine makes enforcement decisions. Execution tokens are cryptographically bound to approved actions and verified at the OS level.

No single agent holds unilateral power over consequential actions. Not because AI is dangerous, but because *any* autonomous actor — human or artificial — operating without institutional constraints will eventually cause harm.

---

## The Problem in Two Sentences

AI agents with system access self-govern: they decide what's safe, approve their own actions, and execute them. When that agent is compromised by prompt injection or model drift, every layer of protection falls at once — because they're all the same layer.

## What This Is

A **protocol for governing AI agent actions before they execute**, using multi-instance architecture with separation of duties:

- **Operator** proposes actions (full tools, sandboxed)
- **CISO** reviews for security risks (read-only, different LLM provider)
- **Arbiter** enforces deterministic policy and issues execution tokens (no LLM, rule engine)

The governance bus connects them via schema-validated JSON. The policy is deterministic: no LLM in the decision path. Enforcement is configurable from application-level hooks to kernel-level controls the agent cannot bypass.

```
                        Governance Bus
                   (schema-validated JSON)
                            |
         +------------------+------------------+
         |                  |                  |
   +-----+-----+     +-----+-----+     +-----+-----+
   |  Operator  |---->|   CISO    |---->|  Arbiter  |
   |            |     |           |     |           |
   | Proposes   |     | Reviews   |     | Decides   |
   | actions    |     | security  |     | via rules |
   |            |<----|           |     |           |
   | Full tools |     | Read-only |     | No LLM    |
   | Sandboxed  |     | Different |     | Enforces  |
   |            |     | provider  |     | policy    |
   +-----------+     +-----------+     +-----------+
```

When no rule covers a situation, the Arbiter escalates to a human. When a human makes a decision, the Operator logs it. Over time, repeated patterns become deterministic rules — reducing human load without introducing LLM non-determinism.

---

## Key Differentiators

1. **Protocol-first design** — Works at any deployment layer (kernel, container, or application). Same 4-verb API everywhere. Kernel enforcement is the strongest option, not a requirement.

2. **Deterministic policy engine** — No LLM in the enforcement decision path. Rules are explicit, auditable, and not susceptible to prompt manipulation. Unknown situations escalate to humans, never to another model.

3. **Multi-instance isolation** — Operator, CISO, and Arbiter are separate processes with different credentials, different capabilities, and (optionally) different LLM providers. Compromising one doesn't compromise the others.

4. **Content-hash-bound execution tokens** — The Arbiter issues time-limited, scope-pinned HMAC tokens tied to the exact action content. OS-level enforcement verifies tokens before execution. Replay attacks and token reuse are cryptographically prevented.

5. **Framework-agnostic** — Not Nonaxis-specific, not AutoGen-specific. Any agent framework can integrate via the governance bus API (`classify`, `submit`, `execute`, `report`).

---

## Quick Start

```bash
# Clone and set up
git clone https://github.com/nonaxis/protocol.git
cd nonaxis
python3 -m venv .venv && source .venv/bin/activate
pip install -e ".[dev]"

# Run the demo (see governance in action)
python3 demo/run_demo.py

# Run tests (136 passing)
pytest

# Start dev environment (Docker Compose)
cd deploy/
docker compose up -d
```

See [QUICK-START.md](QUICK-START.md) for detailed setup and integration guides.

---

## Architecture Overview

The protocol defines **four verbs**:

1. **`classify`** — Operator analyzes an action and assigns a risk tier (0-3)
2. **`submit`** — Operator sends the classified action to the CISO for review
3. **`execute`** — Arbiter issues an HMAC token; enforcement layer verifies and runs the action
4. **`report`** — Operator logs the result to a tamper-evident audit chain

**Risk tiers:**

| Tier | Examples | Governance |
|------|----------|-----------|
| 0 — Passive | Read, search, browse | Async audit log |
| 1 — Active | Write to workspace, known commands | Async CISO review |
| 2 — Sensitive | Unknown exec, new APIs, memory edits | Sync CISO + Arbiter approval |
| 3 — Critical | Skill install, credentials, self-modify | Sync + Human approval required |

**Enforcement options:**

| Tier | Enforcement | Use Case |
|------|------------|----------|
| **Kernel** | AppArmor + nftables + signed tokens | Self-hosted, security-critical |
| **Container** | seccomp + network policies + eBPF | Cloud, Kubernetes |
| **Application** | Process isolation + API scoping | SaaS, managed environments |

All three use the same governance API. The protocol is deployment-agnostic.

Full architecture: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

---

## Status (What Works Today)

✅ **Core protocol** — Schema-validated bus, 4-verb API, message signing  
✅ **Arbiter** — Deterministic rule engine, HMAC token issuance, escalation handling  
✅ **CISO** — Pattern analyzer, security review logic  
✅ **Operator** — Risk classifier (prototype)  
✅ **Audit trail** — Chain-hashed JSONL with tamper detection  
✅ **Container deployment** — Docker Compose + systemd  
✅ **Tests** — 136 passing (unit + integration)

🚧 **In progress:**  
- Kernel enforcement layer (AppArmor profiles, nftables rules)
- Nonaxis integration module
- Production hardening (mutual TLS, secrets management)

❌ **Not ready:**  
- Enterprise deployment (multi-tenancy, HA, observability)
- Non-Linux platforms
- Real-world validation (zero customers)

See [ROADMAP.md](ROADMAP.md) for detailed project status and timeline.

---

## What This Is NOT

- **Not production-ready.** This is a research prototype. It has survived adversarial reviews and passes 136 tests, but it has never governed a real agent in production.
- **Not a prompt guardrail.** We don't filter LLM inputs or outputs. We govern what agents *do* — shell commands, file writes, network requests.
- **Not Nonaxis-specific.** The architecture is agent-framework agnostic. Nonaxis is the first integration target.
- **Not magic.** If you give an AI agent root access to production servers, no governance layer can fully protect you. This reduces risk; it doesn't eliminate it.

---

## Documentation

**Core concepts:**
- [THESIS.md](docs/THESIS.md) — Why institutional design matters for AI cognition
- [PROBLEM-STATEMENT.md](docs/PROBLEM-STATEMENT.md) — The governance gap in detail
**Technical:**
- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — Full design with data flow
- [BUS-PROTOCOL.md](docs/BUS-PROTOCOL.md) — Message schemas and routing
- [OPERATOR-INTEGRATION.md](docs/OPERATOR-INTEGRATION.md) — How to integrate your agent
- [THREAT-MODEL.md](docs/THREAT-MODEL.md) — Attack scenarios and mitigations

**Project:**
- [CONTRIBUTING.md](CONTRIBUTING.md) — Development setup and pull request process
- [ROADMAP.md](ROADMAP.md) — What's planned

---

## Project Structure

```
docs/                   # Architecture, ADRs, threat model
  decisions/            # Architecture Decision Records
  proposals/            # Active proposals (NIST RFI, etc.)
schemas/                # JSON schemas for bus messages
specs/                  # OpenAPI spec, sandbox spec
examples/               # Sample policy rules
```

**Reference implementation** ([nonaxis/protocol](https://github.com/nonaxis/protocol)):
```
src/governance/         # Python package
  arbiter/              # Deterministic rule engine
  bus/                  # Schema validation, routing
  ciso/                 # Security pattern analysis
  operator/             # Risk classification
  models/               # Pydantic data models
  audit/                # Chain-hashed audit trail
tests/                  # pytest suite (136 passing)
services/               # FastAPI service wrappers
deploy/                 # Docker, systemd
sandbox/                # AppArmor profiles, nftables rules
```

---

## License

This **specification** is Apache License 2.0. See [LICENSE](LICENSE) for full text.

The reference **implementation** ([nonaxis/protocol](https://github.com/nonaxis/protocol)) is proprietary. See its LICENSE file for terms.

Contributions require DCO sign-off (Developer Certificate of Origin). See [CONTRIBUTING.md](CONTRIBUTING.md).

Copyright 2026 Nonaxis LLC

---

