# Threat Model

Mapped to OWASP Top 10 for Agentic Applications 2026 (ASI01-ASI10).

## Threat Actors

1. **Malicious skill author** — publishes backdoored skills to ClawHub
2. **Prompt injection via content** — malicious instructions in web pages, emails, documents
3. **Supply chain compromise** — legitimate skill updated with malicious payload
4. **Insider (compromised Operator)** — the Operator instance is jailbroken or manipulated

## OWASP ASI Threat Mapping

### ASI01: Agent Goal Hijack (CRITICAL)
**Scenario:** Prompt injection alters the Operator's objectives, causing it to execute unauthorized actions.

**Mitigations:**
- CISO scans all code for prompt injection patterns (5 static patterns)
- Operator SOUL.md mandates governance submission for Tier 2+
- OS-level sandbox (AppArmor + nftables) limits blast radius even if hijacked
- Arbiter uses deterministic rules — cannot be prompt-injected

**Residual risk:** Operator can act within Tier 0/1 autonomy before the bus catches it. Post-hoc CISO audit mitigates.

### ASI02: Tool Misuse and Exploitation (HIGH)
**Scenario:** Agent uses exec, network, or filesystem tools in unsafe ways.

**Mitigations:**
- Deterministic risk tier classification catches dangerous patterns
- AppArmor blocks dangerous exec patterns (nc, socat, etc.)
- nftables blocks outbound to unknown hosts without token
- CISO reviews all Tier 2+ tool usage

### ASI03: Identity and Privilege Abuse (HIGH)
**Scenario:** Credential reuse or escalation across agent instances.

**Mitigations:**
- Each instance runs in its own container with unique credentials
- Execution tokens are HMAC-signed and scope-pinned
- Credential access is always Tier 3 (human confirmation)

**Residual risk:** Shared filesystem bus — compromised host could read tokens.

### ASI04: Supply Chain Vulnerabilities (HIGH)
**Scenario:** Compromised plugins, skills, or MCP servers.

**Mitigations:**
- All skill installs go through full CISO review + Arbiter decision
- Content hash pinned at review time (TOCTOU protection)
- Skill install always Tier 3 (human confirmation required)

**Residual risk:** No reputation tracking for skill authors.

### ASI05: Unexpected Code Execution (MEDIUM)
**Scenario:** Agent generates or runs unsafe code.

**Mitigations:**
- AppArmor restricts what can execute
- RCE pattern scanning catches curl|bash, eval, etc.
- Unfamiliar exec always Tier 2+

### ASI06: Memory and Context Poisoning (MEDIUM)
**Scenario:** Attacker modifies agent memory or RAG sources.

**Mitigations:**
- Memory modification is Tier 2 (sync review)
- Prompt/system-prompt modification is Tier 3
- Chain hash integrity on audit trail

**Residual risk:** Workspace file writes at Tier 1 are post-hoc audited only.

### ASI07: Insecure Inter-Agent Communication (MEDIUM)
**Scenario:** Message tampering between governance instances.

**Mitigations:**
- JSON schema validation on all messages
- Chain hashing for tamper evidence
- Deterministic routing (not LLM-decided)
- Invalid messages quarantined in dead letter queue

**Residual risk:** Filesystem bus lacks TLS. Per-instance chains don't detect cross-stream deletion.

### ASI08: Cascading Failures (MEDIUM)
**Scenario:** Error in one instance propagates across the system.

**Mitigations:**
- Fail-closed: any component failure blocks Tier 2+ actions
- Timeout escalation on all stages
- CISO produces fail-safe assessment (risk_score=100) on analysis error
- Instance isolation prevents blast radius

### ASI09: Human-Agent Trust Exploitation (LOW-MEDIUM)
**Scenario:** Agent manipulates human reviewers during escalation.

**Mitigations:**
- Transparency requirement in Operator SOUL.md
- Human confirmation required for all Tier 3 actions
- Escalation auto-rejects after timeout (fail-closed)
- Full audit trail for post-hoc review

### ASI10: Rogue Agents (LOW-MEDIUM)
**Scenario:** Compromised agent acts harmfully while appearing legitimate.

**Mitigations:**
- Model diversity reduces correlated failure probability
- Regular canary testing validates detection capability
- CISO and Arbiter are powerless by design (can't act on the world)
- OS-level sandbox limits what even a rogue Operator can do

## Accepted Risks

- **LLM API as data channel:** Operator could encode exfiltrated data in API prompts. Requires content inspection (not in scope).
- **Slow poisoning:** Many benign skills → one malicious update. No reputation system yet.
- **DoS via review flooding:** Rate limiting planned but not implemented.
