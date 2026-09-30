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

**KYA core v1** – the identity/authorization layer, and a real, deployable connector server around it. 187 passing tests. Mock identity provider; no real money yet.

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
- **Automate**: a customer can describe a rule in plain language ("if a better ISA comes up, move it"); it's translated into a precise, structured trigger they explicitly approve with a real passkey ceremony, then runs itself — matched against real rates every day, executed with no per-instance confirmation, and the customer told afterwards. See "Autonomy" below for why this is the safe half of a much bigger idea.
- **Stay informed**: `get_events`/`ack_event`/`register_agent_webhook` — every event is a real, independently-verifiable signed JWS, including automatic ones from account-opening and standing-instruction executions.
- **Manage**: a real customer-facing control room — sign in with the existing passkey (no agent involved at all), see every linked agent's scopes and limits, revoke one or change its limits, each its own real approval ceremony. The first screen in this project built for the customer directly rather than for an agent or for onboarding.

**Honest gaps, tracked, not hidden:** no browse-only scope tier yet (every session goes through full KYA); no real open-banking payments; no real identity provider or real KMS (both need signing up with an external service); live MCP session notifications and email/push escalation for events both need capabilities not built yet.

## Autonomy, decided deliberately — and now the safe half is real

A short design note worth surfacing on its own, since it's a real product/regulatory decision, not just an engineering one.

1. **Facts, never a recommendation** — an internal monitor watches holdings and the market and reports facts, never suggesting or acting. Designed, not yet built.
2. **Propose, confirm every time** — the discover→open→switch loop above: an agent proposes, nothing executes until the customer confirms.
3. **A standing instruction, stated once — now built.** The customer states a rule in their own words; it's translated into a precise, structured trigger they see and explicitly approve (a real WebAuthn ceremony over the *parsed* rule, catching a misunderstood phrasing before it's live, not after); the system then matches and executes it automatically, with zero discretion beyond that literal match, and tells the customer afterwards. Verified end to end: a real account, a real approved rule, a real automatic switch to a genuinely better rate in the catalogue — and confirmed it correctly does nothing when the gap is too small, when the mandate's been revoked, or on a repeat evaluation after it already fired.

Explicitly **not** built, and flagged rather than quietly attempted: an internal agent that decides, from an open-ended goal rather than a rule the customer wrote word-for-word, what to do and executes it. That crosses into discretionary-management/advice territory and needs a legal read before any design work starts. The standing-instruction work above is the concrete, working proof of where that line actually sits in code — "smart input, dumb execution" — not just a principle on paper: a natural-language model is genuinely used to *understand* the rule, and is given zero say in whether to *act* on it.

## Real market data: moving off the mock catalogue, live in the product

Started sourcing real UK savings rates, deliberately small and low-risk rather than scraping the whole market at once. Rejected aggregators (MoneySavingExpert, Moneyfacts, Raisin) outright as scrape *targets* — they compile other providers' rates into a curated database, carrying real legal exposure a primary source doesn't. MoneySavingExpert's best-buy table WAS used once, by hand, purely as a lookup for who currently ranks well, the way a human analyst would before going to primary sources.

**Eight providers real and live** (Chip, Atom Bank, Tandem Bank, Charter Savings Bank, Cynergy Bank, Oxbury Bank, NS&I, Skipton), each checked by hand before writing a line of scraper code. Five more candidates were checked and rejected outright — two actively blocked the first request, one's own `robots.txt` names and blocks Node's fetch stack directly, one was a parked domain, one never connected — all treated as hard stops, clearer signals than any terms-of-service text.

**Now genuinely wired into what agents see**, not a side pipe: a mapping layer turns a scraped rate into a real catalogue product, but only the fields actually verified (provider, name, rate, type) become real values — everything not independently confirmed (deposit limits, withdrawal terms) stays an honest, visibly-generic placeholder rather than an invented plausible number, flagged with its own `dataQuality` marker so nothing downstream mistakes a placeholder for fact. Runs automatically, coded into the server itself: once at startup, then every 24 hours, for as long as the process is alive — the interim measure until there are official provider partnerships, not a manual step anyone has to remember. Right now that's **16 real products alongside the 12 mock ones**, ranked and compared together.

## Run it (Node 20+)
```bash
cd ~/bank
npm install
npm test                 # 187 tests
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
