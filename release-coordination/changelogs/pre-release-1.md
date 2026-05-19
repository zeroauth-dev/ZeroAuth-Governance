# Changelog — pre-release-1

**Date:** 2026-05-12
**Components:** API (`zeroauth-dev/ZeroAuth` @ `0d1741d`), Dashboard (bundled in API), Governance (`pre-release-1`)

## Summary

First production deploy. Central API live at `https://api.zeroauth.dev/v1/*`. Dashboard live at `https://console.zeroauth.dev/*`. TLS via Caddy + Let's Encrypt. Hosted on VPS at `104.207.143.14`.

## API

- All `/v1/*` endpoints scaffolded: auth (zkp/saml/oidc), identity, devices, users, verifications, attendance, audit
- Tenant API key authentication via `Authorization: Bearer za_<live|test>_<hex>` or `X-API-Key`
- Console JWT for `/api/console/*` (HS256, 24h)
- Admin x-api-key for `/api/admin/*`
- Audit log writes on every state change
- 82 tests passing (Jest 50 + Vitest 32 + Playwright 1)

## Dashboard

- 10 pages: Login, Signup, Overview, ApiKeys, Users, Devices, Verifications, Attendance, Audit, Settings
- Vite 7 + React 19 + Tailwind v4 + TanStack Query 5
- Bundled in the API repo at `dashboard/`, served by Express at `/dashboard/*`

## Governance

- First version of this repo (`zeroauth-dev/ZeroAuth-Governance`) published
- Shared docs: security-policy, coding-standards, naming-conventions, incident-response, breach-notification (DPDP §8(7))
- Canonical threat model with A-01 through A-10
- Compliance mappings: DPDP, IRDAI, RBI, MeitY
- ADR index pointing to the 4 ADRs in the API repo

## Known gaps (carried into pre-release-2)

- No issued-nonce binding (A-02 within-window replay possible)
- No zod input validation (B01 quality bar)
- No Prisma migrations — bootstrap schema only (B01 quality bar)
- No `Idempotency-Key` handling on state-changing endpoints (B01 quality bar)
- No automated `security-reviewer` invocation on PRs (B07 quality bar)
- Verifier inline in API (B02 splits it in Week 2)
- No off-host Postgres backup
- No multisig on `DIDRegistry`
- No hash chain on `audit_events`

## Cooperating PRs

- `zeroauth-dev/ZeroAuth` #22 (Central API console — Day 1 deliverable)
- `zeroauth-dev/ZeroAuth` #24 (Dashboard @types/node hotfix)
- `zeroauth-dev/ZeroAuth` #25 (Marketing-site refactor — operational Block B item)

---

LAST_UPDATED: 2026-05-13
