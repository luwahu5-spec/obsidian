---
date: 2026-05-04
source: Codex (VS Code)
project: core-web
tags: [keycloak, sso, oidc, google-idp, audience-mapper, jwt, auth]
---

# Keycloak SSO — Enterprise IdP Brokering & the `aud` Claim Fix

## Context
Core already logs in via Minnovare/Hexagon Keycloak (OIDC). What "complete SSO functionality" means for Product: **true enterprise/customer SSO** — customers log in with *their own IdP* (Azure AD/Entra, Okta, Ping, ADFS), with Keycloak acting as a **broker**. Extra work: realm config, identity provider mapping, domains, claims, roles, groups, provisioning, tenant rules.

## Setup learned (Google as test IdP)
- Provider type for Google: **OpenID Connect v1.0** (used the discovery endpoint = a well-known URL that auto-fills all OIDC endpoints).
- Local test realm `localhost`, clients `web` + `api`, test flow: login page shows Google option → Google login → redirected back into Core.

## The token/audience problem (the big lesson)
After SSO login, Core Web kept looping through `/BTokenRefresh`. Chain:
```
SitesList.razor.cs → SiteRepository → ApiService.GetAsync → Core API 401
→ UnauthorizedException → Utils.cs navigates /BTokenRefresh → rethrow → circuit refresh loop
```
Cause: access token had `"aud": "account"` but **Core API requires `aud: api`** — API rejects the token before any controller runs (breakpoints never hit).

### Fix — Keycloak Audience mapper
Common mistake: setting **Included Client Audience = web** adds `aud: web`, not `api`. Correct mapper:
```
Clients → web → Client scopes → web-dedicated → Mappers → add Audience mapper
Mapper type: Audience
Included Client Audience: (empty)
Included Custom Audience: api
Add to access token: On
```
Expected result: `"aud": ["account", "api"]`.

### Debugging tool
`Clients → web → Client scopes → Evaluate` → pick test user → **Generate access token** — inspect the token Keycloak would actually issue, instead of guessing from live sessions (stale tokens mislead).

## Note (Blazor Server)
API requests come from the **Core Web server process**, not the browser — remember this when reasoning about where auth headers originate.

## Related
- [[2026-04-29 Keycloak GetUsersForSite 403 Debugging]]
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — full auth chain + SSO rollout plan built on this PoC
