---
date: 2026-07-07
source: Claude Code
project: core-web / core-web-api
tags: [planning, prd-front-end, mining-methods, core-refactor-2.23, users, site-creation, rig-creation, settings]
---

# PRD Front End §15/19/20/21/22 — Code Gap Analysis & Executable Plan

> Analyzed against: PRD - Front End (Confluence §15 Users, §19 Site Creation, §20 Rig Creation, §21 Settings, §22 General), the Core Refactor 2.23 architecture plan, and the current code on `core-web@MC-1773` + `core-web-api@MC-1773-selected-site-main-page-rf`.

---

## Page 1 — Where the code is today (foundation already built)

The refactor foundation from the 2.23 architecture plan is partially landed, which changes the shape of this work — several PRD items are *extensions of existing scaffolding*, not greenfield:

| Foundation piece | State | Evidence |
|---|---|---|
| `MiningMethod` enum (`DefaultProduction`, `DefaultDevelopment`, `CutAndFill`) | ✅ Done (Shared 2.23) | `Minnovare.Core.Shared/Models/MiningMethod.cs` |
| `UserSiteMiningMethodScope` table (UserId + SiteId + MiningMethod) | ✅ Migrated | migration `20260406080328_UserSiteMiningMethodScope` |
| `Rig.MiningMethod` column | ✅ Exists, **not backfilled, not written by UI** | `Rig.cs:132` |
| `GET /MiningMethods/Assigned?userId&siteId` endpoint | ⚠️ Exists but **stubbed** — returns all 3 methods hardcoded | `MiningMethodsController.cs:30`, `IUserMiningMethodScopeService.cs:15` (`StubUserMiningMethodScopeSerivce`) |
| Site Details page with per-method tabs | ✅ Done (MC-1773) | `SiteDetails.razor.cs:191` `LoadMiningMethodTabs()` + temporary bridge `FilterMiningMethodTabsByLegacySiteFeatures` (line 273) |
| Config-driven upload module | ✅ Done (MC-1822) | `Sites/Uploads/SiteUploadConfigProvider.cs` |
| Backend table-data contracts (column metadata driven tables — §22) | ⚠️ Contracts only, **no provider/endpoint** | `Shared/Contracts/TableDataRequest.cs`, `TableData/TableDataResult.cs` |
| Site drilling methods storage | ❌ Still legacy `SiteFeatures` (`FeatureType.Production/Development`) — no `CutAndFill` feature | `SiteCreateEdit.razor.cs:356-392` |
| User drilling-method assignment UI | ❌ None | `EditUser.razor` only has Administrator checkbox (line 53-63) |
| Rig method selection UI | ❌ Method inferred from route `/Site/{SiteId}/Rig/{RigSiteType}` | `RigEdit.razor:2` |

**Critical path insight:** everything downstream (Site Details tabs, landing page, rig sync payloads) already calls `/MiningMethods/Assigned` — but it lies (stub returns everything). The **real scope service + data seeding is the single most load-bearing ticket** in this whole PRD. This matches the agreed implementation order note (Users first — build the source of truth before the UIs that consume it).

---

## Page 2 — §15 Users: gap analysis & tickets

Current code: `Pages/Blazor/Users/DetailsUser.razor` (entity page), `EditUser.razor(.cs)`, `CreateGlobalUser.razor(.cs)` (global/admin creation), `Pages/Blazor/Sites/AddUser.razor.cs` (per-site user creation via `CreateSiteUser`), `UserList.razor`.

> **FINAL DESIGN (2026-07-07): methods are stored globally per user; the API stays per-site.** `GetAllowedMethodsAsync(userId, siteId)` returns *user's global methods ∩ site's enabled methods* — the endpoint signature and all its consumers (SiteDetails tabs, `MiningMethodRepository`) are unchanged; only storage and service internals are decided here. If product ever demands per-user-per-site granularity, we add a SiteId column back and change service internals + an edit UI — no endpoint consumer changes. Future-proof at the seam, minimal at the storage.
>
> Context that drove it — there are **three user-creation paths**, and global storage is the only design where two of them need zero work:
> 1. **Users page** (super admin only, `CreateGlobalUser`) — user has **no sites yet** at creation, so per-site storage has nowhere to put their methods; global storage just writes rows.
> 2. **Add User from a site's drilling-method tab** (`Sites/AddUser`) — auto-assign the method of the tab the admin is standing in (`MiningMethodKey` already travels in navigation state, `SiteDetails.razor.cs:418`). "Cannot save without a method" satisfied by construction; existing form UI unchanged.
> 3. **Security page** (add existing user to a site) — zero change: the user keeps their global methods and the intersection starts returning the right tabs the moment the site claim exists. The "user added to a new site later" edge case ceases to exist.

### Backend tickets (core-web-api) — do these first

**U-API-1 — Real `UserMiningMethodScopeService`** (replaces stub)
- Storage: small `UserMiningMethod` table (UserId + MiningMethod). The already-migrated `UserSiteMiningMethodScope` table has no real consumers (only the stub) — replace/repurpose it via migration.
- `GetAllowedMethodsAsync(userId, siteId)` = user's global methods ∩ site's enabled methods (site methods from S-API-1; until then, bridge via `SiteFeatures`). Admin rule server-side: `Administrator` role → all methods regardless of rows.
- Keep `StubUserMiningMethodScopeSerivce` side by side until cutover (existing convention), swap DI registration last; fix the `Serivce` typo while touching it.

**U-API-2 — User-method CRUD endpoints**
- `GET /MiningMethods/User/{userId}` (the user's global set — feeds DetailsUser display and EditUser form), `PUT` to replace the set.
- Server-side validation: reject empty set for non-admins; admin → force all. Authorization: admin-only, consistent with user management.

**U-API-3 — Deployment seeding (data migration)**
- Existing Site Users → `DefaultProduction` + `DefaultDevelopment` rows; existing Administrators → all methods; Drill Plan Users → all methods (not role-identifiable — in practice "any user we can't classify"; see Q4).
- One row per user per method — no per-site fan-out to compute. Ship model change + migration together (MiningMethod missing-migration incident, 2026-06-24 note).

**U-API-4 — Creation-path writes**
- `CreateSiteUser` (path 2): accept/derive the drilling method from the calling tab's context and write the row on create.
- Global create (path 1): accept the selected method set in the create payload.
- Path 3 (Security page add-to-site): explicitly **no work** — verify with a test that a newly-claimed site shows the intersected tabs.

### Frontend tickets (core-web)

**U-WEB-1 — User Entity page (`DetailsUser.razor`)**
- "None" → "Site User" under User Roles (display-level mapping; not a Keycloak role change).
- Administrator ticked → always display "Administrator".
- New "Drilling Methods Available" section nested under User Roles — show only methods assigned to the user (from U-API-2).
- "User claims" heading → "Sites"; resolve `site_id` claim values → Site names (needs a site-name lookup — `SiteRepository` list is available); Administrator → display "All".
- Inline **Edit** button top-right → navigates to `/Users/Edit?id=` (route already exists).
- Back navigation to Users page (breadcrumb/header — `ConfigurePageHeader` at `DetailsUser.razor.cs:51`).

**U-WEB-2 — Edit User (`EditUser.razor`)**
- Drilling-method checkboxes; ticking Administrator auto-selects all + disables unticking methods until Administrator is unticked.
- Save blocked unless ≥1 method (client + server).
- Persist via U-API-2; effect on next page load / next CORE Sync (see Q5 on "immediately").

**U-WEB-3 — Create flows**
- `CreateGlobalUser` (path 1): add the shared `DrillingMethodCheckboxes` component — default **unchecked**, Administrator → all checked + locked, ≥1 required. The only creation path that gets new UI.
- `Sites/AddUser` (path 2): **no UI change** — auto-assign the drilling method of the tab the admin came from; methods adjustable afterwards via Edit User.
- Security page (path 3): **no change** — intersection handles it.

**U-WEB-4 — Landing page intersection rule**
- Method tab shown only when enabled for **both** site and user. `SiteDetails.razor.cs` already intersects assigned methods with site features via the temporary bridge — once site methods (S-API-1) and user scope (U-API-1) are real, replace `FilterMiningMethodTabsByLegacySiteFeatures` with the true intersection and delete the bridge.

**U-WEB-5 — Localization + tests**
- Every new/changed label (Site User, Sites, Drilling Methods Available, Edit) = component + all 7 `.resx` files.
- bUnit tests for DetailsUser/EditUser (new `[Inject]`s on shared/base components → run affected bUnit projects).

### Frontend story-point estimates (§15, core-web only — backend assumed delivered; re-estimated 2026-07-07)
Scale: 1 = 4–12h · 2 = 12–18h · 3 = 18–24h · 5 = 24–30h · 8 = 30–38h. Buffer for testing/fixes included.
Scope note: Sites/AddUser auto-assign and the SiteDetails bridge cutover were declared **out of scope** for this estimate by the user.

| Ticket | Points | Hours | Buffer rationale |
|---|---|---|---|
| U-WEB-1 DetailsUser rework | **2** | 12–18 | tiny page, additive rendering; site-name lookup = one repository call; 4 labels × 7 resx is mechanical |
| U-WEB-2 EditUser + shared `DrillingMethodCheckboxes` | **2** | 12–18 | real risk = sequencing method-save with Keycloak role add/remove in `HandleSubmit` (EditUser.razor.cs:136-147); component itself is a checkbox group with two rules |
| U-WEB-3 CreateGlobalUser integration | **1** | 4–12 | component exists by then; watch create-then-assign sequencing (user must exist in Keycloak before methods PUT) |
| **Total** | **5** | **~28–48** | ≈ 1 week solo |

Dependency risk: all three block on backend GET/PUT user-method endpoints being stable — starting earlier adds contract-churn cost to tickets 1–2.

---

## Page 3 — §19 Site Creation: gap analysis & tickets

Current code: `Pages/Blazor/Sites/SiteCreateEdit.razor(.cs)`. Production/Development are Yes/No selects mapped to `SiteFeatures` (`SiteCreateEdit.razor.cs:356-392`); per-method settings already live in `TabControl` tab pages hidden by `_productionEnabled/_developmentEnabled` — the PRD's "show only relevant config" behaviour **already exists structurally**; the work is generalizing it to N methods.

> **DECISION (2026-07-07): per-method configuration values are INDEPENDENT (Interpretation B).** A site running Production + Cut and Fill holds two separate sets of Dip/Dump settings. The design below reflects that; confirm with product Tuesday, but plan and build to B.

**S-API-1 — Site drilling-methods storage**
- New `Site → SiteMiningMethods` collection per the architecture plan (§4 entity relationship) — with B decided, the per-method config table is being built anyway, so the "enabled methods" list rides on the same structure instead of stretching `SiteFeatures` further.
- Migration preserving existing sites' current methods (NFR "existing Sites retain configuration"): sites with `FeatureType.Production` → Production row, `FeatureType.Development` → Development row.
- `SiteFeatures.Production/Development` stay in place (read-only compatibility) until the SiteDetails bridge (`FilterMiningMethodTabsByLegacySiteFeatures`) and any API consumers are cut over, then deprecate.

**S-API-2 — Per-method configuration table (committed, Interpretation B)**
- New entity, e.g. `SiteMiningMethodConfiguration`: `SiteId` + `MiningMethod` (composite key) + the 8 orientation fields — `DipTolerance`, `DumpTolerance`, `PositiveDirectionDip`, `PositiveDirectionDump`, `ZeroOffsetDip`, `ZeroOffsetDump`, `DipLimit`, `DumpLimit`. (Fields that stay genuinely method-specific and single-owner — e.g. `DrilingTolerance`/`GradeFormat` for Development, MWD/ChargePlan flags for Production — can stay where they are for now; only the fields Cut and Fill duplicates *must* move.)
- **Backfill migration:** copy the legacy `Site.DipTolerance` etc. columns into the Production row for every Production-enabled site. Model change + migration ship together (MiningMethod missing-migration incident, 2026-06-24 note).
- **Legacy columns kept and dual-written for one release:** the tablet sync and any report queries still read `Site.DipTolerance` today. Saving the Production tab writes both the Production config row and the legacy columns until R-API-2 (method-filtered sync) reads from the new table — then legacy columns are dropped in a follow-up migration. This avoids coordinating web + API + mobile in a single big-bang release.
- Repository + UnitOfWork contract (`ISiteMiningMethodConfigurationRepository` or hang off `SiteRepository`), CRUD through the existing Site save endpoint (config travels with the Site payload).
- Unit tests: per-method round-trip, backfill correctness, dual-write consistency.

**S-API-4 — Sync payload per method (coordinates with R-API-2)**
- Tablet sync for a rig resolves configuration from `SiteMiningMethodConfiguration[rig.MiningMethod]` instead of flat Site columns. Cross-team item with mobile: payload shape ideally unchanged (same field names, values sourced per method) so the tablet needs no change — confirm.

**S-WEB-1 — Drilling Methods Available checkbox group**
- Replace the two Yes/No selects with a checkbox group (Development / Production / Cut and Fill), rendered from the backend method list (never hardcode names — refactor principle: frontend renders whatever the backend returns).
- Keep field order per PRD (methods between Timezone and Next Shift Threshold).

**S-WEB-2 — Orientation settings sub-component + Cut and Fill tab (Interpretation B)**
- Extract the 8 orientation fields from the Production tab into one reusable component (e.g. `OrientationSettingsEditor`) that binds to a `SiteMiningMethodConfiguration` instance — **not** to `_site.DipTolerance` directly.
- Production tab renders `OrientationSettingsEditor(configs[Production])` + its Production-only extras (MWD, ChargePlan, Ream Sizes…); Cut and Fill tab renders `OrientationSettingsEditor(configs[CutAndFill])` and nothing else initially. Two tabs, two independent value sets — no third copy of the markup, and a future method 4 gets this for free.
- First-enable defaults: when Cut and Fill is ticked on an existing site, pre-fill its config from the Production row if one exists (sensible site standards as starting point), otherwise model defaults — cheap UX win, confirm with product.
- Validation identical to Production (same `Validate(nameof(...))` pattern, per-tab error highlighting like `DoesProductionTabPageHaveError()`).

**S-WEB-3 — Disable-method prompt**
- Confirmation modal (existing `GenericModal`) when unticking a method on an existing site: "all rigs assigned to {method} will no longer receive configuration after the next CORE Sync". Only on edit, only when rigs exist for that method (needs a cheap rig-count-by-method call).

**S-WEB-4 / S-API-3 — Inheritance & sync NFRs**
- "Site config is the default for rigs / changes inherited / disabled method stops config updates at next CORE Sync" — today rigs *reference* site config live (no copies), so most of this is **verification of existing behaviour**, not new code. New work: sync payload must filter by the rig's `MiningMethod` (R-API-2). Add integration tests around the sync endpoint per method.

---

## Page 4 — §20 Rig Creation: gap analysis & tickets

Current code: `Pages/Blazor/Rigs/RigEdit.razor` — method comes from the **route** (`/Site/{SiteId}/Rig/{RigSiteType}`, line 2) and rig-type dropdown filters by `RigTypeClassification` (lines 52-65). PRD changes this to a single Add Rig page with a **radio button** for drilling method.

**R-WEB-1 — Unified Add Rig page**
- One entry point from any Site page/tab; radio group Development / Production / Cut and Fill, options limited to the **site's enabled methods** (from S-API-1).
- Selecting a method reveals its section:
  - Development: Name, Rig Type radio Single/Twin Boom (today it's a dropdown — becomes radio).
  - Production: Name, Rig Type radio Boom/Horseshoe, Ream Sizes, Offset Type radio Collar/Pivot, Alternate Dump radio **On/Off** (label change from Yes/No — dropdown→radio, ×7 resx).
  - Cut and Fill: Name only (initial scope per DC Jul 2 comment).
- Route note: the legacy `{RigSiteType}` route stays until callers are migrated, then redirect.

**R-WEB-2 — Edit rules**
- All fields editable after creation **except Drilling Method** (render read-only on edit).

**R-API-1 — Persist `Rig.MiningMethod`**
- Write on create (column exists at `Rig.cs:132`); validate against site's enabled methods server-side.

**R-API-2 — Backfill + method-filtered sync**
- Migration: existing rigs get MiningMethod from `RigTypeClassification` (ProductionRigs → DefaultProduction, DevelopmentRigs → DefaultDevelopment).
- Tablet sync returns only configuration applicable to the rig's method (this is the API contract change the mobile team must confirm — flag as cross-team dependency).

---

## Page 5 — §21 Settings & §22 General

### §21 Settings — smallest epic, mostly regression scope
Pages: `/Settings/ShiftSettings` (Configure Shift Report Variables), `/Settings/ConsumableTypeReason` (Link Consumable Types and Reasons), `Blazor/Drillers/DrillerList` (Driller Overview).
- **SET-1**: regression test pass confirming new drilling-method data points flow through unchanged (no code expected).
- **SET-2**: verify/add back-navigation (NFR says "back to Reports Landing Page" — see Q8, likely a copy-paste error).
- Schedule **last**; it's a verification gate on the other epics, not parallel feature work.

### §22 General — the actual refactor payload (config-driven tables)
This is the architecture-plan §5 work (unify plan-summary tables; fixed shell for shift pages). Contracts already exist (`TableDataRequest`, `TableDataResult` with `IColumnMetaData`) but **no backend provider** — §22's success criterion ("method 4 = config only, no new component code") is *unachievable by frontend work alone*.

**G-API-1 — Table-data provider endpoint** — serves `TableDataResponse` per (site, method, tableType); columns carry visibility/label/sortable metadata. This is the prerequisite for everything below.
**G-WEB-1 — Generic table renderer** — extend `Shared/Table/TableControl` to render from `IColumnMetaData`: columns absent from metadata are **not rendered at all** (PRD: no empty "-" columns), sortable flags drive asc/desc via existing `SortableHeader`.
**G-WEB-2 — Migrate plan-summary tables** (production 6-col / development 3-col components → one config-driven table), then shift pages keep fixed shell with configurable drilling-details section.
**G-WEB-3 — Method-keyed navigation** — Plan Summary → Plan Data → Entity → Sub-Entity links resolve per method key through the web layer (BFF) routing; `SiteDetails` already carries `MiningMethodKey` in navigation state (`SiteDetails.razor.cs:418`).
**G-WEB-4 — Delete the bridges** — remove `FilterMiningMethodTabsByLegacySiteFeatures`, `GetLegacySiteType`, and remaining `SiteType`/`ProductCategory` conditionals in tabular components. Acceptance: grep for `SiteType.Production|SiteType.Development` in UI components returns only the legacy pages scheduled for deletion.

### Recommended sequencing (dependency-ordered, matches the agreed Users-first note)
```
Sprint 1:  U-API-1/2/3 (real scope service + seeding)  ──┐
Sprint 1-2: U-WEB-1..5 (Users pages)                     ├─ unblocks landing-page intersection
Sprint 2:  S-API-1/2/4 (method table + per-method config) │  (Q1 & Q2 decided by engineering; confirm Tuesday)
Sprint 2-3: S-WEB-1..4 (Site create/edit)                │
Sprint 3:  R-API-1/2 + R-WEB-1/2 (Rig)                   │  (mobile team dependency for sync)
Sprint 3-4: G-API-1 + G-WEB-1..4 (tables refactor)       │  (largest, can start G-API-1 early in parallel)
Sprint 4:  SET-1/2 regression gate + bridge deletion
```

---

## Page 6 — Requirements to challenge / questions for Product (Tuesday)

**Q1 (§15) — RESOLVED by engineering 2026-07-07, confirm with product: drilling methods are global per user (reading A), intersected with site methods for display.** Storage is a per-user set; the API seam stays per-site (`userId + siteId`) so per-site granularity remains a service-internals change if ever demanded. Decision drivers: a globally-created user has no sites yet (per-site storage has nowhere to write), the Security-page add-to-site flow and the "user joins a new site later" edge case both become zero-work under global storage, and the site-tab Add User flow auto-assigns its tab's method with no new UI. (Context: user↔site is many-to-many via `site_id` claims — one claim per accessible site; admins hold none and bypass by role.) Remaining product confirmations below.

Ready-to-send wording for product:

```
Question on the User drilling methods (PRD §15) before we build:

The mockup shows one "Drilling Methods Available" checkbox list on the User page. We want to confirm what that list means, because there are two possible readings:

A) Global per user — ticking Production means the user can use Production at every site they have access to. What they actually see at a given site is then: user's methods AND site's methods (the intersection rule already in the NFRs).

B) Per user per site — a user could have Production at Site A but only Development at Site B. This would need a bigger UI (a sites-by-methods grid per user) and more Support admin every time a user is added to a site.

Our recommendation is A: it matches the mockup, keeps Support admin low, and per-site visibility differences are already handled by the site-level toggles. The database is built so we can move to B later without a migration if it's ever needed.

Related follow-up we need answered either way: when an existing user is later added to a NEW site, what should their drilling methods be at that site?
- Option 1: they carry their existing methods (consistent with reading A — our recommendation)
- Option 2: Support picks methods at the moment of adding them to the site

And for go-live seeding, we'll apply "existing Site Users get Production + Development" across all sites each user currently has access to — shout if that's not the intent.
```

**Q2 (§19) — RESOLVED by engineering 2026-07-07, confirm with product: per-method values are independent (Interpretation B).** A site running both Production and Cut and Fill keeps two separate sets of Dip/Dump settings, stored in a new per-method configuration table (see S-API-2). Remaining product confirmations: (a) two independent value sets is the intended behaviour, (b) when Cut and Fill is first enabled on an existing site, pre-filling from the Production values is an acceptable default.

**Q3 (§19 + the Site-vs-Rig email question) — Engineering position: keep configuration at Site level.** Full argument below, written to be presented Tuesday.

> **Site vs Rig level configuration — engineering position**
>
> **The scenario that decides it: a site standard changes.** A site runs 10 production rigs with Dip Tolerance 5°; the survey team tightens the standard to 3°. Tolerance is a *site standard* — it defines what counts as a compliant hole for everyone drilling in that mine, not a per-machine preference.
>
> - *Config at Site level (today):* Support edits one field. All 10 rigs pick up 3° at their next CORE Sync. One change, one audit trail, zero ways to get it wrong.
> - *Config copied into each Rig:* Support edits 10 rig pages. Rig 7 is mid-recalibration, its update gets skipped "for later," and two weeks on it still drills against 5° while nine rigs use 3°.
>
> **Why that drift is poisonous, not just untidy:** the tolerance feeds compliance calculations. The same hole quality is now judged by two yardsticks on the same site — rig 7 shows suspiciously better compliance, nothing errors, nothing logs, the data is quietly wrong until a customer notices. Whoever investigates will chase drilling data and sensors long before thinking to diff 10 rig config pages field by field. Drift bugs are invisible, silent, and discovered by the customer.
>
> **Scale:** 30 sites × 10 rigs = **300 copies** of every setting instead of 30, and every future config field multiplies by rig count forever.
>
> **The PRD itself argues against rig-level:** §19 requires *"changes to site-level configuration must be applied to all existing rigs assigned to the corresponding drilling method."* With site-level config this is true **by construction** — rigs read site values at sync; nothing to propagate. With rig-level copies we'd have to *build* propagation machinery (fan-out updates, partial-failure handling, "which rigs didn't get it" reporting) just to simulate what site-level gives for free. Rig-level config doesn't just add Support admin — it adds engineering work whose only purpose is to fight the drift it created.
>
> **Why rig-level does NOT make future methods easier:** "drilling method 4 is cheap" comes from config being keyed by *method* (a `SiteMiningMethodConfiguration` row, an adapter entry, metadata-driven tables). Whether that config hangs off Site or is copied into every Rig changes nothing about how much new code a method needs — it only changes how many rows Support maintains (sites × methods vs. sites × methods × rigs). Rig-level storage multiplies admin without removing a single `if` from the codebase.
>
> **The principle to adopt:**
> - **Site = standards** — tolerances, orientations, limits: things the mine decides.
> - **Rig = physical facts about one machine** — selected ream sizes, IP configuration, snapshot setting: things that genuinely differ per rig, and which already live on the Rig today.
> - Litmus test: *if changing a value would ever trigger "now update it on all the other rigs too," it belongs at the Site.*
>
> **Conceded counterpoint:** if one rig ever legitimately needs to deviate from the site standard, site-level can't express that. The answer is a per-rig **override** for that specific field when a real need appears (rig value wins *if set*, otherwise inherits the site standard) — not pre-emptively copying everything down to rigs for a need nobody has demonstrated.
>
> This aligns with DC's gut feel in the kickoff email: site standards stay at the site, Support doesn't wear extra admin, and the refactoring's method-keyed config architecture — not rig-level storage — is what makes future methods cheap.

**Q4 (§15) — Drill Plan Users get all methods by assumption.** They're not role-identifiable, so operationally this means "any user without explicit scope rows gets everything," which quietly weakens the site user visibility rule. Acceptable as an interim + Support training item, but please commit in the SSO PRD to an explicit Drill Plan role so this can be tightened later — and confirm how seeding should classify them (we cannot distinguish them from ordinary site users in data).

**Q5 (§15) — "Changes take effect immediately after saving."** Realistic semantics: web landing page → on next navigation/refresh (Blazor Server won't live-push tab changes to an already-open session without extra work); tablets → next CORE Sync. Confirm this definition of "immediately" is acceptable; live-push to open sessions is possible but is real extra scope.

**Q6 (§15) — "None → Site User" scope.** The PRD names the User Entity page only, but the same "None" renders on the Users list page. Recommend the label mapping be global for consistency; confirm.

**Q7 (§20) — Cut and Fill rig has Name only, but §19 gives Cut and Fill site-level tolerance settings.** A rig with no type/config receives which parts of the site payload at sync? Suggest explicitly defining the minimal Cut and Fill sync payload now to avoid the mobile team guessing.

**Q8 (§21, minor) — "Navigation back to Reports Landing Page" under Settings NFRs** looks like a copy-paste from a Reports section — Settings pages aren't reached from Reports. Confirm intended back-target (Settings/home?).

**Q9 (§22, dependency risk) — "Adding method 4 requires only configuration changes"** is a backend-owned success criterion: it requires the table-data provider (columns/labels/sources from the API) which today exists only as contracts. If the backend provider isn't committed in this cycle, §22 can only be partially met (frontend renderer ready, configs still shipped in frontend code as an interim), and the PRD should say which flavor it accepts.

**Q10 (§20, minor) — Rig Type UI changes from dropdown to radio** while "no functionality change" — fine, but note Single/Twin Boom radio for Development is a *new* explicit choice (today it's the `RigType` dropdown filtered by classification). Confirm the radio maps 1:1 onto existing `RigType` values and no new types are implied.

---

*Prepared for the Tuesday requirements discussion. Each "Page N" section above is sized to paste as one Loop page.*
