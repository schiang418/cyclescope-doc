# CycleScope Authentication Gap Analysis Summary

> **Quick-reference summary** of all 16 cross-project authentication discrepancies, their resolutions, and per-project impact.
>
> For full details and rationale, see [CROSS_PROJECT_DISCREPANCIES.md](./CROSS_PROJECT_DISCREPANCIES.md).
> For the authoritative spec, see [UNIFIED_AUTH_STRATEGY.md](./UNIFIED_AUTH_STRATEGY.md).
>
> Last updated: 2026-02-18

---

## Overview

Three CycleScope projects were analyzed for authentication alignment:

| Project | Document |
|---------|----------|
| cyclescope-member-portal | `docs/AUTHENTICATION_AND_API_ARCHITECTURE.md` |
| SwingTrade | `docs/AUTH_STRATEGY.md` |
| OptionStrategy | `docs/SUB_PORTAL_AUTH_INTEGRATION.md` |

**16 discrepancies** were found — **3 critical, 5 high, 5 moderate, 3 low**.

---

## Discrepancies by Severity

### CRITICAL (3)

| # | Issue | Current State | Resolution |
|---|-------|---------------|------------|
| 1 | **Tier Values Mismatch** | Portal: `free`, `basic`, `premium` / SwingTrade: `free`, `stocks`, `stocks_and_options` / OptionStrategy: undefined | Adopt `basic`, `stocks_and_options` (no free tier; both sub-portals are premium services — tier access is a business decision; currently all tiers get access) |
| 2 | **Token Secret Strategy** | Portal: single shared `PREMIUM_TOKEN_SECRET` / SwingTrade: per-service secrets / OptionStrategy: single shared | Per-service secrets — `SWINGTRADE_TOKEN_SECRET`, `OPTION_STRATEGY_TOKEN_SECRET` |
| 6 | **Handoff Token JWT Claims** | Portal: `userId` in body, includes `service` / SwingTrade: `sub` claim, includes `patreonId` / OptionStrategy: `userId` in body | Standardize: `sub` (userId), `email`, `tier`, `service`; drop `patreonId` |

### HIGH (5)

| # | Issue | Current State | Resolution |
|---|-------|---------------|------------|
| 3 | **Portal Env Var Names** | Portal: `PREMIUM_TOKEN_SECRET`, `PREMIUM_STOCKS_URL`, `PREMIUM_OPTIONS_URL` / SwingTrade: `SWINGTRADE_TOKEN_SECRET`, `SWINGTRADE_URL` | Service-specific names: `SWINGTRADE_TOKEN_SECRET`, `OPTION_STRATEGY_TOKEN_SECRET`, `SWINGTRADE_URL`, `OPTION_STRATEGY_URL` |
| 4 | **Cookie Names** | Portal: `stocks_session` / SwingTrade: `session` / OptionStrategy: `swingtrade_session` | Sub-portals: `swingtrade_session`, `option_strategy_session`. Portal: keep `app_session_id` (no rename) |
| 7 | **Local Session Token Claims** | Portal: `userId`, `email` (no tier) / SwingTrade: `sub`, `email`, `tier` / OptionStrategy: `userId`, `email`, `tier` | Standardize: `sub`, `email`, `tier` |
| 13 | **Tier Mapping Function** | Portal maps from Patreon tier *names* returning `basic`/`premium` | Map from Patreon tier *IDs* returning `basic`/`stocks_and_options` |
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

### Portal (12 changes)

| # | Severity | Change |
|---|----------|--------|
| 1 | CRITICAL | Rewrite `mapPatreonTierToAccess()` to return `basic` / `stocks_and_options`; update DB tier column values and all tier checks |
| 2 | CRITICAL | Replace `PREMIUM_TOKEN_SECRET` with `SWINGTRADE_TOKEN_SECRET` + `OPTION_STRATEGY_TOKEN_SECRET`; use correct secret per service |
| 6 | CRITICAL | Use `.setSubject(String(user.id))` for handoff tokens; add `service` claim; remove `userId` from payload body |
| 3 | HIGH | Rename env vars: `PREMIUM_STOCKS_URL` -> `SWINGTRADE_URL`, `PREMIUM_OPTIONS_URL` -> `OPTION_STRATEGY_URL` |
| 4 | HIGH | Rename sub-portal cookie references: `stocks_session` -> `swingtrade_session`, `options_session` -> `option_strategy_session` (portal keeps `app_session_id`) |
| 7 | HIGH | Update sub-portal session token example to use `sub` claim and include `tier` |
| 13 | HIGH | Rewrite tier mapper to use Patreon tier IDs, returning `basic` / `stocks_and_options` |
| 15 | HIGH | Update sub-portal example code to include tier check after token verification |
| 5 | MODERATE | Replace `GET /api/premium/access-token?service=xxx` with per-service `POST /api/launch/*` endpoints |
| 12 | ~~MODERATE~~ | ~~Rename `app_session_id` cookie~~ — **Resolved: keep `app_session_id`, no change needed** |
| 14 | MODERATE | Use `'swingtrade'` and `'option_strategy'` as service identifiers in token claims |
| 16 | MODERATE | Implement webhook handler per unified strategy |
| 8 | LOW | Update sub-portal example redirect from `/dashboard` to `/` |

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

## Recommended Implementation Order

1. **Portal tier values & mapping** (#1, #13) — Everything depends on correct tier values
2. **Portal token secrets & env vars** (#2, #3) — Security foundation
3. **Handoff token claims** (#6) — Portal + both sub-portals
4. **Sub-portal auth validation** (#15, #11) — Defense in depth
5. **Sub-portal cookie names** (#4) — SwingTrade and OptionStrategy only (portal keeps `app_session_id`)
6. **Session token claims** (#7) — After handoff tokens are stable
7. **Launch endpoints & service IDs** (#5, #14) — API contract changes
8. **CORS, redirects, error format** (#9, #8, #10) — Low-risk cleanup
9. **Webhook handling** (#16) — Portal-only, can be done independently
