# CycleScope Authentication Gap Analysis Summary

> **Quick-reference summary** of all 16 cross-project authentication discrepancies, their resolutions, and per-project impact.
>
> For full details and rationale, see [CROSS_PROJECT_DISCREPANCIES.md](./CROSS_PROJECT_DISCREPANCIES.md).
> For the authoritative spec, see [UNIFIED_AUTH_STRATEGY.md](./UNIFIED_AUTH_STRATEGY.md).
>
> Last updated: 2026-02-20

---

## Overview

Three CycleScope projects were analyzed for authentication alignment:

| Project | Document |
|---------|----------|
| cyclescope-member-portal | `docs/AUTHENTICATION_AND_API_ARCHITECTURE.md` |
| SwingTrade | `docs/AUTH_STRATEGY.md` |
| OptionStrategy | `docs/SUB_PORTAL_AUTH_INTEGRATION.md` |

**16 discrepancies** were found — **3 critical, 5 high, 5 moderate, 3 low**. Additionally, **6 gaps (A–F)** were identified by comparing the golden docs against the actual codebase.

> **Important**: Most premium service features (handoff tokens, launch endpoints, tier checks) are **not yet implemented**. The migration effort is largely **greenfield work**, not refactoring.
>
> **Auth model**: Portal authentication is email/password only. Patreon OAuth code exists but is **disabled** via feature flag (`ENABLE_PATREON_OAUTH` not set). Patreon is a subscription data source only (daily sync + webhooks).

---

## Discrepancies by Severity

### CRITICAL (3)

| # | Issue | Current State | Resolution |
|---|-------|---------------|------------|
| 1 | **Tier Values Mismatch** | Portal: `free`, `basic`, `premium` / SwingTrade: `free`, `stocks`, `stocks_and_options` / OptionStrategy: undefined | Adopt `basic`, `stocks_and_options` (no free tier; both sub-portals are premium services — tier access is a business decision; currently all tiers get access) |
| 2 | **Token Secret Strategy** | Portal: single shared `PREMIUM_TOKEN_SECRET` / SwingTrade: per-service secrets / OptionStrategy: single shared | Per-service secrets — `SWINGTRADE_TOKEN_SECRET`, `OPTION_STRATEGY_TOKEN_SECRET` |
| 6 | **Handoff Token JWT Claims** | Portal: `userId` in body, includes `service` / SwingTrade: `sub` claim, includes `patreonId` / OptionStrategy: `userId` in body | Standardize: `sub` (userId), `email`, `tier`, `service`; drop `patreonId`. Portal *session* JWT keeps `userId` for backward compat (handoff/sub-portal tokens are `sub`-only) |

### HIGH (5)

| # | Issue | Current State | Resolution |
|---|-------|---------------|------------|
| 3 | **Portal Env Var Names** | Portal: `PREMIUM_TOKEN_SECRET`, `PREMIUM_STOCKS_URL`, `PREMIUM_OPTIONS_URL` / SwingTrade: `SWINGTRADE_TOKEN_SECRET`, `SWINGTRADE_URL` | Service-specific names: `SWINGTRADE_TOKEN_SECRET`, `OPTION_STRATEGY_TOKEN_SECRET`, `SWINGTRADE_URL`, `OPTION_STRATEGY_URL` |
| 4 | **Cookie Names** | Portal: `stocks_session` / SwingTrade: `session` / OptionStrategy: `swingtrade_session` | Sub-portals: `swingtrade_session`, `option_strategy_session`. Portal: keep `app_session_id` (no rename) |
| 7 | **Local Session Token Claims** | Portal: `userId`, `email` (no tier) / SwingTrade: `sub`, `email`, `tier` / OptionStrategy: `userId`, `email`, `tier` | Standardize: `sub`, `email`, `tier` |
| 13 | **Tier Mapping Function** | Portal maps from Patreon tier *names* returning `basic`/`premium` | Map from Patreon tier *names* (portal team approach accepted) via `mapPatreonTierNameToAccess()`, returning `basic`/`stocks_and_options` |
| 15 | **Tier Check on `/auth`** | SwingTrade checks tier; Portal example and OptionStrategy do not | All sub-portals MUST check tier (defense in depth) |

### MODERATE (5)

| # | Issue | Current State | Resolution |
|---|-------|---------------|------------|
| 5 | **Portal Launch Endpoint** | Portal: `GET /api/premium/access-token?service=xxx` / SwingTrade: `POST /api/launch/swingtrade` | Per-service `POST` endpoints |
| 9 | **CORS Origin Variable** | SwingTrade: `PORTAL_ORIGIN` / OptionStrategy: `MEMBER_PORTAL_URL` | Use `MEMBER_PORTAL_URL` everywhere |
| 11 | **Health Check Exclusion** | Only OptionStrategy excludes `/api/health` from auth | All sub-portals MUST exclude `/api/health` |
| 12 | **Portal Session Cookie** | Currently `app_session_id` | **Keep `app_session_id`** — no functional benefit to renaming; would log out all users |
| 14 | **Service Identifier Values** | Portal uses generic `'stocks'`, `'options'` | Use actual names: `'swingtrade'`, `'option_strategy'` |

### LOW (3)

| # | Issue | Current State | Resolution |
|---|-------|---------------|------------|
| 8 | **Redirect After Auth** | Portal example: `/dashboard` / Sub-portals: `/` | Redirect to `/` |
| 10 | **Error Response Casing** | SwingTrade: `"unauthorized"` / OptionStrategy: `"Unauthorized"` | Lowercase snake_case: `unauthorized`, `session_expired` |
| 16 | **Webhook Coverage** | Only SwingTrade doc covers webhooks in detail | Webhooks are portal-only responsibility; documented in unified strategy |

---

## Impact Per Project

| Project | Total | Critical | High | Moderate | Low |
|---------|-------|----------|------|----------|-----|
| **Portal** | 12 | 3 | 5 | 3 | 1 |
| **SwingTrade** | 6 | 2 | 1 | 2 | 1 |
| **OptionStrategy** | 6 | 2 | 2 | 1 | 1 |

---

## Changes Required Per Project

### Portal (12 changes + 6 gaps)

> **Already existing**: Email/password login, user database, bcrypt, cookie utility (`server/_core/cookies.ts`), daily Patreon sync (`patreonSync.ts`), admin session cookie. Most items below are **NEW** (greenfield) or **MODIFY** (update existing code).

| # | Severity | Type | Change |
|---|----------|------|--------|
| 1 | CRITICAL | NEW | Implement `mapPatreonTierNameToAccess()` to return `basic` / `stocks_and_options`; add `tier` enum + `patreonTier` columns; update all tier checks |
| 2 | CRITICAL | NEW | Add `SWINGTRADE_TOKEN_SECRET` + `OPTION_STRATEGY_TOKEN_SECRET` env vars; use correct secret per service |
| 6 | CRITICAL | NEW | Handoff tokens: use `.setSubject(String(user.id))`, add `service` claim, no `userId`. Portal session JWT: keep `userId` alongside `sub` for backward compat |
| 3 | HIGH | NEW | Add env vars: `SWINGTRADE_URL`, `OPTION_STRATEGY_URL` |
| 4 | HIGH | NEW | Configure sub-portal cookie references: `swingtrade_session`, `option_strategy_session` (portal keeps `app_session_id`) |
| 7 | HIGH | NEW | Sub-portal session token uses `sub` claim and includes `tier` |
| 13 | HIGH | NEW | Create `mapPatreonTierNameToAccess()` in `server/subscription.ts`; case-sensitive Patreon tier name mapping |
| 15 | HIGH | NEW | Sub-portal handoff endpoint includes tier check |
| 5 | MODERATE | NEW | Create per-service `POST /api/launch/*` endpoints |
| 12 | RESOLVED | — | Keep `app_session_id` — no change needed |
| 14 | MODERATE | NEW | Use `'swingtrade'` and `'option_strategy'` as service identifiers in token claims |
| 16 | MODERATE | NEW | Implement webhook handler (after daily sync is stable) |
| 8 | LOW | NEW | Sub-portal example redirect to `/` |
| A | HIGH | MODIFY | Fix `patreonStatus` check: use `!== 'active'` (not `'active_patron'`) |
| C | LOW | MODIFY | Uniform 7d session TTL (OAuth is disabled) |
| D | MODERATE | — | Document `admin_session` cookie (already exists) |

### SwingTrade (6 changes)

| # | Severity | Change |
|---|----------|--------|
| 1 | CRITICAL | Update tier values from `free`/`stocks`/`stocks_and_options` to `basic`/`stocks_and_options`; set `ALLOWED_TIERS = ['basic', 'stocks_and_options']` (current business policy) |
| 6 | CRITICAL | Add `service` claim validation; remove `patreonId` from expected claims; read user ID from `payload.sub` |
| 4 | HIGH | Rename cookie from `session` to `swingtrade_session` |
| 9 | MODERATE | Replace `PORTAL_ORIGIN` env var with `MEMBER_PORTAL_URL` |
| 11 | MODERATE | Add `/api/health` exclusion to `requireAuth` middleware |
| 14 | MODERATE | Add `service` claim validation: `payload.service === 'swingtrade'` |

### OptionStrategy (6 changes)

| # | Severity | Change |
|---|----------|--------|
| 1 | CRITICAL | Define `ALLOWED_TIERS = ['basic', 'stocks_and_options']` explicitly (current business policy — all tiers) |
| 6 | CRITICAL | Read user ID from `payload.sub` instead of `payload.userId`; add `service` claim validation |
| 7 | HIGH | Change from `userId` in payload to `.setSubject()` for local session tokens |
| 15 | HIGH | Add tier check on `/auth` endpoint: reject if tier not in `ALLOWED_TIERS` |
| 10 | LOW | Lowercase error responses: `'unauthorized'`, `'session_expired'` |
| 14 | MODERATE | Add `service` claim validation: `payload.service === 'option_strategy'` |

---

## Additional Gaps (Code Reality)

These gaps were identified by comparing the golden docs against the **actual codebase**.

| Gap | Issue | Severity | Resolution |
|-----|-------|----------|------------|
| A | `patreonStatus` check used `'active_patron'` but users table stores `'active'` | HIGH | Use `patreonStatus !== 'active'` in tier mapping |
| B | `patreonMembers.tierId` exists but not populated by sync | LOW | Future enhancement; keep name-based mapping |
| C | Session TTL was 7d/365d in doc, 7d in code; OAuth disabled | LOW | Uniform 7d for all sessions |
| D | `admin_session` cookie undocumented | MODERATE | Added to cookie table |
| E | Railway proxy-aware `secure` flag not documented | MODERATE | Documented `X-Forwarded-Proto` pattern |
| F | Login history (`loginHistory` table) not mentioned | LOW | Optional task in Phase 4 |

---

## Recommended Implementation Order

The implementation follows a 4-phase plan:

**Phase 1: Database** — Add `tier` enum + `patreonTier` columns, migrate, backfill (#1, #13)

**Phase 2: Tier Mapping & JWT** — Create `mapPatreonTierNameToAccess()`, update `generateToken()`/`verifyToken()`, update login flow and daily sync (#1, #13, #6, #7, Gap A)

**Phase 3: Launch Endpoints** — Add per-service env vars (#2, #3), create `POST /api/launch/*` endpoints (#5, #14), generate per-service secrets

**Phase 4: Sub-Portals** — Implement `/auth/handoff` (#6), `requireAuth` middleware (#11), CORS (#9), cookie names (#4), tier checks (#15), error format (#10), frontend 401 (#8), optional login history (Gap F)
