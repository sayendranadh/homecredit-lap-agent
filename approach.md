# Home Credit LAP Voice Agent — Approach Document

**Author:** [Your name]
**Date:** [Submission date]

---

## 1. The Problem, Stated Precisely

The assignment looks like a scripted questionnaire. It isn't.

A scripted bot asks seven questions in order, captures seven answers,
and ends. That bot fails this assignment because real customers don't
behave that way. They volunteer information before it's asked, correct
themselves mid-flow, ask questions of their own, and sometimes say
five relevant things in one sentence.

The real challenge: **collect a fixed set of data points from a
free-form conversation, in any order, without losing track of what's
already been said, and without ever letting the customer feel like
they're being interrogated.**

That is a state-management problem disguised as a prompt-writing
problem. This solution treats it that way.

---

## 2. The Core Principle

Every customer turn is processed by a single four-beat loop:

> **Extract first. Validate second. Update state third. Ask next fourth.**

Stated as a mnemonic and as pseudocode in the system prompt's
Section 20 (The Per-Turn Loop). It is the heartbeat of the agent —
every other rule either feeds this loop or is triggered by it.

**Why this ordering matters:**

- **Extract first** — the agent reads every slot the customer
  addressed in *this turn*, regardless of what it asked. This is
  what makes out-of-order capture possible.

- **Validate second** — after extraction, the agent checks whether
  the utterance triggers a hard disqualification or transfer signal.
  Both checks fire before state is committed.

- **Update state third** — valid facts are written into the state
  block. Corrections overwrite. Silent — never narrated.

- **Ask next fourth** — the next question is recomputed from state,
  not recalled from "where I was." This prevents diversions,
  interruptions, and corrections from causing the agent to lose
  its place.

Two rules encapsulate the hardest requirement:

> *Always ask the earliest unanswered item.*
> *Never re-ask a question the customer has already clearly answered.*

---

## 3. State Management

The system prompt maintains an internal state block containing
fourteen variables across three groups:

**Gate flags** — `customer_verified`, `existing_property_loan`,
`emi_reduction_request`. Control flow before the eligibility
checklist begins.

**Eligibility slots** — eight variables across seven checklist items.
Item 5 ("Occupation + Income Mode") welds two variables into one
checklist item, exactly as the assignment specifies.

**Terminal flags** — `disqualified`, `transfer_required`,
`callback_time`. Set once; drive the exit branch.

Every slot is binary: `UNKNOWN` or `FILLED`. No partial state. This
makes the "don't re-ask" rule mechanical — if a slot is not UNKNOWN,
the agent does not ask about it.

The state block is never spoken aloud. The customer experiences a
warm conversation; internally, the agent is a slot-filling engine.

---

## 4. The Business Rule Taxonomy

Not all eligibility criteria behave the same way on failure. Early
in the design, treating them uniformly produced two kinds of bugs:
the agent would kill a call for a rule that should have been
renegotiated (loan amount), or it would try to renegotiate a rule
that should have killed the call (tenure).

The prompt therefore classifies every rule into one of four types:

| Type | Rules | Behavior on fail |
|---|---|---|
| **Hard disqualify** | Agricultural property, no original documents, cash income, tenure out of range | Immediate polite end-call |
| **Soft / renegotiable** | Loan amount > ₹75L | Counter-offer the cap; branch on answer |
| **Never fails** | Ownership (sole or joint), market value | No fail condition exists |
| **Transfer signal** | Existing property loan, EMI-reduction request | Route to loan-transfer specialist, end call |

The taxonomy resolves an ambiguity in the assignment: it says
"immediate disqualification" for eligibility fails, but separately
says over-limit loan amounts should trigger a counter-offer. Both
are true; they apply to different rule types. The prompt states the
taxonomy up front.

A separate rule sits above the taxonomy: **when a single utterance
triggers both disqualification and transfer, disqualification wins.**

---

## 5. Two Distinct Transfer Branches

The assignment uses the word "transfer" for two different things:

- **Loan-transfer specialist** — customers who already have a loan
  on the property, or who want to reduce an existing EMI. Not LAP
  leads. Message acknowledges the customer's situation and promises
  a callback from a different team.

- **Senior loan expert** — customers who complete all seven
  eligibility items and pass every rule. The *qualified* handoff.
  Message promises a callback specifically to discuss next steps
  and exact interest rates.

Both branches end with a callback. Both mention a specialist. If
phrased identically, a qualified lead could be routed to the wrong
team. The prompt uses two distinct templates with deliberately
different framing.

---

## 6. Handling the Four Hard Conversation Cases

**Out-of-order information.** Extraction is unconditional. If a
customer answers Q1 by also volunteering Q2 and Q6, all three slots
fill. When the flow reaches Q2 and Q6, the slots are already filled
and the questions are skipped.

**Corrections.** Slots are mutable. A later clear statement
overwrites an earlier one — even after the flow moved past the slot.
The new value is re-evaluated against the rules. A customer who says
"residential… actually, agricultural" gets disqualified on the
overwrite.

**Diversions.** When the customer asks a question the agent cannot
answer, the agent defers to the senior loan expert rather than
inventing an answer, then recomputes the next question from state.

**Interruptions.** When the customer cuts the agent off, the agent
stops speaking, answers briefly, then returns to the first UNKNOWN
item — recomputed from state, not from position.

---

## 7. Anti-Hallucination Guardrails

Two specific hallucinations are common and damaging here:

**Inventing an interest rate.** The assignment specifies that the
senior loan expert provides exact rates. The agent defers, always.

**Inventing a minimum market value.** The assignment explicitly says
no threshold exists — but an LLM trained on typical lending products
will produce one. The prompt states the negative explicitly.

Both guards follow a single principle: **factual answers come only
from `additional_context_from_rag`; if the answer isn't there, the
agent defers rather than fabricates.** This makes the prompt
RAG-swappable.

---

## 8. Testing Approach

28 scenarios across four tiers:

- **Baseline (12)** — one test per terminal branch, plus the core
  out-of-order requirement.
- **Adversarial (4)** — contradiction, maximum extraction density,
  mid-question interruption, and ambiguity calibration.
- **Extended (8)** — the conversational edge cases: Q5 half-fill
  (both directions), diversions, RAG-answerable questions,
  verification refusal, sensitive-info prevention, corrections to
  closed slots, mid-flow deferral.
- **Hindi (4)** — language and gender enforcement.

**Critical must-pass tests:** the happy path, all four hard
disqualifications, the out-of-order test, the contradiction test,
the sensitive-info test, and the Hindi gender test.

### Six bugs found and fixed during testing

1. **Transfer check skipped** (Test 03) — the agent went straight
   from offer to Q1 without asking about existing loans. Fix:
   mandated transition line in Section 8.

2. **`end_call` not firing** (Test 03) — the agent delivered the DQ
   message but left the call open. Fix: added Section 25 (Call
   Termination Enforcement) and configured the Retell `end_call` tool.

3. **Market value re-asked after ASR garble** (Test 11) — "one
   point two crore" transcribed as "one point two floor"; the agent
   dropped the fact, then re-asked Q6. Fix: ASR-tolerance rule in
   Section 13 and pre-ask slot check in Section 11.

4. **Transfer branch fired on garbled input** (Test 16) — the agent
   heard unintelligible audio and triggered the transfer branch
   immediately. Fix: clarity gate in Section 10 — must confirm
   before firing a terminal branch.

5. **Extraction missed during gate phase** (Test 19) — the agent
   extracted loan-status from the transfer-check answer but missed
   the embedded property type. Fix: extended Section 13's extraction
   scope to cover pre-eligibility gates.

6. **Intro missing / redundant** (Tests 18, 22) — the
   self-introduction appeared inconsistently and sometimes
   duplicated. Fix: mandated sequence in Section 6.

All 28 scenarios pass after the fixes.

---

## 9. What I Would Improve With More Time

**Multi-turn persistence.** State is currently in-context per call.
A callback after a busy deferral re-verifies and re-collects from
scratch. Production version: persist the state block to a CRM and
reload on callback.

**Confidence-aware extraction.** The "clearly provided" qualifier
gives the LLM latitude without a numeric anchor. A production
system would score extraction confidence and clarify below a
threshold.

**Analytics on terminal outcomes.** Distribution of outcomes per
prompt version is the primary signal for field behavior. Building
that dashboard is a day-two task.

**Token budget.** Current working set is ~7–9k tokens at
$0.198/min. A production version would trim few-shot examples and
prune redundant rule restatements.

---

## 10. Summary

The assignment tests whether the candidate understands that a
production voice agent is a state machine, not a script. This
submission makes that explicit: the state block, the four-beat
per-turn loop, the rule taxonomy, and the two distinct transfer
branches are the architecture. The prompt is the expression of the
architecture.

The single most important sentence in the prompt:

> *Always ask the earliest unanswered item. Never re-ask a question
> the customer has already clearly answered.*

Everything else exists to support those two rules.
