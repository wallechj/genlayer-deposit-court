# DepositCourt — test flow

Two Studio test accounts: A = landlord (deployer), B = tenant.
Deployed contract address: (fill in after deploy)
Amounts are in wei. Deposit 1 GEN = 1000000000000000000.

Fixture files must be live on the repo's main branch BEFORE running the adjudication steps (validators fetch them at adjudication time):
- lease: https://raw.githubusercontent.com/wallechj/genlayer-deposit-court/main/examples/fixtures/lease.md
- window evidence: https://raw.githubusercontent.com/wallechj/genlayer-deposit-court/main/examples/fixtures/evidence_window.md
- wear evidence: https://raw.githubusercontent.com/wallechj/genlayer-deposit-court/main/examples/fixtures/evidence_wear.md
- utility evidence: https://raw.githubusercontent.com/wallechj/genlayer-deposit-court/main/examples/fixtures/evidence_utility.md

## 1. Open case

As A, call open_case with examples/sample_case.json:
- tenant: 0x7078Da91596B3b8e5578Acfa5C35B911E42BC6A8
- title: "Deposit dispute — 12 Oak Street Apt 4"
- property_desc: "2-bed apartment, 12 Oak Street Apt 4, 2-year tenancy"
- lease_url: the lease fixture URL above
- deposit: 1000000000000000000

Expected: returns 0. Read back get_case(0): landlord = A, tenant = B, status "open", refund 0.

Negative tests (must fail with the exact message):
- deposit 0 → "deposit must be greater than zero"
- lease_url "ftp://example.com/lease" → "lease_url must be an http(s) URL"
- tenant = A's own address → "landlord cannot open a case against themselves"

## 2. File deductions

As A, call file_deduction three times (examples/sample_deduction_0/1/2.json):
- 0: "Broken living-room window", 300000000000000000, evidence_window.md
- 1: "Repainting the whole apartment", 500000000000000000, evidence_wear.md
- 2: "Unpaid final utility bill", 200000000000000000, evidence_utility.md

Expected: returns 0, 1, 2. Read back get_deduction(i): status "filed".

Negative tests:
- as B, file a deduction → "only the landlord can file deductions"
- amount 0 → "amount must be greater than zero"
- amount 2000000000000000000 (over deposit) → "deduction cannot exceed the deposit"
- evidence_url "ftp://example.com/x" → "evidence_url must be an http(s) URL"

## 3. Tenant responds

As B, call respond(0, "I accept the utility bill and the window repair. The repainting charge is normal wear after two years — the walls only have light scuffs, and the lease says normal wear is the landlord's cost.").

Expected: null. Read back get_case(0): tenant_statement set.

Negative tests:
- as A, respond → "only the tenant can respond"
- as B, respond again → "tenant has already responded"

## 4. Adjudicate

Call adjudicate(0) from either account.

Expected: returns "decided". Read back:
- get_deduction(0).status = "upheld" (cracked window, intact at move-in)
- get_deduction(1).status = "rejected" (light scuffs = normal wear per lease)
- get_deduction(2).status = "upheld" (final bill, utilities are tenant's)
- get_case_refund(0) = 500000000000000000 (1 - 0.3 - 0.2)
- get_case(0).summary and .evidence are one-sentence notes citing the lease terms and the inspection notes

Negative tests:
- adjudicate(0) again → "case is not open for adjudication"
- file_deduction(0, ...) after decision → "case is not open for deductions"

## 5. Mark paid and close

As A, call mark_paid(0). Expected: null; get_case_status(0) = "paid".

Negative test:
- as B, mark_paid(0) → "only the landlord can mark a case paid" (test this before the A call, or on case 1)

As A, call close_case(0). Expected: null; get_case_status(0) = "closed".

## 6. Second case — no deductions, full refund

As A, open_case(tenant B, "Deposit — 7 Pine Lane", "1-bed flat, 7 Pine Lane, 1-year tenancy", lease fixture URL, 1000000000000000000). Expected: returns 1.

Negative test: mark_paid(1) before adjudication → "only decided cases can be marked paid".

Call adjudicate(1). Expected: "decided"; get_case_refund(1) = 1000000000000000000 (full refund, no deductions filed).

Negative test: close_case(1) while decided → "case cannot be closed in its current state".

Call mark_paid(1) as A, then close_case(1) as A. Expected: status "closed".

## 7. Final read-backs

- get_case_count() = 2
- get_deduction_count() = 3
- get_case(0): status "closed", refund 500000000000000000
- get_case(1): status "closed", refund 1000000000000000000
- get_case_evidence(0) names the lease terms and inspection notes
