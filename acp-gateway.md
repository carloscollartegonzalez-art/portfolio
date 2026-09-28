# company X Agentic Commerce Gateway — ACP + AP2 prototype

> **Portfolio summary. The source code and supporting documents are private.**

**Thesis:** Agents will act as the new browser. Google and the Amazon-style marketplaces have been gamed into uselessness by ads and SEO farming, and people are tired of them. Once agentic assistants spike - and they will, because they solve exactly that problem - you need a new world: from the app store to the connector, from the app to the API.

## What this is

The roadmap: **ACP capability** (done, below), then **AP2 + UCP** (AP2 done, below; UCP
deliberately deferred — see "Scope decisions"), then a full merchant suite. One gateway, one
underlying risk/liability design philosophy, multiple protocol adapters — the same "evolutions
of one product, not separate rebuilds" shape as the wallet connector's adapter/connector split.

**ACP**: company X as a PSP implementing the [Agentic Commerce Protocol](https://agenticcommerce.dev)
(co-developed by Stripe, OpenAI, and Meta), so a company X-acquired merchant gets agent-checkout
compatibility "with one integration" (a proposed approach) instead of building it themselves.

**AP2**: company X as a Payment *Processor* (a deliberate, narrower role choice — see below) in
Google's [Agent Payments Protocol](https://ap2-protocol.org), settling already-signed Payment
Mandates rather than issuing them.

## Two products, not one, sharing one foundation

Merchants split into two situations, and they need different things:

- **Merchants that don't have (and may never build) their own agent connector.** An agent
  reaching them falls back to browser automation — clicking through the regular checkout,
  exactly the risk the top of this README describes. These merchants need something they can
  adopt *today*, independent of ACP/AP2/UCP entirely: **the Payment Gate**
  (`POST /gate/charge`, `src/gateEngine.ts`) — a single endpoint that takes a payment regardless
  of how the buyer arrived, and is honest about the fact that a browser-driven payment with no
  protocol backing carries real, un-shiftable liability. This is the thing to ship first — light
  integration, immediate value, no protocol adoption required.
- **Merchants that want the full thing.** Discovery through a real connector *plus* payment
  (ACP, AP2, or the Gate) integrated together, so the agent discovers **and** pays through one
  company X integration. This is **the Suite** — Connector + Gate.

**One correction worth being precise about: the Gate half of that sentence is true; the
Connector half is not, yet.** The Gate is genuinely generic — protocol-agnostic, merchant-agnostic,
a finished product today. The Connector is not. What exists is Duffel hardcoded directly inside
`catalogSearch.ts` and `checkoutSessions.ts` — real proof the translation pattern works for one
genuinely complex backend (airline NDC), not a pluggable "any merchant's catalog becomes an
ACP-readable API" layer. A second merchant type means writing new route handlers today, not
registering a new adapter. "Packaging, not re-invention" is true for the Gate; turning the
Connector into something that deserves the word "Suite" needs a real adapter interface + registry
(`interface CatalogAdapter { search, createSession, bookOrder }`, Duffel as one implementation)
— and proof via a second, structurally different adapter, not another airline. See the artifact
for the concrete missing-abstraction sketch.

The Gate is the foundation both share. It doesn't re-invent risk scoring — a `connector`-origin
charge re-surfaces a decision this same gateway's `delegate_payment` or `ap2/settle` already
made; a `browser`-origin charge runs a deliberately harsher assessment because there is nothing
to verify at all, only the merchant's own suspicion.

This is the **merchant/acquirer side** — the counterpart to the earlier
`wallet-connector.md`, which was the consumer/agent side. Two
different directions on the same problem: that one let an agent *spend from* a wallet; this one
lets a *merchant accept* an agent-initiated payment.

## Why agent-initiated payments need a protocol at all

Without ACP/AP2/UCP, an agent without an API integration has to fall back to browser
automation — clicking through checkout like a human. That's not just fragile (UI changes break
it, CAPTCHAs block it, merchants can't distinguish it from bot traffic and may block it
outright). The sharper problem is where payment data has to flow: without a structured,
token-based handoff, the agent (or whatever's driving the browser) ends up typing raw card
numbers into a form — pulling sensitive payment data through a channel never designed to carry
it securely. ACP's `delegate_payment` primitive exists specifically to avoid that.

## The design decision this prototype makes visible: fpan vs. network_token

ACP's real spec (`PaymentMethodCard.card_number_type`) allows either `fpan` (the raw card
number, optionally with raw CVV) or `network_token` (a card-network-issued token, the same
mechanism Apple Pay/Google Pay use). company X, as the PSP implementing this endpoint, decides
which to accept. That single field is the fork point for PCI scope, fraud exposure, and
chargeback liability — three things this prototype makes explicit and observable rather than
leaving as an implementation detail nobody sees until an audit.

**The strategic reasoning, not just the technical one:** company X doesn't write ACP, AP2, or UCP
— Stripe/OpenAI, Google, and Google+Shopify do. company X is a protocol *implementer* in a
landscape where three different well-funded parties are still competing over which standard
wins, and has no market power to force the ecosystem toward one credential type. Betting the
whole platform on "we'll only ever need fpan" or "we'll only ever need tokens" is a bet on which
spec dominates — a bet company X can't influence. **This prototype is deliberately agnostic: it
accepts both.** See `GET /agentic_commerce/policy` for that stance as a checkable fact, not a
claim.

One nuance worth being honest about: supporting both isn't "free" just because it's possible.
PCI DSS scope isn't an average — the moment any part of a system *can* receive `fpan`, that
code path (and everything network-connected to it) is in full PCI DSS scope, even if 95% of
traffic is tokenized. The real architectural requirement (not fully modeled in this prototype —
see below) is keeping the `fpan` path in its own deliberately isolated, segmented component so
it doesn't drag the rest of the platform into that scope.

**Whether this even matters for the *build* cost is an open question, not a settled one.**
An existing acquirer may already have PCI DSS infrastructure (HSMs, segmentation, QSA relationships), but this needs verification before any cost claim. If the PSP checkout supports Apple Pay or Google Pay — **unconfirmed** — then network-token/cryptogram validation
is probably *also* already-existing infrastructure, not new either. If both are true, the real
incremental cost for either path is concentrated in the agent-facing API surface and new
fraud/liability handling, not in rebuilding payment primitives from scratch. That's a materially
different pitch than "tokens are cheap, fpan is expensive" — the difference between the two
paths is mostly in *ongoing* exposure (audit scope, breach cost, engineering velocity), not
upfront build cost.

## Fraud: what's actually different, made visible in `src/riskEngine.ts`

**This is a deliberately simple, transparent, explainable scoring function — not a real fraud
model, and it must not be mistaken for one.** ⚠️ **Flagging this explicitly: before this becomes
a real product, this engine needs to be replaced or substantially enhanced with production risk infrastructure** (whatever combination of rules engines, ML scoring, and network-provided
signals a production fraud team already runs for card-not-present transactions). What's here exists
to demonstrate *what changes* about fraud assessment for agent-initiated payments, not to be
shipped as-is:

- A verified network-token cryptogram is a real, structural advantage — cryptographic proof a
  transaction is fresh and not replayed, not just "trust the token." The engine rewards it
  accordingly (see the scoring walkthrough in the code comments).
- A raw-PAN transaction has no equivalent — it falls back to whatever AVS/CVV checks were
  performed, and gets penalized when neither was done.
- Critically: **a card-testing risk signal blocks a network-token transaction too.** Scenario 3
  in the demo proves this on purpose — tokenization lowers baseline risk and PCI burden, it does
  not make fraud detection optional. A demo that only ever showed tokens sailing through would
  be a strawman, not evidence.
- Agent-specific fraud patterns this simple model does *not* yet cover, and a real build would
  need to: agent-driven card testing at automated velocity (an attacker can attempt far more
  guesses per minute than a human typing into a form), and fraud models trained on human
  behavioral signals (typing cadence, mouse movement) systematically misreading legitimate agent
  traffic as bot-like unless specifically retrained.

## Liability: two separate questions, kept on separate axes (`src/liabilityEngine.ts`)

1. **`chargebackLiability`** — for a classic "this wasn't the real cardholder" dispute, who eats
   it. This mostly follows established card-network liability-shift practice: a verified
   cryptogram shifts liability toward the issuer (the same mechanism behind EMV and tokenized
   wallet liability shifts); a raw-PAN transaction without strong authentication defaults to the
   merchant/PSP under standard card-not-present rules. Marked `confidence: "established"`
   because this is standard, well-understood industry practice — not something invented for
   this prototype.
2. **`relatedOpenQuestions`** — a *different* dispute shape, new to agent-initiated commerce,
   that applies **regardless of card_number_type**: "I never authorized my agent to make this
   purchase at all." Whether existing card-network liability-shift rules were ever designed to
   cover a dispute about delegated agent authority — as opposed to a stolen card — is not
   settled industry-wide, and this prototype does not pretend otherwise. It's the second-order
   consequence of `delegate_authentication` (ACP's OAuth 2.0-based "let an agent act on a
   buyer's behalf" primitive): stealing or misusing delegated *authority* is a new fraud/dispute
   category distinct from stealing a card number, and nobody's liability-shift rules were
   written with it in mind yet.

## AP2: a different mandate model, a different risk story

AP2 (confirmed directly from [ap2-protocol.org](https://ap2-protocol.org), not from a search
summary — see the earlier session's note on why that distinction matters) is built around two
mandate types, each with an **open** stage (constraints/goals, captured before anything is
finalized) and a **closed** stage (the specific, finalized authorization): a **Checkout Mandate**
and a **Payment Mandate**. Mandates are cryptographically signed Verifiable Digital Credentials
(VDCs). AP2 never moves money itself — it produces a verifiable authorization record any rail
settles against. Roles: User, Agent, Merchant, Credential Provider, Payment Processor.

### Scope decisions made for this prototype

- **company X modeled as Payment Processor only**, not Credential Provider. This is the role
  closest to what company X already is — an acquirer executing settlement — and it's the fastest
  path to serving a broad merchant base: a processor can settle *any* trusted mandate regardless
  of which credential provider issued it, without first onboarding every end user as a company X
  credential holder. The Credential Provider role is a different, bigger bet — and notably, it's
  architecturally close to what the wallet connector prototype already does for Muse (managing a
  consumer's authorization for an agent to spend their money). Worth revisiting together, not
  starting from zero, if company X wants that role later.
- **UCP deliberately deferred.** UCP's own scope — product discovery, checkout, *and*
  post-purchase support — is broader than ACP's checkout layer was, so "just enough to close the
  loop" isn't a small addition the way `checkout_sessions` was for ACP. AP2's mandate model is
  where the new fraud/liability substance lives; UCP's discovery/cart layer is comparatively
  generic e-commerce plumbing. Same sequencing logic as ACP: build the substantive part first.
- **Mandate trust chain (verifying the *issuer*, not just the signature) is deferred too.** AP2
  separates Credential Provider from Payment Processor explicitly because a real ecosystem has
  many credential providers — whether company X trusts a *specific* one is a real, PKI-shaped
  question this prototype doesn't model (`src/routes/ap2Settle.ts` checks only that a signature
  field is present, not that it's from a recognized issuer). A good next addition, not core to
  the headline story chosen for this phase.

### The risk story: constraint violation as the primary gate, human-presence as a secondary factor

AP2 expects whoever signs a closed Payment Mandate to have already enforced the open mandate's
constraints (max amount, currency, expiry). **A Payment Processor trusting that blindly repeats
the exact gap the ACP prototype's Scenario 3 warned against** — a cryptographically valid
signature proves the mandate wasn't tampered with, it doesn't prove its contents make sense.
`src/ap2RiskEngine.ts`'s `checkConstraints` is this prototype’s independent, defense-in-depth check,
and it's a hard gate: if the closed mandate's amount exceeds the open mandate's `max_amount` (or
the currency doesn't match, or it's issued after the open mandate expired), settlement is
**rejected outright** — regardless of amount, regardless of whether a human was present. See
Scenario 2 in `npm run demo:ap2`.

Human-presence is a *secondary* factor, not a gate: AP2 explicitly supports both human-present
and human-not-present authorization, and this maps directly onto the oldest risk distinction in
card payments — card-present (strong real-time proof) vs. card-not-present (weaker proof, relies
on what was authorized earlier). A human-present, in-bounds mandate is about as strong a
non-repudiation position as exists. A human-not-present, in-bounds mandate is legitimate, but its
liability exposure depends entirely on how tightly the *earlier* standing authorization was
scoped — which is precisely where the ACP prototype's abstract "delegated authority" open
question becomes concrete: AP2's signature chain proves the mandates are authentic, it does not
settle whether "I authorized this" covers a purchase no human actually confirmed in the moment.
`src/liabilityEngine.ts`'s `attributeAp2Liability` keeps this as `open_industry_question`
confidence rather than picking a comfortable answer.

⚠️ Same caveat as `riskEngine.ts`: **`ap2RiskEngine.ts` is a deliberately simple, explainable
scoring function, not a real model, and must not be mistaken for one before this becomes a real
product.**

## Phase 3 preview: a real merchant catalog, not a mock (`src/duffelClient.ts`)

Both `checkout_sessions` implementations above took client-supplied `line_items` directly — no
real product/catalog behind them, explicitly flagged as deferred. This is what closes that gap
for one real, complex merchant category (flights), and it's the first piece in the private source repo that
calls a genuinely live, external, third-party API instead of a mock we wrote ourselves.

**Is "turn a merchant's catalog into an agent-consumable API" a real product category?** Yes —
and it's already forming. UCP itself exists because every merchant will eventually need this.
Big commerce platforms (Shopify) are building the *native* version for merchants already on
their platform. The harder, more valuable gap is merchants whose catalog does **not** live on a
platform like that — a legacy ERP, a custom booking system, or (the case here) an airline's
NDC/GDS stack. The strategic choice for company X: build this standalone (competing with
Shopify/commercetools for catalog ownership they already have — a hard, crowded lane), or
**bundle it with the payment capability already built in the private source repo** — "we already process your
agent payments, we'll also make your catalog agent-ready with the same integration." The second
is the stronger position: it leverages a relationship company X already has instead of competing
for one it doesn't.

**Why Duffel, not Norwegian directly.** Norwegian's own NDC API is real and documented
(`services.dev.norwegian.com`, NDC v17.2 — confirmed by direct fetch), but access requires
registering as a "Party" with an `AgencyID`, which is a partnership/certification step, not
self-service. [Duffel](https://duffel.com/docs/api) is an NDC *aggregator* — real self-service
signup (about a minute, confirmed from their docs), a free test token, no accreditation for
sandbox use, and their sandbox runs against a real dummy airline ("Duffel Airways") on their own
live servers. That's the realistic integration shape for most merchants in complex verticals:
not a direct certified connection to every merchant individually (doesn't scale), but a
connection to an aggregator that already speaks the merchant's native protocol to hundreds of
them — with company X's own value-add being the payment + ACP/UCP protocol translation on top,
which is what the rest of the private source repo already is.

**The flow**, confirmed from Duffel's real docs, not invented: `POST /catalog/search`
(→ Duffel `offer_requests`, real priced offers) → `POST /checkout_sessions` with a
`duffelOfferId` (the offer is **re-validated live** against Duffel before the session is
created, because "offers get stale fairly quickly" per their own docs) → `.../complete` (a real
`POST /air/orders` booking attempt, returning a real airline booking reference).

**Confirmed live, not just designed.** The `payments` object shape for order creation wasn't in
Duffel's docs during research, so the first live run used a placeholder amount (`"0.00"`) — and
got back exactly the kind of real signal this design decision predicted: a genuine Duffel 422,
`"Field 'payments' must match the order total amount."` Fixed by fetching the offer's real total
fresh (`getOffer`) at completion time rather than trusting an earlier response, and passing that
through. Re-ran end to end: a real booking, confirmed on Duffel's own servers, with a real
airline booking reference (`5UDCLL` in the run that confirmed this) — search → live price
re-validation → ACP `delegate_payment` → real order creation → `liabilityAttribution` attached
to a real order. Every step in that chain is real; nothing in this specific path is mocked.

```bash
# get a free test token first: duffel.com signup, Developers > Access tokens
export DUFFEL_API_TOKEN=<your test token>
npm run demo:catalog
```

Without a token set, this still runs and shows exactly what's missing (a 424, with instructions)
— it does not fall back to fake data, because the entire point of this path is that it's real.

## The Payment Gate: one endpoint, two trust situations (`src/gateEngine.ts`)

`POST /gate/charge` takes an `origin` field, and that field alone decides everything:

- **`origin: "connector"`** — the payment already went through this gateway's own ACP
  `delegate_payment` or AP2 `settle`. The Gate does not re-score. It looks up the vault token or
  AP2 settlement by id and re-surfaces the decision that already exists — re-deriving it would
  either duplicate work or, worse, silently produce a different answer than the connector gave.
  Demo Scenarios 1 and 2 confirm this: same risk score, same liability attribution, just
  presented through the Gate's shape instead of ACP's or AP2's.

  **The connector fraud protection, specifically:** a valid, correctly-signed token or mandate is
  not sufficient on its own — it has to actually authorize *this* charge. `chargeFromConnector`
  verifies the merchant, currency, and amount ceiling on every request before trusting a token or
  settlement, not just that it exists and is real. Scenario 1b proves the catch: the exact same,
  genuinely valid vault token from Scenario 1, reused against a different merchant and an amount
  far past what it was authorized for — declined, risk score 100, with the specific mismatch
  named in the response. A real signature answers "was this credential legitimate," never "does
  it authorize what's being charged right now" — without this check, a correctly-verified token
  from one transaction could be replayed against a completely different one and the Gate would
  have approved it purely because the underlying cryptography checked out.
- **`origin: "browser"`** — the buyer went through the merchant's own regular checkout, possibly
  driven by an agent using browser automation. There is no cryptogram, no signed mandate,
  nothing to verify — only the merchant's own suspicion (`browserSignals.automationSuspected`).
  This is scored closer to worst-case card-not-present risk, and — this is the important part —
  **liability can only ever land on the merchant here**, in both Scenario 3 (declined for review)
  and Scenario 4 (approved). There's no mechanism to shift it anywhere else, because there's
  nothing cryptographic to point to. That's not a scoring choice; it's a direct structural
  consequence of skipping the connector.

**That asymmetry is the actual business case for the Suite, not just a nice-to-have.** A
merchant who only adopts the Gate gets payment processing for agent traffic, but keeps 100% of
the chargeback risk on every browser-origin transaction, forever. A merchant who adopts the
Suite (Connector + Gate) gets transactions that carry real liability protection instead. The
Gate makes that gap visible and quantifiable rather than an abstract sales pitch — see
`relatedOpenQuestions` in Scenarios 3 and 4, which name the incentive problem directly: without
a reason to adopt a connector, why would a merchant ever move off the cheaper, riskier path?

```bash
npm run demo:gate   # all five scenarios, run end to end (including the mismatch catch)
```

## The Suite, actually wired together (not just a diagram)

Building the Gate and the Connector as separate products doesn't make a Suite by itself — it
makes two things a merchant has to remember to call correctly, in the right order, or the
protection quietly doesn't apply. That gap was real in the private source repo until now: `checkout_sessions`
`.../complete` finalized a real Duffel booking without ever calling the Gate, meaning the
connector fraud protection built for the standalone Gate demo didn't actually protect the one
real, live booking flow that exists.

**Fixed by making completion call the Gate internally**, using the live-refreshed offer amount
(the same value used for the real Duffel charge, not a client-supplied one) — automatically, not
as a step the caller has to remember:

- **`declined`** blocks completion outright (402, before the vault token is consumed or Duffel is
  ever called).
- **`review`** does *not* block completion — this caught a real bug while wiring it in. The first
  version treated `review` the same as `declined`, which broke a working scenario:
  `manual_review` has always meant "hold for a human to look at," not "reject," and orders have
  always completed under it elsewhere in the private source repo (with `psp` liability attached). Fixed by
  only hard-blocking on `declined`, and marking `review` orders with `flaggedForReview: true`
  instead of blocking them. Caught by running the demos after wiring, not by reasoning about it
  in the abstract — exactly why "type-checks" and "is correct" are different claims.
- The completed order's `liabilityAttribution` is now sourced directly from the Gate's decision,
  not re-derived separately — one authority, not two copies of the same logic that could drift.

Verified live end to end after the fix: a real Duffel booking (`npm run demo:catalog`), with the
Gate's `"verified to match this charge"` decision now present in the same response, automatically.
That's what makes this the Suite rather than the Gate and the Connector coincidentally existing
in the same repo.

## Who finishes this — engineering vs. organizational sign-off

Getting from this prototype to a real, pluggable "merchant just integrates with one line of
code" product is mostly a large but ordinary engineering effort — merchant onboarding and
multi-tenancy, SDKs, webhook infrastructure, a merchant dashboard, protocol-version handling.
That part, engineering can carry end to end. But a few pieces are not engineering problems no
matter how much of them get built, and planning around that distinction matters more than
writing more code:

| Who does it | What they own |
|---|---|
| **Engineering** | Every line of code: the gateway, integrations, SDKs, dashboard, webhook infra, multi-tenancy, protocol versioning — the entire technical surface. |
| **Compliance/security team + an external QSA** | PCI DSS certification. Structurally not an engineering deliverable — a certification is a third party's independent attestation, by definition. Engineering can build the segmented architecture and prepare every piece of evidence an audit needs, but can't be the auditor. |
| **Legal team** | The delegated-agent-authorization liability question (see above). Engineering can flag it and model both outcomes in code so it's never silently assumed away — but the legal position would need independent review. |
| **Risk team** | Signing off on real fraud thresholds and model behavior once real money is at stake. Engineering builds the engine and the tooling to tune it; the risk tolerance itself is a business decision that needs an accountable human owner. |
| **Infra/security team** | Granting access to real production systems and cardholder data in the first place, and reviewing security posture before anything goes live. This gate exists independent of who wrote the code. |
| **Ops/support org** | On-call, SLAs, incident response once this serves real merchants. |

The practical consequence: this project has real checkpoints where it has to leave engineering's
hands and go through one of those other teams before it can move forward — a security review
before real data flows, a QSA engagement before compliance can be claimed, a legal sign-off
before liability terms are published, a risk-team approval before fraud thresholds go live.
None of those are things more/better engineering removes.

## What's in the private source repo

- **`src/types.ts`** — field names mirror the real ACP OpenAPI spec (`delegate_payment`,
  `agentic_checkout`, spec version 2026-04-17) wherever practical. `dataHandling`,
  `fraudAssessment`, and `liabilityAttribution` are prototype-specific additions layered on top —
  not part of ACP itself.
- **`src/pciVault.ts`** / **`src/riskEngine.ts`** / **`src/liabilityEngine.ts`** — the three
  things this prototype exists to make visible.
- **`src/routes/`** — `GET /agentic_commerce/policy` (a prototype-specific addition, not ACP), `POST
  /agentic_commerce/delegate_payment` (mirrors the real spec), `POST /checkout_sessions` +
  `.../complete` (a minimal slice of the real Agentic Checkout API, just enough to close the
  loop from policy declaration to a finished order with liability attached), `POST /ap2/settle`
  (AP2, Payment Processor role only).
- **`src/ap2RiskEngine.ts`** — the constraint-check-as-hard-gate, human-presence-as-secondary-
  factor logic described above.
- **`src/duffelClient.ts`** — a real HTTP client for Duffel's live API (flight search, offer
  revalidation, order creation). Not a mock.
- **`src/routes/catalogSearch.ts`** — `POST /catalog/search`, the discovery/cart layer the ACP
  section above deferred, now backed by a real merchant catalog.
- **`src/gateEngine.ts`** / **`src/routes/gate.ts`** — `POST /gate/charge`, the standalone
  product: origin-aware, re-surfaces connector decisions or scores browser-origin ones from
  scratch.
- **`src/demo.ts`** — the three ACP scenarios, run end to end.
- **`src/demo-ap2.ts`** — the three AP2 scenarios, run end to end.
- **`src/demo-catalog.ts`** — real flight search → live-revalidated checkout → real booking
  attempt, requires a free Duffel test token (see above).
- **`src/demo-gate.ts`** — five Gate scenarios: connector re-surfacing, the connector fraud
  protection catching a mismatched replay, and two browser-origin scorings, run end to end.

## Running the demo

```bash
npm install
npm run dev         # starts the server on :4100
npm run demo         # in a second terminal: ACP -- policy check, then all three scenarios
npm run demo:ap2     # or this instead: AP2 -- all three mandate-settlement scenarios
npm run demo:catalog # or this: a real Duffel flight search + booking (needs DUFFEL_API_TOKEN)
npm run demo:gate    # or this: the Payment Gate -- connector re-surfacing + browser-origin scoring
```

## What's intentionally out of scope here

- **The fraud engine is a placeholder, not a model** — see the flagged warning above. This is
  the single most important caveat in the private source repo.
- No real PCI-scoped infrastructure — the fpan and network_token paths are two branches of one
  function (`pciVault.ts`), not two genuinely network-segmented services. A real build must make
  that segmentation an actual deployment boundary, not an if-statement.
- No real card-network integration (VTS/MDES detokenization, actual cryptogram validation) — the
  cryptogram check here is "is this field non-empty," not real cryptographic verification.
- No 3DS/step-up authentication flow modeled, even though `capabilities.interventions` in the
  real ACP spec supports it — a real build almost certainly needs this, especially for the fpan
  path.
- No merchant-side catalog/inventory logic for the generic `line_items` path — that
  `checkout_sessions` branch is still the minimum needed to demonstrate the payment flow. The
  Duffel/flights path (below) is the one real exception.
- **AP2's fraud engine has the same placeholder caveat as ACP's** — `ap2RiskEngine.ts` demonstrates
  what changes about risk assessment for mandate-based, agent-initiated payments; it is not a
  model to ship.
- No mandate signature/trust-chain verification — `ap2Settle.ts` checks that a signature field is
  present, not that it's cryptographically valid or issued by a recognized credential provider.
  Deliberately deferred; see "Scope decisions" above.
- No UCP implementation yet — deliberately deferred; see "Scope decisions" above. UCP's
  discovery/cart layer would sit in front of the AP2 module the same way `checkout_sessions` sits
  in front of `delegate_payment` for ACP.
- No Credential Provider role modeled — company X here only ever settles mandates someone else
  issued. Modeling the issuing side is a different, bigger role — see "Scope decisions" above.
- The assumption that existing wallet support includes network-token validation infrastructure is **unconfirmed** and must be checked before it is used as a cost argument.
- **Duffel's `payments` shape is now confirmed live** (`amount` must match the offer total,
  fetched fresh via `getOffer` rather than trusted from an earlier response) — no longer a gap.
- **The `balance` payment method is still a placeholder, and the two payment steps are still not
  reconciled.** `createOrder` uses Duffel's `balance` payment type, which settles against
  Duffel's own account balance — it is not connected to the vault token collected upstream via
  company X's `delegate_payment` in the same request. Right now the user's payment (via company X)
  and the merchant's settlement (via Duffel) are two disconnected steps, not one coherent
  money-movement story. A real build needs to resolve how the collected payment actually
  funds the merchant-side booking — e.g. Duffel's other payment methods (per their docs,
  `arc_bsp_cash` requires registered IATA travel agent status) or a different settlement path
  entirely. This is a real, unresolved architecture question, not a cosmetic gap.
- One Duffel API call per demo run creates a real object on Duffel's sandbox servers (an offer
  request, potentially an order) — harmless in test mode, but worth knowing this isn't purely
  local like every other demo in the private source repo.
- **No automation-detection logic in the Gate** — `browserSignals.automationSuspected` is
  entirely the merchant's own input. This prototype doesn't attempt to actually distinguish a
  human from an agent driving a browser; it only models what happens to risk/liability once a
  merchant claims to know.
- **The `merchant` liability floor is a deliberate, not obviously correct, design stance.**
  Whether that's the right incentive structure (versus, say, some intermediate liability-sharing
  scheme to encourage connector adoption without stalling Gate-only adoption entirely) is a real
  product/commercial question this prototype surfaces but doesn't resolve.
