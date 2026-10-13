---
date: 2026-05-05
updated: 2026-09-24
source: Codex (VS Code), updated by Claude Code
project: both
tags: [auth, jwt, bearer, keycloak, oidc, authorization-code, architecture, debugging-401, mobile]
---

# How authorization actually flows (Core Web → Core API)

> **Takeaway:** Core Web signs users in with the OIDC **Authorization Code** flow ("Standard flow" in Keycloak). Only a one-time **code** comes back through the browser. Core Web's **server** exchanges it, with the client secret, for the ID, access and refresh tokens. The tokens ride in the encrypted sign-in cookie, are loaded into `TokenStore` per Blazor session, and the **access token alone** is forwarded to the API as `Bearer`. None of this is custom code: the ASP.NET Core OpenID Connect handler builds the Keycloak redirect and does the exchange. Our code only configures it, and can hook the outgoing redirect in `OnRedirectToIdentityProvider`. This redirect-based flow is also why SSO works on the web but not on the Android app, which uses Direct access grants (password posted straight to Keycloak).

## The login flow, step by step (without SSO)

1. **Request a page.** Every page requires an authenticated user (core-web `Startup.cs:151`). No cookie means ASP.NET issues a **challenge**.
2. **Redirect to Keycloak.** The OIDC handler sends a 302 to Keycloak's authorization endpoint, roughly:
   `…/realms/corelocal/protocol/openid-connect/auth?client_id=web&response_type=code&scope=openid roles&redirect_uri=https://localhost:5005/signin-oidc&state=…&nonce=…`
   `redirect_uri` is the handler's default callback path `/signin-oidc`, and must be listed in the `web` client's **Valid redirect URIs** in Keycloak.
3. **User signs in on Keycloak's page.** Core Web never sees the password. Keycloak also sets its **own** session cookie, which is why a second sign-in can succeed without re-entering credentials.
4. **Keycloak redirects back to `/signin-oidc?code=…&state=…`.** Only a short-lived, single-use **code** crosses the browser, never a token.
5. **Server-side code exchange.** Core Web's server calls Keycloak's token endpoint directly with the code, `client_id=web` and the **client secret** (`Startup.cs:259`, `ResponseType = Code` `:260`). It receives an **ID token** (who the user is), an **access token** (what the API accepts) and a **refresh token** (to renew the access token). The secret makes an intercepted code useless on its own; this is why the web client runs `UsePkce = false` (`:261`).
6. **User details.** `GetClaimsFromUserInfoEndpoint = true` (`:264`) fetches claims from Keycloak's userinfo endpoint. `OnUserInformationReceived` (`:289-298`) creates the per-user app state.
7. **Core Web's own sign-in.** `SignInScheme = "Cookies"` (`:256`). With `SaveTokens = true` (`:262`) the three tokens are stored **inside the sign-in cookie, encrypted**. The browser carries them but cannot read or use them. Cookie: 60 min, sliding (`:251-252`).
8. **Back to the original page**, now with the cookie.
9. **Tokens into Blazor.** `_Host.cshtml:12-13` reads the access and refresh tokens from the cookie. `App.razor:33-34` puts them into `TokenStore`, held in server memory for the Blazor session.
10. **Calls to the API.** `ApiService` attaches `TokenStore.AccessToken` as `Authorization: Bearer` (`Repositories/ApiService.cs:47`). The call is server-to-server (Core Web → Core API); `HttpContext.User` is **not** passed.
11. **Renewal.** When the access token expires, `/BTokenRefresh` uses the refresh token. Because the Blazor session holds the token in memory, this needs a full page load (`forceLoad`).
12. **Sign-out.** The cookie is cleared and the browser is sent to Keycloak's logout endpoint, ending the Keycloak session too.

## Who actually executes the redirect (step 2)

No custom code builds or sends it: ASP.NET Core's **OpenID Connect handler** does (`OpenIdConnectHandler.HandleChallengeAsync`, registered as `"oidc"` by `.AddOpenIdConnect("oidc", …)`, `Startup.cs:255`).

- **What triggers it.** Originally it had two triggers:
  - Automatically: `DefaultChallengeScheme = "oidc"` (at HEAD before the SSO work) meant any unauthenticated request went straight to the handler.
  - Explicitly: the original `/Login` page called `Challenge(new AuthenticationProperties { RedirectUri = returnUrl }, "oidc")`.
- **What the handler does.**
  1. Reads `{Authority}/.well-known/openid-configuration` to find the login URL. Authority is set at `Startup.cs:257`.
  2. Builds the query from our settings: `client_id` (`:258`), `response_type` (`:260`), scopes (`:267-269`), `redirect_uri`, `state`, `nonce`.
  3. Raises **`OnRedirectToIdentityProvider`**, the only hook on the outgoing redirect (`:272-287`).
  4. Sends the 302.

## What the SSO work changed (and what it didn't)

- The handler, the code exchange and everything from step 4 onward are **unchanged**.
- **What changed is the trigger.** `DefaultChallengeScheme` is now `Cookies` with `LoginPath = "/Signin"` (`Startup.cs:245`, `:253`), so unauthenticated users first land on the email page. `Signin.cshtml.cs` then calls `Challenge(properties, "oidc")`, the same call the old `/Login` made.
- `OnRedirectToIdentityProvider` now adds `kc_idp_hint`, which sends SSO users straight to their company's IdP, and `login_hint`, which pre-fills the email (`:272-287`).
- `DefaultSignOutScheme = "oidc"` is pinned (`:248`). It would otherwise follow the challenge scheme, and sign-out would stop ending the Keycloak session.

## API side: validation and client check

- The API validates the bearer JWT against Keycloak: `Authority = OpenIdConfiguration:Authority`, **`Audience = "api"`** (core-web-api `Startup.cs:261-262`). `app.UseAuthentication()` runs this **before any controller**. That's why the Keycloak **audience mapper** is required ([[2026-05-04 Keycloak SSO and Audience Mapper]]).
- `FullAccess` is a **client** check, not a user check: the token's `client_id` must be `web` or `app` (core-web-api `Startup.cs:277`). A new Keycloak client, e.g. a future mobile SSO client, must be added there or the API rejects its tokens.
- Role and site checks follow: `Administrator` role, then `SiteAuthorizationHandler` for per-site access (`site_id` / `drill_plan_site_id`).

## Why the Android app is different

- Keycloak client `app` has **Direct access grants** only (confirmed in the Keycloak admin console, 2026-09-24). The legacy IdentityServer config shows the same design: `AllowedGrantTypes = GrantTypes.ResourceOwnerPassword` (core-web-api `Config.cs:131-133`).
- With Direct access grants, the app collects username + password itself and posts them to Keycloak's token endpoint. There's **no browser redirect**, so Keycloak can never hand the user to Entra: **no SSO**, and an SSO user (no password) can't sign in at all.
- **The rule of thumb:** SSO needs a redirect-based flow, and a redirect-based flow needs a browser. The web app is in a browser *and* uses Standard flow. The app could open a browser but doesn't.
- **Getting SSO on the app is an Android code change**, not a Keycloak switch. Ticking Standard flow only *allows* it. The app itself must open the system browser (e.g. AppAuth), handle the redirect back, and exchange the code.

| | Web (`web` client) | Future mobile Standard-flow client |
|---|---|---|
| Redirect URI | `https://localhost:5005/signin-oidc` | custom scheme, e.g. `com.minnovare.optimiser:/oauth2redirect` |
| Who exchanges the code | Core Web **server** | the **app itself** |
| Proves itself with | client **secret** | **PKCE** (S256); an app can't keep a secret, so it should be a **public** client |
| Token storage | encrypted cookie + `TokenStore` in server memory | app secure storage; `offline_access` refresh token for offline rigs |

Other Keycloak flows for the app: **Device Authorization Grant** also works with SSO (sign in on a second device), which suits shared rig tablets. **Implicit** is deprecated and gives no refresh token. **CIBA** and **Service accounts** don't apply.

## Debugging playbook for 401s
1. A 401 means failure **before** the controller, so breakpoints in `SitesController.Get()` will never hit.
2. Break at `ApiService.cs` (~line 49) before send and inspect `request.Headers.Authorization`. Null means Blazor isn't forwarding the token.
3. Inspect what the API sees: **`GET /Identity`** (`IdentityController`) returns all claims from the bearer token. Perfect for comparing expectation vs reality.
4. Decode the token: if `"aud": "account"` but the API requires `Audience = "api"`, it's rejected regardless of roles. Fix with the Keycloak audience mapper (see [[2026-05-04 Keycloak SSO and Audience Mapper]]).

## Questions asked
- 2026-05-05: why does the API return 401 when the web is signed in? → Audience mapper; see the playbook.
- 2026-09-24: "Does Keycloak redirect to localhost:5005 and forward only the access token?" → Redirect URI yes (`/signin-oidc`); what comes back is a one-time **code**, exchanged server-side for three tokens. See *The login flow*.
- 2026-09-24: "Which code executes the redirect to Keycloak?" → ASP.NET Core's OIDC handler; our code only triggers/configures it. See *Who actually executes the redirect*.
- 2026-09-24: "Is web SSO possible just because web can open a browser? Does the app need code changes or only Keycloak config?" → A redirect-based flow is what matters; the app needs code changes. See *Why the Android app is different*.

## Related
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]]: living doc for the auth/authz chain and SSO brokering
- [[2026-05-04 Keycloak SSO and Audience Mapper]]
- [[2026-04-29 Keycloak GetUsersForSite 403 Debugging]]
