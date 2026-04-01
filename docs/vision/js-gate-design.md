# JS Gate — Soft Content Gating via Embeddable Script

**Status:** Design — Under Review
**Date:** 2026-03-31
**Related:** [docs/x402-wallet-payment-design.md](x402-wallet-payment-design.md)

---

## Table of Contents

1. [Motivation](#motivation)
2. [What "Soft Gate" Means](#what-soft-gate-means)
3. [Architecture Overview](#architecture-overview)
4. [Server-Side: ContentPricingRule](#server-side-contentpricingrule)
5. [Domain Verification](#domain-verification)
6. [Content Registration Flow](#content-registration-flow)
7. [The `gate.js` Bundle](#the-gatejs-bundle)
8. [Purchase Flow Inside the Snippet](#purchase-flow-inside-the-snippet)
9. [Token Lifecycle](#token-lifecycle)
10. [Security Considerations](#security-considerations)
11. [Store Owner Setup Experience](#store-owner-setup-experience)
12. [Open Questions](#open-questions)
13. [Implementation Checklist](#implementation-checklist)

---

## Motivation

### The onboarding problem

Ledewire's existing content gating requires a store owner to manually create a
`Content` record in the dashboard for every article, page, or post they want to
sell. For a publisher with hundreds of existing articles — or one who publishes
daily — this is impractical. It also creates an ongoing operational burden:
every new post needs to be registered before it can be gated.

This design eliminates that burden entirely. A store owner:

1. Verifies they own their domain (one-time, DNS TXT record)
2. Creates a pricing rule covering a URL pattern (e.g. `https://blog.example.com/posts/*`)
3. Drops a single `<script>` tag into their page template

That's it. Every matching page is automatically gated from that point forward.
Content records are created lazily on first visit — the store owner never touches
the Ledewire dashboard again for day-to-day publishing.

### What this design does and does not change

**Changes — content registration:**
- New `ContentPricingRule` model: URL pattern + price per unlock, set once for an entire section of the site
- New `StoreDomainVerification` model: proves ownership before rules can fire
- Extended `verify-origin` endpoint: lazily registers a `Content` record on first visit to any matching URL

**Does not change — the purchase flow:**
- `POST /purchases` is used as-is (authenticated REST, existing endpoint)
- `Transactions::PurchaseContent`, wallet accounting, and author payouts are untouched
- The `@ledewire/gate` embed (`page-blocker`) already handles the full buyer UX — login modal, paywall overlay, Stripe add-funds, token lifecycle

The only purchase-adjacent addition is emitting an `accessToken` in the `POST /purchases`
response so the embed can skip the `verifyPurchase()` network call on warm page loads.

### The gate is soft by design

The `@ledewire/gate` embed hides content via a full-viewport overlay rather than
server-side access control. A determined user can bypass it by disabling JavaScript.
This is intentional and appropriate for articles, newsletters, and commentary — the
same model used by the NY Times, Medium, and The Atlantic. For content where the
payload itself must remain secret (downloadable files, video stream URLs), the x402
`external_ref` or Generic Proxy paths are the correct choice.

---

## What "Soft Gate" Means

The embed overlay **does not prevent a determined user from accessing the raw HTML**.
Soft gating is appropriate when the value is in the reading experience, not in
keeping the HTML bytes secret.

**Appropriate for:**
- Articles, newsletters, blog posts
- Ad-free reading passes
- Commentary / opinion content

**Not appropriate for:**
- Downloadable files (use the x402 `external_ref` path — the file URI is never
  in the HTML)
- Video streams gated via URL (use `external_ref` + x402 — `content_uri` is only
  revealed post-purchase)
- Any content where the HTML itself is the secret

For hard gating, the Generic Proxy or CMS Plugin paths (documented in
`x402-wallet-payment-design.md`) are the correct choice.

---

## Architecture Overview

```
Store owner drops <script> into page template
          │
          ▼
┌───────────────────────────────────────────────────────┐
│  gate.js loads on page                                │
│  1. Read current URL                                  │
│  2. GET /v1/x402/verify-origin?url=<current URL>      │──► Already built ✅
│     { verified: true, content_id, amount, title }     │
│  3. Check session storage for unexpired accessToken   │
│     for this content_id (TokenManager)                │
│     ├── Valid token  → reveal gated nodes             │
│     └── No token     → render paywall overlay         │
│  4. User clicks "Unlock for $X"                       │
│  5. POST /purchases (authenticated REST)              │
│  6. Store accessToken in TokenManager (per session)   │
│  7. Reveal gated nodes                                │
└───────────────────────────────────────────────────────┘
          │
          ▼
  BUT: step 2 only works if the page URL is registered
  as a Content.resource_url. How does that happen?
          │
          ▼
┌───────────────────────────────────────────────────────┐
│  ContentPricingRule  (new model)                      │
│  store_id | url_pattern | price_cents | gate_selector │
│  e.g. "https://theirsite.com/articles/*"              │
│       price_cents: 100                                │
│       gate_selector: ".post-content"                  │
│                                                       │
│  On first verify-origin hit for a matching URL:       │
│    FindOrCreateContent service lazily creates a       │
│    Content record with resource_url set               │
└───────────────────────────────────────────────────────┘
```

The key insight is that `verify-origin` is already the entry point for the
extension anti-phishing check. We extend it to also do lazy content registration
when a `ContentPricingRule` matches — turning a single cached GET into the
registration mechanism.

### Frontend: `@ledewire/gate`

The JS bundle described in this document is **[`@ledewire/gate`](https://github.com/ledewire/embed)**
(the `page-blocker` entry point). This package already exists with a v0.0.1 release:
Preact + TypeScript + Tailwind + Vite, served as an IIFE from jsDelivr. Most of the
frontend work is already done — see the [Phase 2 checklist](#phase-2--js-bundle-separate-repo--frontend-work)
for what is existing vs. net-new.

---

## Server-Side: ContentPricingRule

### Model

```ruby
# content_pricing_rules
# id               uuid PK
# store_id         uuid FK → stores, NOT NULL
# url_pattern      string NOT NULL    -- glob: "https://theirsite.com/articles/*"
# price_cents      integer NOT NULL
# gate_selector    string             -- CSS selector for gated nodes, e.g. ".post-content"
# paywall_position integer            -- paragraph/character offset for paywall insertion
# positioning_type string             -- "paragraph" | "character"
# active           boolean NOT NULL DEFAULT true
# created_at / updated_at
#
# Indexes:
#   (store_id, url_pattern) UNIQUE
#   (store_id) WHERE active = true
```

**Pattern syntax:** `*` matches any path segment sequence. `?` matches a single
character. No regex — deliberately limited to prevent DoS on pattern evaluation.
Matching is case-insensitive on scheme+host, case-sensitive on path.

Examples:

| Pattern | Matches | Does not match |
|---|---|---|
| `https://blog.example.com/posts/*` | `.../posts/my-article` | `.../posts/sub/nested` |
| `https://blog.example.com/posts/**` | `.../posts/sub/nested` | (double-star = recursive) |
| `https://blog.example.com/posts/*` | `.../posts/my-article?utm=1` | (query string stripped before match) |

### MatchUrl Service

```ruby
# app/services/content_pricing/match_url.rb
module ContentPricing
  class MatchUrl
    # Returns the first active ContentPricingRule whose pattern matches `url`,
    # ordered by pattern specificity (longer patterns win).
    # Returns nil if no rule matches.
    def call(store_id:, url:)
      # ...
    end
  end
end
```

Pattern specificity: exact URLs beat wildcard paths beat root wildcards:
- `https://site.com/articles/my-post` — score 3
- `https://site.com/articles/*` — score 2
- `https://site.com/**` — score 1

### FindOrCreateContent Service

```ruby
# app/services/content_pricing/find_or_create_content.rb
module ContentPricing
  class FindOrCreateContent
    # Finds an existing Content matching resource_url, or creates one from
    # the matched ContentPricingRule. Returns SuccessResponse(content).
    def call(rule:, url:)
      content = Content.publicly_visible.find_by(
        store_id: rule.store_id,
        resource_url: url
      )
      return SuccessResponse.new(content) if content

      title = derive_title(url)  # slug-to-title heuristic; overridable later
      content = Content.create!(
        store_id:     rule.store_id,
        content_type: "external_ref",
        content_uri:  url,          # the resource_url IS the content URI for external pages
        resource_url: url,
        title:        title,
        price_cents:  rule.price_cents,
        visibility:   "public",
        metadata: {
          paywall: {
            gate_selector:    rule.gate_selector,
            position:         rule.paywall_position,
            positioning_type: rule.positioning_type
          }
        }
      )
      SuccessResponse.new(content)
    end
  end
end
```

### Extended verify-origin behaviour

The existing `V1::X402::VerifyOriginController` currently only looks up
`Content.by_resource_url`. It needs one additional step: if no Content is found,
check `ContentPricingRule` for the store and lazily register.

```
GET /v1/x402/verify-origin?url=https://theirsite.com/articles/my-post

Current:  Content.by_resource_url(url) → found? return verified
Extended: Content.by_resource_url(url)
            └── not found?
                  └── ContentPricingRule.match(url)
                        ├── no rule → { verified: false }
                        └── rule matched
                              └── FindOrCreateContent → { verified: true, ... }
```

This keeps the endpoint's contract identical — callers never know whether the
content record pre-existed or was just created.

**Important:** `verify-origin` must not eagerly register content for every random
URL on the internet. The domain verification gate (next section) is what prevents
this: pricing rules are only active for verified domains.

---

## Domain Verification

Before a `ContentPricingRule` can fire, the store owner must prove they control
the domain. Without this, a malicious store could create a pricing rule for
`https://nytimes.com/**` and intercept purchases for content they don't own.

### Verification method: DNS TXT record

Simplest to implement, works for any domain, no HTTP access required.

```
TXT  _ledewire-verify.theirsite.com  "ledewire-site-verify=<token>"
```

Where `<token>` is a store-scoped random hex string generated by Ledewire and
shown to the store owner in their dashboard.

**Alternative: `<meta>` tag**

```html
<meta name="ledewire-site-verification" content="<token>">
```

Easier for store owners who can't edit DNS (shared hosting), but requires an
HTTP fetch to the site's root URL, which introduces network dependencies and
potential SSRF risk (see Security section).

**Recommendation:** Implement DNS TXT first; add meta tag as a fallback for
stores that request it.

### Verification lifecycle

```
store_domain_verifications
  id           uuid PK
  store_id     uuid FK → stores
  domain       string NOT NULL     -- "theirsite.com"
  token        string NOT NULL     -- random hex, unique
  status       enum: pending | verified | failed
  verified_at  datetime
  checked_at   datetime            -- last verification attempt
  created_at / updated_at

  UNIQUE (store_id, domain)
```

Verification is triggered on-demand (store owner clicks "Verify" in dashboard)
and re-checked periodically by a background job to catch token removal.

A `ContentPricingRule` is only `active` if its domain has `status: verified`.

---

## Content Registration Flow

The full sequence from "store owner wants to gate an article" to "user sees paywall":

```
1. Store owner logs into Ledewire dashboard
2. Adds domain "theirsite.com" → gets verification token
3. Places TXT record: _ledewire-verify.theirsite.com → "ledewire-site-verify=<token>"
4. Clicks "Verify" → Ledewire checks DNS → domain marked verified
5. Creates ContentPricingRule:
     url_pattern:   "https://theirsite.com/articles/*"
     price_cents:   100
     gate_selector: ".post-content"
6. Adds <script> tag to their page template

--- First reader visits the page ---

7. gate.js loads → calls GET /v1/x402/verify-origin?url=https://theirsite.com/articles/my-post
8. verify-origin: no Content for that URL
   → ContentPricingRule.match("https://theirsite.com/articles/my-post") → rule found
   → FindOrCreateContent → Content created (resource_url = full URL)
   → returns { verified: true, content_id: "...", amount: "100", title: "My Post" }
9. gate.js hides .post-content, renders paywall overlay
10. Reader clicks "Unlock for $1.00"
11. POST /purchases → accessToken issued → stored in TokenManager
12. gate.js dismisses paywall overlay
```

Steps 7–8 happen on every cold page load until the Content exists (then it's a
direct lookup against the partial index, served from the 1h HTTP cache).

---

## The `@ledewire/gate` Bundle (page-blocker)

This is the [`@ledewire/gate`](https://github.com/ledewire/embed) package, specifically
the `page-blocker.iife.js` entry point. It already exists as v0.0.1 and is served via
jsDelivr. The following defines the expected integration contract; where the existing
implementation differs, see the Phase 2 checklist for the delta work.

**Overlay approach:** `page-blocker` renders a **full-viewport frosted-glass blur overlay**
(`100vw × 100vh`, `z-index: 2147483647`) over the entire page, not a per-selector node
hide. This is harder to bypass than CSS class removal on individual nodes and provides
better UX parity with professional paywalls. The gate selector is used to identify *which
content region* the paywall overlays (affecting paywall copy and positioning), not to hide
DOM nodes.

### Embed interface

```html
<!-- Minimal -->
<script
  src="https://cdn.jsdelivr.net/npm/@ledewire/gate@latest/dist/page-blocker.iife.js"
  data-api-key="<publishable_api_key>"
  data-creator-id="<store-uuid>">
</script>

<!-- Full options -->
<script
  src="https://cdn.jsdelivr.net/npm/@ledewire/gate@latest/dist/page-blocker.iife.js"
  data-api-key="<publishable_api_key>"
  data-creator-id="<store-uuid>"
  data-external-url="https://theirsite.com/articles/my-post"
  data-theme="light"
  data-locale="en">
</script>
```

| Attribute | Required | Default | Description |
|---|---|---|---|
| `data-api-key` | Yes | — | Store's publishable API key. Used to authenticate `verify-origin` lookup. |
| `data-creator-id` | Yes | — | Store UUID. Scopes the content lookup. |
| `data-external-url` | No | `window.location.origin + pathname` | Override the URL used for content lookup. Useful for query-string-heavy or hash-routed pages. |
| `data-theme` | No | `"light"` | Paywall overlay theme. |
| `data-locale` | No | `"en"` | UI strings locale. |

> **Future work (noted in embed architecture doc as Low Priority #6):** `data-match-pattern`
> attribute to allow per-page gate selector overrides without re-deploying the script.

### Script behaviour

```
ON COMPONENT MOUNT (page-blocker renders its full-viewport blur overlay immediately)
  1. Resolve current URL: data-external-url attr, else window.location.origin + pathname
     (strip query string, normalise trailing slash)
  2. GET /v1/x402/verify-origin?url=<url>          (HTTP GET, 1h cached)
     ├── { verified: false } → remove overlay, exit (not a Ledewire content page)
     └── { verified: true, content_id, amount, title, gate_selector? }
  3. Check session storage for accessToken keyed by content_id
     (TokenManager.getAccessToken(content_id))
     ├── Present + not expired → removeGate() → done   (zero additional network calls)
     ├── Expired + buyer session present
     │     → POST /v1/purchases { content_id }         (idempotent — no charge if already owned)
     │       ├── 200 → fresh accessToken stored → removeGate()
     │       └── 422 (insufficient funds) → render paywall modal
     └── Absent, or expired with no session → render paywall modal

ON USER CLICKS "UNLOCK"
  4. Check Ledewire buyer session (TokenManager.getBuyerToken())
     ├── No session → open Ledewire login modal (email/password, Google OAuth)
     └── Has session → proceed to step 5
  5. POST /v1/purchases { content_id }              (Bearer: buyer JWT)
     ├── 200 → extract accessToken from response body
     │        → TokenManager.storeAccessToken(content_id, accessToken)
     │        → removeGate()
     └── 422 → insufficient funds → open add-funds modal

removeGate():
  - Dismiss full-viewport blur overlay
  - Token is retained in session storage for subsequent loads
```

**Note on x402:** The browser snippet uses `POST /purchases` (authenticated REST), not
the x402 protocol. x402 is the machine-to-machine protocol for unauthenticated clients
(browser extensions, AI agents, Cloudflare Workers). The x402 endpoint continues to work
for those contexts; the embed never uses it.

### Flash prevention

The full-viewport overlay is applied synchronously on component mount, before any API
calls complete. This prevents flash of unblurred content without requiring `<head>`
placement. Placing the `<script>` at the end of `<body>` is acceptable; placing it in
`<head>` is still recommended to minimise time-to-overlay on slow pages.

```html
<!-- Recommended: <head> placement for fastest overlay -->
<head>
  <script
    src="https://cdn.jsdelivr.net/npm/@ledewire/gate@latest/dist/page-blocker.iife.js"
    data-api-key="<key>"
    data-creator-id="<uuid>">
  </script>
</head>
```

---

## Purchase Flow Inside the Snippet

The snippet uses the authenticated REST purchase path, not x402. All required API
endpoints are either already built or have a small net-new addition:

| Step | API call | Status |
|---|---|---|
| Verify page is registered Ledewire content | `GET /v1/x402/verify-origin` | ✅ Built |
| Authenticate buyer | `POST /v1/merchant/auth/login` (managed by embed TokenManager) | ✅ Built |
| Purchase content + receive `accessToken` | `POST /v1/purchases` | ⚠️ `accessToken` field not yet in response |
| Verify `accessToken` offline (expiry check) | Client-side JWT decode (no network) | No server work needed |

The `POST /purchases` endpoint currently returns purchase metadata but not an
`accessToken`. Adding it is a small change to `DetailedPurchasePresenter` and the
`PurchaseResponse` OpenAPI schema — this is tracked in the Phase 1 checklist below.

The snippet does **not** need the JWKS endpoint for client-side expiry checking —
`exp` is a plain claim in the JWT payload, readable without signature verification.
JWKS is only needed for third-party servers that need to *cryptographically verify*
the token before trusting it (CMS plugin, Cloudflare Worker, etc.).

---

## Token Lifecycle

```
accessToken issued on POST /purchases response
  └── exp = now + 24h (config.x402_access_token_ttl)
  └── stored in TokenManager (InMemoryStorage default, or SessionStorageAdapter)
      keyed by content_id

On next page load (same tab / session):
  ├── token present in TokenManager, exp > now → gate removed immediately (no network call)
  └── token absent or expired
        └── GET verify-origin → confirmed Ledewire content
              └── if buyer session JWT present:
                    POST /v1/purchases { content_id }  (idempotent)
                    ├── 200 + already owned → fresh accessToken, no charge → gate removed
                    ├── 200 + new purchase  → accessToken, balance debited → gate removed
                    └── 422 (insufficient funds) → show paywall overlay
              └── if no session: show paywall overlay
```

**Storage adapter:** The embed uses `InMemoryStorage` by default (XSS-safe; tokens
lost on page refresh) and `SessionStorageAdapter` as an opt-in (tab-scoped;
cleaned up on tab close). Neither writes to `localStorage`. This means the zero-
network fast path currently only applies within a single page session; across page
loads the bundle always calls `verifyPurchase()`. Adding the `accessToken` to the
`POST /purchases` response and storing it in `SessionStorageAdapter` by default
would extend the fast path to the full browser session.

**Token expiry and re-issue (Model A — one-time purchase):** Ledewire purchases are
permanent — a user who bought an article owns it forever. The 24h `accessToken`
TTL is purely a *caching mechanism*, not a re-billing window. When a token
expires, the embed calls `POST /purchases` again. If the purchase record already
exists, the endpoint returns the existing purchase with a fresh `accessToken` and
does **not** debit the buyer's balance. The user is charged at most once per
content item. Future content types with time-windowed access (e.g. a daily digest
pass) would use a different model; this design does not prevent that extension but
does not implement it.

**Storage scope:** `SessionStorageAdapter` is tab-scoped; tokens for
`theirsite.com` are isolated from `ledewire.com` by the browser's same-origin
policy. This is correct isolation.

**Token accumulation and eviction:** Each purchased content item produces one
`accessToken` entry in `TokenManager`, keyed by `content_id`. A reader who buys
50 articles accumulates 50 entries (~500 bytes per JWT × 50 = ~25KB — well within
`SessionStorage`'s 5MB limit). However, `SessionStorageAdapter` has no eviction
strategy for expired entries: tokens past their 24h `exp` remain in storage until
the tab closes. For most users this is harmless, but `TokenManager` should prune
entries with `exp < now` on every mount to keep storage tidy. This is a
`ledewire/embed` repo concern, not an API concern — tracked in the Phase 2 delta
checklist below.

---

## Security Considerations

### Soft gate bypass

Accept it. Gate bypass is inherent to any client-side content gate. The business
value is real: the vast majority of users will not bypass paywalls. Sites with
truly secret content should use the hard-gate proxy path.

### Domain verification — SSRF risk (meta tag method)

If the meta tag verification method is implemented, the Ledewire server must fetch
`https://<domain>/` to check for the meta tag. This is a classic SSRF vector. Mitigations:
- Resolve domain to IP first; reject private ranges (RFC 1918, loopback, link-local)
- Use a short timeout (3s)
- Strip redirects (do not follow) or limit to HTTPS-only redirects to the same domain
- Run the fetch in a background job (not synchronously on the verification request)

DNS TXT verification has none of these risks and should be the primary method.

### CORS on x402 endpoints

`PAYMENT-REQUIRED` and `PAYMENT-RESPONSE` are already in `expose:` in `cors.rb`.
The `verify-origin` endpoint must also be accessible cross-origin from `theirsite.com`.
Because it is unauthenticated and returns no sensitive data, `Access-Control-Allow-Origin: *`
is appropriate (already set by the Rails CORS config for that namespace).

### Token storage: SessionStorage vs localStorage vs cookie

The `@ledewire/gate` embed uses `InMemoryStorage` by default (lost on page refresh,
but XSS-safe) and `SessionStorageAdapter` as an opt-in (tab-scoped, cleared on close).
Neither adapter uses `localStorage`, which avoids the classic XSS token harvest risk.

For the `accessToken` specifically:
- It is entitlement-only (proves a past purchase). It cannot log in or transfer funds.
- Storing it in `SessionStorageAdapter` (tab-scoped) is safe and appropriate.
- `localStorage` would extend the fast path across tabs and sessions but increases
  XSS exposure window. Defer this unless store owners report friction with the
  session-scoped model.

The Ledewire buyer session JWT should **not** be stored in `localStorage` — it is
already handled by the embed's `TokenManager` which uses `InMemoryStorage` or
`SessionStorageAdapter`, never `localStorage`.

### Content ID enumeration via verify-origin

The verify-origin endpoint returns a `content_id` UUID for any matched URL. This
is necessary for the snippet to perform the `POST /purchases` call, but it means
a caller can discover whether a URL has a Ledewire pricing rule. This is acceptable —
the `content_id` alone cannot be used to purchase (requires a funded Ledewire account
and a valid buyer session).

### Rate limiting

`verify-origin` already has a 60/min rate limit per IP. The lazy content creation
path in `FindOrCreateContent` must be idempotent (the `resource_url` partial unique
index guarantees this at the DB level) and should not be callable faster than the
rate limit allows — no additional guard needed.

---

## Store Owner Setup Experience

From the merchant dashboard perspective, onboarding a site for soft-gating is a
three-step flow:

**Step 1 — Add domain**

```
Domains → Add Domain → "theirsite.com"
→ Ledewire generates verification token
→ Shows instruction: "Add this TXT record to your DNS:
  _ledewire-verify.theirsite.com  TXT  ledewire-site-verify=abc123..."
→ "Verify" button → checks DNS
```

**Step 2 — Create pricing rule**

```
Content → Pricing Rules → New Rule
  Origin URL pattern: https://theirsite.com/articles/*
  Price per unlock:   $1.00
  Gate selector:      .post-content   (optional)
  Paywall position:   After paragraph 3  (optional)
→ Save
```

**Step 3 — Embed script**

```
Get embed code → copy/paste to site header:
  <script
    src="https://cdn.jsdelivr.net/npm/@ledewire/gate@latest/dist/page-blocker.iife.js"
    data-api-key="<publishable_api_key>"
    data-creator-id="<store-uuid>">
  </script>
```

That's the entire integration. The `data-api-key` is a publishable (read-only) key —
safe to embed in public HTML. No server-side changes, no CMS plugin required.

---

## Open Questions

1. ~~**Silent re-purchase threshold.**~~ Resolved: Ledewire supports **one-time
   purchases only** (Model A). `accessToken` expiry triggers a free re-issue via
   the idempotent `POST /purchases` endpoint — the buyer is never charged again for
   content they already own. The silent-threshold question was predicated on
   repeated charging and does not apply. If time-windowed access models are
   introduced in future, this question should be reopened.

2. **What happens if a rule is updated after content is created?** If a store owner
   changes `price_cents` on a `ContentPricingRule`, existing `Content` records
   created by that rule retain the old price. Should rule updates propagate to all
   associated Content records? Recommendation: propagate to contents that have not
   been manually edited (a `rule_managed: boolean` flag on `Content`). Contents
   edited directly in the dashboard are exempt from propagation.

3. **`gate_selector` from the rule vs. from `Content.metadata.paywall`.** The
   `FindOrCreateContent` service writes the rule's `gate_selector` into
   `Content.metadata.paywall.gate_selector`. Should the script read from
   verify-origin response (adds a field to that endpoint) or re-derive from the
   Content metadata (requires an extra API call)? Simplest: add `gate_selector`
   and `paywall_position` to the verify-origin response payload for rules-backed
   content. No new endpoint needed.

4. **Multi-rule overlap.** If a URL matches two rules (e.g. a specific-path rule
   and a wildcard root rule), which wins? Recommendation: most-specific wins
   (longest literal prefix before the first `*`). Document explicitly.

5. **Rule deactivation and Content visibility.** If a `ContentPricingRule` is
   deactivated, should all associated `Content` records be set to `visibility:
   private`? This would break any existing `purchase_id` links. Recommendation:
   deactivating a rule only stops lazy creation of new Content; existing records
   remain untouched. Store owner must manually archive Content if desired.

6. ~~**`gate.js` hosting.**~~ Resolved: `@ledewire/gate` is served from jsDelivr at
   `https://cdn.jsdelivr.net/npm/@ledewire/gate@<version>/dist/page-blocker.iife.js`.
   Semver-pinned paths (`@1`, `@latest`) are already the release strategy. Store owners
   who pin to `@latest` get non-breaking updates automatically; those who pin to a major
   version get only compatible updates.

---

## Implementation Checklist

### Phase 1 — Server-side backbone

- [ ] Design review sign-off on this document
- [ ] Migration: `content_pricing_rules` table
- [ ] Migration: `store_domain_verifications` table
- [ ] Model: `ContentPricingRule` with pattern matching logic
- [ ] Model: `StoreDomainVerification` with DNS TXT check
- [ ] Service: `ContentPricing::MatchUrl`
- [ ] Service: `ContentPricing::FindOrCreateContent`
- [ ] Extend `V1::X402::VerifyOriginController` with lazy registration path
- [ ] Add `gate_selector` and `paywall_position` to verify-origin response when rule-backed
- [ ] **Make `POST /purchases` idempotent** — if a purchase record already exists for `(user, content_id)`, return the existing purchase with a fresh `accessToken` and do **not** debit the buyer's balance (Model A: one-time purchase, permanent access)
- [ ] **Add `accessToken` to `POST /purchases` response** (`DetailedPurchasePresenter`, `PurchaseResponse` OpenAPI schema, `X402::AccessTokenService.mint`)
- [ ] Merchant API: CRUD for `ContentPricingRule` (`/v1/merchant/:store_id/pricing-rules`)
- [ ] Merchant API: domain verification flow (`/v1/merchant/:store_id/domains`)
- [ ] Background job: periodic re-verification of domain TXT records
- [ ] OpenAPI spec: new schemas and paths for the above
- [ ] Request specs for all new endpoints
- [ ] Service specs for `MatchUrl` and `FindOrCreateContent`

### Phase 2 — `@ledewire/gate` integration (separate repo, frontend work)

The frontend bundle ([`ledewire/embed`](https://github.com/ledewire/embed)) is already
built and released as v0.0.1. The following distinguishes what exists from what is
net-new delta work.

#### Already in `@ledewire/gate` ✅

- Full-viewport frosted-glass blur overlay with paywall modal (page-blocker)
- Login modal: email/password auth, Google OAuth UI
- Stripe add-funds modal
- Token lifecycle: `TokenManager` with 5-minute proactive refresh buffer
- Storage adapters: `InMemoryStorage` (default, XSS-safe) and `SessionStorageAdapter`
- Shadow DOM style isolation (overlay CSS cannot be clobbered by host page styles)
- CDN hosting via jsDelivr at semver-pinned paths
- CI + release workflows (GitHub Actions)

#### Net-new changes to `@ledewire/gate` (delta work)

- [ ] **Replace `searchContentByMetadata` with `GET /v1/x402/verify-origin`** in
      `page-blocker.tsx` — one function swap in `AuthService`; removes the
      requirement for content to be manually pre-registered in the dashboard
- [ ] **Store `accessToken` from `POST /purchases` response** in `TokenManager`
      (keyed by `content_id`) and check it on mount to skip `verifyPurchase()` call
      on warm loads within the same session
- [ ] **Implement `data-match-pattern` attribute evaluation** — 10-line fix already noted
      as Low Priority #6 in the embed architecture doc; enables per-page gate selector
      overrides without re-deploying the script
- [ ] **Prune expired `accessToken` entries from `TokenManager` on every mount** —
      currently no eviction; entries with `exp < now` accumulate until tab close
- [ ] Integration test: full flow against staging API (verify-origin → purchase → unlock)
