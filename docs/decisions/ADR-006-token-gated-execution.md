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
token_id:nonce:expires_at:content_hash:allowed_hosts_csv:allowed_paths_csv:max_duration_seconds
```

Rules:
- `expires_at` is RFC3339 UTC (e.g., `2026-02-14T18:00:00Z`)
- `allowed_hosts_csv` is hosts joined by `,` in list order (empty string if omitted)
- `allowed_paths_csv` is paths joined by `,` in list order (empty string if omitted)
- `max_duration_seconds` is an integer, or empty string if omitted

### Test vector (for cross-language parity)

| Field | Value |
|---|---|
| Secret (ASCII) | `test_hmac_secret` |
| token_id | `11111111-1111-1111-1111-111111111111` |
| nonce | `22222222-2222-2222-2222-222222222222` |
| expires_at | `2026-02-14T18:00:00Z` |
| content_hash | `sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa` |
| allowed_hosts | `["api.github.com"]` |
| allowed_paths | `["/tmp"]` |
| max_duration_seconds | `60` |

Canonical string:
```
11111111-1111-1111-1111-111111111111:22222222-2222-2222-2222-222222222222:2026-02-14T18:00:00Z:sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa:api.github.com:/tmp:60
```

Expected HMAC-SHA256 (hex): `a0bea13e1887bdcff8f05e446c848a4a381e2b8d6a26ebcf20cf9924bb759501`

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
