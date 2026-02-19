# ADR-006: Token-Gated Execution with HMAC + Nonce + Content Hash

Date: 2026-02-11
Status: Accepted

## Context
When the Arbiter approves an action, the Operator needs a verifiable proof of approval that is tamper-proof, replay-resistant, scope-limited, and time-bounded.

## Decision
Tokens are HMAC-signed, include a nonce for replay protection, a content_hash pinning the approved action, a TTL for time-bounding, and scope fields limiting what hosts/actions are authorized.

## Canonical Token Format (P0)

**Source of truth:** `schemas/execution-token.schema.json` and `specs/governance-api.yaml#/components/schemas/ExecutionToken`.

### Canonical HMAC string

To prevent Python/Bash mismatch, the HMAC input MUST be computed over this exact canonical string:

```
token_id:nonce:expires_at:scope_json
```

Where `scope_json` is the JSON-serialized scope object with **keys sorted alphabetically** (`json.dumps(scope, sort_keys=True)`).

Rules:
- `expires_at` is RFC3339 UTC (e.g., `2026-02-14T18:00:00+00:00`)
- `scope_json` contains: `action_type`, `allowed_hosts`, `content_hash` (and any other scope fields), serialized with `sort_keys=True` and no extra whitespace
- In Python: `f"{token_id}:{nonce}:{expires_at}:{json.dumps(scope.model_dump(), sort_keys=True)}"`
- In Bash/other languages: replicate the same JSON key ordering and separator format (`, ` between items, `: ` between key-value pairs)

### Test vector (for cross-language parity)

| Field | Value |
|---|---|
| Secret (ASCII) | `test_hmac_secret` |
| token_id | `11111111-1111-1111-1111-111111111111` |
| nonce | `22222222-2222-2222-2222-222222222222` |
| expires_at | `2026-02-14T18:00:00+00:00` |
| scope | `{"action_type": "exec_unfamiliar", "allowed_hosts": ["api.github.com"], "content_hash": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"}` |

Canonical string:
```
11111111-1111-1111-1111-111111111111:22222222-2222-2222-2222-222222222222:2026-02-14T18:00:00+00:00:{"action_type": "exec_unfamiliar", "allowed_hosts": ["api.github.com"], "content_hash": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"}
```

Expected HMAC-SHA256 (hex): `13d08030e37fa1610d9ae3172ee342b460dd01e9371a4664bd1d86ae998690a9`

This directly resolves the "Python generator vs bash verifier" mismatch by making the canonical string + test vector explicit.

## Rationale
HMAC prevents forgery without a shared secret. Nonces prevent replay within the TTL window. Content hash ensures the Operator executes exactly what was approved (not a modified version). TTL limits the blast radius of a leaked token.

## Evidence
- **Gemini-I:** Identified replay risk without nonces. Nonce field was implemented.
- **Gemini-E:** Recommended content_hash verification at execution time, not just approval time.
- **Opus:** "Token design is solid. HMAC-signed, time-limited, scope-pinned to content_hash, nonce for replay protection."
- **GPT-E (critical bug):** Python token generator doesn't include nonce; bash verifier expects it. Tokens would be rejected in practice. **This must be fixed — the token system is broken at the integration boundary.**
- **Opus:** `TOKEN_SECRET = "CHANGE-ME-IN-PRODUCTION"` default should fail-closed.

## Consequences
- Must unify token format across Python generator and bash verifier (P0 fix)
- Must fail-closed on default TOKEN_SECRET value
- Execution harness must re-verify content_hash at execution time (not just approval time)
- Token nonce log must persist across restarts
