# Roadmap

**Timeframe:** 90 days from 2026-02-11
**Decision point:** May 11, 2026 — continue, pivot, or open-source and move on.

---

## Phase 1: Foundation ✅ Complete

- Architecture design and threat model
- Three-instance topology (Operator, CISO, Arbiter)
- JSON schemas for bus protocol
- Arbiter: deterministic policy rules, token generation, audit logging
- CISO: static pattern analysis (30+ rules)
- Bus: schema validation, directory-based routing
- Operator: governance interceptor skill
- AppArmor and nftables sandbox profiles
- Docker Compose dev environment
- Canary test skills (exfiltrator, prompt injector)
- Chain-hashed audit trail
- Four-reviewer adversarial review cycle (GPT, Gemini, Opus, GPT-external)
- 10 Architecture Decision Records
- Full documentation suite

## Phase 2: Fix & Validate — In Progress

### P0 Fixes (must complete before demo)
- [x] `TOKEN_SECRET` default → fail-closed (refuse to start)
- [x] Persist chain hash state to disk (survive restarts)
- [x] Fix Docker Compose volume isolation (`:ro` mounts)
- [ ] Unify token format: Python generator + bash verifier must match
- [ ] Remove duplicate implementations (one canonical path, prototypes archived)
- [ ] Authenticate or remove `/reload-rules` endpoint
- [ ] First real end-to-end test on filesystem bus

### P1 Fixes (must complete before trusting with real data)
- [ ] Harden filesystem bus (O_NOFOLLOW, atomic writes, flock)
- [ ] Execution harness: re-verify content_hash at execution time
- [ ] Egress proxy for data exfiltration mitigation
- [ ] Human escalation notifications (Telegram)
- [ ] Test AppArmor profiles in enforcement mode

### Integration Testing
- [ ] End-to-end: skill install → CISO review → Arbiter decision → Operator response
- [ ] Escalation flow → human approval
- [ ] Failure injection: crash each instance, verify fail-closed
- [ ] TOCTOU test: modify skill between review and install

## Phase 3: Package & Validate (Weeks 3-8)

- [ ] 5-minute demo: show exploit, show system blocking it
- [ ] Blog post: problem-focused, not product-focused
- [ ] 20 conversations with potential users
- [ ] Move tier classification to Arbiter (currently in Operator — known issue)
- [ ] Evaluate OPA/Rego as Arbiter rule backend

## Phase 4: Decide (Weeks 9-12)

If 5+ humans want this → form entity, open-source reference architecture, build first integration.

If not → publish as blog series, open-source everything, move on.

---

_Informed by adversarial reviews from GPT-4o, Gemini, and Claude Opus. See `docs/REVIEW-SYNTHESIS-ALL-2026-02-11.md` for the full analysis._
