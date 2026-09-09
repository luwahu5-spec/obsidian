---
date: 2026-08-27
updated: 2026-08-27
source: Claude Code
project: core-web
tags: [domain-knowledge, auth, blazor-server, token-refresh, oidc, signalr]
---

# BTokenRefresh — what the feature actually does

> **Takeaway:** A Blazor Server circuit captures the access token **once**, when the circuit starts, and holds it in memory for the life of the SignalR connection. It cannot refresh that token itself, because the auth cookie holding the refresh token is only reachable during a real HTTP request. `BTokenRefresh` is the escape hatch: a plain Razor Page that a component force-navigates to when the API returns 401, so a genuine HTTP round trip can refresh the tokens, rewrite the cookie, and bounce the user back — starting a fresh circuit with a fresh token. That is why so many components reference it: every API-calling component needs the same escape.

## The physical reality being modelled
Blazor Server splits a user's session across two very different channels:

| Channel | Can read/write the auth cookie? | Lives how long? |
|---|---|---|
| The initial HTTP page request (`_Host.cshtml`) | **Yes** — `HttpContext` exists | one request |
| The SignalR circuit (everything after) | **No** — no HttpContext, no Set-Cookie | minutes to hours |

The access token expires on Keycloak's schedule (short — minutes), while the circuit can live far longer. So the circuit inevitably ends up holding a token the API no longer accepts. Refreshing requires writing a new cookie, which requires an HTTP response — something the circuit does not have. **The only way out is to leave the circuit, do an HTTP round trip, and come back.**

## How the token gets into the circuit in the first place
1. `_Host.cshtml:12-13` — during the page request, reads `access_token` / `refresh_token` out of the auth cookie via `HttpContext.GetTokenAsync(...)`.
2. `_Host.cshtml:37` — passes them into the root component: `param-InitialAccessToken` / `param-InitialRefreshToken`.
3. `App.razor:29-36` — builds a `TokenStore` from those values.
4. `Repositories/ApiService.cs:47` (and every other call site) — `_httpClient.SetBearerToken(TokenStore.AccessToken)` on every API call.

That `TokenStore` is a snapshot. Nothing inside the circuit can update it from the cookie.

## The refresh cycle, end to end
1. **Detection** — an API call returns 401 → `UnauthorizedException`.
2. **Escape** — `Shared/Utils.cs:29` / `:52` (`RunOrRefresh`, the shared wrapper) catches it and calls
   `navManager.NavigateTo("/BTokenRefresh?returnUrl=" + navManager.Uri, true)` (`Utils.cs:39`).
   The `true` is **forceLoad** — a real browser navigation, not Blazor routing. Without it the circuit would never be left and nothing would be fixed.
3. **Refresh** — `Pages/BTokenRefresh.cshtml.cs:21-74` now runs as a Razor Page *with* an `HttpContext`:
   - `HttpContext.AuthenticateAsync()` (`:29`) → reads `expires_at` from the auth properties (`:30`).
   - If expired (`:37`), takes `refresh_token` (`:39`) and calls `ApiService.RefreshAccessTokenAsync` (`:43`, implemented at `Repositories/ApiService.cs:581`) — discovery document → `RequestRefreshTokenAsync` against Keycloak's token endpoint.
   - Stores the new tokens back (`:44`) and calls `HttpContext.SignInAsync` (`:48`) — **this is the step that rewrites the cookie**, and it is only possible here.
   - Redirects to `returnUrl` (`:50`, `:76-86`).
   - On failure: `SignOutAsync()` + redirect home (`:55-56`).
4. **Fresh circuit** — the redirect is a new page load, so `_Host.cshtml` reads the *new* cookie, `App.razor` builds a *new* `TokenStore`, and the user resumes with a valid token.

The page itself is nearly empty — `BTokenRefresh.cshtml` renders one heading (`:6`). Users see it as a brief flash.

### Worked example
Access token lifetime 5 min, auth cookie 60 min sliding (`Startup.cs:244-245`). User opens Plan Data at 09:00; the circuit holds a token minted 09:00, expiring 09:05. At 09:07 they click Save → API 401 → `RunOrRefresh` force-navigates to `/BTokenRefresh?returnUrl=…/PlanData` → the cookie (valid until 10:00) still carries a usable refresh token → new access token → cookie rewritten → redirect back → new circuit, valid token, Save works. Total user experience: one page flash.

## Why it appears in so many places
Every component that calls the API has to handle the same 401. Two patterns coexist:
- **Shared wrapper (preferred)** — `Utils.RunOrRefresh` (`Shared/Utils.cs:29`, `:52`; also `:143`, `:285`).
- **Direct navigation (older/local)** — `Pages/Blazor/Rigs/ShiftList.razor.cs:136` & `:346`, `Pages/Blazor/Rigs/ComplianceStateView.razor.cs:117` & `:227`, `Shared/UserTimeService.cs:140`.

So the repetition is not duplicated logic; it is many call sites reaching the same escape hatch.

## Interaction with the identity-first sign-in change (2026-08-27)
The `/Signin` work changed `DefaultChallengeScheme` to Cookies with `LoginPath = "/Signin"` (`Startup.cs`). Effect on this feature:

- **Normal case — unaffected.** `BTokenRefresh` requires auth (it is *not* `[AllowAnonymous]`), but in the case it exists for, the cookie is still valid and only the *access token* has expired. No challenge occurs; the page runs exactly as before.
- **Expired-cookie case — destination changed, by design.** If the cookie itself has lapsed, the user previously got the Keycloak login page; now they get `/Signin`. That is the intended new front door, not a regression.
- **One line was genuinely at risk and is now protected.** `BTokenRefresh.cshtml.cs:55` calls `HttpContext.SignOutAsync()` **with no scheme**, so it resolves via `DefaultSignOutScheme` — which silently falls back to `DefaultChallengeScheme`. Switching that to Cookies would have made a failed refresh clear only the local cookie while leaving the Keycloak session alive (user "signed out", then logged straight back in). Fixed by pinning `options.DefaultSignOutScheme = "oidc"` explicitly. `Logout.cshtml.cs:49-51` names both schemes and was never at risk.

## Design subtleties & edge cases
- **`forceLoad: true` is load-bearing.** Ordinary Blazor navigation stays inside the circuit and would refresh nothing.
- **Only refreshes when actually expired** — if `expires_at` is in the future the page just redirects back (`:59-62`), so a 401 from a *different* cause (e.g. wrong `aud`) produces a redirect loop rather than a fix. That is the documented `aud: api` symptom in [[2026-05-04 Keycloak SSO and Audience Mapper]].
- **Failure path does two redirects** — `SignOutAsync()` (RP-initiated, itself a redirect) immediately followed by `DoRedirect(null)` (`:55-56`). Pre-existing; worth knowing when reading logs.
- **SSO users** hitting a failed refresh land on `/Signin`, re-enter their email, are routed to their IdP and silently re-linked — no password involved.

## Questions asked
- 2026-08-27 — "What does BTokenRefresh do, lots of features reference it — and will the new identity-first procedure affect it?" → It is the escape hatch from the Blazor circuit for token refresh (see *The refresh cycle*); the many references are call sites, not duplicated logic. The new procedure leaves it working; the one at-risk line (`SignOutAsync()` with no scheme) is protected by pinning `DefaultSignOutScheme` — see *Interaction with the identity-first sign-in change*.

## Related
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — the wider auth chain this sits inside
- [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] — the `/Signin` change and the `DefaultSignOutScheme` trap
- [[2026-05-04 Keycloak SSO and Audience Mapper]] — the `aud: api` failure that manifests as a `/BTokenRefresh` loop
- [[2026-06-12 When the Blazor Error Banner Appears]] — the other circuit-level failure surface
