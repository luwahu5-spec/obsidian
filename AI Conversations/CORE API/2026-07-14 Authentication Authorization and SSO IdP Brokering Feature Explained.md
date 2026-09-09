---
date: 2026-07-14
updated: 2026-08-31
source: Claude Code
project: both
tags: [domain-knowledge, auth, keycloak, oidc, sso, idp-brokering, site-authorization, user-management]
---

# Authentication, Authorization & SSO IdP Brokering — what the feature actually does

> **Takeaway:** Authentication is fully delegated to Keycloak (OIDC code flow); the app never sees passwords. Authorization is claim-driven: `site_id` / `drill_plan_site_id` claims decide site access, `Administrator` role bypasses everything. The non-obvious constraint: **user management is a dual-store design** (ASP.NET Identity DB + Keycloak, joined by `KcUserMapping`) — SSO breaks nothing in authorization, but an SSO user auto-created by Keycloak would have no local row, no mapping, no site claims. Hence the link-or-reject first-login flow (Step 3), verified end-to-end with Google and Entra ID.

## The business reality
One CORE deployment serves many sites owned by different customers. Site admins see their site(s); `Administrator` sees all; drill-plan users touch drill plans for one site only. "Enterprise SSO" = a customer's employees log in with the customer's own IdP; the customer controls onboarding/offboarding at their end.

## The authentication chain
1. Every page requires login — global `RequireAuthenticatedUser()` filter (core-web `Startup.cs:139-145`); unauthenticated → `oidc` challenge (`:234`) → redirect to Keycloak.
2. OIDC client (core-web `Startup.cs:240-267`): Authority `https://auth.minnovare.com/realms/core` (`appsettings.json:12`), client `web` + secret, code flow, `SaveTokens = true`, scopes `openid roles`. Session = 60-min sliding cookie (`:235-239`).
3. Blazor Server: API calls originate from the web SERVER; token lives in `TokenStore`, attached via `SetBearerToken` (`Repositories/ApiService.cs:47` et al.); refresh via `/BTokenRefresh`. The browser never holds the bearer token.
4. API validates plain JWT bearer with **`Audience = "api"`** (core-web-api `Startup.cs:239-252`) — this is why the Keycloak **audience mapper** (`aud: api` on client `web`) is mandatory ([[2026-05-04 Keycloak SSO and Audience Mapper]]); without it every request 401s before any controller and the web loops `/BTokenRefresh`.

## The authorization chain
- `ClaimsTransformer` (core-web-api `Services/ClaimsTransformer.cs:20-41`): flattens `realm_access.roles` → .NET role claims; `azp` → `client_id`.
- Client-level policies (`Startup.cs:257-275`) gate which APPLICATION may call (`FullAccess` = App/Web; Aegis/Deswik/LiveMine/Newtrax get narrower ones). Machine clients use client credentials — SSO work never touches them.
- Site-level: `SiteAuthorizationHandler` (`Shared/Security/SiteAuthorizationHandler.cs:13-39`) — `Administrator` allows all; otherwise token must carry `site_id`/`drill_plan_site_id` == siteId (`MinClaimTypes.cs:6-9`), invoked via `SiteAccessAuthorizationService`.
- Site claims are **Keycloak user attributes** exposed by protocol mappers (KC config, not code). The only KC realm role in use is `Administrator` (`KcClient.cs:335-353`).

## User management — the dual-store design
| Store | Holds | Written by |
|---|---|---|
| ASP.NET Identity DB | profile, local roles, local claims copy | `UserManager` |
| Keycloak realm | credentials, `site_id` attributes, realm roles | `KcClient` (admin REST) |
| `KcUserMapping` table | KC GUID ↔ Identity GUID join | `UsersController` on create |

- Create (`UsersController.cs:328-370`, site variant `:423-521`): Identity `CreateAsync` → `KcClient.MigrateUser` (`KcClient.cs:241-366`; password credential `:280-288`, site claims as attributes `:297-315`) → `KcUserMapping` insert. Password `[Required]` (`UsersController.cs:31`); null → generated + `UPDATE_PASSWORD` forced (`KcClient.cs:292-295`).
- Claims/roles written to BOTH stores (`UsersController.cs:769-812`, `:649-713`); drift possible — `GetUsersForSite` trusts KC attributes (`:118-133`), `LoadUsersAsync` reads the local copy (`:157`).
- A KC user with no local row = corruption path "This should not be possible" (`UsersController.cs:204,221`) — exactly what an auto-provisioned SSO user would be. Such orphans fail SOFT: they log in but see "no permission to view sites" (no site claims).

## Configuring SSO IdPs — verified recipe (Google + Entra ID, 2026-07-14/17)
Keycloak = identity broker; CORE talks only to Keycloak; token still issued by the same realm/client → audience mapper and all authorization apply unchanged. Zero CORE code change for login.

### Step 1 — Entra ID side (the customer's identity admin)
1. Test tenant if needed: **Azure free signup** creates a Default Directory with you as Global Admin (a bare personal account has no directory and can't register apps).
2. **App registration**: single tenant; Web redirect URI = `https://<kc-host>/realms/<realm>/broker/<alias>/endpoint`.
3. **New client secret** → copy the **Value** immediately (shown once; the "Secret ID" GUID is not the secret).
4. Note **Application (client) ID** + **Directory (tenant) ID**.
5. Test users must be **native tenant users** (not the tenant-creator `#EXT#` MSA — unreliable claims). On EVERY user set **Properties → Contact information → Email** (primary field, not "other emails") = exactly the CORE user's email — this feeds the `email` claim; bare test tenants never auto-populate it (twice-confirmed failure cause), corporate tenants do via Exchange/HR. Put "users must have `mail` populated" on the customer onboarding checklist (FMI §4.1).

→ Items 2–4 are the **customer onboarding checklist**: tenant ID + client ID/secret from them, redirect URI from us.
   - **The tenant ID is NOT a secret** — derive it yourself from the customer's email domain via `https://login.microsoftonline.com/<domain>/v2.0/.well-known/openid-configuration` (the `issuer` field carries the GUID), then confirm it back to them. Removes transcription errors (a mistyped GUID locks out every user with a confusing tid mismatch) and removes a trust step. Ask only for the **domain(s)**, which routing needs anyway.
   - Only the **client secret** needs a secure channel (never email) — and only in the per-customer model; a shared multi-tenant app takes no secret from the customer at all.
   - Several domains → one tenant is normal; several tenants per customer (post-acquisition) is why `CustomerIdentityProvider` is a separate table — multiple rows, no schema change. Also collect **named identity-admin contacts** (needed for the re-bind procedure). The whole exchange fits FMI's existing intake (§5.1 + Appendix A questionnaire).

### Step 2 — Keycloak provider
1. **Add provider → OpenID Connect v1.0** — never the built-in "Microsoft" social provider (consumer endpoints; single-tenant apps fail with `unauthorized_client: not enabled for consumers`).
2. **Alias = customer identifier** (e.g. `freeport`; `microsoft` was PoC-only). Alias is baked into the customer's Azure redirect URI and any `kc_idp_hint`/Organizations routing — permanent, lowercase, no spaces. **Display name** = login-button label, freely renameable.
3. **Discovery endpoint** = `https://login.microsoftonline.com/<tenant-GUID>/v2.0/.well-known/openid-configuration` — never `common`/`consumers`.
4. Client ID + secret Value.
5. **Scopes = `openid profile email`** (space-separated; field is in the "Advanced" group under OpenID Connect settings, NOT "Advanced settings"). Without them the token carries only opaque `sub` → linking can never match.
6. **Trust Email = On**; **First login flow = Step 3 flow** — a fresh provider silently defaults to `first broker login` (auto-create returns!).
7. **Verify essential claim** (KC 22+ only — **our KC 20 lacks it**): rejects tokens missing claim/value (e.g. `tid` = tenant GUID) in the callback. KC 20 equivalents: single-tenant registration already locks the tenant; Entra **"Assignment required = Yes"** on the Enterprise app gates by user/group before token issuance (FMI §5.2 standard practice). NB: KC 20 also lacks Organizations (KC 26+) and is out of security support — KC upgrade belongs under PRD FR-A 14 (ECS image bump, DB migrates on startup).
8. **tid/oid hardening (verified, works on KC 20)** — the chain that carries tenant + person identity from Entra into CORE:
   1. Entra issues the ID token: `tid` = tenant GUID, `oid` = user's object id. **Both are stamped by Microsoft and cannot be set by a tenant admin** (unlike `email`/`mail`, where only the UPN domain is verified).
   2. **IdP Mappers** (provider → Mappers → Add), type **Attribute Importer**, **Sync mode override = Force**: `tid`→`entra_tid`, `oid`→`entra_oid`. Force matters because plain Import only fires for broker-CREATED users — ours are pre-created and *linked*, so Import may never apply; Force also re-imports on every login.
   3. **User Attribute protocol mappers** on the `web` dedicated client scope (add to access token) — same mechanism as `site_id`.
   4. CORE reads `entra_tid` / `entra_oid` from the bearer token.
   5. CORE compares them to the values stored on the user at pre-creation → allow, or reject + audit.
   **`ExpectedTid` always comes from customer configuration at user creation — never trust-on-first-use.** TOFU is acceptable for `oid` only (it guards email recycling *within* an already-verified tenant). This distinction becomes critical in any shared multi-tenant setup, where `tid` is the sole customer identifier ([[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] T10). Enforce for SSO-type users only — local logins carry no such claims. KC linking itself stays email-based; the tid/oid check is a CORE-side second factor after linking. `oid` doubles as the future SCIM `externalId`.
9. **Hide on login page** (Advanced settings): removes the provider's button WITHOUT disabling it — `kc_idp_hint` still routes to it. With identity-first routing in CORE ([[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] T10) every customer provider gets this On, so the login page never grows a button per customer and local users see a password-only form. No `login.ftl`/theme edit needed. Verify routing BEFORE hiding, or SSO users have no way in; users hitting the Keycloak URL directly then see only the password form.
10. Google variant: built-in Google social provider is fine (`/broker/google/endpoint`, same Trust Email + flow settings). Note: Google's test-user list is testing-mode-only — once published, ANY Google account authenticates; the gate is always Step 3.

### Step 3 — custom first-login flow (link-or-reject; the access gate)
Two gates: the IdP answers "who is this?"; this flow answers "are they allowed in?" (a KC user with the same email must pre-exist). Without it the default flow auto-creates orphans.

1. Correct realm → `Authentication → Flows → Create flow` — e.g. `sso first login (no auto-create)`, **Basic flow**.
2. `Add step` → **Detect existing broker user** → **Required** (rejects unknown emails).
3. `Add step` → **Automatically set existing user** → **Required**, below it (silent link). Two Required rows, nothing else — any row on Alternative/Disabled silently admits everyone.
4. Attach as First login flow on EVERY SSO provider; delete orphans from earlier logins.

Rejection message key `federatedIdentityUnavailableUser` ("User … does not exist…"); reword via custom login theme (`themes/<name>/login/messages/messages_en.properties`, `parent=keycloak`) — **our KC is ECS/ECR-deployed (IP-type ALB targets, no server to hand-edit): theme changes = image rebuild**, same work item as Hexagon branding (PRD FR-A 17). Matching email → silent link, KC GUID preserved, `KcUserMapping` stays valid.

### Step 4 — verify & debug (no server-log access)
- Test matrix: matching pre-created user → straight in (link visible under *Identity provider links*); unknown account → rejected, no user created.
- **Login Events**: Realm settings → Save events → error codes in the Events view.
- **Claims viewer**: temporarily restore the default first-login flow — the "Update Account Information" page pre-fills exactly what the token carried; cancel without submitting.
- **One user rejects while others work** (rejection shows opaque `sub`): claim presence is per-user — check (1) cached Microsoft session logged in a DIFFERENT account → retest incognito typing the UPN; (2) that user's Email property unset (Step 1.5); (3) user is Guest/`#EXT#` instead of native.
- Email still missing despite scopes + property: App registration → Token configuration → optional claim `email` (+`upn`) on the ID token.
- Token inspection: `Clients → web → Client scopes → Evaluate → Generate access token`.
- "Unexpected error when authenticating with identity provider" = broker **callback** failure (secret/issuer/endpoint) — fires BEFORE the first-login flow, a pre-created user can't help; recheck Step 2.

## Q1 — What must change in user management
Authorization: nothing. Provisioning:
1. **CORE stays provisioning master**; user email = IdP email (the linking key).
2. **Password optional for SSO users** — `[Required]` input (`UsersController.cs:31`) + forced credential (`KcClient.cs:280-295`) need an SSO path that skips both.
3. **Hide password/confirm-email features** for SSO-linked users (IdP owns credentials; set `emailVerified` on link).
4. **Offboarding gap**: IdP-side disable does NOT remove the CORE user — needs reconciliation or an explicit story (SCIM later).
5. **Not phase 1**: IdP mappers Entra-groups→`site_id` would hand site-access ownership to the customer's IdP admin and worsen claims drift.

## Q2 — SSO for only some sites?
Yes — reframed as per-customer/per-email-domain (site membership is only known AFTER login; one realm, one login page):

| Option | How | Caveat |
|---|---|---|
| A. IdP buttons on shared login page | just add providers | zero code; buttons visible to all |
| B. Keycloak Organizations (KC ≥26) | email-first login, domain-routed to customer IdP | needs KC upgrade; the clean answer |
| C. `kc_idp_hint` per entry URL | core-web adds hint in `OnRedirectToIdentityProvider` (`Startup.cs:240`) | small code change + per-customer URL |
| D. Realm per customer | full isolation | **avoid** — API validates ONE Authority (`Startup.cs:244`); rearchitecture |

Force SSO-only for a customer: create their users with no password credential (needs Q1.2) — the password form can then never succeed.

## Questions asked
- 2026-07-14 — end-to-end auth/authz explanation + SSO IdP config + user-mgmt impact + per-site SSO → this note.
- 2026-07-14 — stop auto-creation, reject unknown SSO users → Step 3 flow; verified with Google.
- 2026-07-14 — test Entra without being org admin → own free tenant (Step 1); Entra verified end-to-end.
- 2026-07-17 — customize rejection message + Verify essential claim → theme-in-ECR-image + Step 2.7; tid/oid mappers designed & verified (Step 2.8); per-user email-claim trap confirmed twice (Step 4).
- 2026-08-31 — "Multi-tenant test shows 'Approval required / KeyCloak unverified' — normal? How do we stop any tenant's user getting in just because a matching email exists in CORE? How does tid flow?" → "Approval required" = enterprise tenants disable user consent (admin consent needed per customer); "unverified" = no verified publisher (MPN/Partner Center linkage, a Hexagon corporate task), which many tenants block outright. Email is NOT proof of customer — a tenant admin can set `mail` freely — so the `tid` check is mandatory; 5-step flow now in Step 2.8 with the never-TOFU rule for `ExpectedTid`. Consent friction previews a customer security review — the practical case for per-customer registrations ([[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] T10).

## Related
- [[2026-07-14 PRD Enterprise Authentication User Management Analysis]] — the Freeport/Entra PRD mapped onto this architecture
- [[2026-05-04 Keycloak SSO and Audience Mapper]] — the `aud: api` mapper fix + Google IdP PoC
- [[2026-05-05 How Authorization Flows from Core Web to Core API]] — bearer forwarding; /Identity endpoint; 401 playbook
- [[2026-04-29 Keycloak GetUsersForSite 403 Debugging]] — KcClient admin permissions
