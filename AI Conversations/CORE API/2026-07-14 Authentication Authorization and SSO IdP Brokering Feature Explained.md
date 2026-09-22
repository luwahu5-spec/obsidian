---
date: 2026-07-14
updated: 2026-09-09
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
   - **Superseded 2026-09-09:** the domain-collection/`CustomerIdentityProvider`-table plan below is replaced by three columns directly on `Customer` (`IdpAlias`, `TenantId`, `SsoEnabled`) — routing resolves by the user's own `AuthenticationMethod`/`CustomerId`, never by email domain, so a domains table was never actually load-bearing. One tenant per customer is the common case; if a customer ever needs two (post-acquisition), extract into a child table then. Also collect **named identity-admin contacts** (needed for support escalation). The whole exchange fits FMI's existing intake (§5.1 + Appendix A questionnaire). See [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] for the current data model.

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
   4. CORE reads `entra_tid` / `entra_oid` from the bearer token and stores them as `ExpectedTid`/`ExpectedOid` on the user record.
   **Design decision 2026-09-09 — NOT enforced at login.** Originally planned as a CORE-side runtime comparison (reject on mismatch), this was dropped: each customer's Keycloak provider is already bound to that customer's tenant by its own tenant-specific discovery URL, and the link-or-reject first-login flow (Step 3) already requires the user to pre-exist — so the tenant is guaranteed structurally before any CORE code runs. A redundant runtime check would add a failure mode (a wrongly-populated `ExpectedTid` blocking a legitimate user) without adding protection. The columns are captured and kept in the schema, reserved for future provisioning work (`oid` doubles as the future SCIM `externalId`) — see [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]].
9. **Hide on login page** (Advanced settings): removes the provider's button WITHOUT disabling it — `kc_idp_hint` still routes to it. With identity-first routing in CORE ([[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] ticket 3.1–3.3) every customer provider gets this On, so the login page never grows a button per customer and local users see a password-only form. No `login.ftl`/theme edit needed. Verify routing BEFORE hiding, or SSO users have no way in; users hitting the Keycloak URL directly then see only the password form.
10. Google variant: built-in Google social provider is fine (`/broker/google/endpoint`, same Trust Email + flow settings). Note: Google's test-user list is testing-mode-only — once published, ANY Google account authenticates; the gate is always Step 3.

### Step 3 — custom first-login flow (link-or-reject; the access gate)
Three gates now apply: the IdP answers "who is this?"; the first two executions require the Keycloak user to pre-exist; and the conditional sub-flow requires that existing user to have been provisioned by CORE specifically for SSO (`auth_method=sso`). Without this flow, Keycloak's default first-broker-login flow can auto-create orphan users. Without the attribute gate, a pre-existing local user with the same email could be silently linked to an external IdP.

1. Correct realm → `Authentication → Flows → Create flow` — e.g. `sso first login (no auto-create)`, **Basic flow**.
2. `Add step` → **Detect existing broker user** → **Required** (rejects an external identity when no matching Keycloak user exists).
3. `Add step` → **Automatically set existing user** → **Required**, directly below it (selects the matching existing user for silent linking). This must run before the attribute condition because Keycloak must first place the existing user into the authentication context.
4. At the root of the flow, `Add sub-flow` → name it e.g. **Reject users not marked for SSO** → set its Requirement to **Conditional**. Place it below **Automatically set existing user**.
5. From the `+` menu on that sub-flow, choose **Add condition** (not **Add step**) → **Condition - user attribute** → **Required**, then configure:
   - Attribute name: `auth_method`
   - Expected attribute value: `sso`
   - Negate output: **On**
   - Include group attributes: **Off**
6. From the same sub-flow's `+` menu, choose **Add step** → **Deny Access** → **Required**. Configure the error message, for example: `This CORE account is not enabled for enterprise SSO.`
7. Attach this flow as **First login flow** on EVERY SSO provider; delete any orphan users created by earlier use of the default flow.

The final tree must be:

```text
CORE SSO First Broker Login
├── Detect Existing Broker User             REQUIRED
├── Automatically Set Existing User         REQUIRED
└── Reject users not marked for SSO         CONDITIONAL
    ├── Condition - User Attribute           REQUIRED
    │   ├── Attribute name: auth_method
    │   ├── Expected value: sso
    │   ├── Negate output: On
    │   └── Include group attributes: Off
    └── Deny Access                          REQUIRED
```

The negation is intentional: when the selected user has exactly `auth_method=sso`, the condition is false, so the rejection sub-flow is skipped and linking continues. When the attribute is absent or has another value, the condition is true and **Deny Access** stops the login before a federated identity link is created. In the Admin Console, conditional authenticators appear under **Add condition** only after a **Conditional** sub-flow has been created; they do not appear in the ordinary **Add step** list.

Unknown users are rejected by **Detect existing broker user** (message key `federatedIdentityUnavailableUser`). Existing users not marked for SSO are rejected by the configured **Deny Access** message. Theme-level rewording uses `themes/<name>/login/messages/messages_en.properties` with `parent=keycloak` — **our KC is ECS/ECR-deployed (IP-type ALB targets, no server to hand-edit): theme changes = image rebuild**, same work item as Hexagon branding (PRD FR-A 17). An eligible matching SSO user is linked silently; its KC GUID is preserved and `KcUserMapping` stays valid.

### Step 3b — custom Reset Credentials flow (G2: blocks self-service password reset for SSO users)
**Built and verified 2026-09-09** in `corelocal`. Removing an SSO user's password credential (G1) is not enough on its own — Keycloak's password-reset page is reachable by direct URL regardless of routing, and without this step an SSO user could use "Forgot your password?" to give themselves a working local password, bypassing their company's MFA/conditional access entirely.

1. **Duplicate the built-in flow** — `Authentication → Flows → reset credentials` (marked `Built-in`) → kebab menu (⋮) → **Duplicate** → name it e.g. `reset credentials (SSO blocked)`. Don't edit the built-in directly.
2. **Add sub-flow** → name it e.g. `Block SSO Users` → Requirement **Conditional**.
3. Inside that sub-flow, add two steps:
   - `Condition - user attribute` → **Required** → configure: Alias = any label (e.g. `sso-user`, purely cosmetic, unrelated to the IdP alias), Attribute name `auth_method`, Expected attribute value `sso`, Negate output **Off**.
   - `Deny Access` → **Required**.
4. **Position matters** — the sub-flow must sit directly below `Choose User` and above `Send Reset Email`, at the same indentation level as `Choose User` (not nested inside the `Reset - Conditional OTP` sub-flow). Use the table view (not the diagram view) to confirm indentation and order reliably. If it fires after `Send Reset Email`, the reset email has already been sent by the time access is denied.
5. **Bind the flow**: kebab menu on the new flow → **Bind flow** → **Reset credentials flow**. Confirm via the Flows list **"Used by"** column — it should now show your copy, not the built-in.

**Test:** tag a user with attribute `auth_method=sso` (this is what ticket 2.1 sets automatically at creation) → "Forgot your password?" → denied immediately, **no email sent**. A user without the attribute gets the normal reset email, unaffected.

**Version note:** `Condition - user attribute` was present and usable on this KC 20 instance — worth reconfirming when repeating this in staging/production (ticket 4.4), since availability isn't guaranteed across every KC 20.x build.

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
- 2026-08-31 — "Multi-tenant test shows 'Approval required / KeyCloak unverified' — normal? How do we stop any tenant's user getting in just because a matching email exists in CORE? How does tid flow?" → "Approval required" = enterprise tenants disable user consent (admin consent needed per customer); "unverified" = no verified publisher (MPN/Partner Center linkage, a Hexagon corporate task), which many tenants block outright. Consent friction previews a customer security review — the practical case for per-customer registrations. (This answer was reached before the team settled on per-customer registrations exclusively; see the 2026-09-09 entry below for the final call on tid.)
- 2026-09-09 — team decided the per-customer provider binding alone is sufficient proof of tenant (each provider only accepts its own tenant's tokens, and the user must pre-exist) — the CORE-side `tid` runtime check from the answer above is dropped as redundant. `ExpectedTid`/`ExpectedOid` remain captured and stored, reserved for future provisioning, not enforced. See [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]].
- 2026-09-09 — "Add the reset-credentials customization step" → Step 3b added: duplicate the built-in Reset Credentials flow, add a Conditional sub-flow with `Condition - user attribute` (auth_method=sso) + Deny Access, positioned between Choose User and Send Reset Email, bind it as the realm's Reset credentials flow. Built and verified working in corelocal.

## Related
- [[2026-07-14 PRD Enterprise Authentication User Management Analysis]] — the Freeport/Entra PRD mapped onto this architecture
- [[2026-05-04 Keycloak SSO and Audience Mapper]] — the `aud: api` mapper fix + Google IdP PoC
- [[2026-05-05 How Authorization Flows from Core Web to Core API]] — bearer forwarding; /Identity endpoint; 401 playbook
- [[2026-04-29 Keycloak GetUsersForSite 403 Debugging]] — KcClient admin permissions
