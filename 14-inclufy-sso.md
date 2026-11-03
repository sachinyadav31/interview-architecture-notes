# 14: Inclufy single sign-on across tenants (Django + React)

## Context
Inclufy sells several products (Procurement, Sales, Finance, AI). The **Django backend** (`Inclufy-Django-Backend`, DRF + SimpleJWT) is the identity hub: users, subscriptions (Stripe), tenant profiles. The **eProcure Spring services** are the product backends, each with **its own** IAM per tenant. The React app (**Ignite**) needs a token for *every* tenant the user can reach.

## Login flow (`MyTokenObtainPairSerializer.validate`)
```mermaid
sequenceDiagram
  participant UI as Ignite (React)
  participant DJ as Django /api/token
  participant GW as eProcure gateway (per tenant)
  UI->>DJ: email + password (+ optional X-TENANT-QUALIFIER)
  DJ->>DJ: authenticate(); is_active? (email verified)
  DJ->>DJ: subscribed & tenants still provisioning (< 20 min)? → "setup in progress, N min left"
  DJ->>DJ: reachable_tenant_profiles(user) = OWNED ∪ ASSOCIATED (UserTenantAssociation)<br/>drop profiles with no domain (half-finished onboarding)
  DJ->>DJ: X-TENANT-QUALIFIER given? → scope to that tenant or 403 "no access to this tenant"
  loop each reachable tenant
    DJ->>GW: POST /user/auth (tenant credentials) with tenant qualifier
    GW-->>DJ: tenant JWT
  end
  DJ-->>UI: {access, refresh (Django), user, tenant_profiles[], tenant_tokens[ {qualifier, type, token} ], procurement_token, sales_token}
```

### The bug I fixed (Aug 2026), a great debugging story
- **Symptom**: a co-founder could log in, but every guarded screen (Settings…) was denied; `my-acls` came back empty. It happened twice in two days.
- **Root cause**: tenants reached through `UserTenantAssociation` were only consulted **when the user owned no TenantProfile**. The user owned a **half-finished onboarding shell with no domain**, so the associated tenants were hidden, no per-tenant token was issued, and the strict ACL context denied everything.
- **Fix**: resolve **owned ∪ associated**, drop domain-less profiles (they can never produce a token), track association per profile (only associated ones use the password fallback), and return **`tenant_tokens[]`, one per reachable tenant**. The old `procurement_token`/`sales_token` held one token per type (last write wins), which can't represent two tenants of the same type; they now keep the first and the array carries all of them.
- This complemented an earlier create-side guard that stopped *new* phantom tenants but couldn't clean up existing rows.
- Frontend side: a **tenant switcher** that only offers tenants the login actually got a token for.

### Token refresh fix (Jun 2026)
- **Symptom**: users were logged out within minutes.
- **Cause**: the 30 s expiry checker looked at the **per-tenant gateway token** (the shortest-lived) and logged out as soon as it expired, even though the Django session was refreshable.
- **Fix**: refresh first. Renew the Django access token with the refresh token, then **re-mint the per-tenant tokens**; log out only if there's no refresh token or the refresh is rejected. An **in-flight guard** stops overlapping refreshes across timer ticks and window-focus events.

### Gateway routing (Jul 2026)
All tenant API traffic goes through **one domain** (`inclufy.com`): the gateway reads an explicit `X-TARGET-TENANT-QUALIFIER` header first, then falls back to Host-based lookup. Django moved to `api.inclufy.com`. That's why tenant domains became optional.

## Weak spots and how I'd improve them (be ready; it's a real design smell)
- Django logs into each tenant **with a password** (the user's, or a stored per-tenant one, or a default fallback for associated users). That's credential propagation. The correct design is **token exchange / federation**: Django (or a real IdP like Keycloak/Auth0) issues a signed token that every tenant gateway trusts (shared JWKS), with tenant claims. No passwords between systems.
- `ACCESS_TOKEN_LIFETIME = 1 day` with refresh rotation off is long. Shorter access (5–15 min) plus rotating refresh tokens, stored in an httpOnly cookie rather than localStorage.

## Interview Q&A

**Q: How did SSO work across products?**
A: Django is the identity hub. On login it authenticates the user, works out every tenant they can reach (owned plus associated), gets a tenant-scoped JWT from each tenant's gateway, and returns them all. The React app keeps a token per tenant and switches between tenants without logging in again.

**Q: How would you design it properly from scratch?**
A: A single IdP (OIDC: Keycloak/Auth0/Cognito). The user logs in once and gets an ID token plus an access token with a `tenants` claim (or requests a tenant-scoped token through token exchange, RFC 8693). Every service validates JWTs with the IdP's public keys (JWKS). Authorization per tenant comes from roles in the token or a central policy service.

**Q: Where do you store tokens in the browser?**
A: Ideally the refresh token in an httpOnly, Secure, SameSite cookie (JS can't read it, so XSS can't steal it) and the access token in memory. localStorage is easy but exposed to XSS.

**Q: Tell me about a tricky auth bug.**
A: (The union-of-tenants story above, in STAR form.)
