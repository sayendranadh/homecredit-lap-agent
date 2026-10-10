# Home Credit LAP Agent — Test Matrix

28 scenarios across four tiers. All pass on the final prompt.

Terminal outcome codes:
  A — QUALIFIED_HANDOFF
  B — DISQUALIFIED
  C — TRANSFERRED
  D — DEFERRED (busy)
  E — DECLINED

═══════════════════════════════════════════════════════════════════
PART 1 — BASELINE SCENARIOS (12)
═══════════════════════════════════════════════════════════════════

| # | Scenario | Customer says | Expected state | Outcome | Result |
|---|----------|--------------|----------------|---------|--------|
| 01 | Perfect Eligible | Full happy path | All 8 slots filled | A — QUALIFIED | ✅ |
| 02 | Agricultural | "It's agricultural land" | property_type=Agricultural | B — DISQUALIFIED | ✅ |
| 03 | Cash Income | "I'm self-employed, cash" | income_mode=Cash | B — DISQUALIFIED | ✅ |
| 04 | No Originals | "I only have photocopies" | documents_available=FALSE | B — DISQUALIFIED | ✅ |
| 05 | Tenure Short | "2 years" | tenure=2 | B — DISQUALIFIED | ✅ |
| 06 | Tenure Long | "20 years" | tenure=20 | B — DISQUALIFIED | ✅ |
| 07a | Counter-Offer Accept | "90 lakhs" → "yes" | loan_amount=7500000 (overwrite) | A — QUALIFIED | ✅ |
| 07b | Counter-Offer Reject | "90 lakhs" → "no" | loan_amount=9000000 | E — DECLINED | ✅ |
| 08 | Joint Ownership | "jointly owned" | ownership_status=Joint | A — QUALIFIED | ✅ |
| 09a | Existing Loan (early) | "I have a loan" | existing_property_loan=TRUE | C — TRANSFERRED | ✅ |
| 09b | Existing Loan (late) | "actually I have a loan" | existing_property_loan=TRUE | C — TRANSFERRED | ✅ |
| 10 | EMI Reduction | "reduce my EMI" | emi_reduction_request=TRUE | C — TRANSFERRED | ✅ |
| 11 | Out-of-Order | 3 facts in one turn | 3 slots filled, skip ahead | A — QUALIFIED | ✅ |
| 12 | Busy | "I'm in a meeting" | callback_time captured | D — DEFERRED | ✅ |

═══════════════════════════════════════════════════════════════════
PART 2 — ADVERSARIAL SCENARIOS (4)
═══════════════════════════════════════════════════════════════════

| # | Scenario | Customer says | Expected state | Outcome | Result |
|---|----------|--------------|----------------|---------|--------|
| 13 | Contradiction | "residential… actually agricultural" | property_type=Agricultural (overwrite) | B — DISQUALIFIED | ✅ |
| 14 | Multi-Fact 5 Slots | 5 facts in one turn | 5 slots filled, skip to Q5 | A — QUALIFIED | ✅ |
| 15 | Interruption | "wait, which documents?" | Answer + return to Q3 | A — QUALIFIED | ✅ |
| 16 | Ambiguous Documents | "papers are probably somewhere" | Clarify → extract TRUE | A — QUALIFIED | ✅ |

═══════════════════════════════════════════════════════════════════
PART 3 — EXTENDED EDGE CASES (8)
═══════════════════════════════════════════════════════════════════

| # | Scenario | Customer says | Expected state | Outcome | Result |
|---|----------|--------------|----------------|---------|--------|
| 17 | Q5 Half-Fill (occ) | "I'm salaried" | occupation=Salaried, income_mode=UNKNOWN → ask income | A — QUALIFIED | ✅ |
| 18 | Q5 Half-Fill (inc) | "income is bank" | income_mode=Bank, occupation=UNKNOWN → ask occupation | A — QUALIFIED | ✅ |
| 19 | Diversion | "what interest rate?" | Defer to expert, return to flow | A — QUALIFIED | ✅ |
| 20 | RAG Question | "nearest branch?" | Answer from RAG or defer, return to flow | A — QUALIFIED | ✅ |
| 21 | Verification Refused | "wrong number" | customer_verified=FALSE | E — DECLINED | ✅ |
| 22 | Sensitive Info | "what's my balance?" | No disclosure, redirect to verify | E — DECLINED | ✅ |
| 23 | Correction Closed Slot | "actually 12 years" | tenure=12 (overwrite from 10) | A — QUALIFIED | ✅ |
| 24 | Busy Mid-Flow | "need to go" after Q3 | callback_time + partial state preserved | D — DEFERRED | ✅ |

═══════════════════════════════════════════════════════════════════
PART 4 — HINDI TESTS (4)
═══════════════════════════════════════════════════════════════════

| # | Scenario | Expected | Result |
|---|----------|----------|--------|
| H1 | Happy Path (Hindi) | Full flow in Hindi; female verb forms consistent (रही हूँ, not रहा हूँ) | ✅ |
| H2 | Agricultural (Hindi) | DQ fires; Hindi closing template; end_call fires | ✅ |
| H3 | Out-of-Order (Hindi) | Multi-fact extraction in Hindi | ✅ |
| H4 | Diversion (Hindi) | Hindi deferral + return to flow | ✅ |

═══════════════════════════════════════════════════════════════════
SUMMARY
═══════════════════════════════════════════════════════════════════

| Batch | Count | Passed |
|-------|-------|--------|
| Baseline | 12 | 12 |
| Adversarial | 4 | 4 |
| Extended | 8 | 8 |
| Hindi | 4 | 4 |
| **Total** | **28** | **28** |

Six prompt revisions were made during testing (see approach.md §8).
All scenarios pass on the final version.

Evidence:
- Audio recordings + transcripts: transcripts_urls.csv
- Public call links: (see CSV)

═══════════════════════════════════════════════════════════════════
DETAILED SCENARIO SCRIPTS
═══════════════════════════════════════════════════════════════════

Detailed customer scripts, expected state, and expected utterances
for each scenario are available in the transcripts_urls.csv file
submitted alongside this repository.

To reproduce any scenario: paste the corresponding customer script
into the Retell web test panel and run against the current agent.
