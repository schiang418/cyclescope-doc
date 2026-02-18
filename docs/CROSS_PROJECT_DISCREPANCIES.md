# Cross-Project Authentication Discrepancies Report

> **Purpose**: Documents all discrepancies found between the three CycleScope project authentication documents.
> Each discrepancy includes what each project currently says, the resolution chosen in the unified strategy, and what each project must change.
>
> **Reference**: [UNIFIED_AUTH_STRATEGY.md](./UNIFIED_AUTH_STRATEGY.md) — the single source of truth
>
> Last updated: 2026-02-18

---

## Source Documents Analyzed

| Project | Document | Branch |
|---------|----------|--------|
| cyclescope-member-portal | `docs/AUTHENTICATION_AND_API_ARCHITECTURE.md` | `claude/fix-language-toggle-portal-O5mtJ` |
| SwingTrade | `docs/AUTH_STRATEGY.md` | `claude/stock-swing-trade-ranking-lmugL` |
| OptionStrategy | `docs/SUB_PORTAL_AUTH_INTEGRATION.md` | `claude/option-income-strategy-app-xtFx9` |

---

## Severity Legend

| Severity | Meaning |
|----------|---------|
| **CRITICAL** | Will cause authentication failures or security vulnerabilities if not aligned |
| **HIGH** | Will cause runtime errors or broken redirects |
| **MODERATE** | Inconsistency that causes confusion but may not break functionality immediately |
| **LOW** | Cosmetic or naming inconsistency |

---

## Discrepancy #1: Subscription Tier Values

**Severity**: CRITICAL

The three projects use fundamentally different tier naming schemes. If these don't match, tier checks will fail silently — users will either be denied access they paid for, or granted access they shouldn't have.

| Project | Tier Values Used |
|---------|-----------------|
| **Portal** | `'free'`, `'basic'`, `'premium'` |
| **SwingTrade** | `'free'`, `'stocks'`, `'stocks_and_options'` |
| **OptionStrategy** | References `tier` field but doesn't define specific values |

**Resolution**: Adopt two canonical tiers — `'basic'` and `'stocks_and_options'`. No free tier exists.

**Rationale**: There are only two Patreon subscription levels. `basic` is the base offering (Portal only). `stocks_and_options` is the premium tier (Portal + SwingTrade + OptionStrategy). SwingTrade and OptionStrategy are **premium services** — they nominally require `stocks_and_options`. However, which tiers can access which services is a business decision configured via each service's `ALLOWED_TIERS`. Currently, as a promotional business decision, all `basic` members are given access to all premium services.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Replace `mapPatreonTierToAccess()` return type to `'basic' \| 'stocks_and_options'`. Update database `tier` column values. Update all tier checks in frontend and backend. |
| **SwingTrade** | Update tier values. Define `ALLOWED_TIERS = ['basic', 'stocks_and_options']` (current business policy). |
| **OptionStrategy** | Define `ALLOWED_TIERS = ['basic', 'stocks_and_options']` (current business policy). |

---

## Discrepancy #2: Token Secret Strategy (Single vs Per-Service)

**Severity**: CRITICAL

| Project | Approach | Portal-Side Variable(s) |
|---------|----------|------------------------|
| **Portal** | Single shared secret for ALL services | `PREMIUM_TOKEN_SECRET` |
| **SwingTrade** | Per-service secrets | `SWINGTRADE_TOKEN_SECRET`, `OPTIONS_TOKEN_SECRET` |
| **OptionStrategy** | Single shared secret | `PREMIUM_TOKEN_SECRET` |

**Resolution**: Adopt per-service secrets (SwingTrade's approach).

**Rationale**: If a single shared secret is compromised (e.g., through a breach of one sub-portal), an attacker could forge tokens for ALL sub-portals. Per-service secrets contain the blast radius.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Replace `PREMIUM_TOKEN_SECRET` with `SWINGTRADE_TOKEN_SECRET` and `OPTION_STRATEGY_TOKEN_SECRET`. Use the correct secret when generating tokens for each service. |
| **SwingTrade** | Already aligned. `PREMIUM_TOKEN_SECRET` on SwingTrade side matches portal's `SWINGTRADE_TOKEN_SECRET`. |
| **OptionStrategy** | Already uses `PREMIUM_TOKEN_SECRET` locally. Just ensure it matches portal's `OPTION_STRATEGY_TOKEN_SECRET` (not `SWINGTRADE_TOKEN_SECRET`). |

---

## Discrepancy #3: Portal Environment Variable Names

**Severity**: HIGH

The portal-side variable names for service URLs and secrets differ across documents.

| Variable Purpose | Portal Doc | SwingTrade Doc | Unified Standard |
|-----------------|------------|----------------|------------------|
| SwingTrade handoff secret | `PREMIUM_TOKEN_SECRET` | `SWINGTRADE_TOKEN_SECRET` | `SWINGTRADE_TOKEN_SECRET` |
| Options handoff secret | `PREMIUM_TOKEN_SECRET` | `OPTIONS_TOKEN_SECRET` | `OPTION_STRATEGY_TOKEN_SECRET` |
| SwingTrade URL | `PREMIUM_STOCKS_URL` | `SWINGTRADE_URL` | `SWINGTRADE_URL` |
| Options URL | `PREMIUM_OPTIONS_URL` | `OPTIONS_URL` | `OPTION_STRATEGY_URL` |

**Resolution**: Use descriptive, service-specific names on the portal side.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Rename env vars: `PREMIUM_TOKEN_SECRET` -> `SWINGTRADE_TOKEN_SECRET` + `OPTION_STRATEGY_TOKEN_SECRET`; `PREMIUM_STOCKS_URL` -> `SWINGTRADE_URL`; `PREMIUM_OPTIONS_URL` -> `OPTION_STRATEGY_URL` |
| **SwingTrade** | Rename `OPTIONS_TOKEN_SECRET` reference to `OPTION_STRATEGY_TOKEN_SECRET` (this is portal-side only, informational). |
| **OptionStrategy** | No change needed (sub-portal always uses `PREMIUM_TOKEN_SECRET` locally). |

---

## Discrepancy #4: Cookie Names

**Severity**: HIGH

Three different documents propose three different cookie names for SwingTrade.

| Project | SwingTrade Cookie | OptionStrategy Cookie |
|---------|-------------------|-----------------------|
| **Portal** | `stocks_session` | `options_session` |
| **SwingTrade** | `session` | — |
| **OptionStrategy** | `swingtrade_session` (in convention table) | `option_strategy_session` |

**Resolution**: Use the naming convention from the OptionStrategy doc: `{service_name}_session`.

| Service | Cookie Name |
|---------|-------------|
| Portal | `cyclescope_portal_session` |
| SwingTrade | `swingtrade_session` |
| OptionStrategy | `option_strategy_session` |

**Rationale**: Descriptive, avoids collisions if services share a parent domain, and follows a consistent convention.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Update references from `stocks_session`/`options_session` to `swingtrade_session`/`option_strategy_session`. Keep portal's own cookie as `app_session_id` (no rename — see Discrepancy #12). |
| **SwingTrade** | Change cookie name from `session` to `swingtrade_session`. |
| **OptionStrategy** | Already aligned. Uses `option_strategy_session`. |

---

## Discrepancy #5: Portal Launch Endpoint

**Severity**: MODERATE

| Project | Endpoint | Method |
|---------|----------|--------|
| **Portal** | `/api/premium/access-token?service=xxx` | `GET` |
| **SwingTrade** | `/api/launch/swingtrade` | `POST` |
| **OptionStrategy** | Not specified (relies on portal) | — |

**Resolution**: Use per-service `POST` endpoints.

| Service | Endpoint |
|---------|----------|
| SwingTrade | `POST /api/launch/swingtrade` |
| OptionStrategy | `POST /api/launch/option-strategy` |

**Rationale**: `POST` is correct because this operation has side effects (token generation). Per-service endpoints are more explicit and allow per-service middleware if needed in the future.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Replace generic `GET /api/premium/access-token?service=xxx` with `POST /api/launch/swingtrade` and `POST /api/launch/option-strategy`. |
| **SwingTrade** | Already aligned. |
| **OptionStrategy** | Document the portal endpoint it expects: `POST /api/launch/option-strategy`. |

---

## Discrepancy #6: Handoff Token JWT Claims

**Severity**: CRITICAL

The three projects encode different data in the handoff JWT.

| Claim | Portal | SwingTrade | OptionStrategy |
|-------|--------|------------|----------------|
| `sub` (userId) | Not used — puts `userId` in payload | Uses `.setSubject(user.id)` | Not used — puts `userId` in payload |
| `email` | yes | yes | yes |
| `tier` | yes | yes | yes |
| `service` | yes | no | no |
| `userId` (in payload) | yes | no (uses `sub`) | yes |
| `patreonId` | no | yes | no |

**Resolution**: Standardize on:

```
{
  sub: <userId>,        // via .setSubject()
  email: <email>,
  tier: <tier>,
  service: <serviceId>  // 'swingtrade' or 'option_strategy'
}
```

**Rationale**:
- `sub` is the JWT standard claim for subject/user identity — use it instead of custom `userId`.
- `service` claim allows sub-portals to reject tokens meant for other services.
- `patreonId` is not needed by sub-portals (they don't interact with Patreon).

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Use `.setSubject(String(user.id))` instead of `userId` in payload. Add `service` claim. Remove `userId` from payload body. |
| **SwingTrade** | Add validation of `service` claim. Remove `patreonId` from expected claims. Read user ID from `payload.sub`. |
| **OptionStrategy** | Read user ID from `payload.sub` instead of `payload.userId`. Add validation of `service` claim. |

---

## Discrepancy #7: Local Session Token Claims

**Severity**: HIGH

| Claim | Portal (for sub-portals) | SwingTrade | OptionStrategy |
|-------|-------------------------|------------|----------------|
| `sub` (userId) | Not used | Uses `.setSubject(payload.sub)` | Not used |
| `userId` (in payload) | yes | no | yes |
| `email` | yes | yes | yes |
| `tier` | no | yes | yes |

**Resolution**: Standardize on:

```
{
  sub: <userId>,    // via .setSubject()
  email: <email>,
  tier: <tier>      // included so sub-portal can do tier checks without DB
}
```

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal doc** | Update sub-portal session token example to use `sub` and include `tier`. |
| **SwingTrade** | Already mostly aligned. Ensure `tier` is included. |
| **OptionStrategy** | Change from `userId` in payload to `.setSubject()`. Already includes `tier`. |

---

## Discrepancy #8: Redirect Destination After Auth

**Severity**: LOW

| Project | Redirects To |
|---------|-------------|
| **Portal doc** (sub-portal example) | `/dashboard` |
| **SwingTrade** | `/` |
| **OptionStrategy** | `/` |

**Resolution**: Redirect to `/` (root). Sub-portals can handle their own internal routing from root.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal doc** | Update sub-portal example code: `res.redirect('/')` instead of `res.redirect('/dashboard')`. |
| **SwingTrade** | Already aligned. |
| **OptionStrategy** | Already aligned. |

---

## Discrepancy #9: CORS Origin Variable Name

**Severity**: MODERATE

| Project | Variable Used for CORS Origin |
|---------|-------------------------------|
| **SwingTrade** | `PORTAL_ORIGIN` |
| **OptionStrategy** | `MEMBER_PORTAL_URL` |

**Resolution**: Use `MEMBER_PORTAL_URL` for both CORS origin and redirect URL.

**Rationale**: One variable serves both purposes. No need for a separate `PORTAL_ORIGIN` variable since the portal URL and origin are the same value.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **SwingTrade** | Remove `PORTAL_ORIGIN` env var. Use `MEMBER_PORTAL_URL` for CORS `origin` config. |
| **OptionStrategy** | Already aligned. |

---

## Discrepancy #10: 401 Error Response Casing

**Severity**: LOW

| Project | No Cookie Response | Expired Token Response |
|---------|-------------------|----------------------|
| **SwingTrade** | `{ error: "unauthorized" }` | `{ error: "session_expired" }` |
| **OptionStrategy** | `{ error: 'Unauthorized' }` | `{ error: 'Session expired' }` |

**Resolution**: Use lowercase, snake_case error codes consistently.

```typescript
// No cookie
{ error: 'unauthorized' }

// Expired/invalid token
{ error: 'session_expired' }
```

**Rationale**: Error codes should be machine-readable identifiers, not human-readable messages. Lowercase snake_case is the most common convention for error codes.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **SwingTrade** | Already aligned. |
| **OptionStrategy** | Change `'Unauthorized'` to `'unauthorized'`, `'Session expired'` to `'session_expired'`. |

---

## Discrepancy #11: Health Check Exclusion from Auth

**Severity**: MODERATE

| Project | `/api/health` Excluded from Auth? |
|---------|-----------------------------------|
| **Portal doc** | Not mentioned |
| **SwingTrade** | Not mentioned |
| **OptionStrategy** | Yes — explicitly excluded |

**Resolution**: All sub-portals MUST exclude `/api/health` from auth middleware.

**Rationale**: Health check endpoints are needed by deployment platforms (Railway, etc.) for readiness probes. They must work without authentication.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **SwingTrade** | Add health check exclusion to `requireAuth` middleware. |
| **OptionStrategy** | Already aligned. |

---

## Discrepancy #12: Portal Session Cookie Name

**Severity**: ~~MODERATE~~ **RESOLVED (no change)**

| Project | Portal Cookie Name |
|---------|-------------------|
| **Portal** | `app_session_id` (defined as `COOKIE_NAME`) |
| **SwingTrade** | Not referenced |
| **OptionStrategy** | Not referenced |

**Resolution**: **Keep `app_session_id`.** The unified strategy initially recommended renaming to `cyclescope_portal_session` for naming consistency, but this provides no functional benefit while causing user disruption (all existing sessions would be invalidated, logging out every user). The portal cookie is never read by sub-portals, so the name has no cross-project impact.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | None. Keep `app_session_id`. |

---

## Discrepancy #13: Tier Mapping Function

**Severity**: HIGH

The portal's tier mapping function returns different tier values than what the sub-portals expect.

| Portal's Current Implementation | What Sub-Portals Expect |
|-------------------------------|------------------------|
| `'free' \| 'basic' \| 'premium'` | `'basic' \| 'stocks_and_options'` |
| Maps from Patreon tier *names* (e.g., `'Premium Tier'`) | Maps from Patreon tier *IDs* |
| Returns `'basic'` as default for active patrons | Should default to `'basic'` for unmapped tiers |

**Resolution**: Portal must map from Patreon tier IDs to the two canonical tier values.

```typescript
const PATREON_TIER_MAP: Record<string, SubscriptionTier> = {
  '<patreon_basic_tier_id>': 'basic',
  '<patreon_both_tier_id>': 'stocks_and_options',
};
```

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Rewrite `mapPatreonTierToAccess()` to use tier IDs and return `'basic' \| 'stocks_and_options'`. |

---

## Discrepancy #14: Service Identifier Values

**Severity**: MODERATE

The `service` claim in the handoff token needs consistent identifiers, but the documents don't agree on what to call the options service.

| Context | SwingTrade ID | Options Service ID |
|---------|---------------|--------------------|
| **Portal doc** | `'stocks'` (in generic service param) | `'options'` (in generic service param) |
| **SwingTrade doc** | Not included in token | Not included in token |
| **OptionStrategy doc** | Not included in token | Not included in token |

**Resolution**: Define canonical service identifiers:

| Service | `service` Claim Value |
|---------|----------------------|
| SwingTrade | `'swingtrade'` |
| OptionStrategy | `'option_strategy'` |

**Rationale**: Use the actual project/service names, not generic labels like `'stocks'` or `'options'`.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Use `'swingtrade'` and `'option_strategy'` in token `service` claim. |
| **SwingTrade** | Add `service` claim validation: `payload.service === 'swingtrade'`. |
| **OptionStrategy** | Add `service` claim validation: `payload.service === 'option_strategy'`. |

---

## Discrepancy #15: Tier Check Logic on Sub-Portals

**Severity**: HIGH

Neither sub-portal explicitly checks tier on the `/auth` endpoint in their current docs.

| Project | Tier Check on `/auth`? |
|---------|------------------------|
| **Portal doc** (sub-portal example) | No — the portal's example code doesn't check tier after verifying the token |
| **SwingTrade** | Yes — checks `allowedTiers.includes(payload.tier)` |
| **OptionStrategy** | No — only verifies the token, doesn't check tier |

**Resolution**: All sub-portals MUST check the tier on the `/auth` endpoint. Defense in depth — don't rely solely on the portal to enforce access.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal doc** | Update sub-portal example to include tier check. |
| **SwingTrade** | Already aligned. |
| **OptionStrategy** | Add tier check: `if (!ALLOWED_TIERS.includes(payload.tier)) redirect to portal`. `ALLOWED_TIERS = ['basic', 'stocks_and_options']` (current policy). |

---

## Discrepancy #16: Webhook Handling Coverage

**Severity**: MODERATE

| Project | Covers Webhook Handling? |
|---------|--------------------------|
| **Portal doc** | Briefly, in implementation checklist |
| **SwingTrade doc** | Yes — detailed webhook events and tier mapping |
| **OptionStrategy doc** | No |

**Resolution**: Webhook handling is documented in the unified strategy as portal-only responsibility. Sub-portals do NOT need webhook handling code.

**Changes Required**:

| Project | What to Change |
|---------|---------------|
| **Portal** | Implement webhook handler per unified strategy. |
| **SwingTrade** | Remove any webhook handling references from sub-portal code (webhook is portal responsibility). |
| **OptionStrategy** | No changes needed. |

---

## Summary Table

| # | Discrepancy | Severity | Portal Change | SwingTrade Change | OptionStrategy Change |
|---|------------|----------|---------------|-------------------|-----------------------|
| 1 | Tier values | CRITICAL | Yes | Yes | Yes |
| 2 | Token secret strategy | CRITICAL | Yes | No | No |
| 3 | Portal env var names | HIGH | Yes | Info only | No |
| 4 | Cookie names | HIGH | Yes | Yes | No |
| 5 | Launch endpoint | MODERATE | Yes | No | Info only |
| 6 | Handoff token claims | CRITICAL | Yes | Yes | Yes |
| 7 | Session token claims | HIGH | Yes | No | Yes |
| 8 | Redirect destination | LOW | Yes | No | No |
| 9 | CORS variable name | MODERATE | No | Yes | No |
| 10 | Error response casing | LOW | No | No | Yes |
| 11 | Health check exclusion | MODERATE | No | Yes | No |
| 12 | Portal cookie name | RESOLVED | No (keep `app_session_id`) | No | No |
| 13 | Tier mapping function | HIGH | Yes | No | No |
| 14 | Service identifiers | MODERATE | Yes | Yes | Yes |
| 15 | Tier check on /auth | HIGH | Yes | No | Yes |
| 16 | Webhook coverage | MODERATE | Yes | Info only | No |

### Changes Required Per Project

| Project | Total Changes | Critical | High | Moderate | Low |
|---------|--------------|----------|------|----------|-----|
| **Portal** | 12 | 3 | 5 | 3 | 1 |
| **SwingTrade** | 6 | 2 | 1 | 2 | 1 |
| **OptionStrategy** | 6 | 2 | 2 | 1 | 1 |
