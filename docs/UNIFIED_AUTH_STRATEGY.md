# CycleScope Unified Authentication & Authorization Strategy

> **This is the Single Source of Truth** for cross-project authentication across all CycleScope services.
>
> All three projects MUST reference this document and align to it exactly.
>
> - **Member Portal**: `cyclescope-member-portal`
> - **Stock Service**: `SwingTrade`
> - **Options Service**: `OptionStrategy`
>
> Last updated: 2026-02-18

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Subscription Tiers](#2-subscription-tiers)
3. [Environment Variables (All Projects)](#3-environment-variables-all-projects)
4. [Token Exchange Flow](#4-token-exchange-flow)
5. [JWT Specifications](#5-jwt-specifications)
6. [Cookie Specifications](#6-cookie-specifications)
7. [Portal Implementation](#7-portal-implementation)
8. [Sub-Portal Implementation](#8-sub-portal-implementation)
9. [CORS Configuration](#9-cors-configuration)
10. [Patreon Webhook Handling](#10-patreon-webhook-handling)
11. [Frontend 401 Handling](#11-frontend-401-handling)
12. [Security Summary](#12-security-summary)
13. [Tier-Based Access Matrix](#13-tier-based-access-matrix)
14. [Error Response Format](#14-error-response-format)
15. [Implementation Checklist](#15-implementation-checklist)
16. [Dependencies](#16-dependencies)

---

## 1. Architecture Overview

```
┌──────────┐      OAuth      ┌────────────────────────┐
│  Patreon  │◄──────────────►│  CycleScope Member     │
│  (tiers)  │   webhooks     │  Portal                │
└──────────┘                 │  - authenticates users  │
                             │  - stores tier info     │
                             │  - issues access tokens │
                             └──────────┬─────────────┘
                                        │
                          ┌─────────────┼─────────────┐
                          │  5-min JWT  │  5-min JWT  │
                          ▼             │             ▼
                 ┌──────────────┐      │    ┌──────────────┐
                 │  SwingTrade  │      │    │OptionStrategy│
                 │  (stocks)    │      │    │  (options)   │
                 └──────────────┘      │    └──────────────┘
                                       │
                                  (future services)
```

**Portal** = gatekeeper (authenticates users, determines tier, issues access tokens)
**SwingTrade / OptionStrategy** = enforcers (verify token, check tier, manage local sessions)

### Key Principles

1. Sub-portals **never** handle passwords, signup, or Patreon OAuth directly.
2. All user authentication UI lives **exclusively** in the Member Portal.
3. Each sub-portal receives a short-lived handoff token and creates its own local session.
4. Per-service token secrets ensure compromise isolation.
5. Sub-portal data is shared across all premium users (no per-user data isolation needed in sub-portals).
6. SwingTrade and OptionStrategy are **architecturally independent premium services**. Tier-to-service access mapping is a business-layer decision configured via each service's `ALLOWED_TIERS`.

---

## 2. Subscription Tiers

### Canonical Tier Values

These are the **exact string values** stored in the database and embedded in JWTs. All three projects MUST use these values verbatim.

| Internal Key              | Tier Name           | Access Granted                              |
|---------------------------|---------------------|---------------------------------------------|
| `basic`                   | Basic               | Portal only                                 |
| `stocks_and_options`      | Stocks + Options    | Portal + **SwingTrade** + **OptionStrategy**|

> **No free tier.** All users have at least a `basic` subscription. There is no unpaid/community tier.
>
> SwingTrade and OptionStrategy are **premium services** — the `stocks_and_options` tier is what grants access to them.

### Tier-to-Service Access: Business vs Architecture

SwingTrade and OptionStrategy are **architecturally independent premium services**. Which tiers can access which services is a **business decision**, not an architectural constraint. The mapping is configured per-service via `ALLOWED_TIERS` and can change at any time without code changes.

**Current business decision**: All `basic` members are given access to all premium services (promotional). This may be restricted in the future so that `basic` = Portal only and only `stocks_and_options` members access SwingTrade and OptionStrategy.

### Tier Mapping from Patreon

The portal maps Patreon tier IDs to internal keys:

```typescript
// Portal: server/subscription.ts

// Fill in actual Patreon tier IDs once created on patreon.com/cyclescope
const PATREON_TIER_MAP: Record<string, SubscriptionTier> = {
  '<patreon_basic_tier_id>': 'basic',
  '<patreon_both_tier_id>': 'stocks_and_options',
};

export type SubscriptionTier = 'basic' | 'stocks_and_options';

export function mapPatreonTierToAccess(
  patreonTierId: string | null,
  patronStatus: string
): SubscriptionTier {
  if (patronStatus !== 'active_patron') {
    return 'basic';
  }
  return PATREON_TIER_MAP[patreonTierId || ''] || 'basic';
}
```

### Tier Check Constants (Sub-Portals)

Each sub-portal defines which tiers grant access. This is a **configurable business decision** — update these arrays when access policy changes.

```typescript
// SwingTrade: src/auth.ts (premium service)
// Nominally stocks_and_options only, but currently all tiers get access (business decision).
// To restrict: change to ['stocks_and_options']
const ALLOWED_TIERS: string[] = ['basic', 'stocks_and_options'];

// OptionStrategy: src/auth.ts (premium service)
// Nominally stocks_and_options only, but currently all tiers get access (business decision).
// To restrict: change to ['stocks_and_options']
const ALLOWED_TIERS: string[] = ['basic', 'stocks_and_options'];
```

---

## 3. Environment Variables (All Projects)

### 3.1 Member Portal (`cyclescope-member-portal/.env`)

```env
# ── Patreon OAuth ──
PATREON_CLIENT_ID=<from Patreon developer portal>
PATREON_CLIENT_SECRET=<from Patreon developer portal>
PATREON_REDIRECT_URI=https://portal.cyclescope.com/auth/patreon/callback
PATREON_WEBHOOK_SECRET=<from Patreon webhook setup>

# ── Portal Session ──
JWT_SECRET=<random 256-bit secret, unique to portal>

# ── Per-Service Token Secrets (for issuing handoff tokens) ──
SWINGTRADE_TOKEN_SECRET=<random 256-bit secret, shared with SwingTrade>
OPTION_STRATEGY_TOKEN_SECRET=<random 256-bit secret, shared with OptionStrategy>

# ── Sub-Portal URLs ──
SWINGTRADE_URL=https://swingtrade.up.railway.app
OPTION_STRATEGY_URL=https://option-strategy.up.railway.app

# ── Email Service ──
RESEND_API_KEY=re_xxxxx

# ── Portal Cookie ──
# Cookie name defined in code as constant: 'cyclescope_portal_session'
```

### 3.2 SwingTrade (`SwingTrade/.env`)

```env
# ── Handoff Token Verification ──
# MUST match portal's SWINGTRADE_TOKEN_SECRET
PREMIUM_TOKEN_SECRET=<same value as portal's SWINGTRADE_TOKEN_SECRET>

# ── Local Session ──
# Unique to SwingTrade — NOT shared with any other service
JWT_SECRET=<random 256-bit secret, unique to SwingTrade>

# ── Portal References ──
MEMBER_PORTAL_URL=https://portal.cyclescope.com

# ── Local Cookie ──
# Cookie name defined in code as constant: 'swingtrade_session'
```

### 3.3 OptionStrategy (`OptionStrategy/.env`)

```env
# ── Handoff Token Verification ──
# MUST match portal's OPTION_STRATEGY_TOKEN_SECRET
PREMIUM_TOKEN_SECRET=<same value as portal's OPTION_STRATEGY_TOKEN_SECRET>

# ── Local Session ──
# Unique to OptionStrategy — NOT shared with any other service
JWT_SECRET=<random 256-bit secret, unique to OptionStrategy>

# ── Portal References ──
MEMBER_PORTAL_URL=https://portal.cyclescope.com

# ── Local Cookie ──
# Cookie name defined in code as constant: 'option_strategy_session'
```

### Variable Cross-Reference

| Variable | Portal | SwingTrade | OptionStrategy | Notes |
|----------|--------|------------|----------------|-------|
| `JWT_SECRET` | unique | unique | unique | Each service has its own. Never shared. |
| `SWINGTRADE_TOKEN_SECRET` | yes | — | — | Portal-side name for SwingTrade handoff secret |
| `OPTION_STRATEGY_TOKEN_SECRET` | yes | — | — | Portal-side name for OptionStrategy handoff secret |
| `PREMIUM_TOKEN_SECRET` | — | yes | yes | Sub-portal-side name. Must match its corresponding portal secret. |
| `MEMBER_PORTAL_URL` | — | yes | yes | Portal URL for redirects and CORS origin |
| `SWINGTRADE_URL` | yes | — | — | SwingTrade URL for redirects |
| `OPTION_STRATEGY_URL` | yes | — | — | OptionStrategy URL for redirects |
| `PATREON_CLIENT_ID` | yes | — | — | Only portal handles Patreon OAuth |
| `PATREON_CLIENT_SECRET` | yes | — | — | Only portal handles Patreon OAuth |
| `PATREON_WEBHOOK_SECRET` | yes | — | — | Only portal handles webhooks |

> **Critical**: `PREMIUM_TOKEN_SECRET` and `JWT_SECRET` MUST be different values on each sub-portal. A compromise of the shared handoff secret should not compromise local session cookies (and vice versa).

> **Critical**: Each sub-portal has its own `PREMIUM_TOKEN_SECRET` value that matches a different portal-side variable. SwingTrade's `PREMIUM_TOKEN_SECRET` = Portal's `SWINGTRADE_TOKEN_SECRET`. OptionStrategy's `PREMIUM_TOKEN_SECRET` = Portal's `OPTION_STRATEGY_TOKEN_SECRET`. This means a compromise of one sub-portal's shared secret does NOT affect the other sub-portal.

---

## 4. Token Exchange Flow

### Step-by-Step Sequence

```
┌────────┐       ┌────────────────┐       ┌─────────────────┐
│ User   │       │ Member Portal  │       │  Sub-Portal     │
└───┬────┘       └───────┬────────┘       └────────┬────────┘
    │                    │                         │
    │ 1. Click "Launch   │                         │
    │    SwingTrade"     │                         │
    │───────────────────▶│                         │
    │                    │                         │
    │                    │ 2. Verify user session   │
    │                    │ 3. Check tier access     │
    │                    │ 4. Generate 5-min JWT    │
    │                    │    (signed with per-     │
    │                    │     service secret)      │
    │                    │                         │
    │  5. Redirect with  │                         │
    │  ?token=eyJ...     │                         │
    │◀───────────────────│                         │
    │                    │                         │
    │ 6. GET /auth?token=eyJ...                    │
    │─────────────────────────────────────────────▶│
    │                                              │
    │                      7. Verify handoff token  │
    │                      8. Check tier in token   │
    │                      9. Create local session  │
    │                     10. Set session cookie    │
    │                                              │
    │          11. 302 Redirect to /               │
    │◀─────────────────────────────────────────────│
    │                                              │
    │ 12. Subsequent API calls include cookie      │
    │─────────────────────────────────────────────▶│
    │                                              │
```

---

## 5. JWT Specifications

### 5.1 Portal Session Token (Portal internal use only)

| Field | Value | Notes |
|-------|-------|-------|
| Algorithm | `HS256` | |
| Library | `jose` | |
| Signing Secret | Portal's `JWT_SECRET` | |
| Expiration | `7d` (email/password login) or `365d` (Patreon OAuth) | |
| Cookie | `cyclescope_portal_session` | |

**Payload**:
```typescript
interface PortalSessionPayload {
  sub: string;        // User ID (as string)
  email: string;      // User email
  role: string;       // 'user' | 'admin'
  tier: SubscriptionTier;  // 'basic' | 'stocks_and_options'
  iat: number;        // Issued at (automatic)
  exp: number;        // Expiration (automatic)
}
```

### 5.2 Handoff Access Token (Portal -> Sub-Portal)

This is the short-lived token the portal generates when a user clicks a sub-portal link.

| Field | Value | Notes |
|-------|-------|-------|
| Algorithm | `HS256` | |
| Library | `jose` | |
| Signing Secret | Per-service: `SWINGTRADE_TOKEN_SECRET` or `OPTION_STRATEGY_TOKEN_SECRET` | |
| Expiration | `5m` | Short-lived for security |

**Payload (CANONICAL — all fields required)**:
```typescript
interface HandoffTokenPayload {
  sub: string;           // User ID (as string) — MUST use `sub` claim (JWT standard)
  email: string;         // User email
  tier: SubscriptionTier; // 'basic' | 'stocks_and_options'
  service: string;       // 'swingtrade' | 'option_strategy' — target service identifier
  iat: number;           // Issued at (automatic via .setIssuedAt())
  exp: number;           // Expiration (automatic via .setExpirationTime('5m'))
}
```

**Generation code** (Portal):
```typescript
import { SignJWT } from 'jose';

const token = await new SignJWT({
  email: user.email,
  tier: user.tier,
  service: 'swingtrade',  // or 'option_strategy'
})
  .setProtectedHeader({ alg: 'HS256' })
  .setSubject(String(user.id))  // MUST use .setSubject() for user ID
  .setIssuedAt()
  .setExpirationTime('5m')
  .sign(secret);
```

### 5.3 Sub-Portal Local Session Token

Each sub-portal creates its own session after verifying the handoff token.

| Field | Value | Notes |
|-------|-------|-------|
| Algorithm | `HS256` | |
| Library | `jose` | |
| Signing Secret | Sub-portal's own `JWT_SECRET` | |
| Expiration | `7d` | |

**Payload (CANONICAL — all fields required)**:
```typescript
interface SubPortalSessionPayload {
  sub: string;           // User ID (as string) — from handoff token's `sub`
  email: string;         // User email — from handoff token
  tier: SubscriptionTier; // Tier — from handoff token. Included so sub-portal can do tier checks without DB.
  iat: number;           // Issued at (automatic)
  exp: number;           // Expiration (automatic)
}
```

**Generation code** (Sub-Portal):
```typescript
const sessionToken = await new SignJWT({
  email: payload.email as string,
  tier: payload.tier as string,
})
  .setProtectedHeader({ alg: 'HS256' })
  .setSubject(payload.sub as string)  // MUST use .setSubject()
  .setIssuedAt()
  .setExpirationTime('7d')
  .sign(sessionSecret);
```

---

## 6. Cookie Specifications

### 6.1 Cookie Names (CANONICAL)

| Service | Cookie Name | Notes |
|---------|-------------|-------|
| Member Portal | `cyclescope_portal_session` | Portal's own session |
| SwingTrade | `swingtrade_session` | SwingTrade local session |
| OptionStrategy | `option_strategy_session` | OptionStrategy local session |
| (Future services) | `{service_name}_session` | Follow this naming convention |

### 6.2 Cookie Options (ALL services)

```typescript
{
  httpOnly: true,                                      // Prevents JavaScript access (XSS protection)
  secure: process.env.NODE_ENV === 'production',       // HTTPS only in production
  sameSite: 'lax' as const,                            // CSRF protection
  path: '/',                                           // Sent for all routes
  maxAge: 7 * 24 * 60 * 60 * 1000,                    // 7 days in milliseconds
}
```

### 6.3 Cookie Helper (Shared Pattern)

Each service defines its cookie config as a constant:

```typescript
// SwingTrade: src/auth.ts
export const SESSION_COOKIE_NAME = 'swingtrade_session';

// OptionStrategy: src/auth.ts
export const SESSION_COOKIE_NAME = 'option_strategy_session';

// Portal: server/_core/cookies.ts
export const SESSION_COOKIE_NAME = 'cyclescope_portal_session';

// Shared cookie options function (each service implements its own copy)
export function getSessionCookieOptions(): {
  httpOnly: boolean;
  secure: boolean;
  sameSite: 'lax';
  path: string;
  maxAge: number;
} {
  return {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: 7 * 24 * 60 * 60 * 1000,
  };
}
```

---

## 7. Portal Implementation

### 7.1 Launch Endpoints

The portal exposes **per-service launch endpoints** (not a generic one).

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/launch/swingtrade` | `POST` | Generate handoff token for SwingTrade |
| `/api/launch/option-strategy` | `POST` | Generate handoff token for OptionStrategy |

**Implementation**:

```typescript
// Portal: server/routes/launch.ts
import { SignJWT } from 'jose';
import { Router } from 'express';

const router = Router();

// Service configuration registry
const SERVICE_CONFIG: Record<string, { secretEnvVar: string; urlEnvVar: string; serviceId: string }> = {
  swingtrade: {
    secretEnvVar: 'SWINGTRADE_TOKEN_SECRET',
    urlEnvVar: 'SWINGTRADE_URL',
    serviceId: 'swingtrade',
  },
  'option-strategy': {
    secretEnvVar: 'OPTION_STRATEGY_TOKEN_SECRET',
    urlEnvVar: 'OPTION_STRATEGY_URL',
    serviceId: 'option_strategy',
  },
};

// Tier access rules — which tiers can access which premium services
// This is a BUSINESS DECISION, not an architectural constraint.
// Both services are nominally stocks_and_options only.
// Current policy: all tiers get access (promotional). To restrict: remove 'basic'.
const SERVICE_TIER_ACCESS: Record<string, string[]> = {
  swingtrade: ['basic', 'stocks_and_options'],
  'option-strategy': ['basic', 'stocks_and_options'],
};

function createLaunchHandler(serviceKey: string) {
  return async (req: Request, res: Response) => {
    const config = SERVICE_CONFIG[serviceKey];
    const allowedTiers = SERVICE_TIER_ACCESS[serviceKey];
    const user = (req as any).user;

    // Check tier access
    if (!allowedTiers.includes(user.tier)) {
      return res.status(403).json({
        error: 'insufficient_tier',
        message: 'Your subscription does not include access to this service.',
        currentTier: user.tier,
        requiredTiers: allowedTiers,
      });
    }

    // Generate handoff token
    const secret = new TextEncoder().encode(process.env[config.secretEnvVar]);
    const token = await new SignJWT({
      email: user.email,
      tier: user.tier,
      service: config.serviceId,
    })
      .setProtectedHeader({ alg: 'HS256' })
      .setSubject(String(user.id))
      .setIssuedAt()
      .setExpirationTime('5m')
      .sign(secret);

    const serviceUrl = process.env[config.urlEnvVar];
    return res.json({
      redirectUrl: `${serviceUrl}/auth?token=${token}`,
    });
  };
}

// Mount endpoints
router.post('/launch/swingtrade', requireAuth, createLaunchHandler('swingtrade'));
router.post('/launch/option-strategy', requireAuth, createLaunchHandler('option-strategy'));

export default router;
```

### 7.2 Portal Frontend — Launch Buttons

```typescript
// Portal: src/components/PremiumServices.tsx

// Premium service access config — update when business policy changes
// Both are nominally stocks_and_options only; currently promotional for all tiers
const TIER_ACCESS = {
  swingtrade: ['basic', 'stocks_and_options'],
  option_strategy: ['basic', 'stocks_and_options'],
} as const;

function canAccess(userTier: string, service: keyof typeof TIER_ACCESS): boolean {
  return (TIER_ACCESS[service] as readonly string[]).includes(userTier);
}

async function launchService(service: 'swingtrade' | 'option-strategy') {
  const response = await fetch(`/api/launch/${service}`, {
    method: 'POST',
    credentials: 'include',
  });

  const data = await response.json();

  if (!response.ok) {
    if (data.error === 'insufficient_tier') {
      // Show upgrade prompt
      alert(data.message);
      return;
    }
    throw new Error(data.message);
  }

  // Redirect to sub-portal with handoff token
  window.location.href = data.redirectUrl;
}
```

---

## 8. Sub-Portal Implementation

### 8.1 Token Exchange Endpoint

**Route**: `GET /auth?token=xxx`

This is **identical** for both SwingTrade and OptionStrategy, differing only in constants.

```typescript
// Sub-Portal: src/auth.ts
import { jwtVerify, SignJWT } from 'jose';
import { Request, Response, NextFunction } from 'express';

// ── Constants (differ per service) ──
export const SESSION_COOKIE_NAME = 'swingtrade_session';  // or 'option_strategy_session'
// Premium service — nominally stocks_and_options only.
// Currently all tiers get access (business decision). To restrict: ['stocks_and_options']
const ALLOWED_TIERS = ['basic', 'stocks_and_options'];
const SERVICE_ID = 'swingtrade';                           // or 'option_strategy'

// ── Auth Endpoint ──
export async function handleAuthCallback(req: Request, res: Response) {
  const { token } = req.query;

  if (!token || typeof token !== 'string') {
    return res.redirect(`${process.env.MEMBER_PORTAL_URL}?error=missing_token`);
  }

  try {
    // 1. Verify handoff token from Member Portal
    const premiumSecret = new TextEncoder().encode(process.env.PREMIUM_TOKEN_SECRET);
    const { payload } = await jwtVerify(token, premiumSecret);

    // 2. Validate the token is intended for this service (recommended)
    if (payload.service && payload.service !== SERVICE_ID) {
      console.error(`[Auth] Token service mismatch: expected ${SERVICE_ID}, got ${payload.service}`);
      return res.redirect(`${process.env.MEMBER_PORTAL_URL}?error=invalid_service`);
    }

    // 3. Check tier authorization
    if (!ALLOWED_TIERS.includes(payload.tier as string)) {
      return res.redirect(`${process.env.MEMBER_PORTAL_URL}?error=upgrade_required`);
    }

    // 4. Create local session token
    const sessionSecret = new TextEncoder().encode(process.env.JWT_SECRET);
    const sessionToken = await new SignJWT({
      email: payload.email as string,
      tier: payload.tier as string,
    })
      .setProtectedHeader({ alg: 'HS256' })
      .setSubject(payload.sub as string)
      .setIssuedAt()
      .setExpirationTime('7d')
      .sign(sessionSecret);

    // 5. Set session cookie
    res.cookie(SESSION_COOKIE_NAME, sessionToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      path: '/',
      maxAge: 7 * 24 * 60 * 60 * 1000,
    });

    // 6. Redirect to app root
    return res.redirect('/');

  } catch (error) {
    console.error('[Auth] Token verification failed:', error);
    return res.redirect(`${process.env.MEMBER_PORTAL_URL}?error=invalid_token`);
  }
}
```

### 8.2 Auth Middleware

**Applies to**: All `/api/*` routes **except** `/api/health`

```typescript
// Sub-Portal: src/auth.ts (continued)

export async function requireAuth(req: Request, res: Response, next: NextFunction) {
  const token = req.cookies?.[SESSION_COOKIE_NAME];

  if (!token) {
    return res.status(401).json({ error: 'unauthorized' });
  }

  try {
    const secret = new TextEncoder().encode(process.env.JWT_SECRET);
    const { payload } = await jwtVerify(token, secret);
    (req as any).user = payload;  // { sub, email, tier, iat, exp }
    next();
  } catch {
    return res.status(401).json({ error: 'session_expired' });
  }
}

// Mount in Express app
app.use('/api', (req: Request, res: Response, next: NextFunction) => {
  if (req.path === '/health') return next();
  return requireAuth(req, res, next);
});
```

### 8.3 Service-Specific Constants

| Constant | SwingTrade | OptionStrategy |
|----------|------------|----------------|
| `SESSION_COOKIE_NAME` | `'swingtrade_session'` | `'option_strategy_session'` |
| `ALLOWED_TIERS` | `['basic', 'stocks_and_options']` (nominally `['stocks_and_options']`; currently promotional) | `['basic', 'stocks_and_options']` (nominally `['stocks_and_options']`; currently promotional) |
| `SERVICE_ID` | `'swingtrade'` | `'option_strategy'` |

---

## 9. CORS Configuration

Each sub-portal restricts CORS to the portal origin:

```typescript
import cors from 'cors';

app.use(cors({
  origin: process.env.MEMBER_PORTAL_URL,  // e.g. 'https://portal.cyclescope.com'
  credentials: true,                       // Required: allows cookies on cross-origin requests
}));
```

> **Note**: CORS only restricts browsers. The `requireAuth` middleware is the real security layer.

---

## 10. Patreon Webhook Handling

Handled **exclusively** by the Member Portal. Sub-portals do NOT interact with Patreon.

### Events to Handle

| Event | Action |
|-------|--------|
| `members:pledge:create` | Set user tier based on pledge tier |
| `members:pledge:update` | Update user tier (upgrade or downgrade) |
| `members:pledge:delete` | Downgrade user tier to `basic` |

### Portal Endpoint

```typescript
// Portal: server/routes/webhooks.ts
app.post('/api/webhooks/patreon', express.raw({ type: 'application/json' }), (req, res) => {
  // 1. Verify webhook signature using PATREON_WEBHOOK_SECRET
  // 2. Parse the event type and member data
  // 3. Map the Patreon tier ID to internal tier key using PATREON_TIER_MAP
  // 4. Update user record in portal database
  res.sendStatus(200);
});
```

### Propagation Delay Note

When a user downgrades on Patreon, their existing sub-portal session cookie remains valid for up to 7 days. On next re-auth through the portal, the new (lower) tier will be embedded in the token and they will lose access. For immediate revocation, sub-portals could expose a `/api/revoke` endpoint callable by the portal. This is optional and can be added in a future iteration.

---

## 11. Frontend 401 Handling

Every sub-portal's frontend MUST handle 401 responses by redirecting to the portal.

```typescript
// Sub-Portal: src/lib/api.ts

const MEMBER_PORTAL_URL = import.meta.env.VITE_MEMBER_PORTAL_URL
  || 'https://portal.cyclescope.com';

export async function apiFetch(url: string, options?: RequestInit): Promise<Response> {
  const res = await fetch(url, {
    ...options,
    credentials: 'include',  // MUST include cookies
  });

  if (res.status === 401) {
    window.location.href = MEMBER_PORTAL_URL;
    throw new Error('Session expired');
  }

  return res;
}
```

### Frontend Environment Variable

| Variable | SwingTrade | OptionStrategy |
|----------|------------|----------------|
| `VITE_MEMBER_PORTAL_URL` | `https://portal.cyclescope.com` | `https://portal.cyclescope.com` |

---

## 12. Security Summary

| Measure | What It Does | Where |
|---------|-------------|-------|
| Patreon OAuth | Authenticates user identity | Portal |
| Tier from Patreon API | Determines what the user can access | Portal |
| Per-service handoff secrets | Compromise of one sub-portal doesn't affect others | Portal <-> Sub-Portal |
| 5-min JWT handoff token | Short-lived pass from portal to sub-portal | Portal -> Sub-Portal |
| `service` claim in handoff | Sub-portal can reject tokens intended for other services | In JWT |
| Tier embedded in handoff | Sub-portal can verify authorization without calling portal | In JWT |
| `requireAuth` middleware | Blocks unauthenticated API access | Sub-Portal |
| 7-day local session cookie | Keeps user logged in within the sub-portal | Sub-Portal |
| `httpOnly` cookie flag | Prevents JavaScript access (XSS protection) | All |
| `secure` cookie flag | HTTPS only in production | All |
| `sameSite: lax` | CSRF protection | All |
| CORS lockdown | Prevents random websites from making API calls | Sub-Portal |
| Separate `JWT_SECRET` per service | Compromise of one doesn't affect others | All |
| Patreon webhooks | Keeps portal in sync with subscription changes | Portal |
| `/api/health` excluded from auth | Health checks work without authentication | Sub-Portal |

---

## 13. Tier-Based Access Matrix

> **Note**: SwingTrade and OptionStrategy are **premium services** that nominally belong to the `stocks_and_options` tier.
> However, the current business policy grants all `basic` members access to all services (promotional).
> This can be changed at any time by updating each service's `ALLOWED_TIERS` config.

| Resource | `basic` (nominal) | `basic` (current policy) | `stocks_and_options` |
|----------|--------------------|--------------------------|----------------------|
| Portal dashboard | yes | yes | yes |
| Portal blog / commentary | yes | yes | yes |
| SwingTrade — rankings | no | **yes** (promotional) | **yes** |
| SwingTrade — portfolios | no | **yes** (promotional) | **yes** |
| SwingTrade — EMA analysis | no | **yes** (promotional) | **yes** |
| OptionStrategy — scanner | no | **yes** (promotional) | **yes** |
| OptionStrategy — trade setups | no | **yes** (promotional) | **yes** |

---

## 14. Error Response Format

### 14.1 Sub-Portal API Errors (401)

All sub-portals MUST return errors in this exact format:

```typescript
// Unauthenticated (no cookie or missing cookie)
res.status(401).json({ error: 'unauthorized' });

// Session expired or invalid token
res.status(401).json({ error: 'session_expired' });
```

### 14.2 Portal Launch Errors

```typescript
// Tier insufficient
res.status(403).json({
  error: 'insufficient_tier',
  message: 'Your subscription does not include access to this service.',
  currentTier: user.tier,
  requiredTiers: ['basic', 'stocks_and_options'],  // varies per service (business decision)
});
```

### 14.3 Auth Redirect Error Parameters

When redirecting back to the portal on auth failure, use these query parameter values:

| Query Parameter | Meaning |
|----------------|---------|
| `?error=missing_token` | No token provided in `/auth` request |
| `?error=invalid_token` | Token verification failed (expired, bad signature, etc.) |
| `?error=invalid_service` | Token's `service` claim doesn't match this sub-portal |
| `?error=upgrade_required` | User's tier doesn't include access to this service |

---

## 15. Implementation Checklist

### Member Portal (`cyclescope-member-portal`)

- [ ] Patreon OAuth login flow
- [ ] Patreon API call to fetch user tier and map to internal tier key
- [ ] User database table with `tier` field using values: `basic`, `stocks_and_options`
- [ ] `POST /api/launch/swingtrade` — generate 5-min JWT signed with `SWINGTRADE_TOKEN_SECRET`
- [ ] `POST /api/launch/option-strategy` — generate 5-min JWT signed with `OPTION_STRATEGY_TOKEN_SECRET`
- [ ] `POST /api/webhooks/patreon` — handle subscription changes
- [ ] Frontend: show/hide/disable sub-portal launch buttons based on tier
- [ ] Frontend: handle `?error=upgrade_required` redirect-back from sub-portals
- [ ] Set env vars: `SWINGTRADE_TOKEN_SECRET`, `OPTION_STRATEGY_TOKEN_SECRET`, `SWINGTRADE_URL`, `OPTION_STRATEGY_URL`
- [ ] Portal session cookie named `cyclescope_portal_session`

### SwingTrade

- [ ] Install `jose` and `cookie-parser`
- [ ] `GET /auth?token=xxx` — verify handoff token, check tier, set local session cookie
- [ ] Validate `service` claim equals `'swingtrade'`
- [ ] Allowed tiers: `['basic', 'stocks_and_options']` (current business policy — all tiers)
- [ ] Session cookie named `swingtrade_session`
- [ ] `requireAuth` middleware on all `/api/*` routes (except `/api/health`)
- [ ] CORS locked to `MEMBER_PORTAL_URL`
- [ ] Frontend: 401 handler redirects to `MEMBER_PORTAL_URL`
- [ ] Set env vars: `PREMIUM_TOKEN_SECRET`, `JWT_SECRET`, `MEMBER_PORTAL_URL`
- [ ] Frontend env var: `VITE_MEMBER_PORTAL_URL`

### OptionStrategy

- [ ] Install `jose` and `cookie-parser`
- [ ] `GET /auth?token=xxx` — verify handoff token, check tier, set local session cookie
- [ ] Validate `service` claim equals `'option_strategy'`
- [ ] Allowed tiers: `['basic', 'stocks_and_options']` (current business policy — all tiers)
- [ ] Session cookie named `option_strategy_session`
- [ ] `requireAuth` middleware on all `/api/*` routes (except `/api/health`)
- [ ] CORS locked to `MEMBER_PORTAL_URL`
- [ ] Frontend: 401 handler redirects to `MEMBER_PORTAL_URL`
- [ ] Set env vars: `PREMIUM_TOKEN_SECRET`, `JWT_SECRET`, `MEMBER_PORTAL_URL`
- [ ] Frontend env var: `VITE_MEMBER_PORTAL_URL`

---

## 16. Dependencies

All three projects use these packages for auth:

```json
{
  "jose": "^5.2.0",
  "cookie-parser": "^1.4.6"
}
```

Portal additionally requires:

```json
{
  "bcrypt": "^5.1.1",
  "express": "^4.18.2",
  "zod": "^3.22.4",
  "@trpc/server": "^10.45.0",
  "nanoid": "^5.0.4"
}
```

Sub-portals additionally require:

```json
{
  "cors": "^2.8.5",
  "express": "^4.18.2"
}
```
