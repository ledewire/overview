# x402 Wallet Payments

**Status:** Proposed — Under Review
**Date:** 2026-03-27

---

## Table of Contents

1. [Motivation](#motivation)
2. [What is x402?](#what-is-x402)
3. [Adapting x402 for Ledewire's Fiat Wallet](#adapting-x402-for-ledewires-fiat-wallet)
4. [Request / Response Lifecycle](#request--response-lifecycle)
5. [The `ledewire-wallet` Scheme](#the-ledewire-wallet-scheme)
6. [Implementation Architecture](#implementation-architecture)
7. [Browser / Human Client Integration](#browser--human-client-integration)
8. [Security: Nonce & Idempotency](#security-nonce--idempotency)
9. [What Does Not Change](#what-does-not-change)
10. [Out of Scope](#out-of-scope)
11. [Open Questions](#open-questions)
12. [Implementation Checklist](#implementation-checklist)

---

## Motivation

The current content purchase flow requires three sequential steps:

1. `GET /v1/checkout/state` — fetch checkout state and create a pending `Purchase`
2. Ensure the buyer has sufficient wallet balance (separate funding flow if not)
3. `POST /v1/purchases` — complete the purchase; deduct from wallet

This multi-step state machine adds friction for all client types. It is particularly
poor for:

- **Human browsers** that want a single-click, zero-friction purchase experience
  (e.g. "Watch ad-free" or "Unlock article") without navigating a checkout flow
- **AI agents and automated clients** that need to acquire content programmatically
  in a single round-trip
- **Third-party integrations** where implementing ledewire's checkout state machine
  is a significant integration burden
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
  │    { ...DetailedPurchasePresenter... }       │
```

If the wallet has insufficient funds, `Transactions::PurchaseContent` returns a
`FailureResponse` which the controller renders as `422 Unprocessable Content` —
the same behaviour as the existing purchase endpoint. The x402 spec permits
non-`402` error codes for server-side failures.

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
browsers already operate when the user is logged in. See Open Question 8.

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

## Browser / Human Client Integration

Because the `ledewire-wallet` scheme uses only a Bearer JWT and a server-issued
nonce — no crypto wallet, no MetaMask, no signing key — the entire x402 handshake
can be performed transparently by a thin JavaScript layer running in any browser.
The user experience depends solely on how the client-side interceptor is configured.

### How it works in a browser

The coinbase x402 repository ships `@x402/fetch` and `@x402/axios` — drop-in
wrappers that intercept `402` responses, build a `PAYMENT-SIGNATURE` header, and
retry automatically. Those packages target the canonical `exact` (crypto) scheme;
for `ledewire-wallet` a small custom interceptor is needed that:

1. Reads the `PAYMENT-REQUIRED` header from the `402` response
2. Decodes and parses the Base64 JSON
3. Picks the `ledewire-wallet` entry from `accepts`
4. Builds the `PaymentPayload` using the user's stored JWT and the nonce from
   `accepted.extra.nonce`
5. Retries the original request with the `PAYMENT-SIGNATURE` header attached

Because the JWT is already available in the browser session (same credential used
for all other API calls), no additional login or wallet-connection step is required.

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

8. **Short-circuit `200` for returning purchasers.** Should `GET /v1/x402/contents/:id`
   accept an optional `Authorization: Bearer <jwt>` header and, if the resolved
   user already has a `completed` purchase for the content, return `200` immediately
   without issuing a `402` or nonce? This eliminates one unnecessary round-trip for
   logged-in users revisiting content they already own — the common case for human
   browser sessions over time. The implementation adds a pre-flight check at the
   top of the controller's `show` action (resolve user from `Authorization` header
   if present → query `purchases` → short-circuit) and requires no changes to the
   concern or the `PurchaseContent` service. The `PAYMENT-RESPONSE` header is
   omitted on this path (no new settlement occurred); the response body is the same
   presenter output.

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
  `{ sub: user_id, content_id, resource_url, entitled_for: ["ad_free"], exp }`.
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
