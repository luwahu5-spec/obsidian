---
date: 2026-08-27
updated: 2026-09-09
source: Claude Code
project: core-web-api (Layers 1–2), core-web (Layer 3), infra (Layer 4)
tags: [planning, tickets, sso, enterprise-auth, keycloak, entra-id]
---

# SSO Enterprise Auth — ticket breakdown

> **Takeaway:** Four delivery layers — two data-model tickets, two backend tickets, eight frontend tickets, six Keycloak/infra tasks. Everything through the frontend is dark-shippable — no user-visible behaviour until Layer 3 lands, so it merges continuously with no feature flag. Full design writeup, approved: **CORE Enterprise SSO Design** (Confluence + [artifact](https://claude.ai/code/artifact/a5778522-063a-43d7-9a03-d0128bb403d0)). This note is the engineering-detail companion — file:line references, the KcClient specifics, and what got cut and why.
>
> **Design decided 2026-09-01, finalized 2026-09-09:** per-customer IdP entries + identity-first login. Existing users stay local forever; SSO users are created as SSO — no conversion either direction. Rationale and rejected alternatives: [[2026-07-14 PRD Enterprise Authentication User Management Analysis]]. Verified Keycloak config: [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]].

## The three enforcement gates that are code (G4 is not)
Together with G4 below, these make one requirement true: *an SSO user shall not create, reset or use local credentials.*

| Gate | What it stops | Mechanism | Ticket |
|---|---|---|---|
| **G1** | Password login succeeding at all | The Keycloak user has **no password credential** — nothing to guess or leak | 2.1 |
| **G2** | The user *creating* a password via "Forgot your password?" | Conditional **Deny Access** in Keycloak's **Reset Credentials** flow when `auth_method=sso` | 4.2 |
| **G3** | An admin creating one from inside CORE | Password fields hidden/disabled; change-password and confirm-email refuse for SSO users; email locked on edit | 2.1, 3.6 |
| **G4** | The wrong tenant or wrong person getting in | **Structural, not a runtime check** — each Keycloak provider is bound to one customer's tenant by its own configuration, and the user must already exist in CORE. No code ticket. | — |

**G2 is built and verified** (2026-09-09) against the `corelocal` realm: a user tagged `auth_method=sso` is denied at "Forgot your password?" with no email sent; a normal user's reset is unaffected. Ticket 4.2 is repeating this configuration in staging and production, not building it from scratch.

**G4 was originally a runtime `tid` comparison in code — removed by design decision.** Each customer's Keycloak provider only accepts tokens from that customer's own tenant (its discovery URL is tenant-specific), and the link-or-reject first-login flow requires the user to pre-exist in CORE. Both guarantees exist before any CORE code runs. A second, redundant comparison in CORE would add a failure mode (e.g. a wrongly-populated `ExpectedTid` blocking a legitimate user) without adding protection. The `ExpectedTid`/`ExpectedOid`/`OidCapturedAtUtc` columns remain in the schema, reserved for future provisioning work (SCIM `externalId`), but nothing populates or checks them in this phase.

## Dependency spine
```
1.1 User model ──┬── 2.1 SSO user creation (+ G1, G3)
1.2 Customer model ┴── 2.2 Sign-in lookup
                         │
              3.1–3.3 Sign-in flow (needs 2.2)
              3.4–3.8 User pages (needs 2.1)
Parallel: 4.1 provider checklist · 4.2 G2 (done) · 4.3 claim mappers ·
          4.4 runbook · 4.5 login theme · 4.6 KC upgrade eval
```

---

## Layer 1 — Data model

### 1.1 — Update the user model
**Why:** every other ticket reads these columns.
**Scope:**
- `AuthenticationMethod` enum: `Local = 0` (default), `EnterpriseSSO = 1`.
- User columns: `CustomerId` (nullable — Administrators span customers), `AuthenticationMethod`, `ExpectedTid` (Guid?, reserved), `ExpectedOid` (Guid?, reserved), `OidCapturedAtUtc` (DateTime?, reserved).
- Extend the `User` contract the API returns (and the `MinnovareUser → User` conversion) to carry `AuthenticationMethod` and `CustomerId` — without this the web app cannot tell an SSO user from a local one.
- EF migration ships in the same PR as the model change.
**Design note:** `MinnovareUser.cs:7-11` has no customer link today; this ticket adds it directly on the user rather than resolving it via site claims → Site → Customer at login (fragile, adds a login-path query).
**Acceptance:** migration applies on a prod-shaped DB copy; every existing user reads `Local`; no consumer behaviour changes.
**Size:** S–M.

### 1.2 — Update the customer model
**Why:** an SSO-enabled customer needs a place to hold its provider config; a site needs a way to reach its customer.
**Scope:**
- `Customer` columns: `IdpAlias` (string?, immutable once live), `TenantId` (Guid?, unique index), `SsoEnabled` (bool, default `false`).
- Expose `Site.CustomerId` and a `Customer` navigation on `Site` — the column already exists as an EF shadow property with populated data (`b.HasOne(..., null).WithMany("Sites").HasForeignKey("CustomerId")` in the model snapshot), so this needs **no data migration**, just making it a real property.
**Superseded design:** the original plan (below) used a separate `CustomerIdentityProvider` table with email domains, to support >1 IdP per customer. Dropped 2026-09-09 — routing resolves by the **user's** `AuthenticationMethod` + `CustomerId`, never by email domain, so the domain column was never actually needed; and one IdP per customer is the common case, so three columns on `Customer` is simpler than a join. If a customer ever needs two tenants (post-acquisition), extract these columns into a child table then.
**Acceptance:** a site can resolve its customer in code; a customer can hold one IdP configuration.
**Size:** S.

<details>
<summary>Superseded: original CustomerIdentityProvider table design (kept for history)</summary>

A separate table (`CustomerId`, `IdpAlias`, `TenantId`, `SsoEnabled`, email domain(s) as a child table or delimited column), so a customer could hold more than one IdP. Replaced by 1.2 above.
</details>

---

## Layer 2 — Backend

Each ticket delivers one working capability end to end.

### 2.1 — Creating an SSO user works
**Why:** "SSO users shall not create local credentials" starts at creation.
**Depends:** 1.1, 1.2.
**Scope:**
- **KcClient SSO-variant creation:** no password credential, no `UPDATE_PASSWORD` required action, `emailVerified=true`, `auth_method=sso` attribute — today `MigrateUser` always writes a credential (`KcClient.cs:280-295`).
- Both creation endpoints (`PostUserAsync` `UsersController.cs:328`, `CreateSiteUserAsync` `:423`) accept `AuthenticationMethod` and `CustomerId`; navigation-bar entry takes the customer as input, the site entry derives it from the site.
- Password becomes conditionally required (replaces blanket `[Required]`, `UsersController.cs:31`): required for local, rejected for SSO.
- `ExpectedTid` copied from the customer at creation (reserved column, not enforced — see G4 note above).
- Validation: SSO only when the customer has `SsoEnabled` and an `IdpAlias`; never for Administrator or Drill-Plan users.
- **G3:** `ChangePassword` (`UsersController.cs:571`) and `SendConfirmEmail` (`:523`) return an explicit error for `EnterpriseSSO` users; email becomes non-editable on update for SSO users (it's the Entra link key).
**Acceptance:** SSO create → user in both stores, zero KC credentials, `auth_method=sso` set, mapping row present; local create unchanged; ChangePassword on an SSO user → 4xx with a clear reason.
**Size:** M.

### 2.2 — The sign-in page can ask who an email belongs to
**Why:** the routing decision (password form vs. redirect to the customer's IdP) has to be made by CORE before Keycloak is ever involved.
**Depends:** 1.1, 1.2.
**Scope:**
- One new endpoint: given an email, return whether the user exists, their `AuthenticationMethod`, their customer's `IdpAlias`, and whether the customer's `SsoEnabled` is true.
- Unauthenticated by necessity (called before login) — needs its own access model (see open question below) and rate limiting, since it reveals whether an email is registered.
**Acceptance:** the four cases below each return a distinct, unambiguous answer:

| # | Condition | Result |
|---|---|---|
| 1 | Local user | route to Keycloak password form, username pre-filled |
| 2 | Not found | reject |
| 3 | SSO user, customer `SsoEnabled=true` | route to the customer's IdP via `kc_idp_hint` |
| 4 | SSO user, customer `SsoEnabled=false` | refuse with an explanation — **never** falls through to the password form |

**Size:** M.

---

## Layer 3 — Frontend (core-web)

| # | Page | Change |
|---|---|---|
| 3.1 | Sign-in page — **new** | Email field; calls 2.2; branches per the table above |
| 3.2 | Login entry point | Redirect to 3.1 instead of challenging Keycloak directly |
| 3.3 | Authentication wiring | `kc_idp_hint` + `login_hint` on redirect via `OnRedirectToIdentityProvider`; route unauthenticated requests to 3.1; keep sign-out ending the Keycloak session |
| 3.4 | Add User — nav bar (Admin only) | Authentication method choice + customer selector (SSO-enabled customers only); password enabled only for Local |
| 3.5 | Add User — under a site | Authentication method choice; customer taken from the site; hidden when that customer has no SSO |
| 3.6 | Edit User | SSO users: hide change-password, disable the email field, show method read-only, no revert-to-local option |
| 3.7 | Site create/edit | Read-only indicator that the customer has SSO enabled — informational only |
| 3.8 | Localisation | Every new label/message across all 7 `.resx` files |

**Spike already built** (stashed on `MC-1840`, later moved to `MC-1842`, compiles clean, `/Signin` confirmed working end to end): `SsoRouting.cs`, `Pages/Signin.cshtml(.cs)`, `Login.cshtml.cs` redirecting to `/Signin`, `Startup.cs` `OnRedirectToIdentityProvider` + `DefaultChallengeScheme`→Cookies with `LoginPath`.

**⚠️ Trap handled in the spike:** `DefaultSignOutScheme` silently falls back to `DefaultChallengeScheme`. Switching the challenge scheme to Cookies would have made `SignOutAsync()` with no scheme (`BTokenRefresh.cshtml.cs:55`) stop ending the **Keycloak** session — user "logs out", logs straight back in. Fixed by pinning `options.DefaultSignOutScheme = "oidc"` explicitly.

**Must change before the spike is real:** the domain→alias lookup (`SsoRouting.cs`, backed by an `appsettings` stub) is replaced entirely by calling 2.2 and branching on the returned `AuthenticationMethod` — **not** on email domain, so a local contractor on an SSO customer's domain still gets the password form.

**Acceptance:** all four cases from 2.2 render correctly; deep links route through 3.1; local login and logout behave exactly as before.

---

## Layer 4 — Keycloak & infrastructure (parallel, never blocks code)

| # | Task | Notes |
|---|---|---|
| 4.1 | Per-customer provider checklist | Alias = customer slug (immutable, unique per realm), client ID/secret, tenant-specific discovery URL, Trust Email on, first-login flow attached, **Hide on login page = On** |
| 4.2 | **G2 — reset-credentials deny** | ✅ Built and verified 2026-09-09 in `corelocal`: `Condition - user attribute` (`auth_method`=`sso`) + `Deny Access`, positioned between "Choose User" and "Send Reset Email" in a duplicated + bound Reset Credentials flow. This task is repeating the same config in staging/production |
| 4.3 | Claim mappers | Attribute Importer (Sync mode Force) for `tid`/`oid` → user attributes; protocol mappers exposing them in the access token — built for future use, not consumed by any check yet |
| 4.4 | Realm runbook | Everything above, written up per environment; currently only built by hand in `corelocal`. Includes onboarding order: record the customer's tenant → create their users → enable login (never backfill); derive the tenant ID from their email domain via the discovery endpoint rather than trusting a typed GUID; delete stray test providers |
| 4.5 | Login theme | Rejection message override + Hexagon branding, baked into the Keycloak image (ECR/ECS — no server to hand-edit) |
| 4.6 | Keycloak upgrade evaluation (20 → 26) | Unlocks Organizations and Verify essential claim; KC 20 is out of security support |

---

## Deliberately not in this batch
User Catalogue / User Security UI revamp (blocked on UX designs); SCIM server (phase 2, pending Freeport sign-off); any conversion between Local and SSO (structurally impossible — existing users' emails don't match their Entra identities); a customer self-service admin page for enabling SSO (client secret can only live in Keycloak, so engineering stays involved regardless — revisit if onboarding volume justifies it); enforcing `ExpectedTid`/`ExpectedOid` at login (reserved for future provisioning, not needed given G4 is structural); audit event catalogue (not required now that G4 has no runtime decision to log — revisit if SCIM or the customer admin page ever land).

## Open questions
1. How does the web app authenticate to the sign-in lookup (2.2)? It's called *before* login, so no bearer token exists yet. Leaning: a service credential from core-web, plus rate limiting either way.
2. Wording of the rejection messages an end user actually sees — to be drafted with Product.
