# Operator Client Integration Guide

**Audience:** Developers integrating the Nonaxis governance pipeline into AI agents (Nyx, etc.)

---

## Quick Start

### Installation

```python
# Add src/ to Python path
import sys
from pathlib import Path
sys.path.insert(0, str(Path("/path/to/nonaxis/src")))

from governance.operator.client import OperatorClient
from governance.config import GovernanceSettings
```

### Basic Usage

```python
# Initialize client
settings = GovernanceSettings()  # Uses env vars or defaults
client = OperatorClient(
    settings=settings,
    arbiter_url="http://localhost:8300"  # Arbiter state API endpoint
)

# Session boot: fetch canonical state
boot_data = client.boot()
snapshot = boot_data["snapshot"]      # Current state
ledger = boot_data["ledger"]          # Decision history
last_handoff = boot_data["last_handoff"]

# Submit governed action for review
request_id = client.submit_review(
    action_type="exec_unfamiliar",
    content=b"#!/bin/bash\ncurl https://example.com/api",
    outbound_hosts=["example.com"],
    context="Health check for external API"
)

# Wait for decision
decision = client.poll_decision(request_id, timeout=30.0)
if decision["payload"]["decision"] == "approve":
    # Get execution token
    token = client.get_token(request_id)
    
    # Validate token before executing
    if client.apply_token(token, content):
        # Execute the governed action
        execute_action()

# Session handoff at end
client.submit_handoff(
    snapshot_id=snapshot["snapshot_id"],
    summary=["Completed health check", "No issues detected"],
    task_queue=[{"task": "Schedule recurring check", "priority": "medium"}]
)
```

---

## API Reference

### OperatorClient Constructor

```python
OperatorClient(
    settings: Optional[GovernanceSettings] = None,
    arbiter_url: str = "http://localhost:8300"
)
```

**Parameters:**
- `settings`: Governance settings (defaults to environment variables)
- `arbiter_url`: Base URL for Arbiter state API

**Environment Variables:**
- `GOVERNANCE_BASE_DIR`: Root directory for filesystem bus (default: `/var/lib/governance`)
- `GOVERNANCE_SIGNING_KEY_OPERATOR`: HMAC signing key (default: `CHANGE-ME-OPERATOR`)

---

### Session Boot

```python
boot_data = client.boot() -> dict
```

**Returns:**
```python
{
    "snapshot": {
        "snapshot_id": "uuid",
        "state_hash": "sha256",
        "timestamp": "2026-02-12T15:30:00Z",
        "identity": {...},
        "active_objectives": [...],
        "constraints": [...],
        # ... full canonical state
    },
    "ledger": [
        {
            "decision_id": "D-2026-02-12-01",
            "statement": "...",
            "rationale": [...],
            "tags": ["always_include"],
            # ...
        }
    ],
    "recent_events": [...],
    "last_handoff": {...},
    "consistency_checks": {
        "constraint_check": "pass|warning|error",
        "drift_check": "pass|warning",
        "details": [...]
    }
}
```

**Use Case:** Call at session start to load canonical state + decision history.

**Fallback:** Returns empty state with error details if Arbiter unavailable.

---

### Submit Review Request

```python
request_id = client.submit_review(
    action_type: str,
    content: bytes,
    *,
    outbound_hosts: Optional[list[str]] = None,
    context: str = ""
) -> str
```

**Parameters:**
- `action_type`: Type of governed action (see [Action Types](#action-types))
- `content`: Raw content bytes (code, command, config file, etc.)
- `outbound_hosts`: List of outbound network destinations (e.g., `["api.example.com"]`)
- `context`: Brief explanation of why this action is needed

**Returns:** Request ID (UUID) for polling

**Side Effects:** Writes review request to `{base_dir}/pending-review/{request_id}.json`

**Action Types:**
- `exec_unfamiliar` — Execute command not in allowlist (Tier 2)
- `outbound_network` — Connect to new external host (Tier 2)
- `skill_install` — Install agent skill/plugin (Tier 3)
- `skill_update` — Update existing skill (Tier 3)
- `credential_access` — Read credentials from vault (Tier 3)
- `self_modify` — Modify agent code/config (Tier 3)
- `memory_modify` — Edit governed memory (Tier 2)
- `prompt_modify` — Change system prompt (Tier 3)

**Risk Tiers:**
- **Tier 2:** Synchronous CISO review → Arbiter decision
- **Tier 3:** Synchronous review + human confirmation required

---

### Poll for Decision

```python
decision = client.poll_decision(
    request_id: str,
    timeout: float = 30.0
) -> Optional[dict]
```

**Parameters:**
- `request_id`: Review request ID from `submit_review()`
- `timeout`: Maximum wait time in seconds

**Returns:**
```python
{
    "id": "uuid",
    "timestamp": "2026-02-12T15:30:00Z",
    "type": "decision",
    "source": "arbiter",
    "payload": {
        "review_request_id": "uuid",
        "decision": "approve|reject|escalate",
        "reasoning": "Token issued for 5min TTL with exec+network permissions",
        "policy_rules_applied": ["TIER2_DEFAULT_APPROVE"],
        "escalation": {  # Only present if decision == "escalate"
            "context_for_human": "...",
            "timeout_minutes": 30
        }
    },
    "chain_hash": "sha256"
}
```

**Returns `None`:** If timeout expires before decision arrives

**Blocking:** Polls filesystem every 500ms until decision file appears or timeout

---

### Get Execution Token

```python
token = client.get_token(request_id: str) -> Optional[dict]
```

**Parameters:**
- `request_id`: Review request ID

**Returns:**
```python
{
    "token_id": "uuid",
    "expires_at": "2026-02-12T15:35:00Z",  # Typically +5min from issuance
    "scope": {
        "action_type": "exec_unfamiliar",
        "content_hash": "sha256",
        "allowed_hosts": ["api.example.com"],
        "permissions": ["exec", "network"]
    },
    "signature": "hmac_sha256"
}
```

**Returns `None`:** If no token exists for this request (decision was reject/escalate)

**Token Location:** `{base_dir}/tokens/{request_id}.json`

---

### Apply/Validate Token

```python
valid = client.apply_token(
    token: dict,
    action_content: bytes
) -> bool
```

**Parameters:**
- `token`: Token dict from `get_token()`
- `action_content`: The exact content bytes that will be executed

**Returns:** `True` if token is valid and action can proceed, `False` otherwise

**Validation Checks:**
1. Content hash matches (SHA-256 of `action_content`)
2. Token has not expired (current time < `expires_at`)
3. Token signature is valid (placeholder in current impl)

**Production Note:** In real deployment, this also triggers nftables rule updates to allow temporary network access to `allowed_hosts`.

---

### Submit Session Handoff

```python
handoff_id = client.submit_handoff(
    snapshot_id: str,
    summary: list[str],
    **kwargs
) -> str
```

**Parameters:**
- `snapshot_id`: Current canonical state snapshot ID (from `boot()`)
- `summary`: List of brief statements about what happened this session
- `assumptions_made`: List of dicts with `{"assumption": str, "confidence": str}`
- `unknowns`: List of dicts with `{"question": str, "context": str}`
- `task_queue`: List of dicts with `{"task": str, "priority": str, "est_effort": str}`
- `state_updates_proposed`: List of dicts with state change proposals
- `risks_noted`: List of dicts with `{"risk": str, "severity": str}`

**Returns:** Handoff ID (UUID)

**Use Case:** Call at session end to create continuity artifact for next session.

**Side Effects:** 
- POST to Arbiter `/handoff` endpoint
- Arbiter writes handoff file to `{state_dir}/handoffs/`

---

### Propose State Patch

```python
proposal_id = client.propose_state_patch(
    snapshot_id: str,
    patches: list[dict],
    rationale: str = ""
) -> str
```

**Parameters:**
- `snapshot_id`: Base snapshot ID (from `boot()`)
- `patches`: List of [JSON Patch (RFC 6902)](https://tools.ietf.org/html/rfc6902) operations
- `rationale`: Explanation for the change

**Example Patch:**
```python
patches = [
    {
        "op": "add",
        "path": "/active_objectives/-",
        "value": {"objective": "Integrate with GitHub API", "priority": "high"}
    },
    {
        "op": "replace",
        "path": "/constraints/0/enforcement",
        "value": "strict"
    }
]
```

**Returns:** Proposal ID (UUID)

**Governance:** State patches require Arbiter review. Use for updating canonical state (objectives, constraints, approved skills, etc.).

---

### Convenience: Submit and Wait

```python
decision, token = client.submit_and_wait(
    action_type: str,
    content: bytes,
    *,
    outbound_hosts: Optional[list[str]] = None,
    context: str = "",
    timeout: float = 30.0
) -> tuple[Optional[dict], Optional[dict]]
```

**Returns:** `(decision, token)` tuple
- Both `None` if timeout
- `token` is `None` if decision was reject/escalate

**Use Case:** One-shot submission when you want to block until decision + token.

---

## Integration Patterns

### Pattern 1: Pre-Flight Check

Before executing any Tier 2+ action, submit for review:

```python
# Agent wants to curl external API
command = b"curl -s https://api.example.com/data | jq .status"

# Submit for review
decision, token = client.submit_and_wait(
    action_type="exec_unfamiliar",
    content=command,
    outbound_hosts=["api.example.com"],
    context="Fetch status from monitoring API",
    timeout=30.0
)

if decision is None:
    # Timeout — escalate to human
    log_error("Governance timeout — manual intervention needed")
    return

if decision["payload"]["decision"] == "approve":
    if client.apply_token(token, command):
        # Execute with token
        result = execute_command(command)
    else:
        log_error("Token validation failed")
elif decision["payload"]["decision"] == "reject":
    # Blocked by governance
    log_info(f"Action rejected: {decision['payload']['reasoning']}")
elif decision["payload"]["decision"] == "escalate":
    # Human confirmation required
    escalation = decision["payload"]["escalation"]
    notify_human(escalation["context_for_human"])
    # Wait for human response...
```

---

### Pattern 2: Session Lifecycle

```python
# Session start
def on_session_start():
    boot_data = client.boot()
    load_canonical_state(boot_data["snapshot"])
    apply_decision_history(boot_data["ledger"])
    
    # Check for consistency issues
    checks = boot_data["consistency_checks"]
    if checks["drift_check"] == "warning":
        log_warning(f"State drift detected: {checks['details']}")

# During session
def execute_governed_action(action_type, content, context):
    request_id = client.submit_review(action_type, content, context=context)
    decision = client.poll_decision(request_id, timeout=30.0)
    
    if decision and decision["payload"]["decision"] == "approve":
        token = client.get_token(request_id)
        if client.apply_token(token, content):
            return execute(content)
    
    return None  # Blocked or timeout

# Session end
def on_session_end(snapshot_id):
    summary = generate_session_summary()
    tasks = get_pending_tasks()
    risks = get_noted_risks()
    
    handoff_id = client.submit_handoff(
        snapshot_id=snapshot_id,
        summary=summary,
        task_queue=tasks,
        risks_noted=risks
    )
    log_info(f"Handoff submitted: {handoff_id}")
```

---

### Pattern 3: State Update Workflow

```python
# Propose a change to canonical state
def add_approved_skill(skill_name: str, capabilities: list[str]):
    boot_data = client.boot()
    snapshot_id = boot_data["snapshot"]["snapshot_id"]
    
    # Build JSON Patch
    patches = [
        {
            "op": "add",
            "path": "/approved_skills/-",
            "value": {
                "skill_name": skill_name,
                "capabilities": capabilities,
                "approved_at": datetime.now(timezone.utc).isoformat()
            }
        }
    ]
    
    # Submit proposal
    proposal_id = client.propose_state_patch(
        snapshot_id=snapshot_id,
        patches=patches,
        rationale=f"Adding {skill_name} skill after Tier 3 governance approval"
    )
    
    # Wait for Arbiter to review/commit
    # (In production, this would trigger another governance round)
    return proposal_id
```

---

## Error Handling

### Graceful Degradation

All client methods handle Arbiter unavailability gracefully:

```python
# Boot returns empty state if Arbiter down
boot_data = client.boot()
if boot_data["snapshot"] is None:
    # Fallback to local cache or fail-safe defaults
    snapshot = load_local_snapshot_cache()
    log_warning("Arbiter unavailable — using cached state")

# Handoff returns "error" on failure
handoff_id = client.submit_handoff(...)
if handoff_id == "error":
    # Log locally, retry later
    queue_handoff_for_retry()
```

### Timeout Handling

```python
# Always set reasonable timeouts
decision = client.poll_decision(request_id, timeout=30.0)
if decision is None:
    # Escalate to human — governance system may be down
    alert_ops_team("Governance pipeline timeout")
    # Fail closed — do NOT execute without approval
    return
```

### Chain Hash Verification

```python
# Client automatically maintains chain hash continuity
request1 = client.submit_review(...)  # Uses previous chain hash
request2 = client.submit_review(...)  # Chains to request1

# Verify chain integrity (optional)
from governance.audit.chain import verify_chain_integrity
audit_log = settings.base_dir / "audit" / "audit.jsonl"
if not verify_chain_integrity(audit_log):
    log_critical("Audit chain integrity compromised!")
```

---

## Configuration

### Environment Variables

```bash
# Required
export GOVERNANCE_BASE_DIR=/var/lib/governance
export GOVERNANCE_SIGNING_KEY_OPERATOR=<strong-random-secret>
export ARBITER_TOKEN_SECRET=<strong-random-secret>

# Optional
export GOVERNANCE_AUDIT_LOG_PATH=/var/log/governance/audit.jsonl
export GOVERNANCE_POLL_INTERVAL=2.0
export ARBITER_HOST=0.0.0.0
export ARBITER_PORT=8300
```

### Programmatic Configuration

```python
from governance.config import GovernanceSettings, ArbiterSettings
from pydantic import SecretStr

# Override defaults
settings = GovernanceSettings(
    base_dir=Path("/custom/path"),
    signing_key_operator=SecretStr("my-strong-secret"),
    poll_interval=1.0
)

client = OperatorClient(settings=settings)
```

---

## Testing

### Unit Tests (Offline)

```python
# Test without running services
import tempfile
from pathlib import Path

def test_operator_client():
    with tempfile.TemporaryDirectory() as tmpdir:
        settings = GovernanceSettings(base_dir=Path(tmpdir))
        client = OperatorClient(settings=settings)
        
        # Boot will fail gracefully
        boot = client.boot()
        assert boot["snapshot"] is None
        
        # Submit writes file
        request_id = client.submit_review(
            action_type="exec_unfamiliar",
            content=b"echo test"
        )
        
        request_file = settings.base_dir / "pending-review" / f"{request_id}.json"
        assert request_file.exists()
```

### Integration Tests (With Services)

```bash
# Terminal 1: Start services
$ bash demo/start_services.sh

# Terminal 2: Run demo
$ python3 demo/run_demo.py

# Or run validation suite
$ python3 demo/validate_phase2.py
```

---

## Troubleshooting

### "Connection refused" errors

**Cause:** Arbiter state API not running

**Fix:**
```bash
# Check if Arbiter is running
$ curl http://localhost:8300/docs

# If not, start it
$ python3 -m governance.arbiter.engine
```

### Review requests not moving through pipeline

**Cause:** Orchestrator not running

**Fix:**
```bash
# Start orchestrator
$ python3 -m governance.bus.orchestrator

# Check logs
$ tail -f $GOVERNANCE_BASE_DIR/orchestrator.log
```

### Token validation fails

**Causes:**
1. Token expired (default TTL: 5 minutes)
2. Content hash mismatch (content changed after approval)
3. Clock skew between instances

**Fix:**
```python
# Check token expiry
token = client.get_token(request_id)
print(f"Expires: {token['expires_at']}")

# Verify content hash
import hashlib
content_hash = hashlib.sha256(content).hexdigest()
print(f"Token scope: {token['scope']['content_hash']}")
print(f"Actual hash: {content_hash}")
```

### State drift warnings

**Cause:** Canonical state changed between handoff and next boot

**Expected:** Normal if multiple sessions or state patches committed

**Action:** Review drift details in `consistency_checks["details"]`

---

## Production Checklist

Before deploying Operator client integration:

- [ ] Change `GOVERNANCE_SIGNING_KEY_OPERATOR` from default
- [ ] Change `ARBITER_TOKEN_SECRET` from default
- [ ] Set up monitoring for governance timeouts
- [ ] Configure log rotation for audit logs
- [ ] Test fail-closed behavior (Arbiter down → actions blocked)
- [ ] Verify nftables integration (if using OS-level enforcement)
- [ ] Set up alerts for escalation events
- [ ] Document human confirmation workflow for Tier 3
- [ ] Test disaster recovery (corrupt canonical state)
- [ ] Load test concurrent review submissions

---

## FAQ

**Q: What happens if the Arbiter is down?**
A: The client fails closed — actions are blocked. Boot returns empty state, decisions timeout, handoffs fail. This is intentional (governance must be available for Tier 2+ actions).

**Q: Can I cache decisions locally?**
A: Yes, but be cautious. Decisions are content-hash-bound — a cached approval for `script_v1.sh` does NOT authorize `script_v2.sh`.

**Q: How do I handle Tier 3 human confirmations?**
A: When `decision["payload"]["decision"] == "escalate"`, notify human with `escalation["context_for_human"]`, wait for their approval, then retry with their confirmation token.

**Q: What if canonical state becomes corrupted?**
A: The Arbiter maintains snapshots with `prev_snapshot_id` links. Roll back to last known-good snapshot, investigate drift, re-apply patches manually.

**Q: Can multiple Operators share one Arbiter?**
A: Yes. Each Operator has its own `session_id` and writes to separate handoff files. The filesystem bus is designed for concurrent access.

**Q: How do I rotate signing keys?**
A: Update `GOVERNANCE_SIGNING_KEY_OPERATOR`, restart Operator instance. Old signatures remain valid (no retroactive verification). Consider versioned keys if you need invalidation.

---

## See Also

- [Architecture Deep Dive](../ARCHITECTURE.md) — Three-instance design
- [State API Reference](../specs/governance-api.yaml) — OpenAPI spec
- [Policy Rules Guide](../config/README.md) — Writing Arbiter policy
- [Token Format Spec](../docs/TOKEN-FORMAT.md) — Execution token structure

---

**Version:** 0.1.0 (Phase 2)  
**Last Updated:** 2026-02-12  
**Status:** Production-ready for PoC

