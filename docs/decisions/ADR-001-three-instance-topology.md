# ADR-001: Three-Instance Topology (Operator/CISO/Arbiter)

Date: 2026-02-11
Status: Accepted

## Context
AI agent governance requires separating the entity being governed from the entity doing the governing. Single-instance "self-governance" (where an agent checks its own work) is fundamentally untrustworthy — a compromised agent can compromise its own governance checks. Existing approaches (AutoGen supervisor, NeMo Guardrails) run governance in the same process/memory space as the agent.

## Decision
Run three separate agent instances with distinct roles, separate processes, separate LLM contexts, and kernel-level isolation between them:

- **Operator:** Acts. Executes skills, interacts with the world. The governed entity.
- **CISO:** Analyzes. Reviews proposed actions for risk. Advisory only.
- **Arbiter:** Decides. Deterministic policy engine. The enforcement layer.

## Rationale
The key insight is that even if the CISO or Arbiter are fully compromised, they can only produce bad *advice* or *decisions* — they have no execution capabilities. The Operator can act but cannot approve its own actions. This separation makes single-point compromise insufficient for a full breach.

GPT-I called this "technically justified, operationally heavy." We accept the operational cost because single-instance governance is not governance at all.

## Evidence
- **GPT-I:** "Your real boundaries are: OS sandbox, deterministic Arbiter, file permission hygiene."
- **Gemini-I:** Validated topology as correct. "You are building PAM for AI."
- **Opus:** "Separation of concerns is correct... the key architectural insight and it's well-executed."
- **Competitive landscape (Opus):** No other project does multi-instance governance with kernel-level isolation.

## Consequences
- Higher operational complexity (three processes, three LLM contexts, inter-instance communication)
- Requires a bus transport mechanism (see ADR-003)
- Latency overhead for governed actions (~3-6 seconds per governance loop)
- LLM cost per governed action (CISO analysis)
- But: genuine security separation, not honor-system governance
