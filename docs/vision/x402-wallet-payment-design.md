# x402 Wallet Payment Design

**Status:** Proposed — Under Review
**Date:** 2026-03-23
**Related:** [docs/wallet-transaction-history-and-purchase-fee-breakdown.md](wallet-transaction-history-and-purchase-fee-breakdown.md), [docs/flow-testing-design.md](flow-testing-design.md), [docs/author-payouts-and-accounting.md](author-payouts-and-accounting.md)

---

## Table of Contents

1. [Motivation](#motivation)
2. [What is x402?](#what-is-x402)
3. [Adapting x402 for Ledewire's Fiat Wallet](#adapting-x402-for-ledewires-fiat-wallet)
4. [Request / Response Lifecycle](#request--response-lifecycle)
5. [The `ledewire-wallet` Scheme](#the-ledewire-wallet-scheme)
6. [Implementation Architecture](#implementation-architecture)
7. [Security: Nonce & Idempotency](#security-nonce--idempotency)
8. [What Does Not Change](#what-does-not-change)
9. [Out of Scope](#out-of-scope)
10. [Open Questions](#open-questions)
11. [Implementation Checklist](#implementation-checklist)

---

## Motivation

The current content purchase flow requires three sequential steps:

1. `GET /v1/checkout/state` — fetch checkout state and create a pending `Purchase`
2. Ensure the buyer has sufficient wallet balance (separate funding flow if not)
3. `POST /v1/purchases` — complete the purchase; deduct from wallet

This works well for interactive, browser-based clients. It is poorly suited to:

- **AI agents and automated clients** that need to acquire content programmatically
  in a single round-trip
- **Third-party integrations** where implementing ledewire's multi-step checkout
  state machine is a significant integration burden
- **Machine-to-machine content consumption** where a service purchases content on
  behalf of a workflow, not a human session

[x402](https://docs.x402.org) is an open HTTP payment standard (Apache-2.0) built
around the long-reserved `402 Payment Required` status code. It defines a
two-step HTTP conversation: the server declares its price in a `402` response;
the client retries with a signed payment payload in a `PAYMENT-SIGNATURE` header.
Clients that speak x402 handle this conversation automatically, making paid content
access completely transparent to the calling code.

This document proposes exposing a parallel `/v1/x402/contents/:id` endpoint family
that implements the x402 protocol using the **existing ledewire fiat wallet** as the
payment rail — with no changes to the underlying accounting, fee-splitting, or
`Transactions::PurchaseContent` service.

---

## What is x402?

x402 is a protocol, not a payment network. At its core it is three HTTP headers:

| Header | Direction | Contains |
|---|---|---|
| `PAYMENT-REQUIRED` | Server → Client | Base64-encoded JSON describing accepted payment schemes, price, and destination |
| `PAYMENT-SIGNATURE` | Client → Server | Base64-encoded JSON containing the signed payment payload |
| `PAYMENT-RESPONSE` | Server → Client | Base64-encoded JSON with settlement result; included on the final `200` response |

The canonical x402 `exact` scheme uses ERC-20/SPL token transfers on EVM or Solana
networks, with a third-party **facilitator** service that verifies and settles
on-chain transactions. The `scheme` field is explicitly extensible — any string is
valid, and the facilitator is optional. This is the extension point we use.

---

## Adapting x402 for Ledewire's Fiat Wallet

The key design decision: introduce a **`ledewire-wallet` scheme** in the
`PAYMENT-REQUIRED` response. Under this scheme:

- The "payment payload" the client sends is the user's existing **Bearer JWT** plus
  a **server-issued nonce** — no crypto wallet or signing key required
- Ledewire itself acts as the facilitator: it verifies the JWT, checks the wallet
  balance, runs the existing `Transactions::PurchaseContent` service, and returns
  the purchase result
- No external facilitator service is involved; no crypto network is required

From the client's perspective the HTTP conversation is identical to any other x402
interaction. An x402-aware client only needs to know the scheme name to handle it
correctly.

---

## Request / Response Lifecycle

```
Client                              Ledewire API (/v1/x402/contents/:id)
  │                                         │
  │── GET /v1/x402/contents/:id ───────────►│
  │                                         │  No PAYMENT-SIGNATURE header present.
  │                                         │  Server issues a nonce, stores it in
  │                                         │  Rails.cache with 5-minute TTL.
  │◄── 402 Payment Required ────────────────│
  │    PAYMENT-REQUIRED: <Base64 JSON>      │
  │    {                                    │
  │      accepts: [{                        │
  │        scheme:    "ledewire-wallet",    │
  │        price:     "$0.1000",            │
  │        network:   "ledewire:v1",        │
  │        payTo:     "store:<store-uuid>", │
  │        nonce:     "<uuid>",             │
  │        expiresAt: 1745000000            │
  │      }],                                │
  │      description: "Article title",      │
  │      mimeType:    "application/json"    │
  │    }                                    │
  │                                         │
  │  Client selects the ledewire-wallet     │
  │  scheme, builds payload with its JWT    │
  │  and the received nonce.                │
  │                                         │
  │── GET /v1/x402/contents/:id ───────────►│
  │   PAYMENT-SIGNATURE: <Base64 JSON>      │
  │   {                                     │
  │     scheme:    "ledewire-wallet",       │
  │     network:   "ledewire:v1",           │
  │     token:     "Bearer eyJ...",         │
  │     nonce:     "<uuid>",                │
  │     contentId: "<content-uuid>"         │
  │   }                                     │
  │                                         │
  │            [1] Atomic nonce consume (Rails.cache.delete)
  │            [2] JWT → User (existing AuthToken service)
  │            [3] Transactions::PurchaseContent#purchase (existing)
  │                — pessimistic row lock, balance check, ledger posting
  │                                         │
  │◄── 200 OK ──────────────────────────────│
  │    PAYMENT-RESPONSE: <Base64 JSON>      │
  │    { purchaseId: "...", status: "completed" }
  │    { ...DetailedPurchasePresenter... }  │
```

If the wallet has insufficient funds, `Transactions::PurchaseContent` returns a
`FailureResponse` which the controller renders as `422 Unprocessable Content` —
the same behaviour as the existing purchase endpoint. The x402 spec permits
non-`402` error codes for server-side failures.

---

## The `ledewire-wallet` Scheme

### `PAYMENT-REQUIRED` payload fields

| Field | Type | Description |
|---|---|---|
| `scheme` | `"ledewire-wallet"` | Custom scheme identifier |
| `price` | `"$0.0000"` | Price to 4 decimal places in USD |
| `network` | `"ledewire:v1"` | Custom network identifier for this platform |
| `payTo` | `"store:<uuid>"` | Destination; the store UUID receiving the payment |
| `nonce` | UUID string | Single-use token; expires after `NONCE_TTL` (5 minutes) |
| `expiresAt` | Unix timestamp | Epoch seconds when the nonce expires |

### `PAYMENT-SIGNATURE` payload fields

| Field | Type | Description |
|---|---|---|
| `scheme` | `"ledewire-wallet"` | Must match |
| `network` | `"ledewire:v1"` | Must match |
| `token` | `"Bearer eyJ..."` | The buyer's ledewire JWT |
| `nonce` | UUID string | The nonce received in `PAYMENT-REQUIRED` |
| `contentId` | UUID string | The content being purchased (anti-tampering check) |

### `PAYMENT-RESPONSE` payload fields (on `200`)

| Field | Type | Description |
|---|---|---|
| `purchaseId` | UUID string | The `purchases.id` record |
| `status` | `"completed"` | Purchase status |

---

## Implementation Architecture

Three new files. Zero changes to existing files.

### `app/controllers/concerns/x402_wallet_paywall.rb`

Controller concern encapsulating the full x402 conversation:

```ruby
# frozen_string_literal: true

module X402WalletPaywall
  extend ActiveSupport::Concern

  NONCE_TTL      = 5.minutes
  NONCE_CACHE_NS = "x402:nonce:"
  SCHEME         = "ledewire-wallet"
  NETWORK        = "ledewire:v1"

  # Entry point called by controllers.
  # Returns SuccessResponse(Purchase) or FailureResponse.
  def require_wallet_payment(content)
    raw = request.headers["PAYMENT-SIGNATURE"]
    return issue_payment_required(content) if raw.blank?

    payload = JSON.parse(Base64.strict_decode64(raw))
    verify_wallet_payment(content, payload)
  rescue ArgumentError, JSON::ParserError
    FailureResponse.new("Malformed PAYMENT-SIGNATURE header", status: :bad_request)
  end

  private

  def issue_payment_required(content)
    nonce = SecureRandom.uuid
    Rails.cache.write(
      "#{NONCE_CACHE_NS}#{nonce}",
      content.id.to_s,
      expires_in: NONCE_TTL
    )

    payment_required = {
      accepts: [{
        scheme:    SCHEME,
        price:     format_x402_price(content.price_cents),
        network:   NETWORK,
        payTo:     "store:#{content.store_id}",
        nonce:     nonce,
        expiresAt: NONCE_TTL.from_now.to_i
      }],
      description: content.title,
      mimeType:    "application/json"
    }

    response.headers["PAYMENT-REQUIRED"] = Base64.strict_encode64(payment_required.to_json)
    FailureResponse.new("Payment required", status: :payment_required)
  end

  def verify_wallet_payment(content, payload)
    unless payload["scheme"] == SCHEME && payload["network"] == NETWORK
      return FailureResponse.new("Unsupported payment scheme or network", status: :bad_request)
    end

    unless payload["contentId"] == content.id.to_s
      return FailureResponse.new("Content ID mismatch in payment payload", status: :bad_request)
    end

    # Consume the nonce. Solid Cache's delete is backed by a SQL DELETE —
    # the first caller removes the row; any concurrent caller gets nil on read.
    stored_content_id = Rails.cache.read("#{NONCE_CACHE_NS}#{payload["nonce"]}")
    return issue_payment_required(content) if stored_content_id.nil?

    was_deleted = Rails.cache.delete("#{NONCE_CACHE_NS}#{payload["nonce"]}")
    return issue_payment_required(content) unless was_deleted

    unless stored_content_id == content.id.to_s
      return FailureResponse.new("Nonce was not issued for this content", status: :bad_request)
    end

    Authentication::AuthToken.new
      .user_from_token(payload["token"].to_s.split.last)
      .bind do |user|
        Transactions::PurchaseContent.new(user: user).purchase(content: content).bind do |purchase|
          settlement = { purchaseId: purchase.id, status: purchase.status }
          response.headers["PAYMENT-RESPONSE"] = Base64.strict_encode64(settlement.to_json)
          SuccessResponse.new(purchase)
        end
      end
  end

  def format_x402_price(cents)
    format("$%.4f", cents / 100.0)
  end
end
```

### `app/controllers/v1/x402/contents_controller.rb`

```ruby
# frozen_string_literal: true

module V1
  module X402
    class ContentsController < ApplicationController
      include X402WalletPaywall

      def show
        content = Content.publicly_visible.find_by(id: params[:id])
        return render_response(FailureResponse.new("Content not found", status: :not_found)) \
          unless content

        payment_result = require_wallet_payment(content)
        return render_response(payment_result) unless payment_result.success?

        render_response(
          SuccessResponse.new(DetailedPurchasePresenter.new(payment_result.result).present)
        )
      end
    end
  end
end
```

### Route addition (`config/routes.rb`)

Inside the existing `namespace :v1` block:

```ruby
namespace :x402 do
  resources :contents, only: [:show]
end
```

This yields `GET /v1/x402/contents/:id`.

---

## Security: Nonce & Idempotency

### Threat: Replay attack

A captured `PAYMENT-SIGNATURE` must not be reusable. Mitigation is two-layered:

**Layer 1 — Nonce (primary guard):**
The nonce is written to Rails.cache with a 5-minute TTL. Consumption is a
read-then-delete against Solid Cache (PostgreSQL-backed). Because Solid Cache's
`delete` executes a SQL `DELETE` statement, only one concurrent caller will
observe the row present — a second `delete` call returns `false` and the request
receives a fresh `402`. The narrow TOCTOU window between `read` and `delete` is
acceptable given the second idempotency guard below.

**Layer 2 — `PurchaseContent` service (final guard):**
The existing service acquires a pessimistic row lock (`@user.lock!`) before
reading wallet balance, and uses find-or-create semantics: if a `Purchase` record
already exists at `status: :completed` for this user+content, it returns
`SuccessResponse(existing_purchase)` without a second deduction. This is
identical to the protection the existing `/v1/purchases` endpoint relies on.

### Threat: JWT interception

The `PAYMENT-SIGNATURE` payload contains a Bearer JWT, equivalent in sensitivity
to an `Authorization` header. The nonce binds the JWT to a specific
content+time-window, so an intercepted payload cannot be reused after the nonce
expires (5 minutes) or is consumed. HTTPS must be enforced in all environments.

### Threat: Price manipulation

The price is not taken from the client payload. The server re-reads
`content.price_cents` from the database inside `PurchaseContent#purchase` — the
same as the existing endpoint. The `contentId` field in the client payload is
used only as a tamper-detection sanity check; the authoritative price always
comes from the server.

---

## What Does Not Change

- `Transactions::PurchaseContent` — identical code path, fully reused
- `Accounting::PurchaseLedger` — no changes; Keepr journal entries identical
- `Purchase` model and fee columns — unchanged
- Author fee attribution and payout batches — unchanged
- `DetailedPurchasePresenter` — reused directly for the `200` response body
- Existing `/v1/purchases` and `/v1/checkout` endpoints — untouched

---

## Out of Scope

- **Crypto / stablecoin payments.** The `ledewire-wallet` scheme is fiat-only. A
  future document would design the on-chain `exact` scheme with a crypto
  facilitator if that direction is chosen.
- **Merchant / store authentication.** The `v1/x402` namespace is buyer-facing only.
- **Refunds via x402.** Refunds continue through the existing
  `Transactions::RefundPurchase` path; there is no x402 refund protocol.
- **x402 Bazaar discovery.** The Bazaar extension (discoverability metadata) could
  be added to the `PAYMENT-REQUIRED` response in future but is not designed here.
- **Free content short-circuit.** Content with `price_cents: 0` should bypass the
  paywall. The existing `PurchaseContent` service handles this at the service
  layer; the controller should short-circuit to avoid the unnecessary `402`
  round-trip. Exact behaviour is deferred to Open Question 4.

---

## Open Questions

1. **Nonce atomicity under high concurrency.** The read+delete two-step is safe
   for Solid Cache (PostgreSQL) but is not a true atomic compare-and-delete. If
   traffic patterns warrant it, a DB-backed `x402_nonces` table with a unique
   index and a `consumed_at` column would provide full ACID guarantees without
   relying on cache semantics.

2. **Cache backend assumption.** The design assumes Solid Cache (PostgreSQL). If
   `Rails.cache` is ever swapped to Redis, `SET NX` / `GETDEL` commands would
   provide stronger atomicity and should replace the read+delete pattern.

3. **`DetailedPurchasePresenter` vs `ContentWithAccessPresenter` for `200`.** The
   x402 response currently returns the full purchase detail schema. It may be more
   appropriate to return `ContentWithAccessPresenter` (content body + purchase
   confirmation) so the buyer receives the content in the same response —
   removing the need for a follow-up `GET /v1/contents/:id`.

4. **Free content behaviour.** Should `GET /v1/x402/contents/:id` for zero-price
   content return `200` directly without a `402` round-trip, or should it create a
   `Purchase` record and return it? Consistency with the existing checkout flow
   suggests creating the `Purchase` silently and returning `200` immediately.

---

## Implementation Checklist

- [ ] Create `app/controllers/concerns/x402_wallet_paywall.rb`
- [ ] Create `app/controllers/v1/x402/contents_controller.rb`
- [ ] Add `namespace :x402 { resources :contents, only: [:show] }` inside `namespace :v1` in `config/routes.rb`
- [ ] Add `GET /v1/x402/contents/{id}` to `docs/api/v1/ledewire.yml` (request and response schemas)
- [ ] Request spec: `spec/requests/v1/x402/contents_spec.rb`
  - [ ] `GET` without `PAYMENT-SIGNATURE` → 402, valid `PAYMENT-REQUIRED` header present
  - [ ] `GET` with valid JWT + nonce → 200, `PAYMENT-RESPONSE` header, correct purchase returned
  - [ ] `GET` with replayed nonce → 402 (nonce already consumed)
  - [ ] `GET` with expired nonce (cache TTL elapsed) → 402
  - [ ] `GET` with insufficient wallet balance → 422
  - [ ] `GET` with invalid JWT in signature → 401
  - [ ] `GET` for already-purchased content → 200, idempotent (no double-charge)
  - [ ] `GET` for unknown content → 404
- [ ] Flow spec: `spec/flows/x402_wallet_purchase_flow_spec.rb`
  - [ ] Full round-trip: fund wallet → x402 purchase → assert ledger balance
- [ ] Resolve Open Question 3 (response shape) before implementing the controller
- [ ] Confirm `Rails.cache` is Solid Cache in all environments before shipping
