# ZeroAuth Naming Conventions

> **Last reviewed by:** Pulkit Pareek on 2026-05-13
> **Status:** v1

## Service names

| Service | Repo name | Internal name | Hostname |
|---|---|---|---|
| Central API | `zeroauth-dev/ZeroAuth` | `zeroauth-api` | `api.zeroauth.dev` (planned; today: `api.zeroauth.dev/v1/*`) |
| Verifier | `zeroauth-dev/ZeroAuth-Verifier` (planned) | `zeroauth-verifier` | internal-only, behind API |
| IoT firmware | `zeroauth-dev/ZeroAuth-IoT` (planned) | `zeroauth-iot` | (runs on Orange Pi devices) |
| Mobile SDK | `zeroauth-dev/ZeroAuth-Mobile-SDK` (planned) | `zeroauth-sdk` | n/a (library) |
| Dashboard | currently inside `zeroauth-dev/ZeroAuth: dashboard/` | `zeroauth-dashboard` | `console.zeroauth.dev` |
| Docs site | inside `zeroauth-dev/ZeroAuth: website/` | `zeroauth-docs` | `docs.zeroauth.dev` |
| Governance | `zeroauth-dev/ZeroAuth-Governance` *(this repo)* | `zeroauth-governance` | n/a (docs only) |

## Environment variables

Pattern: `ZEROAUTH_<COMPONENT>_<PURPOSE>` (SCREAMING_SNAKE_CASE).

| Variable | Where set | Example value |
|---|---|---|
| `ZEROAUTH_DATABASE_URL` | API repo `.env` | `postgres://zeroauth:***@db:5432/zeroauth` |
| `ZEROAUTH_REDIS_URL` | API repo `.env` | `redis://redis:6379` |
| `ZEROAUTH_JWT_SECRET` | API repo `.env` | (rotated quarterly) |
| `ZEROAUTH_ADMIN_KEY` | API repo `.env` | (rotated on team change) |
| `ZEROAUTH_BLOCKCHAIN_RPC_URL` | API repo `.env` | `https://sepolia.base.org` |
| `ZEROAUTH_DID_REGISTRY_ADDRESS` | API repo `.env` | from `contracts/deployed-addresses.json` |
| `ZEROAUTH_VERIFIER_ADDRESS` | API repo `.env` | from `contracts/deployed-addresses.json` |
| `ZEROAUTH_API_BASE_URL` | SDK `.env` | `https://api.zeroauth.dev` |
| `ENABLE_DEMO_AUTH` | API repo `.env` | `false` in production; `true` for demo gating |
| `NODE_ENV` | universal | `production` / `development` / `test` |

Reserved prefixes (don't reuse):

- `ZEROAUTH_INTERNAL_*` — internal-service-only, never read by user code
- `ZEROAUTH_DEMO_*` — gated by `ENABLE_DEMO_AUTH`
- `ZEROAUTH_PILOT_<BUYER>_*` — buyer-specific overrides during pilot phase

## API key prefixes

- `za_live_<48 hex>` — production tenant key
- `za_test_<48 hex>` — test environment tenant key
- `zh_<32 hex>` — console JWT signing key (HS256, internal only) — *planned*
- `zr_<base64url>` — refresh token — *planned, RS256 era*

## HTTP header conventions

| Header | Purpose | Source |
|---|---|---|
| `Authorization: Bearer za_<...>` | Tenant API auth | client |
| `X-API-Key: za_<...>` | Alternate tenant API auth | client |
| `X-API-Key: <admin key>` | Admin auth on `/api/admin/*` | client (internal) |
| `Idempotency-Key: <uuid v4>` | Replay-safe writes | client *(planned, Week 1 Day 5)* |
| `X-Request-ID: <uuid v4>` | Request correlation in logs | server-generated, echoed |
| `X-Tenant-Environment: live\|test` | Echo of resolved env (debugging) | server |
| `X-RateLimit-Limit` / `X-RateLimit-Remaining` / `X-RateLimit-Reset` | Rate limit info | server |

## Log fields

Every log line includes:

- `time` — ISO 8601 UTC
- `level` — `debug \| info \| warn \| error \| fatal`
- `service` — one of the internal names from §1 above
- `version` — semver of the running build
- `request_id` — from `X-Request-ID`
- `tenant_id` — UUID if request was tenant-scoped, else absent
- `environment` — `live | test` if tenant-scoped, else absent
- `actor_type` — `tenant_api_key | console_jwt | admin | system`
- `actor_id` — depends on `actor_type`
- `event` — short snake_case verb-phrase: `verification_succeeded`, `device_registered`, `api_key_rotated`, etc.

## Database column conventions

- All tables have: `id` (UUID v4 PK), `tenant_id` (UUID, FK to `tenants` when applicable), `environment` (`'live' | 'test'`), `created_at` (timestamptz UTC), `updated_at` (timestamptz UTC).
- Soft deletes: `deleted_at` (nullable timestamptz). Hard deletes only via admin tooling + audit row.
- Foreign keys: `<entity>_id` (e.g. `device_id`, `user_id`).

## Test naming

- Unit test files: `<unit>.test.ts` next to source.
- Integration tests: `tests/<surface>-<aspect>.test.ts` (e.g. `tests/central-api.test.ts`).
- E2E specs: `e2e/<scenario>.spec.ts` (Playwright).

---

LAST_UPDATED: 2026-05-13
NEXT_REVIEW: 2026-08-13 (quarterly)
