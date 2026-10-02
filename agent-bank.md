# Agent Bank

> **Portfolio summary. The source code and supporting documents are private.**

**Thesis:** Agents will act as the new browser. Google and the Amazon-style marketplaces have been gamed into uselessness by ads and SEO farming, and people are tired of them. Once agentic assistants spike - and they will, because they solve exactly that problem - you need a new world: from the app store to the connector, from the app to the API.

An app for agents. We host one connector that any agent can plug into (MCP + REST). Through it, a person's agent can
discover UK savings products with every condition, get the person's choice, open the account in the person's name
using our KYA, fund it by instructing a payment from the person's own bank, switch it to a better one later or set up
a standing rule that does it automatically, and keep them informed throughout. We never hold customer money.

First market: **UK**. First products: **savings** (investments later). First agents: **phone agents** (e.g. Muse, Instinct) [CHECK integration].

Investments are a real roadmap item, not a live one -- savings was chosen first specifically
because deposit-taking sidesteps investment-advice regulation, a shortcut investments don't get.
Not in the catalogue, no design work started, blocked on a proper regulatory read before anything
gets built.

Status labels in /docs:
- **BUILT** – exists in the private source repo with a passing test
- **PROPOSAL** – our design, not built or validated
- **NEEDS PROOF** – must be tested with users, providers, partners or counsel
- **[CHECK]** – a fact not yet verified

## What is BUILT (30 Sep 2026)

**KYA core v1** – the identity/authorization layer, and a real, deployable connector server around it. 295 passing tests. Mock identity provider by default; a real Didit sandbox integration exists but isn't switched on. No real money yet.

Three things, kept separate in code and in every audit entry (docs/11 "who holds what", docs/12 "assurance levels"):
1. **Customer identity** – a verified person: IDV + a real WebAuthn passkey; private key never leaves a simulated secure enclave. Every customer approval (mandate, revoke, limit change, a standing instruction) is a genuine CBOR-encoded, EdDSA-signed WebAuthn assertion challenged with a hash of exactly what's being approved, with signature-counter clone detection.
2. **Agent identity** – agent key + operator attestation (`signed`), or an OAuth client (`oauth_dpop`: sender-constrained; `oauth_bearer`: bearer credential, weakest). Real RFC 9449 DPoP-over-HTTP backs this at the transport layer, interop-checked against the real MCP SDK client's own proof format.
3. **Delegated authority** – the mandate: scopes, limits, expiry, revocation, checked live on every call regardless of which agent-identity mode issued it.

What this buys: real refresh-token rotation with reuse (theft) detection revoking the entire mandate, not just one token; a real hash-chained audit log, concurrency-safe under simultaneous writers; the issuer's private key never stored in the clear, bootstrapped into Supabase Vault when configured rather than a bare env var; real Supabase-backed storage (9 tables, row-level security, no public policies); and a real shared KV store for every piece of transport-layer state, so this isn't single-process.

## The connector — discover → choose → verify → open → switch → automate, real end to end

`npm run server` starts a real Hono server with a working, tested OAuth + MCP stack. The full customer journey works over real HTTP:

- **Discover, anonymously**: `list_products`/`get_product`/`compare_products`. Ranking is one published, fixed method (best AER first) that never reads whether a provider pays us. No identity needed at all for this stage.
- **Choose, on our own page — never the agent**: `present_choice` returns a link, not a way to confirm it. Confirmation only happens on a real hosted page, gated by a cookie + matching hidden field. Confirming mints a real signed receipt.
- **Verify**: the real OAuth + WebAuthn KYA flow above.
- **Open**: `open_account` spends that receipt, rejecting it outright if missing, tampered, stale-versioned, or already used.
- **Fund**: `set_funding` sets up how an account gets paid into — one-off or standing — on the same "agent prepares, customer authorizes on our own page" pattern. Real open banking isn't built yet, so only the actual bank connection is mocked; the consent state machine is real.
- **Switch**: `rollover` moves a holding to a different product through the same receipt-gated flow as opening one.
- **Withdraw**: `request_withdrawal` takes money out of an account entirely, once the product's own terms actually allow it (easy access, a matured fixed term, or an early-closure penalty the customer accepts) — a real WebAuthn approval, since money is leaving, not arriving.
- **Transfer an ISA, in either direction**: a real UK ISA transfer, not a withdraw-and-redeposit — money never leaves the ISA tax wrapper, and doesn't use fresh annual allowance. Transferring in captures the old provider's name and how much of it was subscribed this tax year (the only honest way the system learns about ISA money held elsewhere); transferring out moves an Agent Bank ISA to a named external provider. Both automatically issue a real, signed transfer-authority document — the actual paperwork, not a stub — and model the other institution's confirmation through a swappable interface, honestly mocked today since no real ISA transfer network exists to connect to yet.
- **Automate**: a customer can describe a rule in plain language ("if a better ISA comes up, move it"); it's translated into a precise, structured trigger they explicitly approve with a real passkey ceremony, then runs itself — matched against real rates every day, executed with no per-instance confirmation, and the customer told afterwards. See "Autonomy" below for why this is the safe half of a much bigger idea.
- **Discover without even a full link**: a `browse`-scoped mandate — zero limits, no customer attached — gets issued at sign-in with no identity check at all. The moment it tries anything that actually needs one, it gets back a real link to verify; completing that upgrades the exact same mandate in place, so the agent's original credential just starts working, no re-issue.
- **Stay informed**: `get_events`/`ack_event`/`register_agent_webhook` — every event is a real, independently-verifiable signed JWS, including automatic ones from account-opening, standing-instruction executions, and a corrected ISA allowance figure. A customer-set weekly cap on non-urgent ones (critical never capped), and each linked agent can filter which types/urgency levels it even wants to see.
- **Manage**: a real customer-facing control room — sign in with the existing passkey (no agent involved at all), see every linked agent's scopes and limits, revoke one or change its limits, and a live ISA-allowance widget for the tax year, each its own real approval ceremony where money moves. The first screen in this project built for the customer directly rather than for an agent or for onboarding.

**Honest gaps, tracked, not hidden:** no real open-banking payments, so a withdrawal or transfer's actual payout is mocked; no real ISA transfer network to send the (real, signed) transfer paperwork to; no real KMS (needs a cloud account); a real identity-provider integration (Didit) exists but isn't DVS-registered, so it stays sandbox/demo only until that's resolved; live MCP session notifications and email/push escalation for events both need capabilities not built yet.

**A security note worth including precisely because it's the boring, honest kind**: adding the lightweight "browse" tier above meant a mandate could now exist with no customer attached — a genuinely new shape, worth a dedicated re-read of the authorization code once it landed rather than trusting the tests alone. That re-read found one real bug (an edge case where an agent could end up stuck pointing at an old, weaker credential instead of a new fuller one — never a security exposure, since the stuck state was always the *more* restrictive one, but a real dead end), fixed in the same pass with a test that fails without the fix and passes with it. Treating "I built it and the tests pass" and "I went back and tried to break it" as two different steps, not one, is the posture this whole project has tried to hold to.

## Autonomy, decided deliberately — and now the safe half is real

A short design note worth surfacing on its own, since it's a real product/regulatory decision, not just an engineering one.

1. **Facts, never a recommendation** — an internal monitor watches holdings and the market and reports facts, never suggesting or acting. Designed, not yet built.
2. **Propose, confirm every time** — the discover→open→switch loop above: an agent proposes, nothing executes until the customer confirms.
3. **A standing instruction, stated once — now built.** The customer states a rule in their own words; it's translated into a precise, structured trigger they see and explicitly approve (a real WebAuthn ceremony over the *parsed* rule, catching a misunderstood phrasing before it's live, not after); the system then matches and executes it automatically, with zero discretion beyond that literal match, and tells the customer afterwards. Verified end to end: a real account, a real approved rule, a real automatic switch to a genuinely better rate in the catalogue — and confirmed it correctly does nothing when the gap is too small, when the mandate's been revoked, or on a repeat evaluation after it already fired.

Explicitly **not** built, and flagged rather than quietly attempted: an internal agent that decides, from an open-ended goal rather than a rule the customer wrote word-for-word, what to do and executes it. That crosses into discretionary-management/advice territory and needs a legal read before any design work starts. The standing-instruction work above is the concrete, working proof of where that line actually sits in code — "smart input, dumb execution" — not just a principle on paper: a natural-language model is genuinely used to *understand* the rule, and is given zero say in whether to *act* on it.

## ISA allowance and transfers — a real regulatory number, enforced, not just displayed

UK ISAs share one £20,000 annual allowance across every ISA type, confirmed live for the 2026/27
tax year. Building this honestly exposed a real gap: nothing in the system tracked how much money
actually went into an account — `open_account`/`rollover` never collected an amount at all. Fixing
that unlocked two real checks for free (a product's own deposit min/max, and the allowance itself),
and made the transfer flows above possible.

- **Enforced, not advisory**: a fresh ISA subscription that would exceed the remaining allowance is
  rejected outright, before an approval page even exists. An ISA-to-ISA transfer is correctly
  exempted — a real transfer doesn't use new allowance — while moving from a non-ISA product into
  an ISA correctly does.
- **Honest about what it can see**: the allowance figure only ever reflects subscriptions made
  through Agent Bank, plus whatever's been named on a transfer brought to it — it says so plainly
  every time, never implying visibility into ISAs held elsewhere that haven't been transferred in.
- **The paperwork is real, the network isn't.** Both transfer directions issue a genuinely signed
  document the moment the customer authorizes them — the same cryptographic mechanism the connector
  already uses for choice receipts, independently verifiable, not asserted. What doesn't exist yet
  is a live connection to any actual receiving or sending institution — modeled behind a clean
  interface so a real one drops in later without touching the logic around it, the same shape as
  every other "real when there's something real to call" piece in this codebase.

## Real market data: moving off the mock catalogue, live in the product

Started sourcing real UK savings rates, deliberately small and low-risk rather than scraping the whole market at once. Rejected aggregators (MoneySavingExpert, Moneyfacts, Raisin) outright as scrape *targets* — they compile other providers' rates into a curated database, carrying real legal exposure a primary source doesn't. MoneySavingExpert's best-buy table WAS used once, by hand, purely as a lookup for who currently ranks well, the way a human analyst would before going to primary sources.

**Eight providers real and live** (Chip, Atom Bank, Tandem Bank, Charter Savings Bank, Cynergy Bank, Oxbury Bank, NS&I, Skipton), each checked by hand before writing a line of scraper code. Five more candidates were checked and rejected outright — two actively blocked the first request, one's own `robots.txt` names and blocks Node's fetch stack directly, one was a parked domain, one never connected — all treated as hard stops, clearer signals than any terms-of-service text.

**Now genuinely wired into what agents see**, not a side pipe: a mapping layer turns a scraped rate into a real catalogue product, but only the fields actually verified (provider, name, rate, type) become real values — everything not independently confirmed (deposit limits, withdrawal terms) stays an honest, visibly-generic placeholder rather than an invented plausible number, flagged with its own `dataQuality` marker so nothing downstream mistakes a placeholder for fact. Runs automatically, coded into the server itself: once at startup, then every 24 hours, for as long as the process is alive — the interim measure until there are official provider partnerships, not a manual step anyone has to remember. Right now that's **16 real products alongside the 12 mock ones**, ranked and compared together.

## Run it (Node 20+)
```bash
cd ~/bank
npm install
npm test                 # 295 tests
npm run build && npm run start  # real production build (tsc, not a dev-time TS runner)
npm run demo              # scripted KYA walkthrough with attacks
npm run demo:connector    # discover -> choose -> verify -> open -> switch, over the real HTTP/MCP stack
npm run scrape:rates      # real, live UK savings rates from eight providers' own pages
npm run server             # http://127.0.0.1:8787
```
A `Dockerfile`/`fly.toml` exist for deployment (agent-bank issue #14) -- prep only, since actually
deploying means creating real, billable accounts only a human should do.

| Doc | What |
|---|---|
| docs/01-product.md | What we are building and why, journey, scope |
| docs/02-spec-v1.md | First release spec, users, failure criteria |
| docs/03-architecture.md | Components, data model |
| docs/04-test-plan.md | Security and validation tests |
| docs/05-risky-assumptions.md | Ranked |
| docs/06-plan.md | 6-week plan to pitch-grade evidence; weeks 1–2 in detail |
| docs/07-protocol-check.md | What ACP, AP2, UCP, Visa, Mastercard actually do |
| docs/08-agent-protocol.md | Agent ↔ connector tools, product record, push events |
| docs/09-uk-regulatory.md | Licence model and open legal questions, incl. investments' own future-scope section |
| docs/10-kyc-kya.md | UK KYC requirements, agent identity landscape, our KYA design |
| docs/11-kya-flow.md | KYA v1: who holds what, customer flow, agent linking, recovery |
| docs/12-connector-spec.md | Build spec: HTTP + MCP connector, OAuth mode + signed mode |
| docs/13-monitor.md | Internal monitor: facts (not recommendations) sent to the customer's agent |
| docs/14-autonomy-tiers.md | How much the system decides vs. executes -- facts, propose-and-confirm, standing instructions (built), and the open (not decided) question of goal-based autonomy |
| docs/15-real-rate-sourcing.md | Why providers' own pages, not aggregators -- the legal reasoning and what's scraped, mapped and wired in |
| docs/archive-v0/ | Earlier "mandate wallet" framing, superseded |
