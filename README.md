# Home Credit LAP Voice Agent

An AI voice agent that qualifies existing Home Credit customers for a pre-approved Loan Against Property (LAP) offer. Built as a single-prompt agent on Retell AI.

**Submitted by:** Sayendranadh
**Platform:** Retell AI (Single-Prompt Agent)
**Languages tested:** English (US) + Hindi

---

## Live Agent

Published Retell agent — test with a web call:

**[Open the agent](https://agent.retellai.com/orb/agent_f302dab1f1c31c6d0ee1482f42?token=bbdb647c854fa7d098c10dd9ebf6fc28)**

Or paste this URL in your browser:

https://agent.retellai.com/orb/agent_f302dab1f1c31c6d0ee1482f42?token=bbdb647c854fa7d098c10dd9ebf6fc28

---

## Call Flow

The agent moves through the following stages on every call. Only one terminal outcome fires per call.

| Step | Stage | Branches |
|---|---|---|
| 1 | Greeting and verification | Customer refuses → **DECLINED** |
| 2 | Busy check | Customer busy → **DEFERRED** (capture callback) |
| 3 | Present LAP offer | — |
| 4 | Transfer check | Existing loan or EMI reduction → **TRANSFERRED** |
| 5 | Eligibility checklist (Q1–Q7) | Hard rule fails → **DISQUALIFIED** |
| 6 | Loan amount check | Over ₹75L → counter-offer → accept (continue) or reject → **DECLINED** |
| 7 | Handoff gate | All 7 items pass → **QUALIFIED_HANDOFF** |

### Terminal Outcomes

| Outcome | Trigger | Message |
|---|---|---|
| QUALIFIED_HANDOFF | All 7 items filled, no disqualify, no transfer | Senior loan expert will call |
| DISQUALIFIED | Hard rule failed | Doesn't meet criteria for this offer |
| TRANSFERRED | Existing loan or EMI reduction | Loan-transfer specialist will call |
| DEFERRED | Customer busy | Callback scheduled |
| DECLINED | Refused, rejected counter-offer, wrong number | Polite close |

---

## What the Agent Does

1. Verifies the intended customer.
2. Handles busy deferrals — captures callback time and ends.
3. Presents the LAP offer as a loyalty reward (up to ₹75 lakh).
4. Detects existing loan / EMI-reduction cases and routes to a loan-transfer specialist.
5. Captures 7 eligibility data points in any order, without re-asking.
6. Applies eligibility rules:
   - **Hard disqualifiers:** agricultural property, no original documents, cash income, tenure outside 3–15 years
   - **Soft fail:** loan amount over ₹75L → counter-offer the cap
   - **Never fails:** ownership (sole or joint), market value
7. Hands off qualified leads to a senior loan expert.

---

## Repository Contents

| Path | Description |
|---|---|
| system-prompt.md | Complete system prompt |
| tests/test-matrix.md | 28 test scenarios |
| approach.md | Design rationale and build process |
| docs/rules-spec.md | Eligibility rule taxonomy |
| docs/state-model.md | State block definition |
| docs/decision-tree.md | Canonical call flow |
| transcripts_urls.csv | Test evidence — call links, transcripts, recordings |

---

## Test Evidence

transcripts_urls.csv contains one row per test call with three columns:

| Column | Content |
|---|---|
| Public Log URL | Shareable Retell call log link |
| Transcript | Full call transcript (English and Hindi) |
| Recording URL | Direct audio recording URL |

28 rows — one per test scenario (24 English + 4 Hindi).

---

## Testing Summary

| Batch | Scenarios | Passed |
|---|---|---|
| Baseline (English) | 12 | 12 |
| Adversarial (English) | 4 | 4 |
| Extended Edge Cases (English) | 8 | 8 |
| Hindi (language + gender) | 4 | 4 |
| **Total** | **28** | **28** |

Six prompt bugs were found and fixed during testing. See approach.md for the full list.

---

## Key Design Decisions

1. **State machine, not a script.** 14 variables (8 eligibility slots, 3 gate flags, 3 terminal flags) maintained silently across the call.

2. **Four-beat per-turn loop.** Extract first. Validate second. Update state third. Ask next fourth.

3. **Rule taxonomy.** Four hard disqualifiers, one soft counter-offer, two never-fail, two transfer triggers.

4. **Two distinct transfer branches.** Loan-transfer specialist (existing loans / EMI reduction) vs. senior loan expert (qualified fresh-loan leads).

5. **RAG-only factual sourcing.** No invented rates, thresholds, or branch locations.

6. **ASR tolerance.** Handles garbled speech by extracting from context.

---

## How to Deploy

1. Create a Single-Prompt Agent on Retell AI.
2. Paste system-prompt.md into the agent's prompt field.
3. Set dynamic variables: agent_name, company_name, customer_name, agent_gender, current_date, current_day, current_time, language_to_speak.
4. Add an end_call tool.
5. Configure post-call extraction fields (see docs/state-model.md).
6. Attach the knowledge base for additional_context_from_rag.
