# Decision Tree

Canonical flow for the Home Credit LAP voice agent, with every
branch and terminal outcome documented.

---

## 1. Top-Level Flow

START
│
├─ Agent greeting: "Hello, may I speak with {{customer_name}}?"
│
├─ Customer responds?
│ ├─ No / wrong number → DECLINED (courteous close)
│ ├─ Asks "who is calling?" → explain identity, re-attempt
│ └─ Confirms → customer_verified = TRUE
│
├─ Agent introduction: "This is {{agent_name}} from {{company_name}}."
│
├─ Agent: "Is now a good time to talk?"
│ ├─ No → BUSY DEFERRAL
│ └─ Yes → continue
│
├─ OFFER PRESENTATION
│
├─ TRANSFER CHECK (mandatory, before Q1)
│ "Do you already have a loan on the property, or are you
│ looking to reduce an existing EMI?"
│ ├─ Yes, existing loan → TRANSFERRED
│ ├─ Yes, EMI reduction → TRANSFERRED
│ └─ No → continue
│
└─ ELIGIBILITY CHECKLIST (Q1 → Q7)
│
└─ Any DQ fires mid-flow → DISQUALIFIED
└─ All 7 items complete + no DQ → QUALIFIED_HANDOFF

---

## 2. Verification Branch (Section 6)

**Question:** "Hello, may I speak with {{customer_name}}?"

| Customer response | Action |
|---|---|
| "Yes, this is Rahul" | `customer_verified = TRUE` → continue |
| "Yes, speaking" | `customer_verified = TRUE` → continue |
| "Who is calling?" | Explain identity briefly, re-attempt verification |
| "No, wrong number" | Courteous close → DECLINED |
| Silence | Prompt once: "Are you still there?" → if silence, close |

**Security rule:** No account, loan, or offer-specific details
are disclosed until `customer_verified = TRUE`. This includes
loan balances, product names, and offer amounts.

---

## 3. Busy Branch (Section 7)

**Trigger:** Customer signals they cannot talk — at any point
before the offer is presented.

**Signals:** "I'm busy", "I'm in a meeting", "Can you call later",
"Not a good time", "I'm driving".

**Behavior:**

1. Acknowledge.
2. Ask: "Certainly. What would be a convenient time for us to
call you back?"
3. Capture callback_time.
4. End Call.

**Terminal state:** DEFERRED

**Note:** Busy is not a decline. The lead is postponed, not lost.
Partial state (if any) is preserved for the callback.

---

## 4. Transfer Branch (Section 10)

**Trigger:** Customer clearly states either:
- They already have a loan on the property
- They want to reduce an existing EMI

**Clarity gate:** If the utterance is ambiguous or garbled, the
agent clarifies before firing the branch:

> "Just to confirm — do you already have an existing loan on
> this property?"

**Behavior:**

1. Acknowledge
2. Explain that a loan transfer specialist will contact them.
3. End call.


**Terminal state:** TRANSFERRED

**Message template:**
> "Thank you for clarifying. Since you already have an existing
> loan on the property, this would need to be handled by our
> loan-transfer specialist. A specialist will contact you shortly
> regarding this. Thank you for your time."

**Note:** Do NOT run the eligibility checklist. Do NOT route to
the senior loan expert — that's a different handoff.

---

## 5. Eligibility Checklist

**Canonical order:**

| # | Item | Question |
|---|---|---|
| 1 | Property Type | "What type of property is it — residential, commercial, or industrial?" |
| 2 | Ownership Status | "Are you the sole owner, or is it jointly owned?" |
| 3 | Original Documents | "Do you have the original property documents available for verification?" |
| 4 | Loan Amount | "How much are you looking to borrow?" |
| 5 | Occupation + Income Mode | "Are you salaried or self-employed, and is your income received via bank transfer or in cash?" |
| 6 | Market Value | "Approximately what is the current market value of the property?" |
| 7 | Tenure | "How many years would you like to take to repay the loan?" |

**Skip-forward rule:** Before asking any question, check the target
slot. If it's already FILLED (from an out-of-order answer), skip
the question.

**Next-question rule:** The next question is always the **earliest
checklist item still UNKNOWN**. Never "resume where I was."

---

## 6. Rule Fails During Checklist

Each eligibility rule has a specific failure behavior:

### Hard Disqualifiers

| Trigger | Point in flow |
|---|---|
| Property type = Agricultural | After Q1 |
| Documents unavailable | After Q3 |
| Income mode = Cash | After Q5b |
| Tenure outside 3–15 | After Q7 |

**Behavior:**

1. STOP the qualification flow immediately.
2. Do NOT ask further eligibility questions.

\
3. Deliver the DQ message:
"Thank you for your time. Unfortunately, you don't meet
the criteria for this specific offer at this time.
Have a good day."
4. Call end_call

**Terminal state:** DISQUALIFIED

### Soft Fail — Loan Amount

**Trigger:** Loan amount > ₹75,00,000

**Behavior:**

1. Deliver counter-offer:
"The current offer is available for up to ₹75 lakh.
Would you like to proceed with the maximum eligible
amount of ₹75 lakh?"
2. Branch on customer's answer:
├─ Yes → loan_amount = 7500000 (overwrite) → continue
└─ No → polite close → DECLINED

**Terminal state:** QUALIFIED (continue) or DECLINED

### Never-Fails

Ownership and market value have no fail condition. The agent
captures the value and continues.

---

## 7. Diversion Handling (Section 16)

**Trigger:** Customer asks a question that isn't an answer to the
current question, isn't a disqualifier, and isn't a transfer signal.

**Examples:** "What's the interest rate?", "Where's your branch?",
"How did you get my number?"

**Behavior:**

1. If the answer is in additional_context_from_rag → answer briefly.
2. Else → defer: "The exact details will be provided by our
senior loan expert after the preliminary qualification."
3. Return to the next missing checklist question, recomputed
from state.

**Note:** Never invent rates, terms, or branch locations.

---

## 8. Interruption Handling (Section 17)

**Trigger:** Customer cuts the agent off mid-question.

**Behavior:**

1. Stop speaking immediately.
2. Answer the customer's question briefly.
3. Return to the next missing checklist item, recomputed from
state (not from position).

---

## 9. Qualified Handoff (Section 19)

**Gate condition:**

all 7 checklist items = FILLED
AND disqualified = FALSE
AND transfer_required = FALSE

**Behavior:**

1. Deliver handoff message:
"Excellent — you meet the preliminary criteria for the offer.
A senior loan expert will call you back shortly to discuss
the next steps and provide the exact interest rates.
Thank you for your time."
2. Call end_call.

**Terminal state:** QUALIFIED_HANDOFF

**Note:** Never say "fully qualified" or "approved" before the
gate condition is met. Use "you meet the preliminary criteria."

---

## 10. Terminal State Summary

| Terminal | Trigger | Message |
|---|---|---|
| QUALIFIED_HANDOFF | All 7 items filled, no DQ, no transfer | Senior loan expert will call |
| DISQUALIFIED | Hard rule failed | Doesn't meet criteria for this offer |
| TRANSFERRED | Existing loan or EMI reduction | Loan-transfer specialist will call |
| DEFERRED | Customer busy | Callback scheduled |
| DECLINED | Refused, rejected counter-offer, wrong number | Polite close |

Every call ends in exactly one of these five states.
