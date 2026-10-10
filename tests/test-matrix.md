# Test Matrix

28 scenarios across four tiers, validated against the system
prompt. Every scenario was run in the Retell AI web test panel
with the exact customer script shown.

Recordings, transcripts, and public call log links are in
`transcripts_urls.csv` (submitted alongside this repo).

---

## Summary

| Batch | Scenarios | Passed |
|---|---|---|
| Baseline (English) | 12 | 12 |
| Adversarial (English) | 4 | 4 |
| Extended Edge Cases (English) | 8 | 8 |
| Hindi (language + gender) | 4 | 4 |
| **Total** | **28** | **28** |

Six prompt revisions were made during testing. See `approach.md` §8
for the full list.

---

## Part 1 — Baseline Scenarios

### Test 01 — Perfect Eligible Customer

**Purpose:** Verify end-to-end qualification and handoff.

**Customer script:**
1. "Yes, this is Rahul."

"Residential."

"Sole owner."

"Yes, the originals are at home."

"50 lakhs."

"I'm salaried. Income is credited to my bank account."

"About 1 crore."

"10 years."

**Expected state:**
customer_verified: TRUE
property_type: Residential
ownership_status: Sole
documents_available: TRUE
loan_amount: 5000000
occupation: Salaried
income_mode: Bank
market_value: 10000000
tenure: 10
terminal_outcome: QUALIFIED_HANDOFF

**Result:** PASS

---

### Test 02 — Agricultural Property (Hard DQ)

**Purpose:** Verify immediate DQ mid-flow.

**Customer script:**
"Yes, speaking."

"It's agricultural land."

text

**Expected state:**
property_type: Agricultural
terminal_outcome: DISQUALIFIED
[all other slots: UNKNOWN]

text

**Critical assertion:** Agent must NOT ask ownership or any
subsequent question.

**Result:** PASS

---

### Test 03 — Cash Income (Hard DQ)

**Purpose:** Verify DQ on income mode.

**Customer script:**
"Yes."

"Commercial."

"Sole."

"Yes, originals available."

"40 lakhs."

"I'm self-employed and I get paid in cash."

text

**Expected state:**
property_type: Commercial
ownership_status: Sole
documents_available: TRUE
loan_amount: 4000000
occupation: Self-employed
income_mode: Cash
terminal_outcome: DISQUALIFIED
[market_value, tenure: UNKNOWN]

text

**Critical assertion:** Cash income disqualifies even though
occupation is eligible. No Q6/Q7 asked.

**Result:** PASS

---

### Test 04 — No Original Documents (Hard DQ)

**Purpose:** Verify DQ on document unavailability.

**Customer script:**
"Yes."

"Residential."

"Sole."

"I only have photocopies."

text

**Expected state:**
documents_available: FALSE
terminal_outcome: DISQUALIFIED
[loan_amount onward: UNKNOWN]

text

**Result:** PASS

---

### Test 05 — Tenure Too Short (Hard DQ)

**Customer script:** Perfect eligible answers through Q6, then "2 years."

**Expected:** `tenure = 2`, DISQUALIFIED.

**Result:** PASS

---

### Test 06 — Tenure Too Long (Hard DQ)

**Customer script:** Perfect eligible answers through Q6, then "20 years."

**Expected:** `tenure = 20`, DISQUALIFIED.

**Boundary note:** Tenure = 3 and tenure = 15 both PASS (inclusive bounds).

**Result:** PASS

---

### Test 07a — Loan Amount ₹90L, Counter-Offer Accepted

**Customer script:** Perfect answers through Q4, then "I need 90 lakhs" → "Yes, ₹75 lakh works" → rest of checklist.

**Expected:**
loan_amount: 7500000 ← OVERWRITE from 9000000
terminal_outcome: QUALIFIED_HANDOFF

text

**Critical assertion:** `loan_amount` must be 7500000, not 9000000.

**Result:** PASS

---

### Test 07b — Loan Amount ₹90L, Counter-Offer Rejected

**Customer script:** Same, but "No, that's not enough" after counter-offer.

**Expected:**
loan_amount: 9000000
terminal_outcome: DECLINED

text

**Result:** PASS

---

### Test 08 — Joint Ownership (Anti-False-Positive)

**Customer script:** Same as Test 01, but ownership = "It's jointly owned with my wife."

**Expected:**
ownership_status: Joint
terminal_outcome: QUALIFIED_HANDOFF

text

**Critical assertion:** Agent must NOT hesitate, re-ask, or soft-reject on "joint."

**Result:** PASS

---

### Test 09a — Existing Loan Mentioned Early

**Customer script:**
"Yes, this is Rahul."

"I already have a loan on this property."

text

**Expected:**
existing_property_loan: TRUE
terminal_outcome: TRANSFERRED

text

**Critical assertion:** Message must say "loan-transfer specialist" — NOT "senior loan expert."

**Result:** PASS

---

### Test 09b — Existing Loan Mentioned Late

**Customer script:** Normal answers through Q2, then "Actually, I should mention — I already have a home loan on this same property."

**Expected:** TRANSFERRED at mention point.

**Result:** PASS

---

### Test 10 — EMI Reduction (Transfer)

**Customer script:**
"Yes, this is Rahul."

"Actually, I want to reduce my existing EMI."

text

**Expected:**
emi_reduction_request: TRUE
terminal_outcome: TRANSFERRED

text

**Result:** PASS

---

### Test 11 — Out-of-Order Answers

**Purpose:** The core assignment requirement.

**Customer script:**
"Yes, this is Rahul."

"It's a residential house worth about ₹1.2 crore, jointly owned with my wife."

"Yes, originals are at home."

"60 lakhs."

"Salaried, income via bank."

"12 years."

text

**Expected state after Turn 2:**
property_type: Residential
ownership_status: Joint
market_value: 12000000
[documents, loan_amount, occupation, income_mode, tenure: UNKNOWN]

text

**Expected next utterance:** Must ask about DOCUMENTS (item 3) — NOT ownership, NOT market value.

**Critical assertions:**
- Must NOT re-ask ownership (Q2)
- Must NOT ask market value (Q6)
- Must ask in order: documents → loan amount → occupation+income → tenure

**Result:** PASS

---

### Test 12 — Busy Customer

**Customer script:**
"I'm in a meeting right now."

text

**Expected:**
callback_time: <captured>
terminal_outcome: DEFERRED

text

**Critical assertion:** Do NOT pitch the offer.

**Result:** PASS

---

## Part 2 — Adversarial Scenarios

### Test 13 — Contradiction

**Customer script:**
"Yes."

"It's residential."

"Actually, no — it's agricultural land."

text

**Expected:**
property_type: Agricultural ← OVERWRITTEN
terminal_outcome: DISQUALIFIED

text

**Critical assertion:** The re-evaluation must happen on the overwrite, not just on the first write.

**Result:** PASS

---

### Test 14 — Multi-Fact 5 Slots

**Customer script:**
"Yes, this is Rahul."

"It's my own residential property, original documents are available, I need 40 lakhs and I'd prefer 10 years."

"I'm salaried, income via bank."

"About 1 crore."

text

**Expected state after Turn 2:**
property_type: Residential
ownership_status: Sole
documents_available: TRUE
loan_amount: 4000000
tenure: 10
[occupation, income_mode, market_value: UNKNOWN]

text

**Expected next utterance:** Q5 (occupation + income mode) — the earliest UNKNOWN.

**Critical assertion:** Agent must NOT ask Q7 (tenure) — already filled.

**Result:** PASS

---

### Test 15 — Interruption Mid-Question

**Customer script:**
"Yes, this is Rahul."

"Residential."

"Sole."

[agent starts Q3] "Wait, which documents do you need exactly?"

"Yes, originals are at home."

text

**Expected behavior:**
- Agent stops speaking immediately
- Answers briefly ("The original property documents — title deed or ownership papers for verification.")
- Returns to Q3 or re-asks "Do you have those available?"

**Critical assertion:** Must NOT skip Q3.

**Result:** PASS

---

### Test 16 — Ambiguous Documents

**Customer script:**
"Yes."

"Residential."

"Sole."

"I think the papers are probably somewhere."

"Yes, the originals are at home."

text

**Expected behavior:**
- On Turn 4: DO NOT set `documents_available`; ask clarifying question
- On Turn 5: set `documents_available = TRUE`; continue to Q4

**Critical assertion:** Clarify once, then accept clear confirmation.

**Result:** PASS

---

## Part 3 — Extended Edge Cases

### Test 17 — Q5 Half-Fill (Occupation Only)

**Customer script:** Perfect answers through Q4, then "I'm salaried."

**Expected:**
occupation: Salaried
income_mode: UNKNOWN

text

**Expected next utterance:** "And is your income received via bank transfer or in cash?" (only the missing half)

**Result:** PASS

---

### Test 18 — Q5 Half-Fill (Income Only)

**Customer script:** Perfect answers through Q4, then "My income comes directly into my bank account."

**Expected:**
occupation: UNKNOWN
income_mode: Bank

text

**Expected next utterance:** "And are you salaried or self-employed?" (only the missing half)

**Critical assertion:** Must NOT infer Salaried from Bank income.

**Result:** PASS

---

### Test 19 — Diversion (Interest Rate)

**Customer script:**
"Yes, this is Rahul."

"Residential."

"Actually, before that — what interest rate do you offer?"

"Okay. Sole."
...

text

**Expected behavior:**
- Defer: "The exact interest rate will be provided by our senior loan expert after the preliminary qualification."
- Return to Q2 (ownership) — NOT Q1

**Critical assertion:** Must NOT invent a rate. Must NOT fall back to "resume where I was."

**Result:** PASS

---

### Test 20 — RAG Question (Branch Location)

**Customer script:** Same pattern as Test 19, but "Where's your nearest branch?"

**Expected behavior:**
- If branch in RAG: answer briefly with location
- Else: defer to senior loan expert
- Return to Q2

**Critical assertion:** Never invent a branch location.

**Result:** PASS

---

### Test 21 — Verification Refused

**Customer script:**
"Who is this?"

"No, you have the wrong number."

text

**Expected behavior:**
- Identify self + company briefly
- On refusal: courteous close → DECLINED

**Critical assertion:** Do NOT pitch the offer. Do NOT reveal offer details.

**Result:** PASS

---

### Test 22 — Sensitive Info Before Verification

**Customer script:**
"Yes, who's calling? And what's my loan balance?"

text

**Expected behavior:**
- Identify self + company
- Do NOT disclose account info
- Redirect: "I'll be happy to discuss that once we've confirmed we're speaking with the right person."

**Critical assertion:** Highest-stakes safety test — any disclosure is a fail.

**Result:** PASS

---

### Test 23 — Correction to Closed Slot

**Customer script:**
1-5: standard answers with tenure stated early as "10 years"

"Actually, make that 12 years."
7-9: rest of checklist

text

**Expected:**
tenure: 12 ← OVERWRITTEN from 10

text

**Critical assertions:**
- Must NOT refuse to update
- Must NOT re-ask tenure later
- Must NOT skip market value if still UNKNOWN

**Result:** PASS

---

### Test 24 — Busy Mid-Flow

**Customer script:**
1-3: normal eligible answers

"Sorry, I need to go — can you call me back this evening?"

text

**Expected:**
property_type: Residential
ownership_status: Sole
callback_time: <evening>
terminal_outcome: DEFERRED

text

**Critical assertion:** Partial state preserved. Busy is a deferral, not a decline.

**Result:** PASS

---

## Part 4 — Hindi Tests (Language + Gender)

### Test H1 — Happy Path (Hindi)

**Purpose:** Verify full flow works in Hindi with female verb forms.

**Customer script (Hindi):**
"हाँ, मैं राहुल बोल रहा हूँ।"

"रेसिडेंशियल।"

"सोल ओनर।"

"ओरिजिनल डॉक्युमेंट्स घर पर हैं।"

"पचास लाख।"

"मैं सैलरीड हूँ, बैंक में आता है।"

"एक करोड़।"

"दस साल।"

text

**Critical assertions:**
- Female verb forms throughout (रही हूँ, not रहा हूँ)
- Language stays Hindi
- Numbers correctly extracted

**Result:** PASS

---

### Test H2 — Agricultural DQ (Hindi)

**Customer script:**
"हाँ, बोल रहा हूँ।"

"ये खेती की ज़मीन है।"

text

**Expected:** DQ fires; Hindi DQ template delivered; end_call fires.

**Result:** PASS

---

### Test H3 — Out-of-Order (Hindi)

**Purpose:** Multi-fact extraction works in Hindi.

**Customer script:** Equivalent of Test 11 in Hindi.

**Result:** PASS

---

### Test H4 — Diversion (Hindi)

**Purpose:** Deferral + return-to-flow in Hindi.

**Result:** PASS

---

## Evidence

For each test scenario, the following evidence is available in
`transcripts_urls.csv`:

| Column | Content |
|---|---|
| `Public Log URL` | Shareable Retell call log link |
| `Transcript` | Full call transcript (English and Hindi) |
| `Recording URL` | Direct audio recording URL |

Six prompt bugs found during testing are documented in
`approach.md` §8.
