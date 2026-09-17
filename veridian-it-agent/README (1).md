# Veridian Corp — Internal IT Service Agent

Tier-1 IT support agent for Veridian Corp, built on the Assignment 2 data pack (week of 21–25 Sep 2026).

**Live demo:** _paste your artifact link here_
**Run locally:** `git clone … && open veridian-it-agent.html` — single file, no build, no install, no keys.

---

## What it does

For any employee message it: classifies intent with a confidence score → retrieves the governing policy from a closed corpus → decides **resolve / clarify / escalate** → drafts the reply with inline citations → asks the follow-ups that actually unblock the case → writes a structured ticket → logs every step to an append-only audit trail (exportable as JSON/CSV).

All 15 employee requests and the 4 active tickets are pre-loaded. There is also a free-text box for a request that isn't in the pack.

## Architecture

```
Intake ─► Classifier ─► Retriever ─► Policy engine ─► Ticket writer
        (intent+conf)  (KB-01…10,   (decision, risk,   (structured JSON,
                        AMP-01)      owner, SLA)        SLA, owner)
                           │              │
                           ▼              ▼
                      Guardrails     LLM drafter
                 (no source→no answer, (tone only — cannot change
                  conflict/gap/breach   the decision or citations)
                  detection)
                           │
                           ▼
              Append-only audit trail (every event, every citation)
```

**Hybrid by design.** Classification, decision, risk and routing sit in a deterministic rules layer so two identical tickets always get the same outcome and every outcome traces back to a policy line. The LLM (Claude Sonnet, called from inside the running app) only drafts employee-facing language, and is handed just the retrieved policy text plus the decision already made. If the model is unavailable the app falls back to deterministic templates — the UI labels each reply *LLM drafted* or *template*.

## Judgement calls this data set is really testing

| Case | What a naive agent does | What this agent does |
|---|---|---|
| REQ-01 laptop dead, 3.5 yrs | Approves replacement (KB-03: 3 yrs) | Flags the conflict with AMP-01 (4-yr cycle + Finance sign-off), cites TK-1043 precedent, escalates |
| REQ-08 phishing | Escalates to Security | Escalates **and** tells the employee to stop forwarding — KB-09 breach in progress — and asks who already received it |
| REQ-10 admin access | Invents an approval path | Reports a knowledge gap (no KB covers privileged access), matches the TK-1050 rejection precedent, routes to a human |
| REQ-15 "its not working" | Guesses | Confidence 0.34 → refuses to classify, asks scoping questions |
| REQ-12 expense tool | Troubleshoots the login | Checks ownership first: account creation is Finance (KB-08), IT only owns login faults on an existing account |
| REQ-13 flickering screen, 2 yrs | Replaces the laptop | Repair path — fails both refresh windows |
| REQ-02 guest Wi-Fi | Raises a ticket | Deflects to self-service kiosk — KB-07 says no ticket required |

## Inputs, sources, assumptions

**Sources:** 10 KB articles, the Asset Management Policy extract, 15 employee requests, 10 ticket rows. Nothing else — no external lookups, no invented policy.

**Assumptions**
1. Where KB-03 and AMP-01 disagree, neither automatically wins; the agent flags it for Finance and notes AMP-01 is newer (Q2 2026) and Finance-owned.
2. Section 2 requesters are full-time employees unless stated (REQ-11 is explicitly a contractor).
3. P1–P4 priorities and SLA windows are agent-side conventions; the only SLA taken from the pack verbatim is the 3–5 business day Security review.
4. Closed tickets are read-only precedent; active tickets run through the same engine as new requests.
5. The agent has no write access to Finance, Security or manager systems — those are handoffs, never auto-approvals.

## AI tools used

| Tool | Where | How |
|---|---|---|
| Claude Opus | Build time | Converted the data pack into the rule set and escalation matrix, wrote the prototype, derived the edge cases above |
| Claude Sonnet (quick tier) | Runtime, in-app | Drafts each employee reply from the retrieved policy + the fixed decision. Cannot alter decision, routing or citations |
| Deliberately *not* an LLM | — | Classification, decision, risk, routing — kept deterministic so outcomes are reproducible and auditable |

## Limits / next steps

- Keyword classification; a far-off paraphrase lands in the low-confidence clarify path (safe but blunt). Fix: embedding retrieval over the same corpus, guardrails unchanged.
- Ticket writes are in-memory; the emitted JSON is the real payload shape for a ServiceNow/Jira create call.
- No identity verification beyond a follow-up question; no live ITSM or mail integration.

---

## 10-slide deck script

1. **The problem** — Tier-1 IT burns time on requests that policy already answers, and escalates the ones it shouldn't. 15 requests, 10 tickets, 11 policy docs, one week.
2. **What I built** — live agent, one link, no install. 30-second walkthrough of the queue.
3. **Architecture** — the diagram above; where deterministic ends and the LLM begins.
4. **Grounding** — closed corpus of 11 documents; every answer carries its citation; no source means no answer.
5. **Decision policy** — resolve / clarify / escalate, and the four triggers that force escalation (other function owns it, policies conflict, no policy exists, risk is irreversible).
6. **Edge case: the policy conflict** — REQ-01, KB-03 vs AMP-01, plus the TK-1043 precedent.
7. **Edge case: the live breach** — REQ-08, containment before escalation.
8. **Edge case: the knowledge gap** — REQ-10, and why "urgent" is not a justification; consistency with TK-1050.
9. **Audit trail & ticket schema** — the exported log, the JSON payload, what a reviewer can prove after the fact.
10. **Limits and what ships next** — embedding retrieval, real ITSM write, identity verification, confidence-threshold tuning from resolution data.
