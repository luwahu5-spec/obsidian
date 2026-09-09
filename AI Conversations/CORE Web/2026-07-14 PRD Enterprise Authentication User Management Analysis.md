---
date: 2026-07-14
updated: 2026-07-17
source: Claude Code
project: both
tags: [prd-analysis, auth, sso, entra-id, keycloak, user-management, freeport]
---

# PRD Enterprise Authentication & User Management — fit against the investigated architecture

> **Takeaway:** The PRD (Confluence page 862781441, Dale Cromarty, driven by Freeport) confirms the Keycloak-brokering approach end-to-end: Keycloak stays, Entra ID is brokered, authorization stays claim-driven in CORE, local logins keep working, and unknown SSO users must be refused — which the Google PoC already proved with the custom first-login flow. The genuinely new/hard piece is **User Type as a first-class immutable concept (Site User / Administrator / SSO User / Drill Plan User)** that must *determine which authentication method a user is presented* (FR-A 6) — per-USER login routing is stronger than stock Keycloak domain-based routing and drives most of the real engineering work, along with a User Catalogue/User Security UI revamp.

## What the PRD locks in (matches the investigation 1:1)
- **Keycloak stays** (Assumptions) → brokering, not replacement. Kills the comment "can Keycloak be removed?": no — the API validates KC-issued tokens (`Startup.cs:245` `Audience="api"`), and machine clients (Deswik/Aegis/LiveMine/Newtrax) authenticate via KC client credentials (`Startup.cs:257-275`).
- **Entra ID only; additional IdPs out of scope** → Google IdP was a PoC vehicle only. Entra = generic OIDC v1.0 provider with tenant discovery URL (see [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]]).
- **FR-A 4 local auth continues** + **FR-A 5 per-customer enable/disable** → the Q2 answer (same realm, routing per customer).
- **FR-A 7/8 authN/authZ separation, permissions stay in CORE** → matches: `SiteAuthorizationHandler` never asks how you authenticated; no IdP group→site mapping in phase 1 (PRD agrees — SCIM/JML out of scope).
- **FR-A 9 map authenticated users to existing CORE roles** → satisfied by pre-create + auto-link (KC GUID preserved → `KcUserMapping` + roles/claims intact).
- **FR-A 10 prevent unauthorised users** → exactly the custom first-login flow (`Detect Existing Broker User` + `Automatically Set Existing User`), verified working 2026-07-14.
- **FR-A 11/16 meaningful auth errors** + **FR-A 17 Hexagon-branded login** → one work item: custom Keycloak login theme (branding + `federatedIdentityUnavailableUser` message override).
- **NFR-A 9 no tokens in browser** → already true by architecture (Blazor Server; token lives in server-side `TokenStore`).
- **FR-A 20 session across modules** → already true (cookie + server-side token).
- **NFR-U 1 SSO users have no local credentials / FR-U 9 no password on SSO create** → the identified change: `InputUser.Password` `[Required]` (`UsersController.cs:31`) and `KcClient.MigrateUser` always writing a password credential + `UPDATE_PASSWORD` (`KcClient.cs:280-295`) must get an SSO path.

## The new/hard requirements (beyond what was discussed)
1. **FR-A 6 + FR-U 4 + NFR-U 3 — User Type is a first-class, immutable, creation-time concept** deciding the auth method presented. Today "user type" doesn't exist: the only role is `Administrator`, drill-plan users are inferred from `drill_plan_site_id` claims, and the UI shows role "None" for everyone else (PRD FR-U 3 renames those to "Site User"). Needs: a UserType on the user model (or KC attribute), creation-time radio, immutability enforcement.
2. **Per-USER login routing** (FR-A 6 + FR-A 15): stock Keycloak routes per email-domain (Organizations, KC≥26) or per `kc_idp_hint`, not per user attribute. Options: (a) domain routing approximates it if all SSO users share customer domains (true for Freeport); (b) identity-first login flow / custom authenticator SPI that looks up the user's type and redirects; (c) CORE-side pre-login screen passing `kc_idp_hint`. Design decision needed — flag early.
3. **FR-A 12 configurable inactivity expiry** — today hardcoded 60-min sliding cookie (core-web `Startup.cs:237-238`) + KC session idle defaults. "Configurable" (per customer?) is new work on both layers.
4. **FR-A 13/18 admins manage enterprise auth config *within CORE*** — implies CORE UI driving KC admin REST for IdP config (KcClient extension). "Subject to implementation approach" — scope risk; manual KC console config is the cheap alternative.
5. **FR-A 14 simplify existing auth configuration** — licence to clean up: duplicate `AddAuthentication` (api `Startup.cs:233-242`), dual claims stores drift (KC attributes vs `AspNetUserClaims`).
6. **User management UI revamp (FR-U 1-10)**: Users → chevron menu (User Catalogue + User Security); site NAMES instead of IDs on user claims ("Sites"); Edit button on user detail; filter/sort by user type (new column); multi-site select at creation (also for SSO users); new per-site User Security page (active sites only); site-page Add User gets Site User vs SSO User choice with no password for SSO.
7. **NFR-A 5 auth events audited** — KC event logging + surfacing; **NFR-A 6 graceful IdP-down behaviour** — broker error handling on the themed login page.

## Answers to the open PRD comments (RThumma, Mar 31)
- *Password changed in AD — sync to CORE?* → Non-issue under NFR-U 1: SSO users have NO CORE/KC password at all; nothing to sync. Entra owns credentials entirely.
- *Both site admin and drill plan user as SSO?* → PRD FR-U 4 models SSO User and Drill Plan User as mutually exclusive types; drill-plan users keep the simplified workflow (they have fake emails — can't SSO; `KcClient.cs:309` already marks them emailVerified because of that).
- *2FA instead of AD?* → Out of scope (MFA explicitly excluded); also doesn't meet Freeport's lifecycle requirement.
- *Remove Keycloak?* → No (see above).

## Implementation traps the PRD doesn't mention (carry into design)
- **`aud: api` audience mapper** must exist on the `web` client or every SSO login 401s at the API and loops `/BTokenRefresh` ([[2026-05-04 Keycloak SSO and Audience Mapper]]).
- **Email is the linking key**: CORE-created user email must exactly equal the Entra email or first-login linking fails (rejection path).
- **Trust Email = On** on the IdP, or linking demands a verification step.
- **Orphan soft-fail**: an auto-created SSO user authenticates fine but sees "no permission to view sites" — provisioning gap, not an authz bug. The custom first-login flow prevents it; keep it mandatory in every environment.
- **Dual-store writes**: every new user-management surface (User Catalogue, site-page Add User) must keep writing BOTH stores like `UsersController` does today, or drift worsens (NFR-U 2 "no errors on user changes" is effectively a dual-store-sync-hardening requirement).

## FMI client standard (SCIM_FMI_Vendor_Integration.pdf) vs the PRD — gap analysis (2026-07-17)
Source: Freeport's vendor-shareable SCIM standard (12 pp., on desktop). Their model: **"Implement SSO first (for login) and SCIM second (for lifecycle)"** (§2.1); §3 "What the Vendor Must Provide" = a SCIM 2.0 **server** on our side (`/Users` (+`/Groups` optional), OAuth bearer, filter by `userName`/`externalId`, `active=false` deactivation, 429+Retry-After, `/ServiceProviderConfig`+`/Schemas`); goal "reduces or eliminates local/manual accounts" via automated Joiner/Mover/Leaver.

**Gaps:**
1. 🔴 **PRD excludes SCIM/JML — the centerpiece of the client's standard.** The PRD's manual pre-create model is exactly what FMI wants eliminated. Needs explicit Freeport sign-off on phasing (their own "SSO first, SCIM second" supports it).
2. 🔴 **No deactivation concept in CORE** — only hard delete (`UsersController.cs:601-647`); SCIM Leaver + FMI test plan (§6.1) require disable/re-enable.
3. 🟡 **No `externalId`** — SCIM matches on it (Entra `objectId`). Capture Entra's `oid` claim as a KC attribute at first link — cheap now, painful to backfill.
4. 🟡 **Ops requirements missing from PRD**: Entra client-secret rotation (secrets expire — all SSO logins break at expiry), sandbox env for FMI testing, support/escalation contacts, rate limits (Appendix A questionnaire needs written answers).
5. 🟢 **Authorization answer to document**: FMI prefers Entra-group authorization but permits Users-only SCIM (Q2) — PRD FR-A 8 (permissions stay in CORE) is defensible; state it explicitly in the questionnaire.

**PRD requirements needing rewording (validated against implementation):**
- **FR-A 6** (auth method by CORE User Type): KC login page can't consult CORE types pre-auth → reword to identity-domain routing (email-domain/IdP hint), with User Type controlling credential issuance only.
- **NFR-U 3** (User Type immutable): blocks password→SSO migration of existing users; delete/recreate would destroy `KcUserMapping`. Reword to allow a one-way admin migration to SSO (verified: existing password users link cleanly).
- **FR-A 13/18** (auth config UI in CORE): IdP config changes ~once per customer (FMI intake is ticket-driven, §7.1) → downgrade to documented ops procedure / Should Have.
- **FR-A 12** (configurable inactivity timeout): define scope — per deployment (web cookie `Startup.cs:237-238` + KC session idle in lockstep), not per customer unless demanded.
- **Out of Scope SCIM** → rephrase as "separate phase; this phase shall not preclude it" + new NFR: provisioning architecture SCIM-compatible (disable semantics, externalId, idempotent create).
- **NFR-A 9** (no tokens in browser): already met by Blazor Server architecture — annotate as met-by-design.

**Key architectural argument for product:** the verified pre-create + link-or-reject model IS SCIM-shaped — SCIM replaces the human admin, not the auth flow. Entra's Joiner push would call the same dual-store creation path (`Identity → KcClient → KcUserMapping`) that `UsersController` runs today; the first-login flow is untouched. Phase 1 and phase 2 are the same architecture with a different caller.

**Action list:** (1) raise Gap 1 with product + get Freeport phasing sign-off; (2) draft Appendix A questionnaire answers; (3) three cheap SCIM-readiness items in phase 1 — user disable/re-enable, store Entra `oid` at link, SSO no-password creation path; (4) ops runbook: secret-expiry calendar, KC login events on, staging realm as FMI sandbox; (5) propose the rewordings above as PRD comments.

## Per-user authentication method — design picture (2026-07-17)
Requirement: auth method determined by customer + user config; SSO users shall not create/reset/use local credentials.

- **"Secret password" is NOT enforcement** — fails 4 ways: requirement says "shall not CREATE"; the creating admin knows it; KC "Forgot password?" / Account console lets the user set one themselves; CORE `ChangePassword` (`UsersController.cs:571`) can mint one. Enforcement = **absence of the credential**, not secrecy. (Local Identity password hash is vestigial anyway — all logins go through KC.)
- **Two-level model**: Customer record → IdP alias + expected `tid`; User → `AuthenticationMethod` (Local|EnterpriseSSO), CORE DB master + KC attribute mirror. SSO only allowed if customer has an IdP; user's expected tid comes from customer config.
- **Four enforcement layers**: (1) no KC password credential / no UPDATE_PASSWORD for SSO users (change `KcClient.MigrateUser`, `KcClient.cs:280-295`); (2) KC conditional Deny (`Condition - user attribute` + `Deny Access`, KC 20-compatible) on browser AND **Reset Credentials** flows — the forgot-password bypass is the one designs forget; (3) CORE hides/refuses password endpoints for SSO users; (4) **tid/oid check enforces "the APPLICABLE IdP"** — KC links by email only, so with multiple customer IdPs a matching email could link via the wrong tenant; per-user expected tid closes this.
- **DECIDED 2026-07-18 — method switching**: not immutable; "not editable, changes only via explicit audited conversion actions" (preserves NFR-U 3 intent). **Convert to SSO**: precondition checks (customer has IdP, not Admin/DrillPlan, domain warning) → CORE `AuthMethod`+`ExpectedTid` (from customer config), `ExpectedOid=null` → KC attribute + DELETE password credential + drop UPDATE_PASSWORD/OTP + emailVerified → revoke sessions → audit; fails closed if user absent in Entra (hence dry-run report for per-customer bulk convert, pilot first per FMI §5.4). **Convert to Local**: CORE fields → KC attribute + **DELETE federated identity link** (else both methods work — the forgotten step) + execute-actions email (user sets own password; admin never types one) → revoke sessions → audit. Asymmetry: →SSO removes a credential; →Local grants one — Administrator-only, never an outage workaround.
- **DECIDED 2026-07-18 — Entra recreation / "Reset identity link"**: recreated account (same email, new oid+sub) fails closed at KC (stale federated link, old sub) or at G4 (oid mismatch) — re-bind must clean BOTH: delete KC federated link + clear `ExpectedOid` (NEVER tid — tid changes are customer-level config). G4 mismatches audit expected-vs-presented oid (support evidence; spike = customer bulk migration). **Mandatory human verification** before re-bind: confirm with the customer's named identity admins (FMI §5.1 contacts) that the account was recreated for the SAME person — resists the recreated-leaver-mailbox takeover. Self-healing finish: next login re-links by email, TOFU recaptures. UI home: "Identity link" panel on Edit User (status, tenant, oid date, last SSO login + both action buttons) — doubles as support diagnostics.
- **Still open**: mismatch UX wording; audit event catalogue (NFR-A 5); offboarding latency = session lifetime; KC 20 shared login page until upgrade (interim: identity-first Username Form flow possible on KC 20).
- **Recommended bonus**: admin never types passwords for ANY user (create → KC reset email) — kills the admin-knows-password smell globally.
- **DECIDED 2026-09-01 — login topology**: per-customer IdP entries + identity-first login (email → user type → Keycloak password form OR the customer's Microsoft IdP). Each customer registers the app in **their own** tenant and provides client ID + secret + tenant ID; every provider is set `Hide on login page = On`, so no SSO buttons appear for anyone. This is how FR-A 5/6/15 are satisfied on KC 20 without Organizations. Tickets: [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]].
  - **Deciding reason:** an SSO button must not appear for customers who haven't configured SSO. Identity-first goes further than any button arrangement — nobody sees a button; each user only meets the one path that applies to them.
  - **Why the Hexagon-owned multi-tenant "unified portal" was rejected:** (1) **Keycloak can't easily consume it** — a multi-tenant app's token issuer carries the *user's* tenant GUID while `/common` discovery publishes the literal template `https://login.microsoftonline.com/{tenantid}/v2.0`; Keycloak stores one expected issuer and validates strictly, so it never matches. (2) **Loses structural tenant proof** — with per-customer entries, arriving via the `freeport` provider *is* proof of tenant; with a shared app the tenant is only a claim, making the `tid` check the single control. (3) **Organizational blocker** — the app registration must live in *Hexagon's* corporate tenant, requiring Hexagon IT to own it, own secret rotation (one expiry downs every customer), and pass a security review; per-customer needs **no Hexagon tenant at all**. (4) **Consent friction, observed in testing 2026-08-31** — signing in from another tenant showed *"Approval required"* (enterprise tenants disable user consent) plus *"unverified"* (no MPN/Partner Center publisher verification, which some tenants block outright). That screen is a preview of what a customer security review sees; the per-customer model has none of it. This also matches FMI §4, which already commits them to providing the Enterprise Application on their side.
  - **Industry validation:** Atlassian runs both — a generic "Continue with Microsoft" social button (their own multi-tenant app; identity only) and, for enterprises, Atlassian Guard: customer verifies their email domain, configures their own IdP, and login is email-first → domain match → redirect to the org's IdP. Enterprises use the second, which is exactly this design. (Atlassian can run the shared button because they wrote their own OIDC client — the blocker for us is Keycloak's issuer validation, not the protocol.)

## Questions asked
- 2026-07-14 — "Walk through the PRD requirements against what we discussed." → Verdict: architecture validated; new work = User Type concept + per-user login routing + user-management UI revamp + login theme; PoC first-login flow is FR-A 10 delivered.
- 2026-07-17 — "Review the client's SCIM_FMI_Vendor_Integration doc, validate the PRD against it, flag FRs/NFRs that no longer make sense, and say what to do." → Gap analysis section above; biggest finding: PRD's SCIM exclusion collides with the client standard; six requirements need rewording; five-step action list.
- 2026-07-17 — "Per-user auth method: is the unknown pre-created password enough to block SSO users from password login? Complete the picture." → No — four bypasses; full design in *Per-user authentication method* section: two-level config, four enforcement layers, eight open decisions.

## Related
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — the living architecture/config doc this PRD lands on
- [[2026-05-04 Keycloak SSO and Audience Mapper]] — the `aud` trap
- [[2026-05-05 How Authorization Flows from Core Web to Core API]] — 401 debugging playbook
