# Home Credit LAP Voice Agent

An AI voice agent that qualifies existing Home Credit customers for
a pre-approved Loan Against Property (LAP) offer. Built as a
single-prompt agent on **Retell AI**.

**Submitted by:** [Your name]
**Date:** [Submission date]
**Platform:** Retell AI (Single-Prompt Agent)
**Model:** GPT-5.6 Terra
**Languages tested:** English (US) + Hindi

---

## What the Agent Does

1. **Verifies** the intended customer.
2. **Handles busy deferrals** — captures callback time and ends.
3. **Presents the LAP offer** as a loyalty reward (up to ₹75 lakh).
4. **Detects existing loan / EMI-reduction cases** and routes to a
   loan-transfer specialist via a separate branch.
5. **Captures 7 eligibility data points** in any order, remembering
   facts provided out of sequence and never re-asking.
6. **Applies eligibility rules**:
   - Hard disqualifiers: agricultural property, no original
     documents, cash income, tenure outside 3–15 years.
   - Soft fail: loan amount > ₹75L → counter-offer the cap.
   - Never-fails: ownership (sole or joint), market value.
7. **Hands off qualified leads** to a senior loan expert only after
   all 7 items are answered and validated.

---

## Repository Contents

| Path | Description |
|---|---|
| `system-prompt.md` | The complete system prompt (Deliverable 1) |
| `tests/test-matrix.md` | 28 test scenarios with expected outcomes |
| `approach.md` | Design rationale and build process |
| `docs/rules-spec.md` | Eligibility rule taxonomy |
| `docs/state-model.md` | State block definition |
| `docs/decision-tree.md` | Canonical call flow |

---

## External Evidence (not in repo)

| Resource | Location |
|---|---|
| Test call recordings + transcripts | `transcripts_urls.csv` (submitted separately) |
| Excel workbook with audio + transcripts | Submitted separately |

The `.gitignore` excludes `*.wav`, `*.mp3`, and `*.xlsx` — audio and
Excel files are hosted outside the repo to keep it lightweight.

---

## Testing Summary

| Batch | Scenarios | Passed |
|---|---|---|
| Baseline (English) | 12 | 12 |
| Adversarial (English) | 4 | 4 |
| Extended Edge Cases (English) | 8 | 8 |
| Hindi (language + gender) | 4 | 4 |
| **Total** | **28** | **28** |

Six prompt bugs were identified during testing and fixed. The prompt
went through 6 revisions. All scenarios pass on the final version.

See `tests/test-matrix.md` for full scenario definitions and
`approach.md` for the testing narrative.

---

## Build Process





---

## External Evidence (not in repo)

| Resource | Location |
|---|---|
| Test call recordings + transcripts | `transcripts_urls.csv` (submitted separately) |
| Excel workbook with audio + transcripts | Submitted separately |

The `.gitignore` excludes `*.wav`, `*.mp3`, and `*.xlsx` — audio and
Excel files are hosted outside the repo to keep it lightweight.

---

## Testing Summary

| Batch | Scenarios | Passed |
|---|---|---|
| Baseline (English) | 12 | 12 |
| Adversarial (English) | 4 | 4 |
| Extended Edge Cases (English) | 8 | 8 |
| Hindi (language + gender) | 4 | 4 |
| **Total** | **28** | **28** |

Six prompt bugs were identified during testing and fixed. The prompt
went through 6 revisions. All scenarios pass on the final version.

See `tests/test-matrix.md` for full scenario definitions and
`approach.md` for the testing narrative.

---

## Build Process
Requirements
↓
Eligibility Rules
↓
State Variables
↓
Decision Tree
↓
Conversation Rules
↓
Edge Cases
↓
System Prompt
↓
Testing Matrix
↓
6 prompt fixes (feedback loop)
↓
28/28 passing

text

---

## Key Design Decisions

1. **State machine, not a script.** 14 variables (8 eligibility
   slots + 3 gate flags + 3 terminal flags) maintained silently
   across the call.

2. **Four-beat per-turn loop.** Extract first. Validate second.
   Update state third. Ask next fourth.

3. **Rule taxonomy.** Four hard disqualifiers, one soft
   counter-offer, two never-fail, two transfer triggers. Different
   failure behaviors per rule type.

4. **Two distinct transfer branches.** Loan-transfer specialist
   (existing loans / EMI reduction) vs. senior loan expert
   (qualified fresh-loan leads). The prompt guards against routing
   a qualified lead to the wrong desk.

5. **RAG-only factual sourcing.** No invented rates, thresholds, or
   branch locations.

6. **ASR tolerance.** Handles garbled speech ("one point two floor"
   → ₹1.2 crore) by extracting from context.

---

## How to Deploy

1. Create a Single-Prompt Agent on Retell AI.
2. Paste `system-prompt.md` into the agent's prompt field.
3. Set dynamic variables: `agent_name`, `company_name`,
   `customer_name`, `agent_gender`, `current_date`, `current_day`,
   `current_time`, `language_to_speak`.
4. Add an `end_call` tool.
5. Configure post-call extraction fields (see `docs/state-model.md`).
6. Attach the knowledge base for `additional_context_from_rag`.
