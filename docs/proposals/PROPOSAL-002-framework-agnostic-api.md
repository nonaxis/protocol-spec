# PROPOSAL-002: Framework-Agnostic Governance API

**Status:** Active — Phase 2 (required by ADR-011)
**Filed:** 2026-02-11
**Priority:** High — this determines whether we're a feature or infrastructure

---

## Problem

The governance architecture currently assumes Nonaxis as the agent framework. The Operator's interceptor, the skill format, and the SOUL.md-based configuration are all Nyx-specific. This limits the addressable market to one framework and makes us a plugin, not a platform.

Per ADR-011: "If this only works with Nonaxis, it's a feature. If it works with AutoGen, CrewAI, LangChain — it's infrastructure."

## Goal

Define a minimal API surface that any agent framework can implement to participate in governance. The API should be:
- Framework-agnostic (no Nonaxis assumptions)
- Transport-agnostic (HTTP, Unix socket, filesystem — adapter's choice)
- Simple enough that an integration takes hours, not weeks
- Complete enough that governance guarantees hold

## Core Abstraction: The Governance Contract

An agent framework participates in governance by implementing one interface:

```
Before executing any tool/action:
  1. CLASSIFY the action → get a tier
  2. SUBMIT the action for review → get a decision
  3. If approved, EXECUTE with the issued token
  4. REPORT the outcome
```

That's it. Four verbs.

## API Surface

### 1. `POST /governance/classify`

Agent asks: "What tier is this action?"

```json
// Request
{
  "action_type": "exec|file_write|network|install|config_modify|memory_edit",
  "command": "ls -la /tmp",
  "context": {
    "agent_id": "operator-1",
    "session_id": "uuid",
    "user_id": "optional",
    "framework": "nonaxis|autogen|langchain|crewai|custom",
    "source": "user_request|autonomous|triggered"
  }
}

// Response
{
  "tier": 0|1|2|3,
  "tier_name": "Passive|Active|Sensitive|Critical",
  "governance_required": "none|async|sync|sync_with_human",
  "classification_id": "uuid"
}
```

**Design decision:** Classification happens outside the agent. The agent submits what it wants to do; the governance layer decides how much scrutiny it gets. This prevents a compromised agent from downclassifying its own actions (the "fox guarding the henhouse" problem GPT flagged).

### 2. `POST /governance/submit`

Agent submits action for review.

```json
// Request
{
  "classification_id": "uuid",
  "action": {
    "type": "exec",
    "command": "ls -la /tmp",
    "working_dir": "/home/user/project",
    "env": {},
    "content_hash": "sha256:..."
  },
  "justification": "User requested dependency installation"
}

// Response (sync for tier 2+)
{
  "decision": "APPROVED|DENIED|ESCALATED|NEEDS_INFO",
  "decision_id": "uuid",
  "token": {  // only if approved
    "token_id": "uuid",
    "nonce": "uuid",
    "expires_at": "ISO-8601",
    "scope": {
      "content_hash": "sha256:...",
      "allowed_hosts": ["example.com"],
      "max_duration_seconds": 30
    },
    "hmac_signature": "hex"
  },
  "reason": "Policy rule POL-003: piped execution from external source requires review",
  "alternatives": [],  // suggested safe alternatives if denied
  "audit_ref": "chain-hash-ref"
}
```

### 3. `POST /governance/execute`

Agent presents token and content hash for execution verification.

```json
// Request
{
  "token_id": "uuid",
  "nonce": "uuid",
  "content_hash": "sha256:...",
  "hmac_signature": "hex"
}

// Response
{
  "verified": true|false,
  "execution_id": "uuid",
  "constraints": {
    "timeout_seconds": 30,
    "network_hosts": ["example.com"],
    "filesystem_paths": ["/home/user/project"]
  }
}
```

**Design decision:** The execution endpoint is verified by a privileged harness, not the agent. The agent presents its token; the harness verifies the HMAC, checks the nonce hasn't been used, confirms the content hash matches, and either executes the command in a constrained environment or returns the constraints for the agent to self-enforce (defense in depth).

### 4. `POST /governance/report`

Agent reports execution outcome.

```json
// Request
{
  "execution_id": "uuid",
  "outcome": "success|failure|timeout|aborted",
  "exit_code": 0,
  "output_summary": "truncated output",
  "duration_ms": 1234
}

// Response
{
  "logged": true,
  "audit_ref": "chain-hash-ref"
}
```

## Framework Adapters

Each agent framework needs a thin adapter that hooks into its tool execution pipeline:

### Nonaxis Adapter
Hook into the exec/tool pipeline. When Nonaxis calls a tool, intercept → classify → submit → execute with token → report.

### LangChain Adapter
Wrap `BaseTool.run()`. Before execution, call governance API. LangChain's `CallbackHandler` can also be used for async audit.

### AutoGen Adapter
Hook into `ConversableAgent.execute_function()`. AutoGen's existing `human_input_mode` maps naturally to our tier system.

### CrewAI Adapter
Wrap `Tool.__call__()`. CrewAI's task delegation model maps to our specialist routing.

### Generic / Custom
Any framework that can make HTTP calls before executing tools can integrate. The API is 4 endpoints.

## What This Means for the Repo

### Now (Phase 2)
- Define the API spec as an OpenAPI document (`specs/governance-api.yaml`)
- Refactor `src/governance/operator/interceptor.py` to implement the API internally
- The existing bus/orchestrator becomes the backend for these endpoints

### Phase 3
- Build the first non-Nonaxis adapter (LangChain or AutoGen)
- Publish the API spec as a standalone document
- The spec becomes the "protocol" that frameworks implement

### Phase 4+
- SDK libraries for common languages (Python first, TypeScript second)
- Certification program: "Governance Compatible" badge for frameworks

## Open Questions

1. **Sync vs async for Tier 1?** Currently async. Should the API support both and let the caller decide?
2. **Streaming actions?** Some tools produce long-running output. How does governance handle a command that runs for 10 minutes?
3. **Batch classification?** An agent planning 5 steps ahead — can it pre-classify all 5 at once?
4. **Multi-agent scenarios?** When Agent A delegates to Agent B, does the governance context transfer?

## Acceptance Criteria

- A LangChain agent can use the governance API with <100 lines of adapter code
- The API spec is complete enough to implement without reading our source code
- Token verification works identically regardless of which framework submitted the action
- The audit trail is framework-agnostic (you can't tell from the audit log which framework was governed)
