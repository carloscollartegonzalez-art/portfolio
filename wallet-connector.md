# Unified Wallet Connector — prototype for a Muse connector on Wallet A/Wallet B

> **Portfolio summary. The source code and supporting documents are private.**

**Thesis:** Agents will act as the new browser. Google and the Amazon-style marketplaces have been gamed into uselessness by ads and SEO farming, and people are tired of them. Once agentic assistants spike - and they will, because they solve exactly that problem - you need a new world: from the app store to the connector, from the app to the API.

## The problem

Meta's Muse opened a connector platform (`muse.ai/platform`) where third parties submit
an integration and Muse's agent calls it on the user's behalf. Muse's technical bar is
low and deliberately generic: **an OpenAPI document or an MCP server URL, reachable
without a login step, authenticated with a bearer token or API key.**

The wallet-side APIs of Wallet A and Wallet B are nothing like that. They're PSD2 Open
Banking APIs (separate wallet PSD2 portals):

- Two **separate** APIs, one per wallet, with different field names and status
  vocabularies, even though both are company X products.
- OAuth2 Authorization Code + **mandatory PKCE**, client registration via **mutual TLS
  with QWAC certificates** (bank-grade, not something Muse's platform speaks).
- Every payment requires **Strong Customer Authentication** — a 3-step
  preview → send → finalize flow with an OTP challenge in the middle.
- (There's also a third, unrelated surface — company X's separate merchant Payments API —
  which is merchant payment-*acceptance*, not wallet access. Easy to mistake for the
  same thing; it isn't.)

Muse cannot be pointed at either PSD2 API directly. Something has to sit in between.

## The proposal

This repo is that "something," as a working prototype: a single connector API that

1. **Normalizes** Wallet A and Wallet B into one resource model — `Wallet`, `Balance`,
   `Transaction`, `Transfer` — instead of two brand-specific ones. See
   `openapi/wallet-connector.yaml` for the full spec.
2. **Simplifies auth for Muse** to a single bearer token (`client_credentials` →
   `access_token`), instead of per-brand OAuth2 + PKCE + QWAC mTLS.
3. **Absorbs the consent + SCA flow** behind a hosted "connect your wallet" redirect
   (`POST /connect/sessions` → `consentUrl` → callback), the same pattern Plaid/TrueLayer
   use for Open Banking aggregation. Muse only ever sees a `walletId` on the other side.
4. **Never throws away provenance**: every response carries `provider`, and every error
   carries the underlying `providerCode`, so support/reconciliation can always trace a
   normalized object back to the real Wallet A or Wallet B record.

This illustrates the interface a wallet provider would need to expose (or approve a partner exposing) for a
Wallet A/Wallet B wallet connector to be viable on Muse — or on any other agent platform
with similar constraints (ChatGPT Actions, Claude/MCP, etc. all want the same thing:
a plain OpenAPI/MCP surface with simple bearer auth).

## Reading a wallet vs. paying with one

The first cut of this prototype only covered wallet self-service: balance, transaction
history, and a P2P-style `/transfers` endpoint. That's necessary but not sufficient —
the actual requirement is letting Muse **pay a merchant with the wallet**, instead of
falling back to Muse's own default rails. Whether that's possible, and how, depends on
whether the merchant already accepts Wallet A/Wallet B:

- **`POST /wallets/{id}/merchant-payments`** — for a merchant that already accepts
  Wallet A or Wallet B wallet-to-wallet (a company X-acquired merchant like Norwegian, or
  any merchant that separately integrated Wallet A/Wallet B Quick Checkout, regardless of
  who their card acquirer is). This is mechanically the *same* PISP "send money" call as
  `/transfers` — The wallet provider documentation describes wallet checkout as paying a merchant
  "directly from your wallet account." It's a distinct resource because a merchant needs
  `merchantId`/`orderId` for reconciliation and a receipt-shaped response, not a P2P
  transfer record. Same SCA/OTP confirm step as a transfer.
- **`POST /wallets/{id}/cards`** — for a merchant that does *not* accept Wallet A/Wallet B
  at all. Both wallet providers issue a wallet-funded virtual Mastercard usable anywhere Mastercard
  is accepted online. A card token handed to Muse's own checkout rails works at *any*
  acquirer — Stripe, Adyen, whoever — because to the merchant it's an ordinary card
  transaction. This is the only acquirer-agnostic path, and it's the one that pulls in
  real compliance weight: **this endpoint never returns a PAN or CVV**, only a masked
  `last4`/expiry for display and an opaque `cardToken` a checkout rail exchanges
  server-to-server with the card network's tokenization service. Raw card data should
  never transit an LLM agent's context, independent of who's asking for it — that's a
  design decision worth testing with the relevant provider, not an implementation
  detail to gloss over.

## Muse decides, not the connector

Neither `/merchant-payments` nor `/cards` can make Muse prefer this wallet over its own
default rail (Stripe Link, via the Agentic Commerce Protocol Meta co-developed with
Stripe and OpenAI). The connector is a tool Muse *can* call — it has no way to intercept
or override Muse's own checkout logic. In practice that means this wallet only gets used
when: the user asks for it by name in that purchase, the merchant has literally no
card/ACP option, or the user has set a **standing default**.

That third case is the one worth building for, because "tell Muse every single time" is
not a real product. So every linked wallet carries:

- **`isDefaultForPurchases`** (boolean, off by default) — set via `PATCH /wallets/{id}`.
  Only one wallet can be default at a time, same shape as a default payment method
  picker. Once set, Muse can check this on `GET /wallets` *before* falling back to Link,
  instead of needing the user to repeat the instruction on every purchase.
- **`defaultWalletPrompt`** — not a tip, a literal yes/no question (*"Do you want to
  always pay with this wallet when possible?"*) plus the exact `PATCH` call for
  each answer, returned in the `/connect/sessions/{state}/callback` redirect right when
  the wallet finishes linking. Earlier drafts of this used a passive `usageHint` string
  ("say X to set this as default") — that relies on the user remembering the right
  phrase days or weeks later. Asking a direct question at the one moment the user is
  already paying attention (right after linking) is a materially better bet, and
  removing the need to interpret free-text phrasing removes a failure mode entirely.
  It's present once and disappears from later `GET /wallets` calls as soon as it's
  answered (yes or no) — it's an onboarding question, not a standing nag.

**Two separate unknowns worth keeping apart.** Whether Muse's own agent logic actually
asks `defaultWalletPrompt.question` and honors the answer is unconfirmed — nothing
publicly documented about Muse's connector platform says either way, and that's a
question for Muse's connector team. But there's a second, more structural question this
signal can't answer at all: **the connector platform and ACP look like two entirely
separate integration tracks.** `isDefaultForPurchases` lives inside *this* API — Stripe
has no visibility into it, and can't "block" it, but by the same token it has no
authority inside Muse's actual ACP checkout either. A connector (the self-serve,
`muse.ai/platform` kind) is a tool Muse can choose to call instead of running checkout;
an ACP payment handler (the Stripe SPT kind) is a formal, merchant-declared,
protocol-level participant that competes directly at checkout time. Nothing found in
research confirms a general connector can become an ACP handler — that's likely a
separate, heavier process with Meta/OpenAI/Stripe as protocol maintainers, not something
achievable through connector API design alone. Worth validating before a real integration
rather than assuming this connector is a path to that.

## What's in this repo

- **`openapi/wallet-connector.yaml`** — the actual artifact to hand to both sides: it's
  what you'd register with Muse's connector platform, and it's the spec to negotiate
  with the provider API team. OpenAPI 3.1, fully self-contained.
- **`src/`** — a runnable Node/TypeScript server implementing that spec, with mocked
  Wallet A and Wallet B backends standing in for the real PSD2 APIs. The mocks
  deliberately use different field names per brand (`customerAccountId`/`emailMasked`/
  `SETTLED` vs `customerId`/`memberEmailMasked`/`COMPLETE`) so the adapter layer
  (`src/adapters.ts`) does real normalization work, not just pass-through.
- **`src/demo.ts`** — scripted walkthrough of the full flow end to end.

## Running the demo

```bash
npm install
npm run dev     # starts the server on :4000
npm run demo     # in a second terminal: links both wallets, sets Wallet A as
                  # the standing default, reads balance/transactions, sends
                  # a P2P transfer, pays a merchant (wallet-to-wallet), and
                  # issues a virtual card
```

## Mapping cheat sheet (illustrative)

| Unified concept                | Wallet A PSD2 | Wallet B PSD2 (expected, same compliance shape)       |
|---------------------------------|-----------------------------------------------------------------|---------------------------------------------------------|
| `GET /wallets/{id}/balance`     | AISP "Get Customer Accounts Information"                        | AIS balance equivalent                                   |
| `GET /wallets/{id}/transactions`| AISP "Get Transaction History for Specified Customer Account"   | AIS transactions equivalent                               |
| `POST /transfers`               | PISP "Send Money Preview" + "Send Money" (returns SCA challenge) | PIS payment initiation (returns SCA challenge)             |
| `POST /transfers/{id}/confirm`  | PISP "Send Money Finalize" (OTP)                                  | PIS confirm (OTP / app push)                               |
| `POST /connect/sessions`        | OAuth2 Authorization Code + PKCE consent redirect                | OAuth2 Authorization Code + PKCE consent redirect          |
| `POST /merchant-payments`       | Wallet Checkout — pay a merchant's Wallet A account from balance  | Wallet Checkout equivalent                                 |
| `POST /cards`                   | a wallet-funded virtual Mastercard                                | a wallet-funded virtual Mastercard                            |

Wallet B's exact PSD2 endpoint names weren't reachable during this research pass (the
reference page 404'd); the mock mirrors the same AIS/PIS shape Wallet A documentation describes, since
both are modeled as PSD2 wallets under the same regulatory framework. **Confirming the real Wallet B PSD2 endpoints and fields is a prerequisite before any production build.**

## What's intentionally out of scope here

- No real wallet credentials or network calls — everything is mocked in memory.
- No persistence (wallets/tokens/transfers live in process memory, reset on restart).
- No token encryption, rate limiting, or production auth hardening — the point of this
  prototype is the *shape* of the API, not a production-ready implementation.
- No MCP wrapper — Muse also accepts MCP server URLs as an alternative to OpenAPI, so if
  the wallet provider or Muse review team prefers that transport, the same routes can be re-exposed
  as MCP tools without changing the underlying model.
- No card capture/usage modeling — `POST /cards` issues a card and reserves the
  spend limit against the mock balance, but doesn't simulate the merchant actually
  charging it. In production, card issuance and network tokenization would go through
  the providers’ real card-issuing APIs, not a mock.
- Merchant payment webhook delivery (`notifyUrl`) is fire-and-forget with no retry —
  fine for a prototype, not fine for real order reconciliation.
