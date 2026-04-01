# Flow Testing Design

**Status:** Proposed — Under Review
**Date:** 2026-03-04
**Related:** [testing/testing-style.md](testing-style.md), [design/funding-transfer-edge-cases.md](../design/funding-transfer-edge-cases.md)

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Why Not Cucumber](#why-not-cucumber)
3. [Proposed Approach: spec/flows](#proposed-approach-specflows)
4. [Distinction from Request Specs](#distinction-from-request-specs)
5. [Directory Structure](#directory-structure)
6. [Supporting Helpers](#supporting-helpers)
7. [Purchase Flow Spec — Reference Implementation](#purchase-flow-spec--reference-implementation)
8. [Flow Catalogue](#flow-catalogue)
9. [Implementation Checklist](#implementation-checklist)

---

## Problem Statement

Several critical business flows — particularly the purchase lifecycle — are currently verified
only through manual testing. This creates three concrete problems:

1. **Slow feedback.** A developer changes `Transactions::PurchaseContent` or
   `Accounting::PurchaseLedger` and has no automated signal that buyer wallet deductions and
   merchant revenue postings still balance correctly.

2. **Coverage illusion.** `spec/requests/v1/purchases_spec.rb` creates purchases via
   `create(:purchase, :completed)` — a factory shortcut that bypasses the service and ledger
   entirely. The HTTP contract is tested; the accounting chain is not.

3. **Missing cross-actor assertions.** No existing test asserts that after a purchase:
   - the buyer's wallet balance decreased by the content price, **and**
   - the merchant's sales revenue increased by `store_net`, **and**
   - the purchase appears in buyer history with `status: completed`.

---

## Why Not Cucumber

Cucumber was evaluated and rejected for this codebase. The primary value of Gherkin is enabling
non-technical stakeholders to author or audit scenarios; this project has an all-technical team.
What Cucumber would introduce in practice:

- A mandatory indirection layer: Gherkin → step definitions → helpers → RSpec. Every bug
  trace passes through four layers.
- Step definition reuse becomes a maintenance trap as scenarios grow and steps acquire boolean
  parameters to handle variations.
- Feature files drift from the running code over time and become documentation theater —
  passing tests that no longer reflect the intent they describe.

The same scenario-oriented coverage can be achieved in plain RSpec with a clear structural
convention and no additional tooling.

---

## Proposed Approach: spec/flows

Flow specs are a named layer within the existing RSpec suite. They differ from request specs
in _mandate_, not in tooling:

| | **spec/requests/** | **spec/flows/** |
|---|---|---|
| **Tests** | Individual endpoint contract | Multi-step business scenario |
| **Actors** | One (the caller) | Multiple (buyer + merchant + platform) |
| **State setup** | Factories are fine for entities under test | Real service calls for financial state |
| **Assertions** | HTTP status, response shape, schema | Ledger balances, database state, multiple HTTP calls |
| **Failure tells you** | "This endpoint is broken" | "This business flow produces wrong outcomes" |

Both layers are necessary. The request specs stay untouched.

---

## Distinction from Request Specs

The clearest way to see the difference: the existing `purchases_spec.rb` tests `GET /v1/purchases`
responses in isolation. A flow spec for purchasing would:

1. Call `POST /v1/purchases` through HTTP
2. Read `GET /v1/wallet/balance` and assert the deduction
3. Read `GET /v1/merchant/:id/sales/summary` as the merchant and assert the revenue delta
4. Assert the purchase appears in `GET /v1/purchases`

Steps 2–4 are currently untested anywhere in the suite. The factory-built purchase in the
request spec never touches `Transactions::PurchaseContent` or `Accounting::PurchaseLedger`.

---

## Directory Structure

```
spec/
  flows/
    purchase_flow_spec.rb          ← priority 1
    wallet_funding_flow_spec.rb    ← refactor out of payment_flow_spec.rb
    author_purchase_flow_spec.rb
    refund_flow_spec.rb
    concurrent_purchase_flow_spec.rb
  requests/                        ← unchanged; owns endpoint contracts
  support/
    flow_helpers.rb                ← new; wallet seeding, balance readers
```

Flow specs carry the `:flow` metadata tag so they can be run independently:

```bash
rspec --tag flow   # smoke suite: all business flows
rspec --tag ~flow  # fast suite: contracts only
```

---

## Supporting Helpers

`spec/support/flow_helpers.rb` — included automatically for `type: :request`.

```ruby
module FlowHelpers
  # Seed a user's wallet using the real ledger path, not factory inserts.
  # This is setup plumbing, so using the service directly is intentional.
  def seed_wallet(user, amount_cents:)
    Transactions::AddFunds.new(user: user, platform: Platform.instance)
      .credit_wallet(amount_cents: amount_cents)
  end

  # Zero out a user's wallet for "insufficient funds" scenarios.
  def drain_wallet(user)
    balance = user.reload.wallet_balance.cents
    return if balance.zero?
    journal = Keepr::Journal.create!(subject: "test drain")
    journal.keepr_postings.create!(
      keepr_account: user.wallet,
      amount: balance / 100.0,
      side: 'credit'
    )
  end

  # Read the store's revenue sub-ledger balance (what the merchant earns,
  # excluding platform fees).
  def store_revenue_cents(store)
    (store.reload.revenue&.balance.to_f * 100).round
  end

  # Convenience: GET seller summary and return parsed JSON.
  def seller_summary(store, headers)
    get "/v1/merchant/#{store.id}/sales/summary", headers: headers
    json_response
  end
end

RSpec.configure do |config|
  config.include FlowHelpers, type: :request
end
```

> **Note:** `credit_wallet` is a direct-credit method that needs to be added (or verified) on
> `Transactions::AddFunds`. It should bypass the Stripe payment intent path and post directly
> to the ledger — acceptable for test setup but must not be exposed via a public API endpoint.

---

## Purchase Flow Spec — Reference Implementation

This spec is the direct replacement for manual purchase testing. Read top to bottom it tells
the story of what correct behaviour looks like across all actors.

```ruby
# spec/flows/purchase_flow_spec.rb
require 'rails_helper'

RSpec.describe "Purchase Flow", :flow, type: :request do
  let(:schema) { JSONSchemer.openapi(YAML.load_file('docs/api/v1/ledewire.yml')) }

  let(:buyer)   { create(:user) }
  let(:store)   { create(:store) }
  let(:content) { create(:content, store: store, price_cents: 1000) }

  let(:buyer_headers)  { { 'Authorization' => "Bearer #{generate_token_for(buyer)}" } }
  let(:seller_headers) { seller_auth_headers(store, permission: 'view') }

  before do
    Setup::Platform.new.setup
    seed_wallet(buyer, amount_cents: 5000)
  end

  # ── Buyer purchases content ────────────────────────────────────────────────

  describe "buyer purchases content" do
    subject(:make_purchase) do
      post "/v1/purchases", params: { content_id: content.id }, headers: buyer_headers, as: :json
    end

    it "deducts buyer wallet and increases merchant revenue atomically" do
      buyer_balance_before = buyer.reload.wallet_balance.cents
      revenue_before       = store_revenue_cents(store)

      make_purchase

      expect(response).to have_http_status(:created)
      expect(schema.schema('PurchaseResponse').valid?(json_response)).to be true

      # Buyer perspective: full price deducted
      expect(buyer.reload.wallet_balance.cents).to eq(buyer_balance_before - 1000)

      # Merchant perspective: net after 15% platform fee (1000 * 0.85 = 850)
      expect(store_revenue_cents(store)).to eq(revenue_before + 850)
    end

    it "makes the content accessible to the buyer after purchase" do
      make_purchase
      purchase_id = json_response['id']

      get "/v1/purchases/#{purchase_id}", headers: buyer_headers

      expect(response).to have_http_status(:ok)
      expect(json_response['status']).to eq('completed')
      expect(json_response['content']['id']).to eq(content.id)
    end

    it "appears in the buyer's purchase history" do
      make_purchase
      purchase_id = json_response['id']

      get "/v1/purchases", headers: buyer_headers

      expect(response).to have_http_status(:ok)
      expect(json_response.map { |p| p['id'] }).to include(purchase_id)
    end

    it "is idempotent — purchasing the same content twice does not double-charge" do
      make_purchase
      expect(response).to have_http_status(:created)

      balance_after_first = buyer.reload.wallet_balance.cents

      make_purchase  # second attempt
      expect(response).to have_http_status(:created)

      expect(buyer.reload.wallet_balance.cents).to eq(balance_after_first)
    end

    context "when the buyer has insufficient funds" do
      before { drain_wallet(buyer) }

      it "returns unprocessable and leaves all balances unchanged" do
        buyer_balance_before = buyer.reload.wallet_balance.cents
        revenue_before       = store_revenue_cents(store)

        make_purchase

        expect(response).to have_http_status(:unprocessable_content)
        expect(json_response['error']).to be_present

        expect(buyer.reload.wallet_balance.cents).to eq(buyer_balance_before)
        expect(store_revenue_cents(store)).to eq(revenue_before)
        expect(Purchase.where(user: buyer, content: content, status: :completed)).to be_empty
      end
    end
  end

  # ── Merchant sees revenue in reporting ────────────────────────────────────

  describe "merchant sees revenue in reporting after purchase" do
    it "updates the seller sales summary" do
      revenue_before = seller_summary(store, seller_headers)['total_revenue_cents']

      post "/v1/purchases", params: { content_id: content.id },
           headers: buyer_headers, as: :json
      expect(response).to have_http_status(:created)

      revenue_after = seller_summary(store, seller_headers)['total_revenue_cents']
      expect(revenue_after).to eq(revenue_before + 850)
    end

    it "increments the seller's total sales count" do
      sales_before = seller_summary(store, seller_headers)['total_sales']

      post "/v1/purchases", params: { content_id: content.id },
           headers: buyer_headers, as: :json
      expect(response).to have_http_status(:created)

      expect(seller_summary(store, seller_headers)['total_sales']).to eq(sales_before + 1)
    end
  end
end
```

---

## Flow Catalogue

The following flows should be implemented, in priority order.

### 1. Purchase Flow — `spec/flows/purchase_flow_spec.rb`
**Priority:** High — replaces current manual testing

**Key assertions:**
- Buyer wallet debited by full content price
- Store revenue credited by `store_net` (price minus platform fee)
- Purchase status `completed` in buyer history
- Repeated purchase is idempotent (no double-charge)
- Insufficient funds returns error, no accounting entries written

---

### 2. Wallet Funding Flow — `spec/flows/wallet_funding_flow_spec.rb`
**Priority:** High — refactor from `spec/requests/payment_flow_spec.rb`

**Key assertions:**
- Stripe payment intent created → `FundingTransfer` in `pending` state
- Webhook `payment_intent.succeeded` → wallet credited, transfer `completed`
- Webhook for failed payment → wallet unchanged, transfer `failed`
- Duplicate webhook is idempotent (no double-credit)

> `payment_flow_spec.rb` already covers this well but is 743 lines and growing.
> Migrating the scenario-level tests to `spec/flows/` and leaving only the security/
> edge-case tests in the original file will improve maintainability.

---

### 3. Author Purchase Flow — `spec/flows/author_purchase_flow_spec.rb`
**Priority:** Medium

**Key assertions:**
- 3-way ledger split: buyer debited full price, author `author_payable` credited `author_net`,
  store credited `store_net`, platform credited `platform_fee`
- All four components sum to the full purchase price (no money created or destroyed)
- Author revenue visible via author reporting endpoint (if one exists)

> The 3-way split in `Accounting::PurchaseLedger` is currently only covered at the unit level.
> A flow spec here catches integration mismatches between the service, the ledger, and the API.

---

### 4. Refund Flow — `spec/flows/refund_flow_spec.rb`
**Priority:** Medium

**Key assertions:**
- Purchase → refund → buyer wallet restored to pre-purchase balance
- Store revenue reversed by `store_net`
- Purchase status transitions to `refunded`
- Content no longer accessible to buyer after refund (if applicable)
- Cannot refund an already-refunded purchase

---

### 5. Concurrent Purchase Flow — `spec/flows/concurrent_purchase_flow_spec.rb`
**Priority:** Low (accounting correctness covered by model/service tests)

**Key assertions:**
- Two threads attempt simultaneous purchases from the same wallet
- Pessimistic lock (`user.lock!`) ensures only valid purchases commit
- Total deducted from wallet equals sum of successful purchases only

---

## Implementation Checklist

- [ ] Create `spec/flows/` directory
- [ ] Create `spec/support/flow_helpers.rb` with `seed_wallet`, `drain_wallet`,
      `store_revenue_cents`, `seller_summary`
- [ ] Verify or add `Transactions::AddFunds#credit_wallet` for test wallet seeding
- [ ] Write `spec/flows/purchase_flow_spec.rb` (reference implementation above)
- [ ] Add `:flow` tag filter to CI — run flow specs as a post-merge smoke suite
- [ ] Write `spec/flows/wallet_funding_flow_spec.rb`, migrating scenarios from
      `spec/requests/payment_flow_spec.rb`
- [ ] Write `spec/flows/author_purchase_flow_spec.rb`
- [ ] Write `spec/flows/refund_flow_spec.rb`
- [ ] Update `docs/testing-style.md` to document the `spec/flows/` layer and its
      distinction from request specs
