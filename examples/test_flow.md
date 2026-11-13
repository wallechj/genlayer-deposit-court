# DepositCourt — test flow (v2)

Two Studio test accounts: A = landlord (deployer), B = tenant.
Deployed contract address: 0x94F7Cf96FA603Bb7df38084d77BC92581D5802B1
Amounts are in wei. Deposit 1 GEN = 1000000000000000000.

Fixture files must be live on the repo's main branch BEFORE running the adjudication steps (validators fetch them at adjudication time):
- lease: https://raw.githubusercontent.com/wallechj/genlayer-deposit-court/main/examples/fixtures/lease.md
- window evidence: https://raw.githubusercontent.com/wallechj/genlayer-deposit-court/main/examples/fixtures/evidence_window.md
- wear evidence: https://raw.githubusercontent.com/wallechj/genlayer-deposit-court/main/examples/fixtures/evidence_wear.md
- utility evidence: https://raw.githubusercontent.com/wallechj/genlayer-deposit-court/main/examples/fixtures/evidence_utility.md

## The filing boundary (what v2 adds)

Adjudication is gated by a non-bypassable intake boundary. A case opens with
`filing_deadline = created_at + 604800` (7 days) and `filing_closed = false`.
Adjudication opens only when intake is final — landlord called `close_filing`
AND tenant filed a response — or when the filing deadline has passed, so a
non-responsive party cannot stall the case forever. Filing methods enforce the
same boundary from the other side: no deduction after the landlord closed
filing or after the deadline, no response before the landlord closed filing
(the tenant answers the final item list) or after the deadline.

The 7-day deadline cannot be fast-forwarded in Studio, so the live demo
exercises the mutual-completion path; the deadline fallback is verified by
read-back (`filing_deadline = created_at + 604800` on every case) and uses the
exact same gate condition in `adjudicate` and `can_adjudicate`.

## 1. Open case

As A, call open_case with examples/sample_case.json:
- tenant: 0x7078Da91596B3b8e5578Acfa5C35B911E42BC6A8
- title: "Deposit dispute — 12 Oak Street Apt 4"
- property_desc: "2-bed apartment, 12 Oak Street Apt 4, 2-year tenancy"
- lease_url: the lease fixture URL above
- deposit: 1000000000000000000

Expected: returns 0. Read back get_case(0): landlord = A, tenant = B, status "open", refund 0.
Read back get_filing_deadline(0) = get_case(0).created_at + 604800, get_filing_closed(0) = false, can_adjudicate(0) = false.

Negative tests (must fail with the exact message):
- deposit 0 → "deposit must be greater than zero"
- lease_url "ftp://example.com/lease" → "lease_url must be an http(s) URL"
- tenant = A's own address → "landlord cannot open a case against themselves"

## 2. Premature adjudication fails (immediately after open)

Call adjudicate(0) from either account, right after opening, before anything is filed.

Expected: ERROR with "adjudication is not open yet: landlord must close filing and tenant must respond, or the filing deadline must pass".
Read back can_adjudicate(0) = false. Case stays "open".

## 3. File deductions

As A, call file_deduction three times (examples/sample_deduction_0/1/2.json):
- 0: "Broken living-room window", 300000000000000000, evidence_window.md
- 1: "Repainting scuffed living-room walls", 500000000000000000, evidence_wear.md
- 2: "Unpaid final utility bill", 200000000000000000, evidence_utility.md

Expected: returns 0, 1, 2. Read back get_deduction(i): status "filed".

Negative tests:
- as B, file a deduction → "only the landlord can file deductions"
- amount 0 → "amount must be greater than zero"
- amount 2000000000000000000 (over deposit) → "deduction cannot exceed the deposit"
- evidence_url "ftp://example.com/x" → "evidence_url must be an http(s) URL"

## 4. Premature adjudication fails (deductions filed, filing not closed)

Call adjudicate(0) again — deductions are filed but the landlord has not closed filing and the tenant has not responded.

Expected: ERROR with the same "adjudication is not open yet: ..." message. can_adjudicate(0) = false.

As B, call respond(0, ...) early, before the landlord closes filing.

Expected: ERROR with "landlord has not closed filing yet" — the tenant answers the final item list, not a moving target.

## 5. Landlord closes filing

As A, call close_filing(0). Expected: null. Read back get_filing_closed(0) = true, can_adjudicate(0) = false (tenant has not responded yet).

Negative tests:
- as B, close_filing(0) → "only the landlord can close filing"
- as A, close_filing(0) again → "filing is already closed"
- as A, file_deduction(0, "Extra item", 100000000000000000, evidence_window.md URL) → "landlord has closed filing"

Call adjudicate(0) once more — filing is closed but the tenant has not responded.

Expected: ERROR with the same "adjudication is not open yet: ..." message. can_adjudicate(0) = false.

## 6. Tenant responds

As B, call respond(0, "The window was already cracked when I moved in, and the lease states normal wear is the tenant's responsibility only after a documented move-in inspection — there was none. The utility bill is for the final month and I accept it.").

Expected: null. Read back get_case(0): tenant_statement set, can_adjudicate(0) = true. Note the response is adversarial — it claims the window was pre-existing damage. Validators still upheld the window deduction, because the fetched move-in checklist shows the window intact at move-in.

Negative tests:
- as A, respond → "only the tenant can respond"
- as B, respond again → "tenant has already responded"

## 7. Adjudicate

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

## 8. Mark paid and close

As A, call mark_paid(0). Expected: null; get_case_status(0) = "paid".

Negative test:
- as B, mark_paid(0) → "only the landlord can mark a case paid" (test this before the A call, or on case 1)

As A, call close_case(0). Expected: null; get_case_status(0) = "closed".

## 9. Second case — no deductions, full refund

As A, open_case(tenant B, "Deposit — 7 Pine Lane", "1-bed flat, 7 Pine Lane, 1-year tenancy", lease fixture URL, 1000000000000000000). Expected: returns 1. Read back get_filing_deadline(1) = created_at + 604800.

Negative test: mark_paid(1) before adjudication → "only decided cases can be marked paid".

Call adjudicate(1) before filing is closed. Expected: ERROR with the same "adjudication is not open yet: ..." message.

As A, call close_filing(1). Expected: null.
As B, call respond(1, "No deductions were filed; I agree the full deposit should be refunded."). Expected: null.

Call adjudicate(1). Expected: "decided"; get_case_refund(1) = 1000000000000000000 (full refund, no deductions filed).

Negative test: close_case(1) while decided → "case cannot be closed in its current state".

Call mark_paid(1) as A, then close_case(1) as A. Expected: status "closed".

## 10. Final read-backs

- get_case_count() = 2
- get_deduction_count() = 3
- get_case(0): status "closed", refund 500000000000000000
- get_case(1): status "closed", refund 1000000000000000000
- get_case_evidence(0) names the lease terms and inspection notes
- get_filing_deadline(0) = get_case(0).created_at + 604800; same for case 1
- can_adjudicate(0) = false (case closed), can_adjudicate(1) = false (case closed)

## Expected transaction count

Deploy (1) + successful calls (open 0, 3 deductions, close_filing 0, respond 0, adjudicate 0, mark_paid 0, close 0 = 9; case 1: open, close_filing, respond, adjudicate, mark_paid, close = 6; total 15) + negative tests (22) = 38 transactions: 16 SUCCESS, 22 intentional ERRORs.
