---
date: 2026-10-20
updated: 2026-10-20
source: Claude Cowork
project: core-web, core-web-api
tags: [sso, sign-in, security, risk-review, keycloak, open-redirect, rate-limiting]
---

# Identity-first sign-in: risk review and decisions (2026-10-20)

> **Takeaway:** Review of the new sign-in flow (core-web `MC-1814` / core-web-api `MC-1880`) after the main logic and the middleware fix were done. One defect was fixed straight away (the lookup dropped `/v1` from the API URL). Of the risks found: rate limiting goes to AWS (to discuss with the lead), auto-created Keycloak users and the same-tenant `oid` gap are accepted for now, the Keycloak post-logout redirect URIs are being added per realm, and the `BTokenRefresh` open redirect (pre-existing since 2020) is deferred to [[Defect]] D-001 with its fix stashed.

## Fixed during the review

- **Lookup URL dropped `/v1`.** `SignInLookupService` set `BaseAddress` to `CoreAPI:BaseUrl` (`https://api.core.<env>.minnovare.com/v1`) and posted to `/SignIn/Lookup`. A leading `/` replaces the base path, so the call went to `https://api.core.<env>.minnovare.com/SignIn/Lookup`. Locally the base URL has no path, so it only showed up on deployed environments. Now builds `<base>/SignIn/Lookup` (`SignInLookupService.LookupUri`), and fails closed if the setting is missing or not http/https.
- `OnRedirectToIdentityProvider` moved from an inline lambda in `Startup.cs` to `SignInRedirectHints.cs` so it can be unit-tested. No behaviour change.
- Unit tests added: `core-web/Minnovare.Core.Web.Test/SignIn/` (53 tests: lookup service, `/Signin` page, account-mismatch handler, redirect hints, chained workflow). Not yet run.

## Decisions

| # | Risk | Decision |
|---|---|---|
| 1 | Anonymous `POST /SignIn/Lookup` has no rate limit (in-app limiter removed 2026-10-07). Allows enumeration: does an email have an account, is it SSO, which customer (IdP alias). The `u.Email.ToUpper()` fallback can't use an index, so each call is a scan. | **Not in code.** Rate limit in AWS (WAF rate-based rule or equivalent). Li to discuss with the lead. Must be in place before SSO goes to production. See [[2026-10-06 Sign-in Lookup Rate Limiting and Forwarded Headers Feature Explained]] for what AWS needs. |
| 2 | Open redirect in `BTokenRefresh` (`returnUrl` not validated). | **Deferred 2026-10-21**: exists since 2020, out of sprint scope. Fix kept in a local git stash; logged as D-001 in [[Defect]]. |
| 3 | Tokens without a `KcUserMapping` row skip the tenant middleware (needed for machine clients). | **Accepted.** The only way an Entra user gets a Keycloak account without a mapping is Keycloak auto-create, and every provider's first-login flow is link-or-reject (no auto-create). Depends on the per-provider setting in item 6. |
| 4 | Same-tenant gap: `entra_oid` not enforced, `ExpectedOid` captured on first sign-in. | **Accepted, documented.** Revisit once SSO is in use. |
| 5 | A cross-tenant sign-in gets a CORE web session; only the API refuses it (403s). | Accepted (safe, confusing UX). |
| 6 | Keycloak config the code can't check. | **Being added per realm.** Checklist below. |
| 7 | API reads JSON with `TypeNameHandling.Objects`, now on an anonymous endpoint. Not exploitable while the request is a single string. | Low. Don't add object-typed properties to `SignInLookupRequest`. |
| 8 | Logout works on GET (another site can sign a user out). | Low, pre-existing. |
| 9 | `UsersController.cs` ~line 251 uses literal `"entra_oid"` instead of `MinClaimTypes.Oid`. | Tidy-up. |

## Item 6: Keycloak settings per realm

**Client `web` → Valid post logout redirect URIs.** When Keycloak signs in a different account than the one typed on /Signin, CORE ends that Keycloak session and returns to `/Signin?reason=account-mismatch`. Keycloak only allows that return if the address is registered on the client. Add the address with a trailing `*` so the `?reason=...` query is covered. If the field is `+` and the Valid redirect URIs already contain `https://<host>/*`, it's already covered.

| Realm (server `https://cloak.minnovare.com`) | CORE web host | Add |
|---|---|---|
| `corelocal` | `https://localhost:5005` | `https://localhost:5005/Signin*` |
| `coredev` | `https://core.dev.minnovare.com` | `https://core.dev.minnovare.com/Signin*` |
| `coretest` | `https://core.test.minnovare.com` | `https://core.test.minnovare.com/Signin*` |
| `corestaging` | `https://core.staging.minnovare.com` | `https://core.staging.minnovare.com/Signin*` |
| `core` (prod, `https://auth.minnovare.com`) | to confirm (API config says `https://production.core.minnovare.com`) | same pattern |

Hosts taken from core-web `appsettings.*.json` (`OpenIdConfiguration.Authority`) and core-web-api `appsettings.*.json` (`Core:BaseUrl`). Test: sign in to CORE as user A, then sign out of CORE only (or open a second tab), go to /Signin, type user B's email. Expected: you land back on /Signin with "You were still signed in as a different account". A Keycloak "Invalid redirect uri" page means the address isn't registered.

**Every customer identity provider, every realm:** first login flow = link-or-reject flow (item 3 depends on this), `entra_tid`/`entra_oid` Attribute Importer mappers with sync mode **Force**, Hide on login page = On. **Realm:** reset-credentials flow blocks SSO users (`ssoResetUnavailable`).

## Item 2: BTokenRefresh open redirect

**Where:** `core-web/Minnovare.Core.Web/Pages/BTokenRefresh.cshtml.cs`, `DoRedirect(returnUrl)` → `Response.Redirect(returnUrl, false)`. Every branch for a signed-in user ends there: token still valid, refreshed, or unreadable expiry. Nothing checks where `returnUrl` points.

**Attack:** `https://core.dev.minnovare.com/BTokenRefresh?returnUrl=https://evil.example/core-login`. A signed-in user who clicks it is sent straight to the attacker's page. The link starts with the real CORE domain, so it survives mail filters and a quick look at the address. The attacker's page can then pose as "session expired, sign in again" and collect the password or Microsoft credentials. `//evil.example` and `/\evil.example` work too. It needs the victim to be signed in (cookie lasts 60 minutes, sliding). No token or cookie leaves CORE, so this is phishing, not session theft (CWE-601).

**Why the usual fix needs care:** the legitimate caller passes an absolute URL. `Utils.RunOrRefresh` does `NavigateTo("/BTokenRefresh?returnUrl=" + navManager.Uri, true)`, and `NavigationManager.Uri` is `https://core.dev.minnovare.com/Sites/12/...`. `Url.IsLocalUrl` alone would reject every real refresh and send users home. Allow relative local URLs **and** absolute URLs on the request's own scheme and host; send anything else to `/`.

**Two small related bugs in the same page:**
- The not-signed-in branch calls `RedirectToPage("./account/accessdenied")` and throws the result away, so an anonymous visitor gets a blank page.
- `navManager.Uri` isn't URL-encoded, so a page URL with its own `&` query values loses them on the way back. Use `Uri.EscapeDataString`.

**Correction:** the anonymous branch was never reached in practice. The global authorize filter sends anonymous visitors to /Signin first. That also means the attack works on signed-out users: /Signin keeps `/BTokenRefresh?returnUrl=…` as its (local) return URL, so after signing in they are bounced through to the attacker.

### Fix (written 2026-10-20, stashed 2026-10-21, not applied; see [[Defect]] D-001)

- `BTokenRefreshModel.SafeReturnUrl(returnUrl, Request)`: keeps a local path (`/x`, `~/x`, not `//x` or `/\x`). Reduces an absolute http(s) address on the request's own host (port included, no user-info) to its path, query and fragment. Everything else goes to `/`. The scheme is deliberately not compared, so a proxy ending TLS in front of the app can't send every refresh home.
- `OnGetAsync` now returns `IActionResult`. The anonymous branch redirects to `/Signin` instead of dropping the result. A failed refresh still signs out and goes home.
- `BTokenRefreshModel.UrlFor(currentUri)` builds `/BTokenRefresh?returnUrl=<escaped>`. All 10 callers use it now: `Utils.cs` ×4, `UserTimeService.cs`, `ComplianceStateView.razor.cs` ×2, `ShiftList.razor.cs` ×2, `AddUser.razor.cs`.
- Tests: `Minnovare.Core.Web.Test/SignIn/BTokenRefreshReturnUrlTests.cs`.
