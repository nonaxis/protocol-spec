# Bus Protocol Specification

## Message Flow

```
Operator                    Bus (Caddy)                CISO                    Arbiter
   │                           │                        │                        │
   │──POST /review ──────────▶│──forward──────────────▶│                        │
   │                           │                        │ analyze code           │
   │                           │                        │ produce assessment     │
   │                           │◀──POST /assess────────│                        │
   │                           │──forward──────────────────────────────────────▶│
   │                           │                                                │ apply rules
   │                           │                                                │ (deterministic
   │                           │                                                │  or human escalation)
   │                           │◀─────────────────────────────POST /decision───│
   │◀──POST /governance/decision│                                               │
   │                           │                                                │
   │  (execute if approved)    │                                                │
```

## Endpoints

| Endpoint | Method | Source | Destination | Schema |
|----------|--------|--------|-------------|--------|
| `/review` | POST | Operator | CISO | review-request.schema.json |
| `/assess` | POST | CISO | Arbiter (via bus) | assessment.schema.json |
| `/decide` | POST | Bus | Arbiter | ArbiterInput (request + assessment) |
| `/governance/decision` | POST | Arbiter | Operator | decision.schema.json |
| `/governance/escalate-to-human` | POST | Arbiter | Operator | decision.schema.json (decision=escalate) |
| `/health` | GET | Any | Arbiter | Health check |
| `/reload-rules` | POST | Admin | Arbiter | Hot-reload policy rules |

## Schema Validation

The bus validates every message against its JSON schema before forwarding. Invalid messages are rejected with HTTP 400 and logged. This prevents:
- Malformed payloads designed to confuse downstream instances
- Missing required fields that could cause silent failures
- Type mismatches

## Timeouts

| Stage | Timeout | On Timeout |
|-------|---------|------------|
| CISO review | 60s | Escalate to human |
| Arbiter decision | 30s | Escalate to human |
| Human escalation (if no rule matches) | 30s | Escalate to human |
| Human escalation | 30min | Reject (default deny) |
| Execution token | 300s | Token expires, action blocked |

## Chain Hash Verification

Any consumer can verify the chain:
```python
for i, msg in enumerate(audit_log):
    if i == 0:
        assert msg["chain_hash"] == compute_hash("0" * 64, msg)
    else:
        assert msg["chain_hash"] == compute_hash(audit_log[i-1]["chain_hash"], msg)
```

A broken chain indicates tampering.
