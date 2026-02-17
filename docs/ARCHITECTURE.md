# Architecture Deep Dive

## Overview

Three air-gapped Nonaxis instances communicate through a controlled message bus to implement inner governance — a self-policing architecture where specialized agents challenge, review, and constrain each other before actions reach the outside world.

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Operator   │────▶│   CISO       │────▶│   Arbiter    │
│  (executor)  │     │  (reviewer)  │     │  (decider)   │
│              │◀────│              │     │              │
│  Full tools  │     │  Read-only   │     │  100%        │
│  OS sandbox  │     │  No exec     │     │  deterministic│
│  User-facing │     │  No network* │     │  No LLM      │
└─────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       └────────────────────┴────────────────────┘
                    Governance Bus
                  (Caddy reverse proxy)
                  Schema-validated JSON
                  Hardcoded routing rules
```

*CISO has network only for its own LLM API calls, enforced by AppArmor.

## Design Principles

### 1. Bus Enforces Governance, Not Agents
The routing rules live in deterministic code (Caddy config + schema validation). A compromised Operator cannot bypass the pipeline because it physically cannot route messages to itself or forge Arbiter decisions.

### 2. OS-Level Enforcement Is Non-Negotiable
AppArmor profiles and nftables rules restrict each instance at the kernel level. The Operator's firewall blocks outbound connections to unknown hosts unless a time-limited token from the Arbiter is present. This is the real security boundary — everything else is defense in depth.

### 3. Fail Closed, Always

> **Note:** The CISO is *advisory in decision authority* (ADR-005), but its *availability* can be required for Tier 2+ actions to preserve fail-closed semantics (ADR-008).

- CISO unreachable → Operator blocks on Tier 2+ actions → escalate to human
- Arbiter unreachable → same
- Bus unreachable → everything stops
- No rule matches → escalate to human (never to LLM)
- Token expired → action blocked

### 4. Defense in Depth
Five layers, each independent:
1. **AppArmor** — kernel-level filesystem and exec restrictions
2. **nftables** — kernel-level network restrictions with token-based temporary allows
3. **Bus routing** — deterministic message flow, schema validation
4. **CISO review** — LLM-based adversarial code analysis
5. **Arbiter rules** — deterministic policy, human escalation for unmatched cases

Any single layer can fail and the others still provide protection.

## Risk Tier Classification

Classification is deterministic (pattern matching on action type + arguments), NOT LLM-decided:

| Tier | Trigger Pattern | Governance |
|------|----------------|-----------|
| 0 | `read`, `search`, `web_search`, `web_fetch` | None. Async audit log |
| 1 | `write` (to workspace), `exec` (allowlisted commands), `message` | Async CISO audit |
| 2 | `exec` (not in allowlist), `web_fetch` (new domain), memory/prompt edits | Sync review pipeline |
| 3 | Skill install/update, credential read, self-modify, `exec` with network, listen on port | Sync review + human confirm |

**⚠️ Implementation note:** The architecture calls for classification in the bus layer, with the Operator's self-classification as a non-authoritative hint. In the current code, tier classification lives in `src/governance/operator/classifier.py` and the bus does **not** independently verify. This is a known discrepancy tracked for Phase 3. The Operator's classification is currently trusted, which means a compromised Operator could downgrade its own action tiers. Moving classification to the Arbiter is the planned fix.

## Token-Based Execution

When the Arbiter approves a Tier 2+ action:

1. Arbiter generates a signed token: `{token_id, expires_at, scope, hmac_signature}`
2. Token is written to a shared mount readable by the Operator
3. The Operator's nftables rules check for valid tokens before allowing governed network connections
4. Token expires after TTL (default 300s)
5. Token scope is pinned to the specific content_hash — can't be reused for different code

## Chain Hashing

Every bus message includes `chain_hash = SHA256(previous_hash + JSON(current_message))`.

Genesis message uses `chain_hash = "0" * 64`.

This creates a tamper-evident log. If any message is modified or deleted post-hoc, subsequent hashes won't validate. Useful for post-incident forensics, not real-time prevention.

## Model Diversity

The three instances should run different LLM providers/models:
- **Operator:** Claude Sonnet/Opus (strong general-purpose)
- **CISO:** GPT-4o or Gemini (different training data → different blindspots)
- **Arbiter:** No LLM. Pure deterministic rule engine. Unmatched cases escalate to human.

Model diversity between Operator and CISO reduces correlated failure probability.

## Instance Isolation

Each instance runs in its own LXC container (production) or Docker container (dev):

| Instance | Filesystem | Network | Exec | Messaging |
|----------|-----------|---------|------|-----------|
| Operator | Read/write own workspace | Governed outbound | Full (sandboxed) | User channels |
| CISO | Read-only | LLM API only | None | None |
| Arbiter | Own config only | None (no LLM) | None | None |

The CISO and Arbiter are intentionally powerless. They can analyze and decide, but cannot act on the world. This is their security guarantee — even if fully compromised, they can only produce bad recommendations (which the other layers catch).
