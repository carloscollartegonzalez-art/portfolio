# Agent Bank

> **Portfolio summary. The source code and supporting documents are private.**

**Thesis:** Agents will act as the new browser. Google and the Amazon-style marketplaces have been gamed into uselessness by ads and SEO farming, and people are tired of them. Once agentic assistants spike - and they will, because they solve exactly that problem - you need a new world: from the app store to the connector, from the app to the API.

An app for agents. We host one connector that any agent can plug into (MCP + REST). Through it, a person's agent can
discover UK savings products with every condition, get the person's choice, open the account in the person's name
using our KYA, fund it by instructing a payment from the person's own bank, switch it to a better one later, and keep
them informed throughout. We never hold customer money.

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

## What is BUILT (29 Sep 2026)

**KYA core v1** – the identity/authorization layer, and now a real, deployable server around it. 135 passing tests. Mock identity provider; no real money yet.

Three things, kept separate in code and in every audit entry (docs/11 "who holds what", docs/12 "assurance levels"):
1. **Customer identity** – a verified person: IDV + a real WebAuthn passkey; private key never leaves a simulated secure enclave. Every customer approval (mandate, revoke, limit change) is a genuine CBOR-encoded, EdDSA-signed WebAuthn assertion challenged with a hash of exactly what's being approved, with signature-counter clone detection.
2. **Agent identity** – agent key + operator attestation (`signed`), or an OAuth client (`oauth_dpop`: sender-constrained, proves possession of its own registered key on every call; `oauth_bearer`: presents a bearer credential with no per-call proof at all, weakest). **Real RFC 9449 DPoP-over-HTTP** now backs this at the transport layer too — no pre-registration, a client embeds its public key fresh in every proof, bound at token issuance and verified per-request, interop-checked against the real MCP SDK client's own proof format.
3. **Delegated authority** – the mandate: scopes, limits, expiry, revocation, checked live on every call regardless of which agent-identity mode issued it.

What this buys:
- Operator (agent platform) registration; OAuth clients register with either a DPoP key or a one-time client secret. `oauth_bearer` gets lower default limits and a step-up on the first payment and any new destination.
- **Switching agents keeps KYA certification**: a customer linking a brand-new agent/operator re-proves who they are with their existing passkey — a real WebAuthn discoverable-credential assertion — instead of redoing ID-document + selfie KYC.
- Every call: identity check, scope, own-name destination, per-payment and monthly limits, and a required, non-empty `reason` on `pay` and `open_account`.
- Full audit trail: principal, operator/OAuth-client, agent thumbprint, mandate, scope, limits, assurance level, on every allowed *and* denied action.
- **Real refresh-token rotation + reuse (theft) detection.** OAuth access tokens are short-lived (15 min) with a 30-day rotating refresh token; presenting an already-rotated token is the theft signal — it revokes the entire mandate, not just that token.
- **Real, hash-chained, tamper-evident audit log**, now concurrency-safe under multiple simultaneous writers (a database-level constraint plus retry, not a lock).
- **The issuer's private key is never stored in the clear** (AES-256-GCM envelope encryption). The master key comes from an env var by default, or — when Supabase is configured — is bootstrapped and stored encrypted in **Supabase Vault**, one shared value across every server instance instead of copying a secret into each environment by hand.
- **Real Supabase-backed storage**: a live Postgres project, 9 tables, row-level security on with no public policies — the service-role key is the only key that can read or write anything, and it has never once been retrieved or handled by the agent doing this work, by design.
- **A real shared KV store** for every remaining piece of transport-layer state (OAuth codes, DPoP replay tracking, refresh tokens, choice sessions) — real atomicity from the database, not from being single-process. No new Redis/Upstash account needed; it reuses the same Supabase project.

## The connector (docs/12) — the discover → choose → verify → open → switch loop, real end to end

`npm run server` starts a real Hono server with a working, tested OAuth + MCP stack. The full customer journey now works over real HTTP, not just at the identity layer:

- **Discover, anonymously**: `list_products` / `get_product` / `compare_products` over a mock catalogue of 12 UK savings products spanning all five product types. Ranking is one published, fixed method (best AER first) that never reads whether a provider pays us — verified by flipping that flag mid-test and checking the order doesn't move. Live on a separate, unauthenticated `/mcp/public` endpoint, since browsing needs no identity at all.
- **Choose, on our own page — never the agent**: `present_choice` returns a link, not a way to confirm it. Confirmation only happens on a real hosted page, gated by a cookie + matching hidden field, so an agent holding only the link it was given can't complete the step itself. Confirming mints a real signed receipt, cryptographically distinct from an agent credential.
- **Verify**: the real OAuth + WebAuthn KYA flow described above.
- **Open**: `open_account` spends that receipt — rejecting it outright if it's missing, tampered, issued against a product version that's since changed, or already used once. A denial for an unrelated reason (e.g. a missing mandate scope) never burns a good, unused receipt — validated before consumed, the same ordering lesson learned from refresh-token rotation.
- **Switch**: `rollover` moves an existing holding to a different product through the identical receipt-gated flow — no separate trust model for "changing your mind" versus "opening the first one." The old holding is marked, not deleted, so its history stays visible.
- **Stay informed**: `get_events` / `ack_event` / `register_agent_webhook` — every event is a real, independently-verifiable signed JWS. Opening or switching an account emits a real one automatically. Webhook registration sends a real signed test event and stays unverified until that specific event is acknowledged, matching the intended "linking completes only after a test event is acked" behaviour. Delivery is fire-and-forget — a dead endpoint never affects anyone else's delivery.

**Honest gaps, tracked, not hidden:**
- No browse-only scope tier yet (every session goes through full KYA today) and no real open-banking payments.
- Live MCP session notifications and email/push escalation for an unacknowledged event both need capabilities this pass doesn't build (the second needs a real email/push provider — the same kind of external-account gap as the identity provider itself).
- **Standing instructions** — a customer pre-authorizing a specific rule in advance ("if a better ISA rate appears, just move it, tell me after") so the system executes without asking each time — are designed (see below) but not built.
- No real identity provider or real KMS yet — both need signing up with an external service, a decision that isn't purely technical.

## Autonomy, decided deliberately: how much the system executes without asking

A short design note worth surfacing on its own, since it's a real product/regulatory decision, not just an engineering one. Three tiers:

1. **Facts, never a recommendation** — an internal monitor watches holdings and the market and reports facts (rate changes, maturities, FSCS exposure), never suggesting or acting. Designed, not yet built.
2. **Propose, confirm every time** — what's built above: an agent (or the system itself) proposes a move, nothing executes until the customer confirms on our own page.
3. **A standing instruction, stated once** — the customer writes the exact rule themselves in advance; the system only ever matches that literal rule and executes it without asking again, then notifies. This is real consent, just given once instead of per-instance — not the system deciding anything.

Explicitly **not** built, and flagged rather than quietly attempted: an internal agent that decides, from an open-ended goal rather than a rule the customer wrote word-for-word, what to do and executes it. That crosses into discretionary-management/advice territory, a different regulatory category from everything else here, and needs a legal read before any design work starts — the same posture every `[CHECK counsel]` item in this project's docs already takes.

## Real market data: moving off the mock catalogue

Started sourcing real UK savings rates, deliberately small and low-risk rather than scraping the whole market at once. Rejected aggregators (MoneySavingExpert, Moneyfacts, Raisin) outright as scrape *targets* — they compile other providers' rates into a curated database, carrying real legal exposure (their own terms almost always prohibit it, and UK database right protects a compiled dataset separately from copyright) that a primary source doesn't. MoneySavingExpert's best-buy table WAS used once, by hand, purely as a lookup for who currently ranks well — the same way a human analyst would casually check before going to primary sources; nothing from it is stored or reproduced.

**Seven providers now real and live** (Chip, Atom Bank, Tandem Bank, Charter Savings Bank, Cynergy Bank, Oxbury Bank, NS&I) — each checked by hand, per provider, before writing a line of scraper code: `robots.txt`, website terms for an anti-scraping clause, and page structure. Five more candidates were checked and rejected: two actively blocked the very first request (a CAPTCHA wall; a Cloudflare block page), one's `robots.txt` explicitly names and blocks Node's own fetch stack, one turned out to be a parked domain rather than a real bank, one failed to connect at all. All treated as hard stops, not obstacles to route around — the clearest signal a site owner can give, clearer than any terms-of-service clause. One more candidate passed the technical checks but has a stricter website-reuse clause than the others; deferred rather than built against with the same confidence.

A one-command script pulls real, live rates from all seven right now. One scraper parses real `schema.org` structured data rather than marketing prose — data a site publishes specifically so automated systems can read it correctly, the strongest signal found. Deliberately **not yet wired into the catalogue or any agent tool** — what's scraped today is a much thinner record (provider, product name, rate, source, timestamp) than the full product schema the catalogue needs; mapping one into the other, and handling the couple of page shapes (multi-term rate ladders, balance-tiered rates) this first pass didn't, is the next piece of work.

## Run it (Node 20+)
```bash
cd ~/bank
npm install
npm test                # 135 tests: attacks, agent-switching, OAuth modes, step-up, key protection,
                         # DPoP-over-HTTP, refresh-token rotation, audit-log concurrency, shared KV store,
                         # Vault master-key bootstrap, product catalogue, choice/receipts, open_account,
                         # rollover/get_holdings, events/webhooks, real-provider scrapers
npm run demo             # scripted KYA walkthrough with attacks (auto-generates a local-dev master key)
npm run demo:connector   # scripted walkthrough of the full discover -> choose -> verify -> open -> switch
                         # loop, over the real HTTP/MCP stack, no shortcuts
npm run scrape:rates     # real, live UK savings rates from seven providers' own pages
npm run server           # http://127.0.0.1:8787
```

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
| docs/09-uk-regulatory.md | Licence model and open legal questions |
| docs/10-kyc-kya.md | UK KYC requirements, agent identity landscape, our KYA design |
| docs/11-kya-flow.md | KYA v1: who holds what, customer flow, agent linking, recovery |
| docs/12-connector-spec.md | Build spec: HTTP + MCP connector, OAuth mode + signed mode |
| docs/13-monitor.md | Internal monitor: facts (not recommendations) sent to the customer's agent |
| docs/14-autonomy-tiers.md | How much the system decides vs. executes -- facts, propose-and-confirm, standing instructions, and the open (not decided) question of goal-based autonomy |
| docs/15-real-rate-sourcing.md | Why providers' own pages, not aggregators -- the legal reasoning and what's scraped so far |
| docs/archive-v0/ | Earlier "mandate wallet" framing, superseded |
