# ADR-010: Canary/Honeytoken Deployment for Compromise Detection

Date: 2026-02-11
Status: Accepted

## Context
Prevention-only security is insufficient. We need detection mechanisms that alert when the Operator has been compromised, even if the compromise bypasses all preventive controls.

## Decision
Deploy canary files (credential-looking files with unique tracking URLs) in the Operator's workspace. If any canary URL is ever accessed, the Operator is compromised. Additionally, deploy honeytokens in governance directories as tamper indicators.

## Rationale
Canary files are zero-cost, zero-false-positive detection. A legitimate Operator has no reason to access `passwords.txt` or exfiltrate its contents. Any access = compromise. This provides detection for the #1 threat (data exfiltration via LLM API) that we can't fully prevent.

## Evidence
- **Gemini-I:** Proposed canary/honeytoken deployment. "Implement immediately. Zero cost, immediate detection."
- **Gemini-E:** Re-recommended canaries (didn't realize they were already implemented — accuracy issue, but validates the approach).
- **GPT-E:** Found that canary files exist but lack documentation. "Credential-looking canary files in tree need loud documentation."

## Consequences
- Canary files must be prominently documented (so new developers don't mistake them for real credentials)
- Canary URL monitoring must be set up (currently: canary files exist, monitoring unclear)
- Honeytokens in governance dirs can detect unauthorized access to policy files
- Does not prevent exfiltration — only detects it after the fact
