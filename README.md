# DepositCourt

On-chain adjudication for rental deposit disputes. A landlord files itemized deductions against a tenant's deposit; validators fetch the lease terms and each deduction's evidence themselves, rule on every item, and the refund is a deterministic on-chain sum of the upheld items.

## Why this exists

Deposit fights are the most common landlord-tenant dispute, and they are almost always decided on two things: what the lease actually says, and whether the damage is real damage or normal wear and tear. Both are checkable from documents. DepositCourt puts the check on-chain: the lease and the inspection evidence are fetched by validators, not taken from either party's word.

## How it works

1. **open_case** — the landlord opens a case against a tenant, naming the lease URL and the deposit amount. The lease URL is declared by the landlord at creation, so validators know exactly which document governs the case.
2. **file_deduction** — the landlord files one deduction per line item: a description, an amount, and an evidence URL (inspection notes, photo page, final bill). Each item is capped at the deposit amount.
3. **respond** — the tenant files one statement for the case (optional). It is a claim, not proof.
4. **adjudicate** — validators fetch the lease and every deduction's evidence page themselves, then decide each item:
   - **upheld** only if the fetched evidence shows damage beyond normal wear and tear that happened during the tenancy, and the lease makes the tenant liable;
   - **rejected** for normal wear (scuffs, fading, minor marks after a long tenancy), pre-existing damage, or no evidence at all — no proof, no deduction.
   - The refund is then computed deterministically on-chain: `refund = deposit - sum(upheld items)`, floored at zero. No LLM ever touches the money math.
5. **mark_paid** — the landlord confirms the refund was paid.
6. **close_case** — the landlord closes the case (withdraw an open case, or archive a paid one).

Every verdict carries a one-sentence evidence note stored on-chain, naming the fetched material that decided it. A case with no deductions adjudicates to a full refund.

## What validators actually fetch

- The lease terms (declared at case creation).
- Each deduction's evidence page (declared per item at filing).
- Nothing else. No party-supplied text is treated as proof; the tenant's response is read as a claim and weighed against the fetched evidence.

## The demo case

`examples/` holds the full demo flow, with fixture files under `examples/fixtures/`:

| Item | Description | Amount | Expected |
| --- | --- | --- | --- |
| 0 | Broken living-room window | 0.3 GEN | upheld — inspection notes show the crack, move-in checklist shows the window intact |
| 1 | Repainting scuffed living-room walls | 0.5 GEN | rejected — only light scuffs after 2 years, which the lease calls normal wear |
| 2 | Unpaid final utility bill | 0.2 GEN | upheld — final bill, utilities are the tenant's responsibility per the lease |

Deposit 1 GEN, upheld total 0.5 GEN, refund 0.5 GEN. The tenant's response claims the window was already cracked at move-in and accepts the utility bill; validators still upheld the window deduction, because the fetched move-in checklist shows the window intact. A second case with no deductions adjudicates to a full refund.

## Files

- `contracts/deposit_court.py` — the contract
- `examples/test_flow.md` — step-by-step verification, including negative tests
- `examples/sample_case.json`, `examples/sample_deduction_*.json` — exact inputs for the demo
- `examples/fixtures/` — lease terms and inspection evidence, as plain fetchable files

## Deploy

1. Upload `contracts/deposit_court.py` to GenLayer Studio and deploy.
2. Follow `examples/test_flow.md` with two test accounts (landlord, tenant).
3. Verify every write with a view read-back (`get_case`, `get_deduction`, `get_case_refund`).
