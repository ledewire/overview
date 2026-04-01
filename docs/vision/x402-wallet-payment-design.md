# x402 Wallet Payment Design

**Status:** Implemented — Reviewed against x402 v2 spec
**Date:** 2026-03-23
**Related:** [design/wallet-transaction-history-and-purchase-fee-breakdown.md](../design/wallet-transaction-history-and-purchase-fee-breakdown.md), [testing/flow-testing-design.md](../testing/flow-testing-design.md), [design/author-payouts-and-accounting.md](../design/author-payouts-and-accounting.md)

---

## Table of Contents

1. [Motivation](#motivation)
2. [What is x402?](#what-is-x402)
3. [Adapting x402 for Ledewire's Fiat Wallet](#adapting-x402-for-ledewires-fiat-wallet)
4. [Request / Response Lifecycle](#request--response-lifecycle)
5. [The `ledewire-wallet` Scheme](#the-ledewire-wallet-scheme)
6. [Implementation Architecture](#implementation-architecture)
7. [Browser Extension / Custom Interceptor Integration](#browser-extension--custom-interceptor-integration)
8. [Security: Nonce & Idempotency](#security-nonce--idempotency)
9. [What Does Not Change](#what-does-not-change)
10. [Out of Scope](#out-of-scope)
11. [x402 Spec Compliance Notes](#x402-spec-compliance-notes)
12. [Open Questions](#open-questions)
13. [Implementation Checklist](#implementation-checklist)

---

## Motivation

Ledewire has two purchase paths, each for a different client context:

**`POST /v1/purchases` (authenticated REST)** — for clients that already have a
Ledewire buyer session (JWT). This is the path used by the Ledewire web app and the
`@ledewire/gate` embed. The buyer is logged in; identity is established before the
request; one round-trip to purchase.

**`GET /v1/x402/contents/:id` (this document)** — for clients that have **no
pre-existing Ledewire session**: AI agents, Cloudflare Workers, browser extensions,
CMS plugins, and any automated pipeline that consumes content programmatically. These
clients cannot hold a session between requests and cannot navigate a login flow.

x402 solves the unauthenticated client problem elegantly. The old checkout state
machine required three steps even for these clients:

1. `GET /v1/checkout/state` — fetch checkout state and create a pending `Purchase`
2. Ensure the buyer has sufficient wallet balance (separate funding flow if not)
3. `POST /v1/purchases` — complete the purchase; deduct from wallet

This is an unrealistic ask for an AI agent or a Cloudflare Worker. [x402](https://docs.x402.org)
is an open HTTP payment standard (Apache-2.0) built around the long-reserved
`402 Payment Required` status code. It defines a two-step HTTP conversation: the
server declares its price in a `402` response; the client retries with a signed
payment payload in a `PAYMENT-SIGNATURE` header. x402-aware clients handle this
conversation automatically, making paid content access completely transparent to
the calling code — no session, no checkout flow, no state machine.

**Human browsers with an active Ledewire session should use `POST /purchases`,
not x402.** x402 can work in a browser via a thin interceptor, but there is no
reason to use the challenge-response round-trip when the buyer is already logged in.
The `@ledewire/gate` embed and the Ledewire web app both use `POST /purchases`
for this reason.

This document designs the `/v1/x402/contents/:id` endpoint family, which implements
the x402 protocol using the **existing ledewire fiat wallet** as the payment rail —
with no changes to the underlying accounting, fee-splitting, or
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

**This design targets x402 protocol version 2** (`x402Version: 2`). The v2 spec
restructures `PaymentRequired` to separate resource metadata into a top-level
`resource` object, uses `amount` (atomic units as a string) instead of a formatted
price string, moves scheme-specific extension data into `extra`, and wraps the
chosen payment method in an `accepted` field within `PaymentPayload`.

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
Client                                   Ledewire API (/v1/x402/contents/:id)
  │                                              │
  │── GET /v1/x402/contents/:id ────────────────►│
  │                                              │  No PAYMENT-SIGNATURE header present.
  │                                              │  Server issues a nonce, stores it in
  │                                              │  Rails.cache with 5-minute TTL.
  │◄── 402 Payment Required ─────────────────────│
  │    PAYMENT-REQUIRED: <Base64 JSON>           │
  │    {                                         │
  │      x402Version: 2,                         │
  │      error: "PAYMENT-SIGNATURE required",    │
  │      resource: {                             │
  │        url:         "/v1/x402/contents/…",   │
  │        description: "Article title",         │
  │        mimeType:    "application/json"       │
  │      },                                      │
  │      accepts: [{                             │
  │        scheme:            "ledewire-wallet", │
  │        network:           "ledewire:v1",     │
  │        amount:            "10",              │
  │        asset:             "USD",             │
  │        payTo:             "store:<uuid>",    │
  │        maxTimeoutSeconds: 300,               │
  │        extra: {                              │
  │          nonce:     "<uuid>",                │
  │          expiresAt: 1745000000               │
  │        }                                     │
  │      }],                                     │
  │      extensions: {}                          │
  │    }                                         │
  │                                              │
  │  Client selects the ledewire-wallet          │
  │  scheme, builds payload with its JWT         │
  │  and the received nonce.                     │
  │                                              │
  │── GET /v1/x402/contents/:id ────────────────►│
  │   PAYMENT-SIGNATURE: <Base64 JSON>           │
  │   {                                          │
  │     x402Version: 2,                          │
  │     accepted: {                              │
  │       scheme:  "ledewire-wallet",            │
  │       network: "ledewire:v1",                │
  │       amount:  "10", asset: "USD",           │
  │       extra:   { nonce: "<uuid>" }           │
  │     },                                       │
  │     payload: {                               │
  │       token:     "Bearer eyJ...",            │
  │       nonce:     "<uuid>",                   │
  │       contentId: "<content-uuid>"            │
  │     },                                       │
  │     extensions: {}                           │
  │   }                                          │
  │                                              │
  │            [1] Atomic nonce consume (Rails.cache.delete)
  │            [2] JWT → User (existing AuthToken service)
  │            [3] Transactions::PurchaseContent#purchase (existing)
  │                — pessimistic row lock, balance check, ledger posting
  │                                              │
  │◄── 200 OK ───────────────────────────────────│
  │    PAYMENT-RESPONSE: <Base64 JSON>           │
  │    { success: true,                          │
  │      transaction: "<purchase-uuid>",         │
  │      network: "ledewire:v1",                 │
  │      payer: "<user-uuid>" }                  │
  │    { ...X402ContentPresenter... }            │
```

If the wallet has insufficient funds, `Transactions::PurchaseContent` returns a
`FailureResponse` which the controller renders as `422 Unprocessable Content` —
the same behaviour as the existing purchase endpoint. This is an intentional
deviation from the x402 v2 HTTP transport spec, which expects a `402` response
with `PAYMENT-RESPONSE: { success: false, errorReason: "insufficient_funds", ... }`
on settlement failure. See [x402 Spec Compliance Notes](#x402-spec-compliance-notes)
for the full rationale and interoperability consequences.

### Returning purchaser (already owns the content)

With the current design, a user who purchased content weeks ago and revisits the
page will still receive a `402` on their first unauthenticated request — because
the server has no identity until the `PAYMENT-SIGNATURE` retry arrives. The full
round-trip plays out correctly (no double-charge — `PurchaseContent#purchase`
detects the existing `completed` record and returns it immediately), but it burns a
nonce and adds visible latency for no reason.

```
Returning user (logged in, content purchased 1 month ago)
  │
  │── GET /v1/x402/contents/:id ────────────────►│
  │   (no PAYMENT-SIGNATURE)                     │  Server has no identity yet.
  │                                              │  Issues 402 + new nonce regardless.
  │◄── 402 Payment Required ─────────────────────│
  │                                              │
  │── GET /v1/x402/contents/:id ────────────────►│
  │   PAYMENT-SIGNATURE: { token: JWT, nonce }   │  Nonce consumed.
  │                                              │  JWT → user resolved.
  │                                              │  PurchaseContent called —
  │                                              │  existing completed purchase found.
  │                                              │  No deduction made.
  │◄── 200 OK (existing purchase returned) ──────│
```

The optimisation is to accept an optional `Authorization: Bearer <jwt>` header on
the **initial** `GET`. If present and the user has a completed purchase for this
content, the server returns `200` immediately — no `402` issued, no nonce created:

```
  │── GET /v1/x402/contents/:id ────────────────►│
  │   Authorization: Bearer eyJ...               │  User already logged in.
  │                                              │  JWT → user → completed purchase found.
  │◄── 200 OK ───────────────────────────────────│  Single round-trip, no 402.
```

This is transparent to x402 clients (they never see a `402`) and aligns with how
browsers already operate when the user is logged in. This optimisation is included
in the initial implementation (resolved in Q8).

---

## The `ledewire-wallet` Scheme

### `PAYMENT-REQUIRED` payload fields

Top-level `PaymentRequired` v2 envelope:

| Field | Type | Description |
|---|---|---|
| `x402Version` | `2` | Protocol version identifier (required) |
| `error` | string | Human-readable reason payment is required |
| `resource` | object | `{ url, description, mimeType }` describing the protected resource |
| `accepts` | array | Payment requirement objects |
| `extensions` | object | Protocol extensions (`{}` by default) |

Each object in `accepts` (`PaymentRequirements`):

| Field | Type | Description |
|---|---|---|
| `scheme` | `"ledewire-wallet"` | Custom scheme identifier |
| `network` | `"ledewire:v1"` | CAIP-2 style custom network identifier |
| `amount` | string | Price in cents as a string (e.g., `"10"` = $0.10) |
| `asset` | `"USD"` | ISO 4217 currency code for the fiat denomination |
| `payTo` | `"store:<uuid>"` | Destination; the store UUID receiving the payment |
| `maxTimeoutSeconds` | `300` | Payment window in seconds (matches `NONCE_TTL`) |
| `extra.nonce` | UUID string | Single-use token; expires after `NONCE_TTL` (5 minutes) |
| `extra.expiresAt` | Unix timestamp | Epoch seconds when the nonce expires |

### `PAYMENT-SIGNATURE` payload fields

Top-level `PaymentPayload` v2 envelope:

| Field | Type | Description |
|---|---|---|
| `x402Version` | `2` | Protocol version identifier (required) |
| `accepted` | object | The chosen `PaymentRequirements` echoed from `PAYMENT-REQUIRED` |
| `payload` | object | Scheme-specific payment data |
| `extensions` | object | Protocol extensions (`{}` by default) |

`payload` object (scheme-specific for `ledewire-wallet`):

| Field | Type | Description |
|---|---|---|
| `token` | `"Bearer eyJ..."` | The buyer's ledewire JWT |
| `nonce` | UUID string | The nonce received in `PAYMENT-REQUIRED` (`accepted.extra.nonce`) |
| `contentId` | UUID string | The content being purchased (anti-tampering check) |

### `PAYMENT-RESPONSE` payload fields (on `200`)

`SettlementResponse` v2 schema:

| Field | Type | Description |
|---|---|---|
| `success` | `true` | Settlement success indicator |
| `transaction` | UUID string | The `purchases.id` record (serves as the transaction identifier) |
| `network` | `"ledewire:v1"` | Network identifier |
| `payer` | UUID string | The buying user's `users.id` |
| `accessToken` | JWT string (optional) | RS256-signed entitlement JWT. Claims: `{ iss: "ledewire", sub: user_id, content_id, store_id, resource_url, entitled_for: ["purchase"], exp, kid: "ledewire-x402-v1" }`. Verifiable offline via `/.well-known/x402-jwks.json`. Omitted if signing key is not configured. |

---

## Implementation Architecture

Three new files. Zero changes to existing files.

### `app/controllers/concerns/x402_wallet_paywall.rb`

Controller concern encapsulating the full x402 conversation:

```ruby
# frozen_string_literal: true

module X402WalletPaywall
  extend ActiveSupport::Concern

  NONCE_TTL          = 5.minutes
  NONCE_CACHE_NS     = "x402:nonce:"
  RATE_CACHE_NS      = "x402:rate:"
  RATE_LIMIT_PER_MIN = 30
  SCHEME             = "ledewire-wallet"
  NETWORK            = "ledewire:v1"

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
      x402Version: 2,
      error:       "PAYMENT-SIGNATURE header is required",
      resource: {
        url:         "#{request.base_url}/v1/x402/contents/#{content.id}",
        description: content.title,
        mimeType:    "application/json"
      },
      accepts: [{
        scheme:            SCHEME,
        network:           NETWORK,
        amount:            content.price_cents.to_s,
        asset:             "USD",
        payTo:             "store:#{content.store_id}",
        maxTimeoutSeconds: NONCE_TTL.to_i,
        extra: {
          nonce:     nonce,
          expiresAt: NONCE_TTL.from_now.to_i
        }
      }],
      extensions: {}
    }

    response.headers["PAYMENT-REQUIRED"] = Base64.strict_encode64(payment_required.to_json)
    FailureResponse.new("Payment required", status: :payment_required)
  end

  def verify_wallet_payment(content, payload)
    unless payload["x402Version"] == 2
      return FailureResponse.new("Unsupported x402 version", status: :bad_request)
    end

    accepted = payload["accepted"] || {}
    inner    = payload["payload"]  || {}

    unless accepted["scheme"] == SCHEME && accepted["network"] == NETWORK
      return FailureResponse.new("Unsupported payment scheme or network", status: :bad_request)
    end

    unless inner["contentId"] == content.id.to_s
      return FailureResponse.new("Content ID mismatch in payment payload", status: :bad_request)
    end

    # Consume the nonce. Solid Cache's delete is backed by a SQL DELETE —
    # the first caller removes the row; any concurrent caller gets nil on read.
    nonce = inner["nonce"].to_s
    stored_content_id = Rails.cache.read("#{NONCE_CACHE_NS}#{nonce}")
    return issue_payment_required(content) if stored_content_id.nil?

    was_deleted = Rails.cache.delete("#{NONCE_CACHE_NS}#{nonce}")
    return issue_payment_required(content) unless was_deleted

    unless stored_content_id == content.id.to_s
      return FailureResponse.new("Nonce was not issued for this content", status: :bad_request)
    end

    Authentication::AuthToken.new
      .user_from_token(inner["token"].to_s.split.last)
      .bind do |user|
        Transactions::PurchaseContent.new(user: user).purchase(content: content).bind do |purchase|
          settlement = {
            success:     true,
            transaction: purchase.id,
            network:     NETWORK,
            payer:       user.id
          }
          response.headers["PAYMENT-RESPONSE"] = Base64.strict_encode64(settlement.to_json)
          SuccessResponse.new(purchase)
        end
      end
  end

  # Per-IP rate limiter using a 1-minute tumbling-window counter in Solid Cache.
  # Called by the controller before any nonce issuance or purchase lookup.
  # Returns nil when within the limit; FailureResponse(:too_many_requests) otherwise.
  def enforce_rate_limit
    bucket = Time.current.to_i / 60
    key    = "#{RATE_CACHE_NS}#{request.remote_ip}:#{bucket}"
    Rails.cache.write(key, 0, expires_in: 2.minutes, unless_exist: true)
    count = Rails.cache.increment(key)
    return if count <= RATE_LIMIT_PER_MIN

    FailureResponse.new("Too many requests", status: :too_many_requests)
  end
end
```

### `app/presenters/x402_content_presenter.rb`

A purpose-built presenter for the x402 `200` response. Returns the content itself
(body or URI, gated to the settled purchase) plus the `purchase_id` for client-side
receipt storage. No `access_info` block — all checkout-state fields are predetermined
on this path and `wallet_balance_cents` would be an unnecessary disclosure.

```ruby
# frozen_string_literal: true

class X402ContentPresenter
  include SchemaBacked
  schema_name "X402ContentResponse"

  def initialize(content:, purchase:)
    @content  = content
    @purchase = purchase
  end

  def present
    content_data = base_fields
    content_data[:purchase_id] = @purchase&.id  # nil for free content
    content_data
  end

  private

  def base_fields
    shared = {
      id:                  @content.id,
      content_type:        @content.content_type,
      title:               @content.title,
      price_cents:         @content.price_cents,
      teaser:              @content.teaser.to_s,
      visibility:          @content.visibility,
      metadata:            @content.metadata,
      external_identifier: @content.external_identifier
    }

    if @content.external_ref_content?
      # content_uri is the unlock — reveal it now that payment is settled.
      shared.merge(content_uri: @content.content_uri.to_s)
    else
      shared.merge(content_body: @content.content_body.to_s)
    end
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

        # Rate-limit before any nonce issuance or DB lookup.
        rate_check = enforce_rate_limit
        return render_response(rate_check) if rate_check

        # Free content: return immediately — no 402, no Purchase record.
        if content.price_cents.zero?
          return render_response(
            SuccessResponse.new(X402ContentPresenter.new(content: content, purchase: nil).present)
          )
        end

        # Returning purchaser: if Authorization header is present and the user already
        # has a completed purchase, serve content directly without a 402 round-trip.
        if (existing = completed_purchase_for(content))
          return render_response(
            SuccessResponse.new(X402ContentPresenter.new(content: content, purchase: existing).present)
          )
        end

        payment_result = require_wallet_payment(content)
        return render_response(payment_result) unless payment_result.success?

        purchase = payment_result.result
        render_response(
          SuccessResponse.new(X402ContentPresenter.new(content: content, purchase: purchase).present)
        )
      end

      private

      # Resolves the optional Authorization header and returns a completed Purchase
      # for this content, or nil. Auth failures are silent — the x402 flow handles
      # authentication for new purchases.
      def completed_purchase_for(content)
        token = request.headers["Authorization"].to_s.split.last
        return nil if token.blank?

        result = Authentication::AuthToken.new.user_from_token(token)
        return nil unless result.success?

        result.result.purchases.find_by(content: content, status: :completed)
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

## Browser Extension / Custom Interceptor Integration

> **Note:** Browsers with an active Ledewire session should use `POST /purchases`
> via the `@ledewire/gate` embed — not x402. The x402 endpoint is designed for
> **unauthenticated or sessionless clients**: browser extensions, AI agents,
> Cloudflare Workers, and CMS plugins. The section below documents how such a
> client would build a `ledewire-wallet` interceptor for cases where one is needed
> (e.g. a browser extension that holds a Ledewire JWT in `chrome.storage` and wants
> to purchase content on any arbitrary site).

Because the `ledewire-wallet` scheme uses only a Bearer JWT and a server-issued
nonce — no crypto wallet, no MetaMask, no signing key — the x402 handshake is
straightforward to implement in any environment that can store a JWT and make HTTP
requests.

### How it works

The coinbase x402 repository ships `@x402/fetch` and `@x402/axios` — drop-in
wrappers that intercept `402` responses, build a `PAYMENT-SIGNATURE` header, and
retry automatically. Those packages target the canonical `exact` (crypto) scheme;
for `ledewire-wallet` a small custom interceptor is needed that:

1. Reads the `PAYMENT-REQUIRED` header from the `402` response
2. Decodes and parses the Base64 JSON
3. Picks the `ledewire-wallet` entry from `accepts`
4. Builds the `PaymentPayload` using the stored JWT and the nonce from
   `accepted.extra.nonce`
5. Retries the original request with the `PAYMENT-SIGNATURE` header attached

### UX states

| Wallet state | Balance sufficient? | Suggested UX |
|---|---|---|
| Logged in, below threshold | Yes | **Silent purchase** — interceptor retries automatically; content appears with no user interaction |
| Logged in, above threshold | Yes | **Confirmation modal** — "Pay $1.00 for lifetime access?" → user clicks OK → interceptor retries |
| Logged in | No | **Top-up prompt** — 422 from server triggers wallet funding flow, then retry |
| Not logged in | N/A | Redirect to login; return to content URL afterward |

**Purchase threshold** — a user-configurable (or store-configurable) per-transaction
amount below which purchases proceed silently. Small micropayments (e.g. $0.10 for
an ad-free article) should be frictionless; larger purchases (e.g. $9.99 for a
video) warrant a confirmation step. This threshold is a client-side concern and
requires no server changes.

### Example: ad-free article button

```javascript
// Thin ledewire-wallet interceptor (illustrative)
async function fetchWithX402(url, options = {}) {
  const res = await fetch(url, options);
  if (res.status !== 402) return res;

  const header = res.headers.get("PAYMENT-REQUIRED");
  const required = JSON.parse(atob(header));
  const offer = required.accepts.find(a => a.scheme === "ledewire-wallet");
  if (!offer) return res; // unsupported scheme — surface error

  if (offer.amount > SILENT_PURCHASE_THRESHOLD_CENTS) {
    const confirmed = await showConfirmationModal(offer);
    if (!confirmed) return res;
  }

  const payload = btoa(JSON.stringify({
    x402Version: 2,
    accepted: offer,
    payload: {
      token:     `Bearer ${getSessionJwt()}`,
      nonce:     offer.extra.nonce,
      contentId: extractContentId(url)
    },
    extensions: {}
  }));

  return fetch(url, { ...options, headers: { ...options.headers, "PAYMENT-SIGNATURE": payload } });
}
```

### Content delivery

Open Question 3 (response shape) is directly relevant here. For human users,
returning `ContentWithAccessPresenter` (content body inline) in the `200` response
is strongly preferred — it means the article text or video stream URL is delivered
in the same request that settled the payment, with no follow-up fetch required.
This is the x402 principle of *HTTP/Transport Native* payments in practice.

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
- Existing `/v1/purchases` and `/v1/checkout` endpoints — untouched

---

## x402 Spec Compliance Notes

This section documents deliberate deviations from the [x402 v2 HTTP transport
specification](https://github.com/coinbase/x402/blob/main/specs/transports-v2/http.md)
and the [x402 v2 core specification](https://github.com/coinbase/x402/blob/main/specs/x402-specification-v2.md),
along with minor gaps identified during a spec review conducted 2026-03-30.

### Deviation 1 — Insufficient funds returns 422, not 402 (intentional)

**What the spec says:** The HTTP transport spec's error table specifies that
any payment settlement failure should be returned as `402 Payment Required`,
with a `PAYMENT-RESPONSE` header containing:
```json
{ "success": false, "errorReason": "insufficient_funds", "transaction": "", "network": "ledewire:v1", "payer": "<uuid>" }
```

**What we do:** `PurchaseContent` returns a `FailureResponse(status:
:unprocessable_content)` on insufficient balance. The controller renders this as
`422 Unprocessable Content` with no `PAYMENT-RESPONSE` header — identical to the
behaviour of the existing `/v1/purchases` endpoint.

**Rationale:** The `ledewire-wallet` scheme is not crypto. A `402` on funds
failure would cause a generic x402 client library to retry (expecting fresh
payment credentials), which is semantically wrong — the user needs to top up their
wallet, not supply a different payment. `422` is an unambiguous terminal failure
that triggers the wallet funding UX flow in the Ledewire browser client.

**Interoperability consequence:** A generic x402 client library (e.g., `@x402/fetch`)
that handles scheme `ledewire-wallet` must treat `422` as a terminal error, not a
retry signal. This must be documented in any future client SDK or browser extension
integration guide.

### Deviation 2 — No `PAYMENT-RESPONSE` header on settlement failure (intentional, follows from Deviation 1)

Because we return `422` and not `402` on insufficient funds, we do not send a
`PAYMENT-RESPONSE` header on the failure path. The spec's failure `PAYMENT-RESPONSE`
is only meaningful when the response code is `402` (so the client knows the
payment was attempted but failed, versus not attempted at all).

For our `422` path the error information is conveyed in the standard JSON body
(`{ error: { code:, message: } }`), which is sufficient for all current Ledewire
clients.

### Gap — `accepted.amount` not validated against server-side price (low priority)

During `verify_payment`, the server checks `accepted["scheme"]` and
`accepted["network"]` from the echoed `PaymentRequirements` object, but does not
verify that `accepted["amount"]` matches `content.price_cents.to_s`. This is not
a security issue — the authoritative price always comes from `content.price_cents`
inside `PurchaseContent`, never from the client payload — but it means a client
could send a modified `accepted.amount` without being rejected at the protocol
boundary.

Adding `accepted["amount"] == content.price_cents.to_s` as a guard in
`WalletService#verify_payment` would tighten the protocol boundary and make the
check explicit. Tracked as a low-priority hardening item.

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
- **Free content short-circuit.** Handled in the controller: `price_cents: 0`
  returns `200` immediately with no `402` and no `Purchase` record (see Q4).

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

3. **Response shape for `200` — ✅ Resolved: use `X402ContentPresenter` / `X402ContentResponse`.**
   A dedicated presenter is used rather than adapting either `ContentWithAccessPresenter`
   or `DetailedPurchasePresenter`. This avoids coupling the x402 response shape to the
   existing checkout or accounting presenters, which carry fields that are either
   redundant on the x402 path (`access_info`, `has_purchased`, `next_required_action`)
   or represent unnecessary information disclosure (`wallet_balance_cents`, fee
   breakdown columns).

   `X402ContentPresenter` returns:
   - The content itself: `content_body` (markdown) or `content_uri` (external ref, revealed post-settlement)
   - Standard content metadata: `id`, `title`, `price_cents`, `teaser`, `visibility`, `metadata`, `external_identifier`
   - `purchase_id` — the UUID of the settled `Purchase` record, for client-side receipt storage

   No `access_info` block. No wallet balance. No fee columns. The OpenAPI schema is a
   new `X402ContentResponse` component defined alongside this implementation.

4. **Free content behaviour — ✅ Resolved: return content directly, no `402`, no `Purchase` record.**
   The controller short-circuits before the paywall when `content.price_cents.zero?`
   and returns the content in a `200` immediately. No nonce is issued and no `Purchase`
   record is created. This preserves the ability to charge for the content in future
   (a phantom `completed` purchase would otherwise block re-pricing) and keeps the
   purchases table free of zero-value records. `purchase_id` in the response is `null`
   on this path.

5. **Multiple license tiers (e.g. rental vs. purchase).** The x402 v2 `accepts`
   array supports multiple `PaymentRequirements` entries with different `amount`
   values and scheme-specific `extra` data — a client selects one and echoes it
   back in `accepted`. This is the natural protocol hook for tiered pricing.
   However, issuing a **separate nonce per tier** (each nonce must be bound to a
   specific content + license outcome to prevent substitution attacks) adds
   meaningful complexity to the nonce issuance and verification logic.

   The simpler alternative is **separate endpoints per license type**
   (`GET /v1/x402/contents/:id/rental`, `GET /v1/x402/contents/:id/purchase`),
   which maps cleanly to distinct Rails routes, keeps the single-entry `accepts`
   array, and avoids multi-nonce state entirely. Initial implementation should use
   this approach. The multi-entry `accepts` approach can be adopted later if
   client-side tier selection within a single request becomes a product requirement.

6. **Silent purchase threshold for browser clients.** The browser interceptor
   should auto-purchase without confirmation below some per-transaction ceiling
   (e.g. $0.50), and show a confirmation modal above it. Should this threshold be:
   (a) a hard-coded client constant, (b) a user preference stored in profile
   settings, or (c) a per-store hint returned by the server in `extra`? Option (b)
   is the most user-friendly long-term but requires a new profile settings field
   and UI. Start with (a) for launch; design (b) as a follow-on.

7. **Browser interceptor library.** Coinbase's `@x402/fetch` and `@x402/axios`
   packages handle the 402 → retry loop but only for the `exact` (crypto) scheme.
   For `ledewire-wallet` the options are: (a) fork/extend those packages to
   register a custom scheme handler, (b) write a bespoke `fetchWithX402` wrapper
   in the ledewire frontend codebase, or (c) contribute `ledewire-wallet` scheme
   support upstream. Option (b) is fastest to ship; option (a)/(c) is better
   long-term if x402 adoption widens across the platform.

   For the browser extension specifically, a fourth option is (d) build on
   [Plasmo](https://github.com/PlasmoHQ/plasmo) — an MV3 build framework with
   first-class OAuth PKCE support and typed `chrome.storage` wrappers — which
   would remove most of the service worker boilerplate and let the team focus on
   the `ledewire-wallet` scheme logic. See the Reference Implementations note in
   the Browser Extension section of Potential Future Efforts.

8. **Short-circuit `200` for returning purchasers — ✅ Resolved: in scope from the start.**
   The controller accepts an optional `Authorization: Bearer <jwt>` header on the
   initial `GET`. If present and the resolved buyer has a `completed` purchase for
   the content, `200` is returned immediately — no `402` issued, no nonce created.
   The `PAYMENT-RESPONSE` header is omitted on this path (no new settlement occurred).
   Auth failures are silent: an invalid or merchant-role token simply falls through
   to the normal x402 flow. Implemented via `completed_purchase_for(content)` in the
   controller; no changes to the concern or `PurchaseContent` service.

---

## Potential Future Efforts

The following directions extend the x402 foundation established here to third-party
sites and generic infrastructure. None of these are in scope for the initial
implementation — they require additional exploration, API design, and likely
separate ADRs.

### Third-Party Site Integration

Stores hosting content on their own domains (WordPress, Ghost, custom CMS, etc.)
need a way to verify that a browser session has a valid Ledewire purchase without
implementing a full ledewire checkout flow themselves.

**What Ledewire would need to add:**

- **Signed access token in `PAYMENT-RESPONSE`** — a short-lived JWT (`accessToken`
  field) issued after purchase settlement, containing claims:
  `{ iss: "ledewire", sub: user_id, content_id, store_id, resource_url, entitled_for: ["purchase"], exp, kid: "ledewire-x402-v1" }`.
  The third-party site verifies this token offline; no per-request call to Ledewire.
- **JWKS endpoint** (`GET /.well-known/x402-jwks.json`) — publishes the RS256/ES256
  public key so any third-party server or proxy can verify `accessToken` signatures
  without calling Ledewire.
- **`resource_url` field on `Content`** — the canonical URL on the origin site, so
  Ledewire can bind a `content_id` to a specific third-party page URL.

**Options for the third-party site:**

| Approach | What the site needs | Trade-offs |
|---|---|---|
| **A. JWT cookie verification** | Read `accessToken` from a cookie; verify against JWKS; conditionally suppress ad nodes | Fully offline after JWKS fetch; requires per-CMS plugin/middleware |
| **B. Entitlement check endpoint** | Call `GET /v1/purchases/entitlement?content_id=&user_token=` server-side per page load | Simpler than key management; one network hop per request |
| **C. Ledewire webhook** | Receive `purchase.completed` webhook; store entitlement locally | No per-request latency; requires persistent user identity mapping between Ledewire and origin |

Option A is the most scalable. Option B is simplest to prototype.

### Generic Proxy (Zero-Change Origins)

The most powerful approach requires **no changes to the origin site** at all. A
reverse proxy sits in front of the origin, handles all x402 logic, and either
forwards a clean request with an entitlement header, or rewrites the HTML response
to strip ad slots.

```
Browser → Ledewire Proxy → Origin (unmodified WordPress/Ghost/etc.)
```

**Flow:**
1. Unauthenticated request hits proxy → proxy issues `402` with `PAYMENT-REQUIRED`
2. Browser JS interceptor purchases via Ledewire API → receives `accessToken` JWT
3. Browser retries with JWT in cookie → proxy verifies JWT offline (JWKS)
4. Proxy forwards request to origin with `X-Ledewire-Entitled: true` header, OR
   performs HTML rewrite to strip ad blocks (`<div class="ad-slot">`) from the
   response using store-configured CSS selector rules

The origin serves its full content to the proxy unconditionally (trusting a shared
secret or private network). The proxy is the only public-facing surface.

**Proxy delivery options — all use the same JWT verification and HTML rewrite logic:**

| Format | Operational footprint | Notes |
|---|---|---|
| **Cloudflare Worker** | Zero — store owner adds one Worker to their zone | Best fit for stores already on Cloudflare; no infrastructure to run |
| **Docker container** (OpenResty/Nginx + Lua, or Go) | Self-hosted; store points a CNAME at it | Suits stores with existing server infrastructure |
| **Caddy plugin** | Single Caddyfile directive | Good for technically sophisticated store owners |

**Ledewire additions required (same for all proxy formats):**
- `accessToken` in `PAYMENT-RESPONSE` (see Third-Party Site Integration above)
- JWKS endpoint
- Store configuration API — maps origin domain + URL patterns → `content_id`, and
  stores per-content ad-selector rules used by the proxy for HTML rewriting

### Browser Extension — Universal Ledewire Wallet

The most ambitious direction: a browser extension that acts as a **universal x402
client** across the entire web. Any site that issues a `402` with a
`ledewire-wallet` `PAYMENT-REQUIRED` header is handled automatically — no plugin,
no proxy, no per-site integration required. From the user's perspective it is a
single wallet balance and a single login, usable everywhere, analogous to what
PayPal attempted for the web and what MetaMask achieved for crypto.

**How it works**

A Manifest V3 service worker intercepts every fetch/XHR response across all tabs.
When it detects a `402` + `PAYMENT-REQUIRED` header containing the
`ledewire-wallet` scheme, it handles the full x402 handshake transparently — the
calling page's JavaScript never sees the `402`:

```
Any site on the web       Extension (background SW)        Ledewire API
      │                           │                              │
      │── fetch('/article') ─────►│── forward ─────────────────►│
      │                           │◄── 402 PAYMENT-REQUIRED ─────│
      │                           │                              │
      │                 Recognises ledewire-wallet scheme        │
      │                 Verifies origin is a registered          │
      │                 Ledewire Content (anti-phishing)         │
      │                 Shows modal if above threshold           │
      │                           │── retry with PAYMENT-SIGNATURE ──►│
      │                           │◄── 200 + content ─────────────────│
      │◄── 200 (transparent) ─────│
```

**Key design requirements**

| Concern | Approach |
|---|---|
| Credential storage | OAuth 2.0 PKCE flow at login; store refresh token in `chrome.storage.local` (encrypted at rest); hold access JWT in memory only — never on disk in plaintext |
| Intercept scope | Declarative Net Request (MV3) to observe responses; service worker handles the 402 → retry loop |
| Confirmation UI | Extension popup overlaid on the page for above-threshold purchases; silent below threshold (see Open Question 6) |
| Balance / top-up | Extension toolbar badge showing current wallet balance; clicking opens top-up flow at ledewire.com |
| Cross-origin requests | Extension requests originate from the background service worker, not page context — origin site's CSP does not apply |
| Anti-phishing | Only process `PAYMENT-REQUIRED` from origins with a verified `Content` record on Ledewire (see origin verification endpoint below) |
| Returning purchaser | Extension holds the JWT persistently; can send `Authorization` header on first request and receive `200` directly (Open Question 8) |

**New server-side requirement: origin verification endpoint**

Before showing any payment UI for a `402` from an arbitrary site, the extension
must confirm the paywall is a legitimate Ledewire-registered content item — not a
rogue site attempting to trick users into paying:

```
GET /v1/x402/verify-origin?url=https://theirsite.com/article-slug
→ { verified: true, content_id: "…", store_id: "…", amount: "10", title: "…" }
→ { verified: false }  ← extension ignores the 402 silently
```

This endpoint cross-references the requested URL against `Content.resource_url`
(the field already identified in the Third-Party Site Integration section). It
requires no authentication and should be heavily cached.

**Relationship to other future efforts**

The browser extension, the generic proxy, and third-party site integration all
converge on the same three Ledewire API additions: `accessToken` in
`PAYMENT-RESPONSE`, the JWKS endpoint, and `resource_url` on `Content`. Building
any one of them unblocks the others. The extension is the highest-leverage
investment: it eliminates the need for a proxy entirely for browser-based users,
and delivers the universal wallet experience with no per-site deployment burden.

**Server-side prerequisites** ✅ All four items are complete.

All three future directions (extension, proxy, third-party CMS) share the same
four server-side additions. None require changes to the existing x402 endpoint,
only additions:

| Addition | Effort | Unlocks |
|---|---|---|
| ~~`resource_url` column on `Content` (migration + field)~~ | ✅ Done | Origin verification, third-party site binding |
| ~~`accessToken` JWT in `PAYMENT-RESPONSE` (new RS256/ES256 key pair, minting logic)~~ | ✅ Done | Offline entitlement verification by proxy/CMS |
| ~~JWKS endpoint (`GET /.well-known/x402-jwks.json`)~~ | ✅ Done | Offline verification of `accessToken` by any third party |
| ~~Origin verification endpoint (`GET /v1/x402/verify-origin?url=`)~~ | ✅ Done | Extension anti-phishing guard; heavily HTTP-cached |

All four items have been delivered. The `resource_url` migration was the natural
first step as it unblocked all the rest.

**Reference implementations (browser extension research)**

Three open source MV3 extensions provide directly applicable patterns:

- **Bitwarden** (`apps/browser` in [github.com/bitwarden/clients](https://github.com/bitwarden/clients))
  — the most directly applicable reference. Completed a full MV3 migration and
  has solved every hard problem: PKCE login via `chrome.identity.launchWebAuthFlow`,
  encrypted token storage in `chrome.storage.local` (master-key-derived encryption,
  no plaintext secrets on disk), and service worker lifecycle management. Their
  `apps/browser/src/background/` directory is the best starting point for the token
  lifecycle, and `AuthService` for the PKCE flow.

- **Plasmo** ([github.com/PlasmoHQ/plasmo](https://github.com/PlasmoHQ/plasmo))
  — an MV3 build framework with first-class OAuth PKCE support and typed storage
  wrappers (`@plasmohq/storage`, `@plasmohq/messaging`). Their docs include an
  end-to-end OAuth PKCE example. Using Plasmo removes most service worker
  boilerplate and is the recommended approach if the team is not familiar with
  raw MV3 extension development.

- **Requestly** ([github.com/requestly/requestly](https://github.com/requestly/requestly))
  — open source request-intercepting MV3 extension. Useful reference specifically
  for the Declarative Net Request + service worker message-passing pattern used
  to intercept responses and hand them to the background worker for processing.

**Critical MV3 service worker lifecycle gotcha**

MV3 service workers are killed after ~30 seconds of inactivity. The access JWT
(held in memory only) is lost on every SW restart. The canonical solution:

1. Store the refresh token in `chrome.storage.local` (encrypted)
2. On every SW wake-up, re-mint the access JWT via a `/auth/refresh` call and
   hold it in memory for the lifetime of that SW invocation
3. Use **`chrome.storage.session`** (available Chrome 102+, Firefox 121+) for
   the access JWT — it survives SW restarts within a browser session without
   hitting persistent storage, and clears automatically on browser close. This
   is cleaner than the `chrome.alarms` keepalive workaround used in older
   Bitwarden versions.

---

## Implementation Checklist

### Pre-implementation decisions

- [x] ~~Resolve Open Question 3~~ — decided: `X402ContentPresenter` / `X402ContentResponse` (new presenter, new schema)
- [x] ~~Resolve Open Question 4~~ — decided: `price_cents: 0` returns `200` directly; no `402`, no `Purchase` record, `purchase_id: null`
- [x] ~~Resolve Open Question 8~~ — decided: in scope from the start; `Authorization` pre-flight in controller

### Server-side changes

- [x] Add `PAYMENT-REQUIRED` and `PAYMENT-RESPONSE` to `expose:` in
      `config/initializers/cors.rb` — without this, browser JavaScript cannot read
      either header (`res.headers.get("PAYMENT-REQUIRED")` returns `null` silently)
- [x] Rate limiting: `rate_limit to: 30, within: 1.minute, only: :show` in the controller
      class body using Rails 8 native rate limiting (no gem required). Backed by
      `config.action_controller.cache_store = :memory_store` in test env.
- [x] Create `app/controllers/concerns/x402_wallet_paywall.rb` (thin HTTP adapter;
      protocol logic lives in `app/services/x402/wallet_service.rb`)
- [x] Create `app/services/x402/wallet_service.rb` — pure service object, no Rails
      request/response objects; returns `Result` struct with `response`,
      `payment_required_header`, `payment_response_header`
- [x] Create `app/controllers/v1/x402/contents_controller.rb`
- [x] Add `namespace :x402 { resources :contents, only: [:show] }` inside `namespace :v1` in `config/routes.rb`
- [x] Add `X402ContentResponse` component schema to `docs/api/v1/ledewire.yml` — fields:
      `id`, `content_type`, `title`, `price_cents`, `teaser`, `visibility`, `metadata`,
      `external_identifier`, `purchase_id`, plus either `content_body` (markdown) or
      `content_uri` (external_ref)
- [x] Create `app/presenters/x402_content_presenter.rb`
- [x] Add `GET /v1/x402/contents/{id}` path to `docs/api/v1/ledewire.yml` — response
      schema is `X402ContentResponse`
- [x] Add `ahoy.track("[Purchase] Create", ..., via: "x402")` on the successful settlement
      path in the controller — x402 purchases appear in analytics alongside standard purchases

### Request spec: `spec/requests/v1/x402/contents_spec.rb`

- [x] `GET` without `PAYMENT-SIGNATURE` → 402, valid `PAYMENT-REQUIRED` header present
- [x] `GET` with valid buyer JWT + nonce → 200, `PAYMENT-RESPONSE` header, content body returned
- [x] `GET` with replayed nonce → 402 (nonce already consumed)
- [x] `GET` with expired nonce (cache TTL elapsed) → 402
- [x] `GET` with insufficient wallet balance → 422
- [x] `GET` with wallet balance exactly equal to price → 422 (documents `PurchaseContent` exact-balance behaviour)
- [x] `GET` with invalid JWT in signature → 401
- [x] `GET` with a merchant-role JWT in payload → 403
- [ ] `GET` with a store-manager-role JWT in payload → 403 _(not yet: no store-manager JWT factory in spec)_
- [x] `GET` for already-purchased content via x402 flow → 200, idempotent (no double-charge via `PurchaseContent`)
- [x] `GET` with `Authorization: Bearer <jwt>` for already-purchased content → 200, no `402` issued, no `PAYMENT-RESPONSE` header
- [x] `GET` for free content (`price_cents: 0`) → 200, `purchase_id: null`, no `402` issued, no `Purchase` record created
- [x] `GET` exceeding rate limit (> 30 req/min from same IP) → 429
- [x] `GET` for unknown content → 404

### Service spec: `spec/services/x402/wallet_service_spec.rb`

- [x] 20 examples covering all `WalletService` branches including protocol guard clauses
      (bad version, bad scheme/network, contentId mismatch, cross-content nonce) not
      reachable via the HTTP layer

### Flow spec: `spec/flows/x402_wallet_purchase_flow_spec.rb`

- [x] Full round-trip: fund wallet → x402 purchase → assert ledger balance

### Final checks

- [x] Confirm `Rails.cache` is Solid Cache in all environments before shipping
- [x] Add `ahoy.track("[Purchase] Create", ..., via: "x402")` on the successful settlement
      path in the controller — x402 purchases appear in analytics alongside standard purchases

### Server-side prerequisites (Phase 2)

- [x] Add `resource_url` column to `contents` (migration `20260330124519_add_resource_url_to_contents.rb`)
- [x] Add `by_resource_url` scope to `Content` model
- [x] Create `app/services/x402/access_token_service.rb` — RS256 JWT minting + JWKS document generation
- [x] Add `config.x402_access_token_ttl` (24h) and `config.x402_verify_origin_cache_ttl` (1h) to `config/application.rb`
- [x] Update `X402::WalletService` to include `accessToken` in settlement hash (`.compact` omits it when key absent)
- [x] Create `app/controllers/x402/jwks_controller.rb` — `GET /.well-known/x402-jwks.json`, 24h Cache-Control
- [x] Create `app/controllers/v1/x402/verify_origin_controller.rb` — `GET /v1/x402/verify-origin?url=`, 1h cache, 60/min rate limit
- [x] Add routes: `GET /v1/x402/verify-origin` (inside v1/x402 namespace) and `GET /.well-known/x402-jwks.json`
- [x] Add `VerifyOriginVerified`, `VerifyOriginUnverified`, `JwksResponse`, `SettlementResponse` schemas + paths to OpenAPI spec
- [x] Store RS256 key pair in `credentials.yml.enc` under `x402.access_token_private_key` / `x402.access_token_public_key`
- [x] Spec: `spec/services/x402/access_token_service_spec.rb` (10 examples — claims, RS256, JWKS modulus, TTL, graceful nil)
- [x] Spec: `spec/requests/x402/jwks_spec.rb` (6 examples — 200, fields, Cache-Control, modulus match)
- [x] Spec: `spec/requests/v1/x402/verify_origin_spec.rb` (11 examples — verified/unverified/private/invalid URLs, no auth required)
- [x] Spec: `accessToken` presence and RS256 verifiability added to `spec/requests/v1/x402/contents_spec.rb`
