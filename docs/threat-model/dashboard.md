# Threat model — Dashboard (currently inside `zeroauth-dev/ZeroAuth: dashboard/`)

> Extends [`canonical.md`](canonical.md).
> **Status:** v1. The dashboard exists today, served by Express at `/dashboard/*`. This file holds the dashboard-specific mitigation detail referenced from the canonical.

## Surfaces

- React SPA served at `https://console.zeroauth.dev/*`
- Console API consumed at `/api/console/*` (authenticated via JWT in `Authorization: Bearer` header)
- 10 pages: Login, Signup, Overview, ApiKeys, Users, Devices, Verifications, Attendance, Audit, Settings

## Attacks (canonical mapping)

### A-01 — Cross-tenant data read (dashboard-specific)

The dashboard NEVER passes a `tenantId` to the API in request body or query. Tenant identity is derived server-side from the console JWT. Frontend code in `dashboard/src/lib/api.ts` enforces this — the typed fetch wrapper takes no `tenantId` parameter.

### A-05 — Credential stuffing on signup (dashboard-specific)

The dashboard signup form in `dashboard/src/routes/public/Signup.tsx` calls `/api/console/signup`. The frontend does NOT implement its own rate-limit (the server's authLimiter is authoritative). If a frontend retry loop hits the 429, the UI surfaces the rate-limit message.

### A-09 — Console JWT theft via XSS

**Storage:** the dashboard stores the console JWT in `localStorage` keyed `zeroauth.console.token`. Trade-off: localStorage is XSS-readable but survives page reloads (better UX than in-memory). Mitigated by:

1. Strict CSP — no inline scripts in the dashboard build
2. React default escape on all user-supplied strings
3. **Hard rule:** no `dangerouslySetInnerHTML` in dashboard code without an ADR
4. Short JWT lifetime (24h)
5. Per-tenant revocation: deleting a tenant invalidates all its console JWTs

**Component-specific gap:** no automated test that asserts the dashboard build output has zero inline `<script>` blocks beyond the Vite entrypoint. Add Vitest test or Playwright assertion.

### A-10 — Cross-tenant data read via console route

The dashboard's API client in `dashboard/src/lib/api.ts` calls `/api/console/{overview,audit,usage,keys,devices,users,verifications,attendance,settings}`. None of these accept a `tenantId` from the frontend. Reviewer-enforced (see [`api.md` A-10](api.md#a-10---dashboard-tenant-inference-from-request) for the server-side test gap).

## Dashboard-specific concerns

- **Auth state machine** — `dashboard/src/lib/auth.tsx` has three states: `loading`, `authenticated`, `unauthenticated`. The `RequireAuth` route guard prevents protected pages from rendering during `loading`.
- **Environment switcher** — `dashboard/src/lib/environment.tsx` lets the user toggle between `live` and `test`. The setting is purely UI; the server resolves the environment from the API key (for `/v1`) or from the console JWT's last-known environment (for `/api/console/*`). The switcher cannot be exploited to "leak" data across environments because the server ignores it.
- **Toast viewport for errors** — error messages from the API are surfaced via the toast component. We never display raw error messages from the server; the dashboard maps server error codes to user-friendly strings in `dashboard/src/lib/api.ts`.

## Future component-specific concerns (when dashboard split into its own repo)

- Subresource integrity (SRI) hashes for any third-party CDN assets (currently none — fonts loaded from Google Fonts; add SRI if we self-host)
- A separate dashboard repo would mean a separate `CLAUDE.md`, a separate threat model, and a `release-coordination/matrix.md` entry — same considerations as the verifier split-out

---

LAST_UPDATED: 2026-05-13
