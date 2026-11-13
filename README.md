# DepositCourt

On-chain adjudication for rental deposit disputes. A landlord files itemized deductions against a tenant's deposit; validators fetch the lease terms and each deduction's evidence themselves, rule on every item, and the refund is a deterministic on-chain sum of the upheld items.

## Why this exists

Deposit fights are the most common landlord-tenant dispute, and they are almost always decided on two things: what the lease actually says, and whether the damage is real damage or normal wear and tear. Both are checkable from documents. DepositCourt puts the check on-chain: the lease and the inspection evidence are fetched by validators, not taken from either party's word.

## How it works

1. **open_case** — the landlord opens a case against a tenant, naming the lease URL and the deposit amount. The lease URL is declared by the landlord at creation, so validators know exactly which document governs the case. Opening sets a filing deadline 7 days out (`filing_deadline = created_at + 604800`).
2. **file_deduction** — the landlord files one deduction per line item: a description, an amount, and an evidence URL (inspection notes, photo page, final bill). Each item is capped at the deposit amount. Filing stops when the landlord closes filing or the deadline passes.
3. **close_filing** — the landlord (only the landlord) declares the item list final. No more deductions can be filed after this.
4. **respond** — the tenant files one statement, after the landlord has closed filing, so the tenant answers the final item list rather than a moving target. It is a claim, not proof.
5. **adjudicate** — opens only once the filing boundary is met: either intake is final (landlord closed filing AND tenant responded) or the 7-day filing deadline has passed, so a non-responsive party cannot stall the case forever. Before that, adjudication fails — no empty or half-filed case can be decided early. Anyone may trigger adjudication once it is open; the verdict is decided by validators, and restricting the trigger to one party would let that party stall the case forever. Validators fetch the lease and every deduction's evidence page themselves, then decide each item:
   - **upheld** only if the fetched evidence shows damage beyond normal wear and tear that happened during the tenancy, and the lease makes the tenant liable;
   - **rejected** for normal wear (scuffs, fading, minor marks after a long tenancy), pre-existing damage, or no evidence at all — no proof, no deduction.
   - The refund is then computed deterministically on-chain: `refund = deposit - sum(upheld items)`, floored at zero. No LLM ever touches the money math.
6. **mark_paid** — the landlord confirms the refund was paid.
7. **close_case** — the landlord closes the case (withdraw an open case, or archive a paid one).

## The filing boundary

The boundary is enforced on both sides of adjudication. Before it opens, `adjudicate` fails with "adjudication is not open yet", and `can_adjudicate` reads false. After the landlord closes filing, no further deduction can be filed; before the landlord closes filing, the tenant cannot respond; after the 7-day deadline, no filing or response is accepted at all. The deadline is a fixed contract constant (`FILING_WINDOW_SECONDS = 604800`), not a per-case parameter, so no party can shorten it. The two opening conditions are visible on-chain: `get_filing_deadline` returns the deadline, `get_filing_closed` the landlord's declaration, and `can_adjudicate` the gate itself.

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

## Verification (v2)

- Deployed contract: 0x94F7Cf96FA603Bb7df38084d77BC92581D5802B1 ([explorer](https://explorer-studio.genlayer.com/address/0x94F7Cf96FA603Bb7df38084d77BC92581D5802B1)), deployed 2026-11-13 on GenLayer Studio.
- Studio verification 2026-11-13: 38 transactions — 16 SUCCESS, 22 intentional ERRORs (deploy 1 + 15 successful calls + 22 negative tests), every step matching `examples/test_flow.md`.
- Premature adjudication fails at all four attempts (right after opening, after deductions, after close before response, and on the second case before close); `can_adjudicate` reads false until the tenant responds, then true.
- `filing_deadline` reads back as `created_at + 604800` on both cases; the 7-day deadline fallback is enforced by the same gate as the mutual-completion path (it cannot be fast-forwarded in Studio, so the live demo exercises mutual completion).
- Verdicts unchanged from v1: window 0.3 upheld, repainting 0.5 rejected as normal wear, utility 0.2 upheld, refund 0.5 of 1 GEN; the no-deduction case refunds in full. 22 negative tests pass.
- v1 contract 0x6C8cd51B936280243E0382613f75BA21B5e5253C was superseded by v2: v1 allowed adjudication immediately after opening, before either party had finished filing. v2 adds the filing boundary described above; the verdict logic is unchanged.
- Studio UI note: the number input shows large u256 values as float approximations (1 GEN displays as 999999984306749440); the submitted value is exact — read-backs confirm deposit 1000000000000000000 on-chain.
- Known boundary (same class as before, not fixed): reading a nonexistent case or deduction fails with a raw `KeyError` (TreeMap lookup) instead of a friendly message — still rejected, not a valid read.
