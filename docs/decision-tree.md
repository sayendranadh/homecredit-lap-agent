# Decision Tree

START
  ↓
Greeting → Verify → Busy?
  ├─ YES → callback time → END
  └─ NO  → Offer → Existing Loan / EMI?
              ├─ YES → transfer message → END
              └─ NO  → 7 Eligibility Qs
                        → Decision
                          ├─ FAIL → explain → END
                          └─ PASS → senior expert handoff → END
