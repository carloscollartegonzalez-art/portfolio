# Agent Bank

> **Portfolio summary. The source code and supporting documents are private.**

**Thesis:** Agents will act as the new browser. Google and the Amazon-style marketplaces have been gamed into uselessness by ads and SEO farming, and people are tired of them. Once agentic assistants spike - and they will, because they solve exactly that problem - you need a new world: from the app store to the connector, from the app to the API.

An app for agents. We host one connector that any agent can plug into (MCP + REST). Through it, a person's agent can
discover UK savings products with every condition, get the person's choice, open the account in the person's name
using our KYA, fund it by instructing a payment from the person's own bank, and keep them informed afterwards.
We never hold customer money.

First market: **UK**. First products: **savings** (investments later). First agents: **phone agents** (e.g. Muse, Instinct) [CHECK integration].

Status labels in /docs:
- **BUILT** – exists in the private source repo with a passing test
- **PROPOSAL** – our design, not built or validated
- **NEEDS PROOF** – must be tested with users, providers, partners or counsel
- **[CHECK]** – a fact not yet verified

## What is BUILT (28 Sep 2026)
**KYA core v0.1** – 46 passing tests. Mock identity provider; no real money yet. **Now has a real network server** (see below) -- no longer CLI-only.

Three things, kept separate in code and in every audit entry (docs/11 "who holds what", docs/12 "assurance levels"):
1. **Customer identity** – a verified person: IDV + a real WebAuthn passkey (`src/webauthn/rp.ts`; private key never leaves a simulated secure enclave, `src/webauthn/mockAuthenticator.ts`, the same way `idv/mock.ts` stands in for a real identity provider). Every customer approval (mandate, revoke, limit change) is a genuine CBOR-encoded, EdDSA-signed WebAuthn assertion challenged with a hash of exactly what's being approved, with signature-counter clone detection.
2. **Agent identity** – agent key + operator attestation (`signed`), or an OAuth client (`oauth_dpop`: sender-constrained, proves possession of its own registered key on every call, same assurance as `signed`; `oauth_bearer`: presents a bearer credential with no per-call proof at all, weakest). Every session/mandate is tagged with its assurance level.
3. **Delegated authority** – the mandate: scopes, limits, expiry, revocation, checked live on every call regardless of which agent-identity mode issued it.

What this buys:
- Operator (agent platform) registration; agent keys must be attested by their operator. OAuth clients (`registerOAuthClient`) register directly with either a DPoP key or get a one-time client secret.
- `oauth_bearer` gets lower default limits (£100/payment, £250/month vs. £500/£500 for `signed`/`oauth_dpop`) and a step-up (`needsCustomerConfirmation: true`, payment not yet recorded) on the first payment and any new destination — the two moments a stolen bearer credential would most want to abuse.
- **Switching agents keeps KYA certification**: a customer linking a brand-new agent/operator (one that has never seen them before) re-proves who they are with their existing passkey -- a real WebAuthn discoverable-credential assertion, verified against whichever principal actually registered that credential -- instead of redoing ID-document + selfie KYC. An earlier version of this trusted a caller-supplied principal id with no proof of possession; that's gone.
- Agent credential: 24h EdDSA JWT bound to the agent/client identity; refresh while mandate active.
- Every call: identity check (signature for `signed`/`oauth_dpop`, credential alone for `oauth_bearer`), scope, own-name destination, per-payment and monthly limits, and a **required, non-empty `reason`** on `pay` and `apply` (this codebase's `open_account` -- applying for a savings product is opening it).
- Every allowed/denied action logs `principalId`, `operatorId`/`oauthClientId`, agent thumbprint, `mandateId`, the `scope` and `limits` in effect, and `assuranceLevel` -- see `authorize()`'s `identifyCaller()` split in `src/kya/service.ts`.
- Customer: change limits, revoke; operator/OAuth-client suspension revokes all its agents; agent can revoke itself, never widen.
- Hash-chained audit log; Supabase schema in `supabase/migrations/0001_kya.sql`.
- **The issuer's private key -- the one that signs every agent credential -- is never stored in the clear.** AES-256-GCM envelope encryption (`src/kya/keyProtection.ts`); the master key comes from `KYA_MASTER_KEY` (env), never from the same file/database as the ciphertext. `npm run keygen` generates a real one. The CLI auto-generates a local-dev-only key into a separate gitignored file (`.data/master.key`, never `db.json`) if none is set -- a real deployment must use a real KMS/secrets manager instead (see the file's own comment for why an env var is only the interim step).

## The connector server (docs/12), first increment

`npm run server` starts a real Hono server (`src/server/`) with a working, tested, end-to-end
OAuth + MCP stack for **oauth_bearer** clients (the realistic default for "any MCP client that
just showed up" -- dynamically-registered clients get bearer credentials, matching docs/12's own
table). `oauth_dpop`/`signed` stay reachable via the CLI/SDK-integration path for now, not this
HTTP flow -- see the honest gap below.

- **Real RFC 9728/8414 discovery**: an unauthenticated `/mcp` call gets a `401` with a correct
  `WWW-Authenticate` challenge pointing at Protected Resource Metadata, which points at
  Authorization Server Metadata (`GET /.well-known/oauth-protected-resource/mcp`,
  `GET /.well-known/oauth-authorization-server`) -- both served by the SDK's own spec-correct
  helper (`oauthMetadataResponse`), not hand-rolled.
- **Real RFC 7591 dynamic client registration** (`POST /oauth/register`) -- any MCP client (Claude,
  ChatGPT) can register itself and get a `client_id`/`client_secret`, mapped straight onto
  `registerOAuthClient`.
- **Real OAuth 2.1 authorization_code + PKCE flow** (`GET /oauth/authorize`, `POST /oauth/token`).
  `/oauth/authorize` *is* the KYA onboarding flow itself (docs/12's own framing) -- a real,
  server-rendered page doing real `@simplewebauthn/browser` WebAuthn ceremonies (returning-customer
  passkey sign-in, or identity-check + new passkey, then a consent screen showing the actual
  proposed scopes/limits), not a stand-in. The `code_verifier`/`code_challenge` (S256), single-use
  authorization codes, and `client_secret_post` client auth are all real and tested, including the
  negative cases (wrong secret, wrong PKCE verifier, code replay).
- **A real MCP Resource Server at `/mcp`** (`@modelcontextprotocol/server`'s `createMcpHandler` +
  `requireBearerAuth`), with three tools that exercise the full safety chain end to end:
  `get_kya`, `pay` (reason required, oauth_bearer step-up, own-name + limit checks all live),
  `revoke_access`. Verified over the real MCP JSON-RPC protocol (`initialize` + `tools/call`), not
  just at the `KyaService` layer.
- `test/server.test.ts` drives the entire thing through Hono's own request/response contract --
  register a client, complete the real WebAuthn ceremonies (via the same `CustomerDevice` used
  everywhere else in this suite), exchange the code for a token, call an MCP tool -- no shortcuts
  taken to make the test pass.

**Honest gaps in this first server increment** (not silently skipped, tracked):
- **No real RFC 9449 DPoP header parsing at the HTTP layer yet.** `oauth_dpop`'s per-call
  proof-of-possession is fully real and tested at the `KyaService.authorize()` level (reused
  directly from `signed` mode's own mechanism) but isn't wired into the MCP bearer-auth gate over
  HTTP yet -- a real DPoP-sending client can't yet get sender-constrained assurance through this
  server. `oauth_dpop` remains usable via the CLI/SDK path (`AgentClient.oauthDpop`).
  `verifyCredential`'s own comment in `src/kya/service.ts` names this explicitly.
- No product catalogue, choice/receipt system, or events -- docs/12's full v0.1 tool list
  (`list_products`, `open_account`, `get_events`, etc.) needs a data model this pass doesn't build.
  `get_kya`/`pay`/`revoke_access` prove the auth + safety chain, not the product surface.
- Still `FileStore` (local JSON), still an in-process auth-code/nonce store -- both fine for one
  process, not yet for more than one (see below).
- Still no real identity provider, still no real payments.

Not built: real identity provider, Supabase store, product catalogue, payments. **Also not solved
yet:** the replay-nonce store, and now also the OAuth authorization-code store, are single-process
(in-memory/file) -- safe today because there's only one process, but both would need a shared
store (Redis-class) before this could run as more than one instance.

## Run it (Node 20+)
```bash
cd ~/bank
npm install
npm test            # 43 tests: attacks, agent-switching, OAuth modes, step-up, acceptance test 7a, key protection
npm run demo        # scripted walkthrough with attacks (auto-generates a local-dev master key)
npm run keygen      # generate a real KYA_MASTER_KEY for an actual deployment (prints, writes nothing)
npm run kya         # list step-by-step commands (state kept in .data/)
npm run kya -- reset
npm run kya -- operator:add Muse
npm run kya -- agent:link Muse
npm run kya -- customer:verify "Jane Doe" 1990-04-02 "1 High St, London E1 1AA"
npm run kya -- customer:approve --monthly 500
npm run kya -- agent:pay 300 "Jane Doe" --reason "customer asked to move savings"
npm run kya -- agent:pay 100 "Mallory" --reason "test"      # denied: not the customer's account
npm run kya -- customer:revoke
npm run kya -- audit

# OAuth mode (docs/12), plain bearer -- lower limits + step-up:
npm run kya -- reset
npm run kya -- oauth:add-secret ChatGPTConnector
npm run kya -- agent:link-oauth-bearer ChatGPTConnector
npm run kya -- customer:verify "Jane Doe" 1990-04-02 "1 High St, London E1 1AA"
npm run kya -- customer:approve            # £100/£250 limits, not £500/£500
npm run kya -- agent:pay 50 "Jane Doe" --reason "first purchase"   # allowed, but customer must still confirm
```

## Run the server
```bash
npm run server      # http://127.0.0.1:8787 -- auto-generates a local-dev master key, same as the CLI
curl http://127.0.0.1:8787/health
curl http://127.0.0.1:8787/.well-known/oauth-authorization-server
curl -i -X POST http://127.0.0.1:8787/mcp    # 401 + WWW-Authenticate, discoverable from there
```
For the actual authorize flow you need a real MCP client (or `test/server.test.ts`, which drives
it end to end); `BASE_URL` and `PORT` are configurable env vars, per docs/12.



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
| docs/13-monitor.md | Monitoring concept: facts (not recommendations) sent to the customer's agent |
| docs/archive-v0/ | Earlier "mandate wallet" framing, superseded |

