# Author Payouts & Accounting (Design Doc)

**Status:** In Progress (Refunds Implemented)
**Owner:** Principal Software Engineer
**Date:** 2026-02-13
**Last Updated:** 2026-02-28

## Implementation Status

- ✅ Fee snapshot columns added to purchases table
- ✅ Purchase service updated to store fee snapshots at transaction time
- ✅ Settlement batch service updated to use snapshots instead of recalculating
- ✅ Refund service implemented with pre/post-payout logic
- ✅ Negative balance threshold monitoring (-$1,000)
- ✅ `Setup::StoreUser` service — creates per-author Keepr sub-accounts on `StoreUser` creation
- ✅ Purchase flow posts to per-author `Author Payable` sub-account (child of platform aggregate)
- ✅ Settlement journal generates per-author debits against sub-accounts for reconciliation
- ✅ Test coverage: 1059 total tests passing
- 🚧 Monthly settlement automation (rake task pending)
- 🚧 Store withdrawal service (ACH transfers)
- 🚧 Stripe Connect onboarding flow
- 🚧 Reconciliation tooling

## Summary

Implement author payouts using Stripe Connect with store‑scoped fee configuration and balanced double‑entry accounting. Purchases create author payable balances held for 7 days, and monthly payout batches settle those liabilities with Stripe transfers.

## Goals

- Attribute each purchase to an author (via `content.store_user_id`).
- Apply platform fee and author fee from gross in a balanced manner.
- Hold funds for 7 days before eligible payout.
- Pay authors monthly using Stripe Connect transfers.
- Keep store‑level reporting intact.

## Non‑Goals

- Multi‑currency support (USD only for now).
- Revenue splits across multiple authors.
- Store‑owner configurable platform fee.

## Data Model (Proposed)

### Store
- `platform_fee_bps` (integer, default: 1500) — platform's fee charged to store, managed by LedeWire only.
- `default_author_fee_bps` (integer, default: 2000) — fee the store charges authors (author pays this, store keeps the difference after paying platform).
- **Validation**: `default_author_fee_bps` must be >= `platform_fee_bps` to prevent negative store margins.
- **Example**: If platform fee is 15% and author fee is 20%, author receives 80%, store keeps 5% after paying platform 15%.

### Store Ledger Accounts (new)
- **Marketplace Fee Expense** (expense) — platform fee paid per transaction.
- **Store Revenue** (revenue) — store's net income (difference between author fee and platform fee).

### StoreUser
- `author_fee_bps` (integer, nullable) — overrides store's default author fee (what store charges this specific author).
- `stripe_connect_account_id` (string, nullable).
- `can_receive_payouts` (boolean, default: true) — set to false to suspend payouts (e.g., high chargeback rate).

### StoreUser Ledger Accounts (per author)
Created by `Setup::StoreUser` on `after_create` when `is_author: true`.
- **Author Payable** (liability, child of platform Author Payable) — the platform's obligation to this specific author. Credited at purchase time; debited at settlement. Enables per-author balance queries and reconciliation against the platform aggregate.
- **Author Bank** (asset, child of platform Cash) — represents this author's external Stripe Connect settlement destination.

### Content
- `store_user_id` (nullable) — author attribution.
- Each content belongs to exactly **one** store (enforced at creation).

### Purchase (required metadata)
- `author_store_user_id` (nullable, denormalized) for reporting snapshots.
- `eligible_payout_date` (datetime) — calculated as `created_at + 7.days`, used for batch eligibility.
- `journal_id` (required) — links to accounting journal entry.
- **Fee snapshots (critical for correctness):**
  - `platform_fee_cents` (bigint) — platform fee at time of purchase
  - `author_fee_cents` (bigint) — author fee at time of purchase
  - `author_net_cents` (bigint) — amount owed to author (gross - author_fee)
  - `store_net_cents` (bigint) — amount owed to store (author_fee - platform_fee)

**Why snapshots are critical:** If fee configurations change after purchase but before payout, recalculating from current rates would pay authors incorrect amounts. Snapshots preserve the exact fee split at transaction time, ensuring accurate payouts and refunds regardless of subsequent configuration changes.

### PayoutBatch (new)
- `id`, `store_id`, `period_start`, `period_end`, `status` (enum: pending, processing, completed, failed, partially_failed), `total_cents`, `journal_id` (links settlement entry), `created_at`, `completed_at`.

### AuthorPayout (new)
- `id`, `payout_batch_id`, `store_user_id`, `amount_cents`, `stripe_transfer_id`, `idempotency_key`, `status` (enum: pending, processing, completed, failed), `error_message`, `created_at`, `completed_at`.

## Target Ledger Accounts

This reflects the **two-phase accounting model**:
1. **Purchase**: Funds move immediately within platform wallets (internal)
2. **Settlement**: Monthly payouts to external bank accounts (external)

### Platform Ledger
- **Cash** (asset) — real money in/out of platform bank account (**created**)
- **Buyer Wallet** (liability) — funds held on behalf of buyers (**created**)
- **Seller Wallet** (liability) — funds held on behalf of stores, available for monthly payout (**created**, currently named "Seller Wallet Balance")
- **Author Payable** (liability) — aggregate funds owed to all authors; individual obligations tracked via per-author sub-accounts (**created**)
- **Marketplace Revenue** (revenue) — platform fee (15% default) (**created**)
- **Marketing Promotional Expenses** (expense) — promotional discounts (**created**)
- **Payment Processing Fees** (expense) — Stripe fees on wallet loading (**created**)

### StoreUser Ledger (per author, created by `Setup::StoreUser`)
- **Author Payable** (liability, child of platform Author Payable) — per-author obligation; credited at purchase, debited at settlement (**created**)
- **Author Bank** (asset, child of platform Cash) — author's Stripe Connect destination (**created**)

### Store Ledger (for store reporting)
- **Bank** (asset) — real money in/out of store's external bank account (**created**)
- **Marketplace Fees** (expense) — platform fee paid per transaction (**created**)
- **Store Revenue** (revenue) — store's net income after platform fee (**created**)

Notes:
- **Immediate flow**: Purchase → Buyer Wallet debited → Seller Wallet + Author's sub-account Author Payable (if author) + Marketplace Revenue credited
- **Monthly settlement**: Per-author Author Payable sub-accounts debited → Platform Cash credited for Stripe Connect transfers
- **Keepr hierarchy**: Child postings roll up into platform aggregate accounts automatically — the platform Author Payable balance always reflects total obligations across all authors
- **Source of truth for payout amounts**: Purchase fee snapshots (`author_net_cents`), not Keepr balances. Keepr balances serve as a cross-check/reconciliation tool.
- **No AR/AP needed**: Internal wallet transfers are immediate; only external monthly payouts are delayed

## Fee Calculation Rules

All fees are calculated from **gross** and moved into their respective accounts. Round to cents after each calculation.

**Fee Structure:**
- **Platform fee**: Charged to store by platform (e.g., 15%)
- **Author fee**: Charged to author by store (e.g., 20%)
- **Store net**: Difference between author fee and platform fee (e.g., 5%)

**Example with $100 gross sale:**
- Gross: $100.00
- Platform fee (15%): $15.00 (store pays platform)
- Author fee (20%): $20.00 (what store charges author)
- Author net: $100.00 − $20.00 = **$80.00** (author receives)
- Store net: $20.00 − $15.00 = **$5.00** (store keeps)

**Default configuration:**
- Platform fee: 1500 bps (15%)
- Default author fee: 2000 bps (20%)
- Resulting store net: 500 bps (5%)

### Fee Precedence

`author_fee_bps` (what store charges author) precedence:
1) `StoreUser.author_fee_bps` (per-author override)
2) `Store.default_author_fee_bps` (store default)

**Critical Validation**: `author_fee_bps >= platform_fee_bps + minimum_store_margin`

If author fee < platform fee, the store loses money on every transaction. Recommended minimum store margin: 100 bps (1%).

**Non-Author Content**: When `content.store_user_id` is NULL, no author fee applies. Store receives full amount after platform fee.

## Accounting Entries (Double‑Entry)

### Phase 1: Purchase (Immediate - Within Platform Wallets)

#### Scenario A: Content with Author

For a completed purchase (gross = $100, platform fee = 15%, author fee = 20%, author net = $80, store net = $5):

**Platform Ledger:**
```
Debit:  Buyer Wallet              $100  (buyer pays)
Credit: Seller Wallet (Store)     $5    (store's net: author fee - platform fee)
Credit: Author Payable            $80   (author's net: gross - author fee, held for payout)
Credit: Marketplace Revenue       $15   (platform's fee)
```

**Store Ledger (Optional - For Store Reporting):**
```
Debit:  Marketplace Fee Expense   $15   (platform fee paid)
Credit: Store Revenue             $5    (store's net income)
```

#### Scenario B: Content without Author (Store-Created)

For a completed purchase (gross = $100, platform fee = 15%, no author):

**Platform Ledger:**
```
Debit:  Buyer Wallet              $100  (buyer pays)
Credit: Seller Wallet (Store)     $85   (store receives gross - platform fee)
Credit: Marketplace Revenue       $15   (platform's fee)
```

**Store Ledger (Optional - For Store Reporting):**
```
Debit:  Marketplace Fee Expense   $15   (platform fee paid)
Credit: Store Revenue             $85   (store's net income)
```

**Key points:**
- Funds move **immediately** within platform wallet system
- Store can request monthly payout from Seller Wallet for purchases 7+ days old
- Author Payable accumulates until monthly payout batch (7+ days old)
- No AR/AP accounts needed for internal wallet transfers
- Author Payable is tracked at **platform level**, not per-store

### Phase 2: Settlement (Delayed - External Bank Transfers)

**Store Withdrawal (On-Demand or Monthly):**
```
Debit:  Seller Wallet (Store)     $X    (reduce store's platform wallet)
Credit: Platform Cash             $X    (actual bank transfer out)
```

**Author Payout Batch (Monthly, for purchases >= 7 days old):**
```
Debit:  Author Payable            $Y    (reduce liability to authors)
Credit: Platform Cash             $Y    (Stripe Connect transfers out)
```

**Totals:** Debits = Credits per phase. Platform Cash decreases only during external settlement, not at purchase time.

### Complete Money Movement (Table View)

Example: gross $100, platform fee $15, author fee $20, author net $80, store net $5.

**Phase 1: Purchase with Author (Immediate)**

| Account | Debit | Credit | Amount | Ledger | Notes |
| --- | --- | --- | --- | --- | --- |
| Buyer Wallet | $100 |  | $100 | Platform | Buyer pays gross |
| Seller Wallet (Store) |  | $5 | $5 | Platform | Store's net (20% - 15%) |
| Author Payable |  | $80 | $80 | Platform | Author's net (held 7 days) |
| Marketplace Revenue |  | $15 | $15 | Platform | Platform's 15% fee |
| **Totals** | **$100** | **$100** | | | Balanced |

**Phase 1: Purchase without Author (Immediate)**

| Account | Debit | Credit | Amount | Ledger | Notes |
| --- | --- | --- | --- | --- | --- |
| Buyer Wallet | $100 |  | $100 | Platform | Buyer pays gross |
| Seller Wallet (Store) |  | $85 | $85 | Platform | Store gets gross - platform fee |
| Marketplace Revenue |  | $15 | $15 | Platform | Platform's 15% fee |
| **Totals** | **$100** | **$100** | | | Balanced |

**Phase 2: Monthly Settlement (Delayed - External Transfers)**

| Account | Debit | Credit | Amount | Ledger | Notes |
| --- | --- | --- | --- | --- | --- |
| Seller Wallet (Store) | $5 |  | $5 | Platform | Store requests payout |
| Platform Cash |  | $5 | $5 | Platform | ACH/wire to store bank |
| Author Payable | $80 |  | $80 | Platform | Monthly batch (≥7 days old) |
| Platform Cash |  | $80 | $80 | Platform | Stripe Connect to authors |
| **Totals** | **$85** | **$85** | | | Balanced |

**Net Effect**: Platform earns $15 immediately, store can withdraw $5 monthly, authors receive $80 via monthly batch.

## Payout Flow

### Author Payout Batch (Monthly)

1. **Eligibility Query**: `Purchase.where('eligible_payout_date <= ? AND status = ? AND author_store_user_id IS NOT NULL', Time.current, 'completed').group(:author_store_user_id).sum(:price_cents)`

2. **Batch Creation**: Create `PayoutBatch` per store with status `pending`

3. **Author Payout Creation**: For each author in batch:
   - Create `AuthorPayout` record with `idempotency_key = "payout_batch_#{batch_id}_author_#{store_user_id}"`
   - Skip if `stripe_connect_account_id` is null (accumulate for next month)
   - Validate minimum payout amount (e.g., $10.00 threshold)

4. **Stripe Connect Transfer**: For each author:
   ```ruby
   Stripe::Transfer.create({
     amount: payout.amount_cents,
     currency: 'usd',
     destination: store_user.stripe_connect_account_id,
     description: "Author payout for #{period_start} to #{period_end}"
   }, {
     idempotency_key: payout.idempotency_key
   })
   ```

5. **Ledger Settlement**: Create journal entry for entire batch:
   ```ruby
   Keepr::Journal.create!(
     keepr_postings_attributes: [
       { keepr_account: platform.author_payable, amount: batch.total_cents, side: 'debit' },
       { keepr_account: platform.cash, amount: batch.total_cents, side: 'credit' }
     ],
     permanent: true
   )
   ```

6. **Update Status**: Mark individual `AuthorPayout` as `completed` or `failed`, update batch status

7. **Partial Failure Handling**: If some transfers fail, batch status = `partially_failed`, retry failed payouts in next cycle

### Store Withdrawal (Monthly)

1. **Request Validation**: Ensure store has sufficient Seller Wallet balance for purchases ≥7 days old
2. **ACH/Wire Transfer**: Initiate bank transfer to store's external account
3. **Journal Entry**: Debit Seller Wallet, Credit Platform Cash
4. **Confirmation**: Update store withdrawal record with transaction ID
5. **Frequency**: Monthly on 1st of month (same as author payouts)

## Business Rules

### Payout Thresholds
- **Minimum payout**: $10.00 (amounts below accumulate until threshold met)
- **Maximum single payout**: $50,000 (amounts above require manual review)
- **Eligibility period**: 7 days from purchase (fraud protection)
- **Payout frequency**: Monthly on 1st of month

### Settlement Schedule
- **Author payouts**: Monthly via Stripe Connect for purchases ≥7 days old
- **Store payouts**: Monthly via ACH/wire for purchases ≥7 days old
- **Processing time**: 2-5 business days for ACH, 1-2 days for Stripe Connect

### Account Closure

**Author Account Closure:**
1. Author can close account and request final payout
2. Final payout includes all purchases ≥7 days old with balance ≥$10
3. Purchases <7 days old remain in Author Payable until eligible, then require manual payout
4. Negative Author Payable balance must be resolved before closure (see Negative Balance Recovery)

**Store Account Closure:**
1. Store content becomes unavailable (separate feature, out of scope)
2. Final payout includes all Seller Wallet balance ≥7 days old
3. Ongoing author obligations (Author Payable) continue until settled by platform
4. Store cannot close with pending refunds or disputes

### Negative Balance Recovery

**For post-payout refunds resulting in negative Author Payable:**

1. **Offset from future sales** (Primary method):
   - Negative balance carries forward
   - Future payouts reduced by negative amount until zeroed
   - Example: Author owes $80, next month earns $100, receives $20 payout

2. **Maximum negative balance**: -$1,000
   - Exceeding this suspends author's content sales until resolved
   - Requires author to contact support

3. **Account closure with negative balance**:
   - Platform sends payment demand via Stripe Connect dashboard
   - 30-day payment window
   - After 30 days, written off as bad debt (Refund Expense)
   - Report to collections if amount >$500

4. **Legal recourse**:
   - Stripe Connect terms include right to recover
   - Platform can deduct from author's Stripe balance
   - Small claims court for amounts >$1,000 (legal review required)

5. **Write-off threshold**: $50
   - Negative balances <$50 with no sales for 90 days auto-written off
   - Reduces administrative overhead

### Dispute & Chargeback Protection

**7-day hold rationale:**
- Balances immediate author needs with fraud protection
- Industry standard chargeback window is 60-120 days
- 7 days catches obvious fraud/refund requests
- Platform absorbs risk for chargebacks after payout (recommended: chargeback insurance)

**Future consideration**: Extend to 30 days for high-risk categories or new authors.

### Payout Suspension

**Platform can suspend payouts (set `can_receive_payouts = false`) when:**
- Chargeback rate exceeds 1%
- Author account flagged for fraud
- Author account under investigation
- Negative balance exceeds -$1,000
- Stripe Connect account suspended/restricted

Suspended payouts accumulate in Author Payable until resolved.

## Refunds & Chargebacks

### Pre-Payout Refund (Purchase < 7 days old or not yet paid out)

**Reverse the original purchase entry:**

**With author:**
```
Debit:  Seller Wallet (Store)     $5    (reverse store credit)
Debit:  Author Payable            $80   (reduce liability)
Debit:  Marketplace Revenue       $15   (reverse revenue recognition)
Credit: Buyer Wallet              $100  (refund to buyer)
```

**Without author (store receives all non-platform revenue):**
```
Debit:  Seller Wallet (Store)     $85   (reverse store's gross - platform fee)
Debit:  Marketplace Revenue       $15   (reverse revenue recognition)
Credit: Buyer Wallet              $100  (refund to buyer)
```

**Implementation Notes:**
- Service: `Transactions::RefundPurchase`
- Uses fee snapshots from purchase record (ensures accuracy regardless of config changes)
- Validates purchase status is `completed` before refunding
- Prevents double refunds by checking `refunded` status first
- Optional `reason` parameter for audit trail

### Post-Payout Refund (Author already received payment)

**Primary Method: Clawback from Future Payouts**
```
Debit:  Seller Wallet (Store)     $5    (reverse store credit)
Debit:  Author Payable            -$80  (negative balance = amount owed by author)
Debit:  Marketplace Revenue       $15   (reverse revenue recognition)
Credit: Buyer Wallet              $100  (refund to buyer)
```

**Negative balance handling:**
- Negative Author Payable accumulates and carries forward
- Next payout batch offsets negative balance before paying author
- Maximum negative balance: -$1,000 (triggers warning and payout suspension)
- System logs warning when author balance drops below -$1,000 threshold
- Automatic suspension prevents further debt accumulation
- Balance calculation includes: completed purchases - refunded purchases - completed payouts
- Balances <$50 with no sales for 90 days → write-off
- Balances >$50 with account closure → 30-day payment demand, then write-off or collections

**Detection Logic:**
```ruby
# Calculated after each post-payout refund
author_balance = (completed_purchases - refunded_purchases - completed_payouts)
if author_balance < -100_000 # -$1,000 in cents
  logger.warn("Author negative balance exceeds threshold")
  # Future: Set can_receive_payouts = false
end
```

**Exception: Platform Bears Cost** (requires approval)
```
Debit:  Refund Expense            $80   (platform absorbs author portion)
Debit:  Seller Wallet (Store)     $5    (reverse store credit)
Debit:  Marketplace Revenue       $15   (reverse revenue recognition)
Credit: Buyer Wallet              $100  (refund to buyer)
```

**Use when:**
- Author account closed and negative balance <$50
- Fraud confirmed (author banned)
- Legal cost exceeds recovery amount
- Executive approval required for amounts >$500

### Chargeback Handling

**Overview:** Chargebacks require two separate accounting operations:
1. **Refund accounting** (handled by `RefundPurchase` service)
2. **Chargeback fee** (handled by dispute webhook - separate concern)

**Refund Accounting (Implemented):**
```ruby
# Use existing RefundPurchase service with reason tracking
Transactions::RefundPurchase.new(platform: platform)
  .refund(purchase: purchase, reason: "Chargeback - #{stripe_dispute.reason}")
```

This creates the same journal entries as post-payout refund:
- Refunds buyer's $100
- Creates negative Author Payable balance for clawback (-$80)
- Reverses store credit ($5) and platform revenue ($15)

**Chargeback Fee Accounting (Not Yet Implemented):**

Stripe charges $15 per chargeback regardless of dispute outcome. This is a separate cost that must be recorded:

```ruby
# In Webhooks::StripeDisputeHandler (future implementation)
Keepr::Journal.create!(
  date: Date.current,
  subject: "Chargeback fee: Purchase #{purchase.id} - #{dispute.reason}",
  keepr_postings_attributes: [
    { keepr_account: platform.payment_processing_fees, amount: 15.00, side: 'debit' },
    { keepr_account: platform.bank, amount: 15.00, side: 'credit' }
  ],
  permanent: true
)
```

**Total Platform Cost for Chargeback:**
- Lost revenue: $15 (reversed marketplace fee)
- Chargeback fee: $15 (Stripe charges platform)
- **Total platform loss: $30** (excluding clawback from author's -$80)

**Architecture Decision:**
- `RefundPurchase` service handles accounting mechanics only
- `StripeDisputeHandler` webhook orchestrates: refund + fee recording + reason tracking
- Keeps refund service reusable for both manual refunds and chargebacks
- Maintains single responsibility principle

**Rate Monitoring (Future):**
- Calculate chargeback rate per author: `chargebacks / total_sales`
- Automatic suspension at 1% threshold
- Dashboard for store managers to track author metrics
- Add `chargeback_reason_code` column to purchases for analysis

## Business Rules

### Payout Thresholds
- **Minimum payout**: $10.00 (amounts below accumulate until threshold met)
- **Maximum single payout**: $50,000 (amounts above require manual review)
- **Eligibility period**: 7 days from purchase (fraud protection)
- **Payout frequency**: Monthly on 1st of month

### Settlement Schedule
- **Author payouts**: Monthly via Stripe Connect for purchases ≥7 days old
- **Store payouts**: Monthly via ACH/wire for purchases ≥7 days old
- **Processing time**: 2-5 business days for ACH, 1-2 days for Stripe Connect

### Account Closure

**Author Account Closure:**
1. Author can close account and request final payout
2. Final payout includes all purchases ≥7 days old with balance ≥$10
3. Purchases <7 days old remain in Author Payable until eligible, then require manual payout
4. Negative Author Payable balance must be resolved before closure (see Negative Balance Recovery)

**Store Account Closure:**
1. Store content becomes unavailable (separate feature, out of scope)
2. Final payout includes all Seller Wallet balance ≥7 days old
3. Ongoing author obligations (Author Payable) continue until settled by platform
4. Store cannot close with pending refunds or disputes

### Negative Balance Recovery

**For post-payout refunds resulting in negative Author Payable:**

1. **Offset from future sales** (Primary method):
   - Negative balance carries forward
   - Future payouts reduced by negative amount until zeroed
   - Example: Author owes $80, next month earns $100, receives $20 payout

2. **Maximum negative balance**: -$1,000
   - Exceeding this suspends author's content sales until resolved
   - Requires author to contact support

3. **Account closure with negative balance**:
   - Platform sends payment demand via Stripe Connect dashboard
   - 30-day payment window
   - After 30 days, written off as bad debt (Refund Expense)
   - Report to collections if amount >$500

4. **Legal recourse**:
   - Stripe Connect terms include right to recover
   - Platform can deduct from author's Stripe balance
   - Small claims court for amounts >$1,000 (legal review required)

5. **Write-off threshold**: $50
   - Negative balances <$50 with no sales for 90 days auto-written off
   - Reduces administrative overhead

### Dispute & Chargeback Protection

**7-day hold rationale:**
- Balances immediate author needs with fraud protection
- Industry standard chargeback window is 60-120 days
- 7 days catches obvious fraud/refund requests
- Platform absorbs risk for chargebacks after payout (recommended: chargeback insurance)

**Future consideration**: Extend to 30 days for high-risk categories or new authors.

### Payout Suspension

**Platform can suspend payouts (set `can_receive_payouts = false`) when:**
- Chargeback rate exceeds 1%
- Author account flagged for fraud
- Author account under investigation
- Negative balance exceeds -$1,000
- Stripe Connect account suspended/restricted

Suspended payouts accumulate in Author Payable until resolved.

## Security & Compliance

- Stripe Connect handles author tax/KYC.
- Platform fee is non‑editable by stores.
- Author fee is editable by store owners only.

## Open Questions

Resolved:

- **Fee snapshots**: ✅ **IMPLEMENTED** - Added 4 columns to purchases table (platform_fee_cents, author_fee_cents, author_net_cents, store_net_cents). Critical for correctness when fee configurations change between purchase and payout/refund. Services now read snapshots instead of recalculating from current rates.
- **Incomplete Connect accounts**: Skip payouts and accumulate author payable balance until onboarding completes.
- **Owner notifications**: Not required; surfaced via reporting later.
- **Negative balance threshold**: ✅ **IMPLEMENTED** - System logs warning when author balance drops below -$1,000. Automatic suspension (setting `can_receive_payouts = false`) pending as next feature.
- **Refund implementation**: ✅ **IMPLEMENTED** - Two-path logic (pre-payout reverses entries, post-payout creates negative balance). Comprehensive test coverage including edge cases (no author, multiple refunds, threshold checks).

Pending:
