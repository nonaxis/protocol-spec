# Operator Integration Guide

## Overview

The **Operator** is the user-facing Nonaxis instance with full tools and OS sandbox. It implements self-governance by submitting high-risk actions through a review pipeline before execution.

This guide covers the Operator's integration with the Arbiter state API and governance pipeline.

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   Operator                       │
│  (Full-featured Nonaxis instance)              │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │         OperatorClient                   │  │
│  │  • boot()          — session init         │  │
│  │  • submit_review() — governance request   │  │
│  │  • poll_decision() — wait for arbiter     │  │
│  │  • get_token()     — retrieve token       │  │
│  │  • apply_token()   — validate + execute   │  │
│  │  • submit_handoff()— session end          │  │
│  │  • propose_state_patch()                  │  │
│  └──────────────────────────────────────────┘  │
│                      ↓                           │
│           File-based Message Queue               │
│  (pending-review/, decisions/, tokens/)         │
└─────────────────────────────────────────────────┘
                      ↓
         ┌────────────────────────┐
         │   Governance Bus       │
         │   (Orchestrator)       │
         └────────────────────────┘
                      ↓
         ┌────────────────────────┐
         │   CISO (Reviewer)      │
         └────────────────────────┘
                      ↓
         ┌────────────────────────┐
         │   Arbiter (Decider)    │
         │   + State API          │
         └────────────────────────┘
```

## Session Lifecycle

### 1. Boot

Every session starts with a call to the Arbiter State API to fetch:

- **Canonical State Snapshot** — authoritative state (identity, objectives, constraints, systems)
- **Decision Ledger** — last 30 decisions + always-include tagged entries
- **Recent Events** — last 20 state events
- **Last Handoff** — previous session's summary and task queue
- **Consistency Checks** — constraint validation, drift detection

```python
from governance.operator.client import OperatorClient

client = OperatorClient(arbiter_url="http://arbiter:8300")
boot_data = client.boot()

snapshot = boot_data["snapshot"]
ledger = boot_data["ledger"]
consistency = boot_data["consistency_checks"]
```

**Consistency checks** include:
- **Constraint check** — validate no constraint violations
- **Dependency check** — verify all referenced IDs exist
- **Drift check** — detect unexpected state changes since last handoff

If drift is detected, the Operator should alert the user before proceeding.

### 2. Governed Action Submission

For Tier 2+ actions (exec_unfamiliar, outbound_network, credential_access, etc.):

```python
# Submit review request
request_id = client.submit_review(
    action_type="exec_unfamiliar",
    content=b"#!/bin/bash\necho 'Hello'\n",
    outbound_hosts=["example.com"],  # Optional
    context="User requested script execution"
)

# Poll for decision
decision = client.poll_decision(request_id, timeout=30.0)

if decision and decision["payload"]["decision"] == "approve":
    # Get execution token
    token = client.get_token(request_id)
    
    # Validate token
    if client.apply_token(token, content):
        # Execute the action
        # In production: nftables/AppArmor integration
        pass
```

### 3. State Updates

When canonical state needs updating (e.g., new objective, constraint change):

```python
# Propose a patch
patches = [
    {
        "op": "add",
        "path": "/active_objectives/-",
        "value": {
            "objective_id": "obj-123",
            "description": "New objective",
            "priority": "high"
        }
    }
]

proposal_id = client.propose_state_patch(
    snapshot_id=snapshot["snapshot_id"],
    patches=patches,
    rationale="User added new objective"
)

# Proposal requires Arbiter review before commit
```

State patches use [JSON Patch (RFC 6902)](https://datatracker.ietf.org/doc/html/rfc6902) format.

### 4. Session Handoff

At session end (timeout, user disconnect, agent completion):

```python
handoff_id = client.submit_handoff(
    snapshot_id=snapshot["snapshot_id"],
    summary=[
        "Completed 3 tasks",
        "Added new constraint C-042",
        "No unresolved issues"
    ],
    task_queue=[
        {
            "task": "Follow up on email from Alice",
            "priority": "medium",
            "est_effort": "10m"
        }
    ],
    assumptions_made=[
        {
            "assumption": "User wants daily summaries at 8 AM",
            "basis": "User said 'every morning'"
        }
    ],
    state_updates_proposed=[
        {
            "type": "constraint_add",
            "proposal_id": "prop-xyz"
        }
    ]
)
```

The next session will load this handoff during boot for continuity.

## OperatorClient API

### Core Methods

#### `boot() -> dict`

Fetch session boot payload from Arbiter.

**Returns:**
```python
{
    "snapshot": {...},           # Canonical state snapshot
    "ledger": [...],             # Decision ledger entries
    "recent_events": [...],      # Recent state events
    "last_handoff": {...},       # Previous session handoff
    "consistency_checks": {      # Validation results
        "constraint_check": "pass",
        "drift_check": "pass",
        "details": [...]
    }
}
```

#### `submit_review(action_type, content, *, outbound_hosts=None, context="") -> str`

Submit a review request to the governance pipeline.

**Args:**
- `action_type` (str): Type of action (exec_unfamiliar, outbound_network, etc.)
- `content` (bytes): Raw content (code, command, etc.)
- `outbound_hosts` (list[str], optional): Network destinations
- `context` (str, optional): Context for the action

**Returns:** Request ID (UUID)

#### `poll_decision(request_id, timeout=30.0) -> Optional[dict]`

Poll for a decision on a review request.

**Returns:** Decision dict if found, None if timeout

#### `get_token(request_id) -> Optional[dict]`

Retrieve execution token for an approved request.

**Returns:** Token dict with scope and expiry

#### `apply_token(token, action_content) -> bool`

Validate an execution token.

**Returns:** True if token is valid and action can proceed

#### `submit_handoff(snapshot_id, summary, **kwargs) -> str`

Submit a session handoff to the Arbiter.

**Args:**
- `snapshot_id` (str): Current canonical state snapshot ID
- `summary` (list[str]): Session summary (brief statements)
- `assumptions_made` (list[dict], optional)
- `unknowns` (list[dict], optional)
- `task_queue` (list[dict], optional)
- `state_updates_proposed` (list[dict], optional)
- `risks_noted` (list[dict], optional)

**Returns:** Handoff ID

#### `propose_state_patch(snapshot_id, patches, rationale="") -> str`

Propose a patch to canonical state.

**Args:**
- `snapshot_id` (str): Base snapshot ID
- `patches` (list[dict]): JSON Patch operations (RFC 6902)
- `rationale` (str): Explanation for the change

**Returns:** Proposal ID

### Convenience Methods

#### `submit_and_wait(action_type, content, *, outbound_hosts=None, context="", timeout=30.0) -> tuple`

Convenience wrapper: submit + poll + get token in one call.

**Returns:** `(decision, token)` tuple (either may be None)

## File-Based Message Queue

The Operator communicates with the governance pipeline via filesystem volumes:

```
$GOVERNANCE_BASE_DIR/
├── pending-review/    # Operator writes review requests here
├── assessments/       # CISO writes assessments here
├── decisions/         # Arbiter writes decisions here
├── tokens/            # Arbiter writes execution tokens here
└── state/
    ├── snapshots/     # Canonical state snapshots
    ├── decision_ledger.jsonl
    ├── events.jsonl
    ├── handoffs/      # Session handoff artifacts
    └── proposals/     # Pending state patch proposals
```

**Why filesystem?** Air-gap enforcement. In production, each instance has its own container with read-only access to peer volumes. The Operator cannot directly write to `decisions/` or `tokens/` — only the Arbiter can.

## Token-Based Execution

When the Arbiter approves a Tier 2+ action, it issues a **time-limited, content-scoped execution token**:

```json
{
  "token_id": "347e567b-...",
  "nonce": "a1b2c3d4-...",
  "expires_at": "2026-02-12T18:25:58Z",
  "scope": {
    "action_type": "exec_unfamiliar",
    "content_hash": "a9d810f9f3734b24...",
    "allowed_hosts": ["example.com"],
    "max_duration_seconds": 60
  },
  "hmac_signature": "f4c8e2..."
}
```

**Token enforcement** (production):
1. **nftables** — firewall rules allow outbound connections to `allowed_hosts` only when a valid token exists
2. **AppArmor** — filesystem/exec restrictions lifted for the token scope
3. **TOCTOU protection** — content hash prevents token reuse for different code

**Token enforcement** (demo/dev):
- `apply_token()` validates structure, content hash, and expiry
- No real OS integration (development only)

## Integration Patterns

### Pattern 1: Synchronous Review (Tier 2)

For actions that can wait (exec_unfamiliar, new outbound domain):

```python
decision, token = client.submit_and_wait(
    action_type="exec_unfamiliar",
    content=script_bytes,
    context="User requested data processing script"
)

if decision and token and client.apply_token(token, script_bytes):
    result = execute_script(script_bytes)
else:
    notify_user("Action denied by governance pipeline")
```

### Pattern 2: Async Review (Tier 1)

For low-risk actions (workspace writes, allowlisted commands):

```python
# Execute immediately
result = write_file(path, content)

# Audit asynchronously (CISO reviews post-hoc)
client.submit_review(
    action_type="filesystem_write",
    content=content,
    context=f"Wrote {len(content)} bytes to {path}"
)
```

### Pattern 3: Human Escalation (Tier 3)

For very high-risk actions (credential access, self-modify):

```python
request_id = client.submit_review(
    action_type="credential_access",
    content=b"access to vaultwarden master password",
    context="User requested password for banking site"
)

# Notify user: "Action requires human approval"
# Poll with longer timeout
decision = client.poll_decision(request_id, timeout=600.0)
```

The Arbiter will escalate to human if no deterministic rule matches.

## Error Handling

### Arbiter Unreachable

If the Arbiter State API is unavailable:

```python
boot_data = client.boot()
if boot_data["snapshot"] is None:
    # Arbiter unreachable — fail-safe mode
    # Only allow Tier 0 actions (read, search)
    # Escalate to human for Tier 1+
```

**Fail-closed by default**: If governance pipeline is down, block governed actions.

### Stale State

If a state patch proposal is based on an old snapshot:

```python
proposal_id = client.propose_state_patch(snapshot_id, patches, rationale)
# Arbiter returns 409 Conflict if snapshot is stale
# Re-fetch latest state and retry
```

### Token Expiry

Tokens expire after TTL (typically 5 minutes). If execution takes longer:

```python
if client.apply_token(token, content):
    # Start execution
    result = long_running_task()
else:
    # Token expired — resubmit review request
    new_request_id = client.submit_review(...)
```

## Testing

### Unit Tests

```bash
pytest tests/operator/test_client.py -v
```

### Integration Tests

```bash
# Start Arbiter State API
python run_state_api.py &

# Run integration tests
pytest tests/integration/test_operator_pipeline.py -v
```

### Demo

```bash
# Phase 2 full integration demo
python demo/phase2_integration_demo.py

# Simple operator client demo
GOVERNANCE_BASE_DIR=/tmp/demo python demo/simple_operator_demo.py
```

## Production Deployment

### Environment Variables

- `GOVERNANCE_BASE_DIR` — base directory for governance state (default: `/var/lib/governance`)
- `GOVERNANCE_SIGNING_KEY_OPERATOR` — HMAC signing key for bus authentication (default: `CHANGE-ME-OPERATOR`)
- `ARBITER_URL` — Arbiter State API endpoint

### Docker Volumes

```yaml
services:
  operator:
    image: nonaxis-operator:latest
    volumes:
      - operator-workspace:/workspace
      - governance-pending:/var/lib/governance/pending-review:rw
      - governance-decisions:/var/lib/governance/decisions:ro
      - governance-tokens:/var/lib/governance/tokens:ro
      - governance-state:/var/lib/governance/state:ro
    environment:
      GOVERNANCE_BASE_DIR: /var/lib/governance
      ARBITER_URL: http://arbiter:8300
```

**Key security properties:**
- Operator has **read-only** access to decisions/tokens/state
- Operator has **write** access to pending-review (submit requests)
- Operator **cannot** write to decisions or tokens (Arbiter-only)

### AppArmor Profile

```
#include <tunables/global>

/usr/local/bin/nonaxis {
  #include <abstractions/base>
  
  # Workspace
  /workspace/** rw,
  
  # Governance bus
  /var/lib/governance/pending-review/** rw,
  /var/lib/governance/decisions/** r,
  /var/lib/governance/tokens/** r,
  
  # Network (governed)
  network inet stream,
  network inet6 stream,
  
  # Deny by default
  deny /etc/shadow r,
  deny /root/** rw,
  deny /** x,
}
```

## Next Steps

- **Phase 3**: Full Docker orchestration with 3 isolated instances
- **Phase 3**: Real CISO static analyzer (not simulated)
- **Phase 3**: Multi-instance message bus with Caddy routing
- **Phase 4**: nftables + AppArmor integration for token-based execution
- **Phase 4**: Human escalation via secure channel (Signal, encrypted email)

---

*Last updated: 2026-02-12 — Phase 2 complete*
