# State Model

The agent maintains an internal state block across the call. The
state block is the source of truth for what has been captured. It
is never spoken aloud.

---

## 1. Full State Block


QUALIFICATION_STATE:
customer_verified: UNKNOWN
property_type: UNKNOWN
ownership_status: UNKNOWN
documents_available: UNKNOWN
loan_amount: UNKNOWN
occupation: UNKNOWN
income_mode: UNKNOWN
market_value: UNKNOWN
tenure: UNKNOWN
existing_property_loan: UNKNOWN
emi_reduction_request: UNKNOWN
disqualified: FALSE
transfer_required: FALSE
callback_time: UNKNOWN

**Fourteen variables across three groups.**

---

## 2. Variable Groups

### Gate Flags

Control flow before the eligibility checklist begins.

| Variable | Type | Domain | Set by |
|---|---|---|---|
| `customer_verified` | Boolean | UNKNOWN / TRUE / FALSE | Verification (Section 6) |
| `existing_property_loan` | Boolean | UNKNOWN / TRUE / FALSE | Transfer detection (Section 10) |
| `emi_reduction_request` | Boolean | UNKNOWN / TRUE / FALSE | Transfer detection (Section 10) |

### Eligibility Slots

The eight values captured across seven checklist items.

| Variable | Domain | Checklist item |
|---|---|---|
| `property_type` | Residential \| Commercial \| Industrial \| Agricultural | Q1 |
| `ownership_status` | Sole \| Joint | Q2 |
| `documents_available` | TRUE \| FALSE | Q3 |
| `loan_amount` | Numeric (₹) | Q4 |
| `occupation` | Salaried \| Self-employed | Q5a |
| `income_mode` | Bank \| Cash | Q5b |
| `market_value` | Numeric (₹) | Q6 |
| `tenure` | Numeric (years) | Q7 |

**Note:** Q5 owns two variables. The checklist item completes
only when BOTH `occupation` and `income_mode` are filled.

### Terminal Flags

Set once. Drive the exit branch.

| Variable | Type | Domain | Meaning |
|---|---|---|---|
| `disqualified` | Boolean | FALSE / TRUE | Hard rule failed |
| `transfer_required` | Boolean | FALSE / TRUE | Transfer signal detected |
| `callback_time` | Text | String | Callback scheduled |

---

## 3. Slot States

Every slot is binary: **UNKNOWN** or **FILLED**.

There is no partial state. There is no "probably". A slot
transitions from UNKNOWN to FILLED only when the customer's
utterance clearly provides the value.

This binary model is what makes the "don't re-ask" rule
mechanical — if a slot is not UNKNOWN, the agent does not ask
about it.

---

## 4. Transition Rules

### Filling a slot

A slot transitions UNKNOWN → FILLED when the customer's turn
clearly provides the value for that slot. Extraction runs on
**every** customer turn, regardless of what question was asked.

### Overwriting a slot

Slots are mutable. A later clear statement overwrites an earlier
one. The latest clearly-stated value wins.

After any overwrite, the new value is **immediately re-evaluated**
against the applicable rule. A disqualification can fire on the
overwrite, not just on the first write.

### Corrections

The customer may correct an answer after the flow has moved past
the slot. The agent overwrites silently, acknowledges briefly if
useful, and continues from the current position.

### Inferring

The agent never infers one variable from another. If the customer
says "I'm salaried", the agent does NOT default `income_mode` to
Bank. It asks the missing half.

### Silent updates

State updates are never spoken aloud. The customer experiences a
natural conversation; internally, the agent is a slot-filling engine.

---

## 5. Checklist Items vs. Variables

There are **7 checklist items** but **8 eligibility variables**.

| # | Checklist Item | Variables Owned |
|---|---|---|
| 1 | Property Type | property_type |
| 2 | Ownership Status | ownership_status |
| 3 | Original Documents | documents_available |
| 4 | Loan Amount | loan_amount |
| 5 | Occupation + Income Mode | occupation AND income_mode |
| 6 | Market Value | market_value |
| 7 | Tenure | tenure |

**Critical rule:** Item 5 is complete only when BOTH variables
are non-UNKNOWN. If the customer provides only one half, the item
remains incomplete and the agent asks only the missing half:

- Occupation filled → "And is your income received via bank
  transfer or in cash?"
- Income mode filled → "And are you salaried or self-employed?"

---

## 6. Next-Question Algorithm

After every customer turn:

1. **Extract** — read every slot the customer addressed.
2. **Validate** — check for disqualify condition, then transfer.
3. **Update** — write valid facts into state. Overwrite on
   correction. Silent.
4. **Ask** — find the earliest checklist item still UNKNOWN.
   Ask ONLY that question.

**Mnemonic:** Extract first. Validate second. Update state third.
Ask next fourth.

The next question is **recomputed from state** every turn — never
recalled from "where I was". A diversion, an interruption, or a
mid-flow correction cannot cause the agent to lose its place.

---

## 7. Terminal States

A call ends in exactly one of five terminal states:

| Terminal state | Set by | Message |
|---|---|---|
| QUALIFIED_HANDOFF | All 7 items filled + no DQ + no transfer | Senior loan expert will call |
| DISQUALIFIED | Hard rule failed | "does not meet the criteria for this specific offer" |
| TRANSFERRED | existing_property_loan OR emi_reduction_request = TRUE | Loan-transfer specialist will call |
| DEFERRED | Customer busy | Callback scheduled |
| DECLINED | Customer refused, rejected counter-offer, or failed verification | Polite close |

The terminal flags `disqualified` and `transfer_required` are
mutually exclusive. Once either is set, the call ends — no further
eligibility collection.

---

## 8. Post-Call Extraction Fields

For evaluation and submission evidence, the following fields are
extracted from the completed call:

| Field | Type |
|---|---|
| `customer_verified` | Boolean |
| `property_type` | Selector (4 options) |
| `ownership_status` | Selector (2 options) |
| `documents_available` | Boolean |
| `loan_amount` | Number |
| `occupation` | Selector (2 options) |
| `income_mode` | Selector (2 options) |
| `market_value` | Number |
| `tenure` | Number |
| `terminal_outcome` | Selector (5 options) |

Empty values are valid — a call that ends in disqualification
before Q6 will have no `market_value`.
