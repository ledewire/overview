# Wallet Transaction History & Purchase Fee Breakdown

**Status:** Proposed — Under Review
**Date:** 2026-03-04
**Related:** [testing/flow-testing-design.md](../testing/flow-testing-design.md), [design/author-payouts-and-accounting.md](author-payouts-and-accounting.md)

---

## Table of Contents

1. [Motivation](#motivation)
2. [Endpoint 1 — GET /v1/wallet/transactions](#endpoint-1--get-v1wallettransactions)
3. [Endpoint 2 — Fee Breakdown on GET /v1/merchant/:store_id/sales/:id](#endpoint-2--fee-breakdown-on-get-v1merchantstore_idsalesid)
4. [Nuances and Edge Cases](#nuances-and-edge-cases)
5. [Effect on Flow Tests](#effect-on-flow-tests)
6. [Out of Scope](#out-of-scope)
7. [Implementation Checklist](#implementation-checklist)

---

## Motivation

Two gaps surface repeatedly when investigating balance discrepancies — in flow
specs, in manual testing, and in production support:

**Gap 1: `GET /v1/wallet/balance` is opaque.**
It returns one number. When a balance looks wrong there is no API path to ask
"what events produced this number?" — you need direct database access. This
makes both automated diagnosis and user-facing balance explainability impossible.

**Gap 2: No merchant-facing purchase detail endpoint exposes the fee split.**
The buyer-facing `GET /v1/purchases/:id` intentionally omits fee columns —
buyers should see what they paid, not how the platform divided it. But merchants
(and internal tooling) have a legitimate need to see `platform_fee_cents`,
`store_net_cents`, and `author_net_cents` for a given transaction. All three are
stored on the `Purchase` record; there is no merchant API path to retrieve them.
The flow specs currently call `Purchase.find(id)` directly to assert the split —
a sign that the information should be in an API response accessible to the store.

---

## Endpoint 1 — GET /v1/wallet/transactions

### Route

```
GET /v1/wallet/transactions
Authorization: Bearer <user JWT>
```

### Purpose

Returns a unified ledger of all events that have changed the authenticated
user's wallet balance, newest first. Clients can use this to render a
statement view and to explain a balance to the user without requiring a
support query.

### Response shape

```json
[
  {
    "id": "uuid",
    "type": "debit",
    "reason": "purchase",
    "amount_cents": 1000,
    "balance_after_cents": 4000,
    "status": "completed",
    "reference_id": "<purchase uuid>",
    "description": "Purchase: Some Content Title",
    "occurred_at": "2026-03-04T12:00:00Z"
  },
  {
    "id": "uuid",
    "type": "credit",
    "reason": "wallet_funding",
    "amount_cents": 5000,
    "balance_after_cents": 5000,
    "status": "completed",
    "reference_id": "<funding_transfer uuid>",
    "description": "Wallet top-up",
    "occurred_at": "2026-03-04T11:55:00Z"
  }
]
```

### Field definitions

| Field | Type | Notes |
|---|---|---|
| `id` | string (UUID) | Internal ID of the transaction entry |
| `type` | `"credit"` \| `"debit"` | Direction relative to the user's wallet |
| `reason` | string | See reason values below |
| `amount_cents` | integer | Always positive; direction is expressed by `type` |
| `balance_after_cents` | integer | Running balance after this event — see nuances |
| `status` | string | `completed`, `pending`, `failed`, `cancelled` |
| `reference_id` | string (UUID) | ID of the source record (Purchase or FundingTransfer) |
| `description` | string | Human-readable label for display |
| `occurred_at` | ISO 8601 | `created_at` of the source record |

### Reason values

| `reason` | `type` | Source model |
|---|---|---|
| `wallet_funding` | `credit` | `FundingTransfer` (direction: inbound, status: completed) |
| `purchase` | `debit` | `Purchase` (status: completed) |
| `refund` | `credit` | `Purchase` (status: refunded) |
| `funding_failed` | — | `FundingTransfer` (status: failed) — **excluded from list**, see nuances |

### Implementation notes

The cleanest implementation assembles two queries and merges them in
application code:

```ruby
# In a new WalletTransactionService or directly in the controller:

credits = user.funding_transfers
              .where(status: :completed)
              .map { |ft| WalletTransactionPresenter.from_funding_transfer(ft) }

debits  = user.purchases
              .where(status: [:completed, :refunded])
              .includes(:content)
              .map { |p|  WalletTransactionPresenter.from_purchase(p) }

(credits + debits).sort_by { |t| t[:occurred_at] }.reverse
```

A single SQL UNION is possible but the mixed types make it awkward; application
merge is simpler to read and maintain at current data volumes. Revisit with
pagination if row counts grow.

---

## Endpoint 2 — Fee Breakdown on GET /v1/merchant/:store_id/sales/:id

### Route

```
GET /v1/merchant/:store_id/sales/:id
Authorization: Bearer <merchant JWT>  |  X-API-Key: <api key>
```

This is a **new** route nested under the existing `sales` collection in the
`scope :merchant` / `scope ":store_id"` block. Using `sales/:id` is consistent
with the existing `GET /v1/merchant/:store_id/sales` list and
`GET /v1/merchant/:store_id/sales/summary` — clients and documentation readers
don’t need to know that “sales” and “purchases” refer to the same underlying
record. The buyer-facing `GET /v1/purchases/:id` is **not** modified; buyers
see only `amount_cents`.

### Response shape

```json
{
  "id": "uuid",
  "content": { ... },
  "buyer": { ... },
  "seller": { ... },
  "amount_cents": 1000,
  "fees": {
    "platform_fee_cents": 150,
    "store_net_cents": 850,
    "author_net_cents": 0
  },
  "status": "completed",
  "timestamp": "2026-03-04T12:00:00Z"
}
```

### Field notes

- The `fees` object is present on all purchase statuses. Values will be `0`
  (not `null`) for `pending` and `failed` purchases — the ledger has not been
  posted yet, but the columns default to `0` in the database (see nuances).
- `author_net_cents` is `0` for content without an author — not `null`. Avoids
  forcing clients to null-check a field that always exists.
- This endpoint requires `owner` role or API key. `is_author` users without owner permission receive
  `403 Forbidden`. Author-scoped reporting is available on the `GET /v1/merchant/:store_id/sales/*`
  and `GET /v1/merchant/:store_id/buyers` endpoints.
- A list equivalent (`GET /v1/merchant/:store_id/purchases`) is out of scope
  for this PR; add it once the detail endpoint is stable.

---

## Nuances and Edge Cases

### `balance_after_cents` calculation

Computing a true running balance requires ordering all events chronologically
and summing. This is straightforward but has two edge conditions:

1. **Pending funding transfers.** A `FundingTransfer` with `status: pending`
   has not yet affected the Keepr ledger — the wallet balance has not changed.
   These should **not** appear in the transactions list (they would produce a
   fake entry with no corresponding balance movement). Only `completed`
   transfers appear.

2. **Failed and cancelled transfers.** Similarly excluded. A future enhancement
   could add a separate `GET /v1/wallet/funding-transfers` endpoint to show the
   full Stripe payment history including failures — but that is a different
   concern from wallet balance history.

3. **Refunds.** A refunded purchase should appear as a `credit` entry with
   `reason: "refund"`. The `amount_cents` is the original purchase price (what
   was returned to the wallet), not a separate fee calculation.

4. **The `credit_wallet` test helper.** `Transactions::AddFunds#credit_wallet`
   (added for flow test seeding) creates a `FundingTransfer` with `vendor: stripe`
   and immediately completes it. This means it **will appear** in the
   transactions list as a `wallet_funding` credit. In tests this is fine — the
   seed is intentional. In production `credit_wallet` must never be called from
   a user-facing path (it is marked with a comment to that effect).

### Fee fields on non-completed purchases

`platform_fee_cents`, `store_net_cents`, and `author_net_cents` are populated
inside the `ActiveRecord::Base.transaction` block in `Transactions::PurchaseContent`
at the same time as the `status` is set to `completed`. For any purchase in
`pending` or `failed` state these columns are `0` in the database (the Rails
default), not `null`.

**Decision:** Return the values as-is. Clients can treat `0` on a non-completed
purchase as "not yet settled" — they should be showing status, not fee detail,
for an in-progress purchase anyway. Do not add conditional `nil` logic; it
would complicate serialisation for no real benefit.

### OpenAPI schema for the new merchant endpoint

A new `MerchantSaleResponse` schema (or a composed variant of
`PurchaseResponse` with an additional `fees` property) should be added to
`docs/api/v1/ledewire.yml`. The existing `PurchaseResponse` schema used by the
buyer endpoint is **not** modified — keeping the two schemas separate makes the
authentication boundary explicit in the contract.

The flow spec that validates the buyer purchase detail:

```ruby
expect(schema.schema("PurchaseResponse").valid?(purchase_detail)).to be true
```

continues to pass unchanged. A new schema assertion is added for the merchant
endpoint once its schema is defined.

### Authentication and ownership

The two endpoints use different authentication mechanisms, matching the existing
pattern in their respective parts of the API:

- **`GET /v1/wallet/transactions`** uses `authenticate_user` (user JWT) — the
  same as `GET /v1/wallet/balance` and the other buyer-facing endpoints.
  The query must be scoped to `current_user` only. No cross-user leakage is
  possible if built from `current_user.funding_transfers` and
  `current_user.purchases`, but this must be verified with a request spec
  context: "cannot view another user's transactions".

- **`GET /v1/merchant/:store_id/sales/:id`** uses `authenticate_merchant` —
  the same mechanism used by all other `scope :merchant` endpoints
  (`seller_summary`, `seller_content_purchases`, etc.). This supports both
  merchant JWT and API key authentication. The owner-only permission gate then
  applies on top of successful merchant authentication, consistent with other
  reporting actions in `V1::Merchant::ReportingController`.

---

## Effect on Flow Tests

Once these changes are live, two flow spec improvements become possible:

**1. Replace direct model access in fee assertions**

Currently [spec/flows/purchase_flow_spec.rb](../spec/flows/purchase_flow_spec.rb)
calls `Purchase.find(id)` to read fee columns:

```ruby
# Current — bypasses the API
purchase = Purchase.find(JSON.parse(response.body)["id"])
expect(purchase.platform_fee_cents).to eq(150)
```

After this change, the flow spec should call the **merchant** endpoint using
`seller_auth_headers`:

```ruby
# Preferred — asserts through the merchant API response
get "/v1/merchant/#{store.id}/sales/#{purchase_id}", headers: seller_headers
merchant_detail = JSON.parse(response.body)
expect(merchant_detail.dig("fees", "platform_fee_cents")).to eq(150)
```

This is the stronger assertion: it confirms the data reaches the merchant
client through the correct authenticated channel, not just that it exists in
the database. The buyer-facing assertion (`GET /v1/purchases/:id`) continues
to validate only `amount_cents` — confirming the fee split is *not* leaked.

**2. Add a `wallet_debited_after_purchase` assertion using the transactions list**

```ruby
get "/v1/wallet/transactions", headers: buyer_headers
transactions = JSON.parse(response.body)

debit = transactions.find { |t| t["reference_id"] == purchase_id }
expect(debit["type"]).to eq("debit")
expect(debit["amount_cents"]).to eq(1000)
expect(debit["reason"]).to eq("purchase")
```

This closes the loop between the purchase endpoint and the wallet endpoint in
a single flow spec, which is exactly what a cross-actor assertion should do.

---

## Out of Scope

**Per-content purchase list for merchants (`GET /v1/merchant/:store_id/content/:id/purchases`)**
This is the third gap identified — no endpoint to drill from a content' total
sales figure down to individual transactions. It is more work (new route,
controller, presenter, schema) and belongs in a separate design doc/PR. The
two endpoints above are additive with low risk and should ship first.

**Pagination**
Not required for the initial implementation at current data volumes. The
`funding_transfers` and `purchases` associations are already scoped to one
user. Revisit when any user is expected to exceed ~200 combined transactions.

**Webhook events / Stripe event log**
Out of scope. The transactions list reflects settled wallet movements only —
it is not a general audit log of Stripe events.

---

## Implementation Checklist

### Endpoint 1 — Wallet transactions

- [ ] Add `GET /v1/wallet/transactions` route in `config/routes.rb`
- [ ] Add `transactions` action to `V1::WalletController`
- [ ] Create `WalletTransactionPresenter` with `from_funding_transfer` and
      `from_purchase` class methods
- [ ] Implement `running_balance` calculation (sort chronologically, accumulate)
- [ ] Add `WalletTransactionItem` schema to `docs/api/v1/ledewire.yml`
- [ ] Add `GET /v1/wallet/transactions` operation to OpenAPI spec
- [ ] Write `spec/requests/v1/wallet_transactions_spec.rb` covering:
  - [ ] Authenticated user sees own credits and debits
  - [ ] Pending / failed transfers excluded
  - [ ] Refunded purchase appears as credit
  - [ ] Unauthenticated request returns 401
  - [ ] Cannot view another user's transactions
- [ ] Update `spec/flows/purchase_flow_spec.rb` to assert purchase debit
      appears in the transactions list (replaces direct model queries)

### Endpoint 2 — Merchant sale detail with fee breakdown

- [ ] Add `GET /v1/merchant/:store_id/sales/:id` route in `config/routes.rb`
      inside the existing `scope :merchant` / `scope ":store_id"` block
- [ ] Add `show` action to `V1::Merchant::ReportingController` (or a new
      `V1::Merchant::SalesController`)
- [ ] Apply owner-only permission gate (same as `seller_summary`)
- [ ] Create `MerchantPurchasePresenter` (or extend `DetailedPurchasePresenter`
      with a `fees` key) — do **not** add `fees` to the buyer-facing presenter
- [ ] Add `MerchantSaleResponse` schema to `docs/api/v1/ledewire.yml` under
      `/v1/merchant/{store_id}/sales/{id}`
- [ ] Write `spec/requests/v1/merchant_sales_spec.rb` covering:
  - [ ] Owner can retrieve sale detail with fee split
  - [ ] Non-owner store manager receives 403
  - [ ] Sale belonging to a different store returns 404
  - [ ] Unauthenticated request returns 401
  - [ ] Buyer-facing `GET /v1/purchases/:id` response does **not** include `fees`
- [ ] Update `spec/flows/purchase_flow_spec.rb` author fee assertions to
      call `GET /v1/merchant/:store_id/sales/:id` with seller headers
      and read from `dig("fees", ...)` rather than `Purchase.find`
