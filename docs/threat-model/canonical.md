# ZeroAuth Canonical Threat Model

> **Last reviewed by:** Pulkit Pareek on 2026-05-13
> **Status:** v1 — synced from `zeroauth-dev/ZeroAuth: docs/threat_model.md` v0 (2026-05-12). Component-specific extensions live in sibling files (`api.md`, `verifier.md`, `iot.md`, `sdk.md`, `dashboard.md`).

This is the **cross-repo source of truth** for ZeroAuth's threat model. Every product repo's component-scoped threat model extends this file. When the canonical and a component file disagree, the canonical wins; the component file updates.

## Threat-surface inventory

| Surface | Component | Exposure | Notes |
|---|---|---|---|
| `https://api.zeroauth.dev/v1/*` | API | Public, tenant-API-key authed | Scoped to `(tenant_id, environment)`. Rate-limit + monthly quota per tenant. |
| `https://zeroauth.dev/api/console/*` | API (console) | Public, JWT-authed for everything except signup + login | Per-IP rate limit on signup/login. Password policy enforced. |
| `https://zeroauth.dev/api/admin/*` | API (admin) | Public, `x-api-key` (single shared admin key) | Read-only today. |
| `https://zeroauth.dev/api/health` | API | Public, unauth | Health + subsystem status only. |
| `https://zeroauth.dev/api/auth/{saml,oidc}/*` | API (demo) | Public, gated by `ENABLE_DEMO_AUTH` flag | Demo stubs; **do not** validate signatures. Off in production. |
| `https://zeroauth.dev/api/leads/*` | API (marketing) | Public, unauth | Writes to `leads` table. |
| Base Sepolia `DIDRegistry` | Contracts | Public RPC, `onlyOwner` writes | Deployer wallet = single owner. Rotation procedure documented. |
| VPS SSH `104.207.143.14:22` | Infra | Internet, key-only | `root` + `zeroauth-deploy` authorized. UFW open on 22 / 80 / 443. |
| Verifier service (planned) | Verifier | Internal (loopback) | Currently inline in API; Week 2 B02 splits it. |
| IoT terminal HTTPS calls (planned) | IoT firmware | Outbound to API only | TLS 1.3, pinned cert. Hardware: Orange Pi 5 / MFS100 / Astra Pro Plus. |
| Mobile SDK calls (planned) | Mobile SDK | Outbound to API only | TLS 1.3. Bundled into customer apps. |

## Identified attacks (A-NN)

For each attack: `Class` is STRIDE classification. `Component(s)` names which component file in this folder holds the component-specific mitigation detail. `Residual` is the post-mitigation residual risk.

### A-01 — Cross-tenant data read

| | |
|---|---|
| Class | Elevation of privilege (E) |
| Component(s) | [api.md](api.md), [verifier.md](verifier.md) (once split), [dashboard.md](dashboard.md) |
| Description | A request authenticated as tenant A receives data belonging to tenant B because a SQL WHERE clause omits the tenant filter. |
| Mitigation summary | Every SQL query takes `(tenant_id, environment)` parameters; enforced in middleware (`src/middleware/tenant-auth.ts`), not in handlers. Router-level integration test exists. |
| Residual risk | Medium — no direct SQL-path test yet. SQL injection in a future raw-SQL handler could bypass the middleware. |

### A-02 — Replayed proof verification

| | |
|---|---|
| Class | Spoofing (S) |
| Component(s) | [api.md](api.md), [verifier.md](verifier.md), [iot.md](iot.md), [sdk.md](sdk.md) |
| Description | An attacker replays a captured Groth16 proof + public signals + nonce after the original session has ended. |
| Mitigation summary | 5-minute timestamp window; nonce format validation. **Gap:** nonce is not bound to an issued-nonce table — replay within the 5-minute window is possible. |
| Residual risk | High until issued-nonce binding lands. Tracked as an open issue. |

### A-03 — Forged SAML assertion via demo callback

| | |
|---|---|
| Class | Spoofing (S) |
| Component(s) | [api.md](api.md) |
| Description | Route mints a session JWT from `nameID` and `email` in request body without validating any SAML signature. |
| Mitigation summary | `demo-auth-gate` middleware returns 503 unless `ENABLE_DEMO_AUTH=true`. Flag is off in production. |
| Residual risk | Low in production; route is dead. **Must not** be re-enabled without real `@node-saml/node-saml` implementation. |

### A-04 — Forged OIDC callback via demo route

| | |
|---|---|
| Class | Spoofing (S) |
| Component(s) | [api.md](api.md) |
| Description | PKCE state lookup is real but the user identity is taken from `req.body.email` without exchanging the code at the IdP. |
| Mitigation summary | Same `demo-auth-gate`. |
| Residual risk | Low in production; same caveat as A-03. |

### A-05 — Credential stuffing / email enumeration on console signup

| | |
|---|---|
| Class | Information disclosure (I) + DoS (D) |
| Component(s) | [api.md](api.md), [dashboard.md](dashboard.md) |
| Description | Without per-IP rate limit, an attacker can probe email addresses (signup) or test password lists (login). 409 vs 201 on signup reveals whether an email is taken. |
| Mitigation summary | `src/routes/console.ts:authLimiter` — 10 attempts per 15 min per IP. Password policy enforced (12 chars, letter+digit, denylist). |
| Residual risk | Medium — no test that 11th attempt returns 429. Enumeration via timing or 409 still possible. |

### A-06 — Replay of revoked API key

| | |
|---|---|
| Class | Spoofing (S) |
| Component(s) | [api.md](api.md) |
| Description | A revoked key is replayed. In-memory rate-limit counters are keyed by tenant ID; if another active key exists, requests share the limiter. |
| Mitigation summary | `authenticateApiKey` re-reads DB on every request; rejects `status != 'active'`. The key itself is rejected. |
| Residual risk | Low — counter sharing isn't a security issue since the request never authenticates. |

### A-07 — Leaked deployer wallet compromises `DIDRegistry`

| | |
|---|---|
| Class | Elevation of privilege (E) |
| Component(s) | Contracts (in `zeroauth-dev/ZeroAuth: contracts/`) |
| Description | Wallet that deployed `DIDRegistry` is contract `owner`; key leak = full registry control. |
| Mitigation summary | Key in `/opt/zeroauth/.env` only. Rotated once post-review. `npm run wallet:rotate` available. **Long-term:** multisig owner. |
| Residual risk | Medium — single-key wallet is acceptable while patent value > stolen identity value, but only until multisig lands. |

### A-08 — Inline event handler bypasses strict CSP

| | |
|---|---|
| Class | Information disclosure / XSS (I) |
| Component(s) | API (marketing surface in `public/`) |
| Description | Helmet sets `script-src-attr 'none'`; the May 2026 review found two `onsubmit=` attributes failing silently in browsers. |
| Mitigation summary | All inline handlers removed; forms use `addEventListener`. CSP enforced. |
| Residual risk | Low — no inline handlers remain in `public/index.html`. CI `curl … grep` confirms. Should be promoted to a real test. |

### A-09 — Console JWT theft via XSS in dashboard

| | |
|---|---|
| Class | Information disclosure + EoP (I + E) |
| Component(s) | [dashboard.md](dashboard.md) |
| Description | Console JWT lives in client memory; XSS reads it and uses it from anywhere. |
| Mitigation summary | Strict CSP from Helmet. React default escape. No `dangerouslySetInnerHTML` without ADR. 24h JWT. |
| Residual risk | Medium — no integration test asserts the dashboard build has no inline scripts; no `auth.token_reuse` audit event on jti replay from new IP. |

### A-10 — Dashboard requests leaking another tenant's data

| | |
|---|---|
| Class | Elevation of privilege (E) |
| Component(s) | [dashboard.md](dashboard.md), [api.md](api.md) |
| Description | If a console handler infers tenant from request body/query rather than JWT subject, a valid JWT can read other tenants' data. |
| Mitigation summary | Every console route reads `tenantId` from `(req as any).console.tenantId` set by `verifyConsoleToken`. Reviewer-enforced; no automated test. |
| Residual risk | Medium — needs automated cross-tenant probe test that constructs a JWT for tenant A and tries body/query overrides. |

## Verifier-component attacks (A-V-NN)

Promoted 2026-05-15 alongside the [`verifier.md`](verifier.md) component file, which holds the per-attack mitigation detail. The verifier shipped to production today as a separate Docker container (`zeroauth-verifier`) per [ADR-0006](https://github.com/zeroauth-dev/ZeroAuth/blob/main/adr/0006-verifier-typescript-not-rust.md). See [verifier.md](verifier.md) for full STRIDE classification, mitigation depth, test status, and residual risk.

| ID | Title | Class | Residual |
|---|---|---|---|
| A-V01 | Audit-log tamper via direct SQLite write | R, T | Medium-low — detectable via `/audit/verify-chain` but not preventable against VPS-root attacker. Cross-chain anchoring is v3. |
| A-V02 | Verification key swap on disk between deploys | S, T | Medium — supply-chain trust in GitHub Actions + committed vkey. Hardens with vkey signature at trusted-setup time. |
| A-V03 | Side-channel attacks on snarkjs (non-constant-time field ops) | I | Medium — accepted trade-off per ADR-0006. Revisit on multi-region or move to arkworks. |
| A-V04 | Resource exhaustion via crafted proof inputs | D | Low — body limit + shape validation + upstream rate limit. |
| A-V05 | Cross-tenant verification via spoofed tenantId in `/verify` request | S, E | Low — trust boundary is Docker network. HMAC headers on API→verifier requests is the roadmap mitigation. |

## Open items (no `A-NN` yet)

- In-memory session store; restart wipes session continuity. Not exploitable today (JWTs are stateless).
- No off-host Postgres backups. VPS disk failure = data loss.
- Audit log is append-only at the table level (no triggers blocking UPDATE / DELETE). Postgres root compromise could rewrite history. **Long-term:** hash chain + cross-chain anchoring per patent.
- No CSP report-uri. Successful CSP blocks go silent.
- Docusaurus build embeds patent number in public docs site (intentional). Verify no pricing or buyer names leak into static assets.

## How to extend

1. New endpoint / endpoint change → identify which existing `A-NN` is in scope. If none, add a new entry.
2. New dependency handling secrets, PII, or network ingress → add an `A-NN` as part of the dep's ADR (`dep-add` skill).
3. New mitigation → update the affected `A-NN`'s Mitigation summary AND the relevant component file in this folder.
4. The `test-from-threat-model` skill (to be installed) generates test scaffolds; each test maps to one `A-NN`.

## Sync with product repos

- `zeroauth-dev/ZeroAuth: docs/threat_model.md` — currently the most-current copy; this canonical was synced from it on 2026-05-13. Going forward, the canonical here is authoritative and the product repo links to it.

---

LAST_UPDATED: 2026-05-13
NEXT_REVIEW: every Friday review (W05) checks for drift between canonical and component extensions
