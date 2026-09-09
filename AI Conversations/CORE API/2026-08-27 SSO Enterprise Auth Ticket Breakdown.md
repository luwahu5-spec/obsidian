---
date: 2026-08-27
updated: 2026-09-01
source: Claude Code
project: core-web-api (primary), core-web (T3, T5), infra (T6–T8)
tags: [planning, tickets, sso, enterprise-auth, keycloak, entra-id]
---

# SSO Enterprise Auth — ticket breakdown

> **Takeaway:** Five delivery tickets (T1–T5), all dark-shippable — no behaviour change until something calls them. Start with **T1**, since every other ticket reads its columns. T6–T8 is a parallel infra track that never blocks code.
>
> **Design decided 2026-09-01:** per-customer IdP entries + identity-first login (email → user type → Keycloak password form *or* the customer's Microsoft IdP). Existing users stay local forever; SSO users are created as SSO. Rationale and rejected alternatives: [[2026-07-14 PRD Enterprise Authentication User Management Analysis]]. Verified Keycloak config: [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]].

## The four enforcement layers (referenced as G1–G4 below)
Together these make PRD NFR-U 1 true — *an SSO user shall not create, reset or use local credentials*. They are independent on purpose: each closes a door the others leave open.

| Gate   | What it stops                                              | Mechanism                                                                                                | Ticket |
| ------ | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ------ |
| **G1** | Password login succeeding at all                           | The Keycloak user has **no password credential** — nothing to guess or leak                              | T2     |
| **G2** | The user *creating* a password via "Forgot your password?" | Conditional **Deny Access** in Keycloak's browser **and Reset-Credentials** flows when `auth_method=sso` | T6     |
| **G3** | An *admin* creating one from inside CORE                   | Password fields hidden; `ChangePassword` / confirm-email refuse for SSO users                            | T2     |
| **G4** | The wrong tenant or wrong person getting in                | CORE checks the token's `entra_tid` against the stored expected value (and records `entra_oid`)          | T3     |

**G2 is the one most designs forget:** G1 removes the credential, but a credential-less user can still hand themselves one through the password-reset email — after which they log in locally, bypassing their company's MFA and conditional access.

## Dependency spine
```
T1 Schema ──┬── T2 SSO user creation + G3 guards
            ├── T3 tid check + oid capture
            ├── T4 Audit events
            └── T5 Identity-first login (core-web)
Parallel infra: T6 realm runbook · T7 login theme image · T8 KC upgrade eval
```

---

## T1 — Add authentication method + identity binding to the data model
**Why:** every other ticket reads these columns.
**Scope:**
- User: `AuthenticationMethod` enum (`Local`=0 default, `EnterpriseSSO`), `ExpectedTid` (Guid?), `ExpectedOid` (Guid?), `OidCapturedAtUtc` (DateTime?).
- New `CustomerIdentityProvider` table (Minnovare.Core.Database/Models, where Customer lives): `CustomerId`, `IdpAlias`, `TenantId`, `SsoEnabled`, **email domain(s)** — customers often have several (`freeport.com`, `fmi.com`), so model as a child table or delimited column. Separate table (not columns on Customer) so a customer can hold >1 IdP.
- EF migration ships in the same PR as the model change (house rule).
**Design note:** there is **no user→customer link** — `MinnovareUser.cs:7-11` has no `CustomerId`, `Customer.cs:9-13` has `Sites` but no `Users`; the only path is site claims → Site → Customer. So `ExpectedTid` is stored **on the user**, copied from customer config at creation. Do not resolve it by walking user→sites→customer at login (fragile, adds a login-path query).
**Out of scope:** any behaviour reading the columns.
**Risk/check first:** if `MinnovareUser` lives in the **Shared submodule**, this carries a submodule bump shipping to BOTH repos — schedule accordingly.
**Acceptance:** migration applies on a prod-shaped DB copy; all existing users read `Local`; snapshot updated; no consumer behaviour changes.
**Size:** S–M (M if submodule).

## T2 — SSO user creation path + Gate 3 guards
**Why:** "SSO users shall not create or reset local credentials" starts at creation; G3 closes the CORE-side reset surfaces.
**Depends:** T1.
**Scope:**
- **KcClient SSO-variant creation:** NO password credential, NO `UPDATE_PASSWORD`, `emailVerified=true`, `auth_method=sso` attribute — today `MigrateUser` always writes a credential (`KcClient.cs:280-295`). Verifiable against the corelocal realm, whose PoC users are ready-made fixtures.
- Creation DTO carries `AuthenticationMethod`; password becomes conditionally required (replaces blanket `[Required]`, `UsersController.cs:31`). SSO → the variant above; `ExpectedTid` auto-filled from `CustomerIdentityProvider`; `ExpectedOid` left null (captured on first login).
- Guards: SSO only when the customer has an `SsoEnabled` IdP; never for Administrator / Drill-Plan users.
- G3: `ChangePassword` (`UsersController.cs:571`) and `SendConfirmEmail` (`:523`) return an explicit error for `EnterpriseSSO` users.
- Both creation paths covered (`PostUserAsync` `:328`, `CreateSiteUserAsync` `:423`).
**Acceptance:** SSO create → user in both stores, no KC credential, `auth_method=sso` attribute set, mapping row present; password create unchanged; ChangePassword on SSO user → 4xx with clear reason; unit tests for the conditional validation.
**Size:** M.

## T3 — Tenant verification (tid) + identity capture (oid)
**Why:** Keycloak links by email only; the `tid` check pins the user to the correct customer tenant. Mappers already verified in KC (Attribute Importer, Sync mode Force → `entra_tid`/`entra_oid` claims).
**Depends:** T1 (columns), T4 (audit).
**Scope:**
- Verification service, `EnterpriseSSO` users only — token `entra_tid` must equal `ExpectedTid`, else reject + audit mismatch (expected vs presented).
- `entra_oid` is **recorded, not enforced**: `ExpectedOid` null → store it + stamp `OidCapturedAtUtc` (audit "binding captured"). A differing oid on a later login is audited, not blocked — this avoids permanent lockout when a customer recreates an Entra account, and keeps the value for audit and the future SCIM `externalId`. Enforcement can be switched on later with no migration.
- Enforcement point decided in-ticket; lean: core-web session establishment (OIDC events, `Startup.cs:240` area) calling one new API endpoint — zero per-request cost.
- Local users: no-op (their tokens carry no such claims).
**⚠️ Implement fail-closed.** `ExpectedTid == null` on an SSO user must **DENY**, not skip the check. The natural implementation is the dangerous one:
```csharp
// WRONG — silently admits any tenant when the tid was never recorded
if (user.ExpectedTid != null && token.Tid != user.ExpectedTid) Reject();
// RIGHT
if (user.AuthMethod == EnterpriseSSO)
    if (user.ExpectedTid == null || token.Tid != user.ExpectedTid) Reject();
```
**Acceptance:** first login captures oid + stamp; wrong-tenant token → blocked + audit row; null `ExpectedTid` on an SSO user → denied; changed oid → audited but still admitted; password users unaffected; cross-repo PR pair if enforcement lands in core-web.
**Size:** M–L (cross-repo).

## T4 — Enterprise auth audit event catalogue
**Why:** PRD NFR-A 5; cheap now, painful to retrofit; T3 writes these events.
**Scope:** via existing `AuditService`: `SsoUserCreated`, `IdentityBindingCaptured`, `IdentityBindingMismatch` (expected vs presented tid/oid), `SsoPasswordEndpointRefused`. Each: actor, target user, timestamp, detail payload.
**Acceptance:** events persisted and queryable; the mismatch event carries both values — it is the evidence trail if oid enforcement is ever switched on, and the early-warning signal that a customer has recreated accounts.
**Size:** S.

## T5 — Identity-first login
**Why:** the routing mechanism for the decided design. Each customer has their own IdP entry; without this, every entry adds a button to the shared Keycloak login page and customers without SSO see buttons that don't apply to them.
**Depends:** T1 (customer/domain data). Independent of T2–T4.
**Solution, two parts:**
1. **Config:** `Hide on login page = On` for every customer IdP. A hidden provider still works via `kc_idp_hint`, so no buttons appear for anyone — SSO users never see that page, local users get a clean password-only form. No `login.ftl`/theme edit needed. **Order matters:** verify routing first, then hide, or SSO users lose their way in.
2. **Code:** core-web `[AllowAnonymous]` email page → API lookup → challenge Keycloak with `kc_idp_hint=<alias>` + `login_hint=<email>` via `OnRedirectToIdentityProvider` (core-web `Startup.cs:240`). No alias → no hint → Keycloak password form with the username pre-filled.
**Spike already built** (stashed on MC-1840 as `stash@{0}`, compiles clean, `/Signin` confirmed working): `SsoRouting.cs`, `Pages/Signin.cshtml(.cs)`, `Login.cshtml.cs` redirecting to `/Signin`, `Startup.cs` `OnRedirectToIdentityProvider` + `DefaultChallengeScheme`→Cookies with `LoginPath`, stub map in `appsettings.Development.json`.
**⚠️ Trap already handled in the spike:** `DefaultSignOutScheme` silently falls back to `DefaultChallengeScheme`, so switching the challenge to Cookies would stop `SignOutAsync()` with no scheme (`BTokenRefresh.cshtml.cs:55`) from ending the **Keycloak** session — user "logs out", then logs straight back in. Pin `options.DefaultSignOutScheme = "oidc"`.
**Must change before this is real:** (a) domain→alias map moves from core-web config to an API endpoint over `CustomerIdentityProvider`; (b) route by the USER's `AuthenticationMethod`, not email domain, so a local contractor on an SSO customer's domain still gets the password form; (c) caching + rate-limiting on what is an unauthenticated login-path lookup; (d) hardcoded English strings → 7 `.resx`.
**Acceptance:** all IdP buttons hidden; SSO user's email → lands directly on their Entra; local user's email → password form with pre-filled username; deep links route through the email page.
**Size:** M (core-web + small API endpoint).

---

## Parallel infra track (never blocks the code track)
- **T6 — Realm SSO config runbook/productionization:** custom first-login flow, audience mapper, Attribute-Importer mappers (tid/oid, Sync mode Force), IdP entry checklist (alias = customer slug, **Hide on login page = On**), login events enabled — scripted or documented for staging + prod. Currently hand-built in corelocal only.
  - **Includes G2 (not yet built anywhere):** a `Condition - user attribute` (`auth_method` = `sso`) + `Deny Access` step added to the **browser flow** and — the one that matters — the **Reset Credentials flow**. Without the reset-flow deny, an SSO user can request a password-reset email and give themselves a working local password, bypassing their company's MFA/conditional access and defeating NFR-U 1. Both authenticators exist on KC 20; verify in our instance. Includes the onboarding sequence: **record the customer's tid → create their users → enable login** (never backfill the tid afterwards), and resolve the tid independently from their email domain via the discovery endpoint rather than trusting a typed GUID. Also delete stray test providers.
- **T7 — Custom login theme in ECR image:** `federatedIdentityUnavailableUser` override + Hexagon branding (PRD FR-A 17) — Dockerfile `COPY themes/core /opt/keycloak/themes/core`, realm Login-theme setting, ECS redeploy. DevOps-owned.
- **T8 — Keycloak upgrade evaluation (20 → 26):** unlocks Organizations and Verify essential claim; KC 20 is out of security support. Staged image bump, DB auto-migration. Under PRD FR-A 14.

## Deliberately NOT in this batch
User Catalogue / User Security UI revamp (blocked on UX designs per PRD); SCIM server (phase 2, pending Freeport sign-off); user disable / offboarding story; "admin never types a password" for local users; oid enforcement plus a clear-binding admin action (only if email recycling proves to be a real incident).

## Open decisions
Mismatch UX wording (T3 error surface); whether verification lives in core-web or API middleware (decided inside T3); audit retention (T4).
