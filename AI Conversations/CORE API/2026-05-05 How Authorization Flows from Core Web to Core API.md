---
date: 2026-05-05
source: Codex (VS Code)
project: core-web-api
tags: [auth, jwt, bearer, keycloak, architecture, debugging-401]
---

# How authorization actually flows (Core Web → Core API)

## The architecture (confirmed from code)
- Core Web does **not** pass `HttpContext.User` to Core API — it forwards only the **access token** as `Authorization: Bearer <token>`.
- Backend: `Startup.cs` — `services.AddAuthentication(...).AddJwtBearer(options => { Authority = OpenIdConfiguration:Authority; **Audience = "api"**; })` (~line 242) + `app.UseAuthentication()` (~line 372). The JWT middleware validates the token **before any controller runs**.
- Role/site checks: `SitesController` role branching + custom `SiteAuthorizationHandler` for per-site access.
- Frontend token capture: OIDC events call `context.GetTokenAsync("access_token"/"refresh_token")` → stored in `TokenStore.AccessToken/RefreshToken` (requires `options.SaveTokens = true`).

## Debugging playbook for 401s
1. A 401 means failure **before** the controller — breakpoints in `SitesController.Get()` will never hit.
2. Break at `ApiService.cs` (~line 49) before send: inspect `request.Headers.Authorization` — null means Blazor isn't forwarding the token.
3. Inspect what the API sees: **`GET /Identity`** (`IdentityController`) returns all claims from the bearer token — perfect for comparing expectation vs reality.
4. Decode the token: if `"aud": "account"` but API requires `Audience = "api"` → rejected regardless of roles. Fix with Keycloak audience mapper (see [[2026-05-04 Keycloak SSO and Audience Mapper]]).

## Related
- [[2026-04-29 Keycloak GetUsersForSite 403 Debugging]]
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — living doc covering the full auth/authz chain and SSO IdP brokering
