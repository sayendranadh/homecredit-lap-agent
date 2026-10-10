# Eligibility Rules Specification

Reference document for the Home Credit LAP voice agent. Defines
every rule, its evaluation timing, and its failure behavior.

---

## 1. Rule Taxonomy

Eligibility criteria do not all behave the same way on failure.
The agent implements four rule types with distinct behaviors:

| Type | Rules | Behavior on fail |
|---|---|---|
| **Hard disqualify** | Agricultural property, no original documents, cash income, tenure out of range | Immediate polite end-call |
| **Soft / renegotiable** | Loan amount > ₹75L | Counter-offer the cap; branch on customer's answer |
| **Never fails** | Ownership (sole or joint), market value (capture only) | No fail condition exists |
| **Transfer signal** | Existing property loan, EMI-reduction request | Route to loan-transfer specialist, end call |

**Ordering rule:** when a single utterance triggers both a
disqualification and a transfer signal, the disqualification wins.

---

## 2. Detailed Rules

### Rule 1 — Property Type

**Domain:** Residential, Commercial, Industrial, Agricultural

**Evaluation:** On slot fill (including overwrites)

**Pass values:** Residential, Commercial, Industrial

**Fail condition:** Agricultural

**Fail type:** Hard disqualify

**Notes:** The whitelist is preferred over a blacklist. Only the
four enumerated types are valid. Colloquial descriptions ("house",
"flat", "shop", "factory", "farmland", "खेती की ज़मीन") map to
the nearest whitelist value.

---

### Rule 2 — Ownership Status

**Domain:** Sole, Joint

**Evaluation:** On slot fill

**Pass values:** Sole, Joint

**Fail condition:** None

**Fail type:** Never fails

**Notes:** Joint ownership is explicitly eligible. The prompt
states this explicitly to prevent false-positive DQs from LLM
pattern-matching "shared" → "complicated".

---

### Rule 3 — Original Documents

**Domain:** Available (TRUE), Unavailable (FALSE)

**Evaluation:** On slot fill

**Pass condition:** Original documents available for verification

**Fail condition:** Originals unavailable

**Fail type:** Hard disqualify

**Notes:** Availability, not possession. The distinction matters:

| Statement | Interpretation |
|---|---|
| "Originals are at home" | Available — eligible |
| "Originals are in a bank locker" | Available — eligible |
| "Originals are with my lawyer" | Available — eligible |
| "I only have photocopies" | Unavailable — disqualify |
| "I lost the originals" | Unavailable — disqualify |
| "I think they're probably somewhere" | Ambiguous — clarify first |

Persistent ambiguity after two clarifications is treated as
unavailable.

---

### Rule 4 — Loan Amount

**Domain:** Numeric, in ₹

**Evaluation:** On slot fill

**Pass condition:** Amount ≤ ₹75,00,000

**Fail condition:** Amount > ₹75,00,000

**Fail type:** Soft / renegotiable

**Counter-offer template:**
> "The current offer is available for up to ₹75 lakh. Would you
> like to proceed with the maximum eligible amount of ₹75 lakh?"

**On acceptance:** Slot is overwritten to ₹75,00,000.

**On rejection:** Polite close, call ends.

**Notes:** This is the ONLY rule that renegotiates rather than
disqualifies. The distinction is stated explicitly in the system
prompt to prevent the LLM from treating all over-threshold answers
as hard fails.

---

### Rule 5 — Occupation + Income Mode

Two variables, one checklist item. The checklist item is complete
only when BOTH variables are FILLED.

**Occupation domain:** Salaried, Self-employed

**Income Mode domain:** Bank, Cash

**Evaluation:** On completion of both variables

**Pass condition:** Any occupation AND income_mode = Bank

**Fail condition:** income_mode = Cash (regardless of occupation)

**Fail type:** Hard disqualify

**Notes:** Occupation never disqualifies on its own. Both values
are eligible. The failure is always and only about income mode.

The 2×2 decision matrix:

| Occupation | Income | Result |
|---|---|---|
| Salaried | Bank | Pass |
| Salaried | Cash | Fail |
| Self-employed | Bank | Pass |
| Self-employed | Cash | Fail |

If only one variable is provided (e.g. "I'm salaried"), the
checklist item is INCOMPLETE. The agent asks only the missing half:
> "And is your income received via bank transfer or in cash?"

---

### Rule 6 — Market Value

**Domain:** Numeric, in ₹ (customer's estimate)

**Evaluation:** None — capture only

**Pass condition:** N/A

**Fail condition:** None. There is no minimum threshold.

**Fail type:** Never fails

**Notes:** The assignment explicitly states no minimum market-value
threshold exists. The agent captures the customer's estimate but
never disqualifies on this basis. This is stated as an explicit
negative in the system prompt because LLMs trained on typical
lending products tend to invent a threshold.

Market value is passed to the senior loan expert as underwriting
input, but it is not an eligibility gate.

---

### Rule 7 — Tenure

**Domain:** Numeric, in years

**Evaluation:** On slot fill

**Pass condition:** 3 ≤ tenure ≤ 15 (both bounds inclusive)

**Fail condition:** tenure < 3 OR tenure > 15

**Fail type:** Hard disqualify

**Boundary cases:**
- 2 years → fail
- 3 years → pass
- 15 years → pass
- 16 years → fail

**Notes:** Tenure is a hard disqualify, not a renegotiable rule.
Unlike loan amount, the agent does NOT counter-offer an in-range
value. The assignment specifies 3–15 as the accepted range, and
values outside it disqualify.

---

## 3. Transfer Signals

### Signal A — Existing Property Loan

**Trigger:** Customer clearly states they already have a loan on
the property being discussed.

**Clarity gate:** If the utterance is garbled, ambiguous, or could
be an ASR error, the agent clarifies:
> "Just to confirm — do you already have an existing loan on
> this property?"

**Behavior:** Route to loan-transfer specialist. End call.

**Message template:**
> "Thank you for clarifying. Since you already have an existing
> loan on the property, this would need to be handled by our
> loan-transfer specialist. A specialist will contact you shortly
> regarding this. Thank you for your time."

### Signal B — EMI Reduction Request

**Trigger:** Customer clearly states they want to reduce their
current EMI.

**Behavior:** Same as Signal A.

**Notes:** Signals A and B are tracked as separate state flags
(`existing_property_loan`, `emi_reduction_request`). Either one
triggers the transfer branch.

---

## 4. Interaction Between Rules

**Disqualify vs. transfer:** If a single utterance triggers both,
disqualify wins. The customer receives the disqualification message,
not the transfer routing.

**Overwrite handling:** When a slot is overwritten (e.g., customer
changes property type from residential to agricultural), the new
value is immediately re-evaluated against the applicable rule. The
DQ can fire on the overwrite, not just the first write.

**Q5 partial fill:** The checklist item completes only when both
occupation and income mode are filled. If only one is provided,
the agent asks the missing half.

---

## 5. Summary Table

| # | Rule | Domain | Fail condition | Fail type |
|---|---|---|---|---|
| 1 | Property Type | Residential, Commercial, Industrial, Agricultural | Agricultural | Hard |
| 2 | Ownership | Sole, Joint | — | Never |
| 3 | Documents | Available, Unavailable | Unavailable | Hard |
| 4 | Loan Amount | Numeric (₹) | > ₹75L | Soft |
| 5a | Occupation | Salaried, Self-employed | — | Never |
| 5b | Income Mode | Bank, Cash | Cash | Hard |
| 6 | Market Value | Numeric (₹) | — | Never |
| 7 | Tenure | Numeric (years) | < 3 or > 15 | Hard |
| — | Existing Loan | Boolean | TRUE | Transfer |
| — | EMI Reduction | Boolean | TRUE | Transfer |
