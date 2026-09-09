---
date: 2026-07-22
updated: 2026-08-05
source: Codex (VS Code)
project: core-web / core-web-api
tags: [planning, architecture, drilling-methods, sites, authorization, decision-points]
status: decision-required
---

# Core Refactor Decision Points

> **Primary decision:** How does a Site declare which Mining Methods it supports? This must be explicit before administrator access, Cut and Fill, Site Edit, Rig creation, and user-method assignment can have one reliable source of truth.

## Decision Point 1 - How does a Site bind to Mining Methods?

### Business requirement

A Site can support one or more Mining Methods, for example:

- Default Development
- Default Production
- Cut and Fill
- Future methods added later

The set belongs to the Site. It is separate from:

- All methods known by the system;
- the methods assigned to a user;

The effective methods shown on a Site page should be:

```text
Administrator = methods enabled for each individual active Site

Regular user = methods enabled for the selected Site
               INTERSECT methods assigned to that user
```

### What exists now

There are currently three incomplete sources:

1. `GET /MiningMethods` returns every value from the `MiningMethod` enum. It is currently a stub and does not accept `siteId`, so it cannot answer which methods are enabled for one Site.
2. `UserSiteMiningMethodScope` stores user/site/method rows. It answers user scope, but a user-scope row should not be the source of truth for Site capability.
3. Legacy `SiteFeatures` identifies Production and Development availability. There is no equivalent Cut and Fill feature.

CORE Web currently uses a temporary compromise:

- Administrators call `GET /MiningMethods` and therefore bypass user-scope rows.
- Regular users call `GET /MiningMethods/Assigned?userId=...&siteId=...`.
- Production and Development results are filtered again using legacy `SiteFeatures`.
- Cut and Fill cannot be site-filtered because no legacy SiteFeature represents it. While `GET /MiningMethods` remains an all-enum stub, administrators can therefore see Cut and Fill for Sites that have not explicitly enabled it.

This is useful for UI development, but it is not the final authorization or domain model.

### Current CORE Web boundary (2026-08-05)

The Site Details tab now preserves the `MiningMethod` enum returned by the API. Provider-driven
Plan Summary content receives that selected enum directly and no longer converts through
`SiteType`:

```text
Mining Methods API
-> MiningMethodTab.MiningMethod
-> selected tab
-> PlanSummaryCard.MiningMethod
-> matching Plan Summary API result/provider
```

`LegacySiteType` remains in CORE Web only as a temporary compatibility bridge:

- Production and Development tabs are suppressed when their corresponding legacy `SiteFeature`
  is disabled. This compensates for method endpoints that are not yet fully Site-aware.
- Existing rig queries and rig cards still divide rigs through Production/Development `SiteType`
  behavior.
- Add Rig, Archived Rigs, Add Users, Security, Manage Drillers, and related child routes still
  include a legacy `SiteType` route value.
- The frontend-owned upload provider still accepts `SiteType` as a compatibility input for the
  existing Production and Development repository workflows. Its stable mining-method key handles
  Cut and Fill without creating another `SiteType`.

`LegacySiteType` must not be used to select Plan Summary metadata or data providers. Cut and Fill
has no legacy `SiteType`; its Plan Summary is selected directly with `MiningMethod.CutAndFill`.

The bridge can be removed when all of the following are true:

1. Backend returns the effective methods enabled for the selected Site and available to the
   current user, including the administrator rule.
2. Rig responses and rig filtering use persisted `Rig.MiningMethod` rather than `SiteType` and
   `RigTypeClassification` as the method source.
3. Rig and Site management routes accept `MiningMethod` or become method-independent.
4. Upload workflows select their implementation from `MiningMethod` without a `SiteType` input.

At that point, remove `LegacySiteType`, `SelectedLegacySiteType`, `ActiveSiteType`, the frontend
legacy-feature filter, and the related conversion helpers together. Removing only one part early
would either expose unsupported tabs or break the remaining rig and route workflows.

### Recommended long-term model

Add an explicit Site-to-MiningMethod entity rather than adding another method-specific flag to `SiteFeatures`:

```csharp
public class SiteMiningMethod
{
    public long SiteId { get; set; }
    public MiningMethod MiningMethod { get; set; }
    public bool Enabled { get; set; }
}
```

Recommended database rules:

- Composite primary key: `(SiteId, MiningMethod)`.
- Foreign key: `SiteId -> Sites.Id`.
- One row per method configured for a Site.
- Keep an explicit `Enabled` value instead of deleting the row when a method is disabled.
- Store method-specific Site configuration in a related record keyed by the same `(SiteId, MiningMethod)` pair, or extend this entity only when ownership remains clear.

Why this is preferred over adding `FeatureType.CutAndFill`:

- A fourth method becomes data plus provider/configuration work, not another legacy feature branch.
- Administrator visibility can be resolved directly from Site data.
- Cut and Fill receives the same enable/disable lifecycle as other methods.
- Disabled methods and their historical configuration can be retained.
- Site Create/Edit can render one dynamic method selector from one source.
- Rig creation can validate `Rig.MiningMethod` against methods enabled for the Site.

Adding `FeatureType.CutAndFill` is acceptable only as a short-lived compatibility bridge. It should not become the permanent extensibility model.

### Recommended API responsibilities

Keep these concepts separate:

```text
GET /MiningMethods
    Catalog: every MiningMethod understood by this API version.

GET /Sites/{siteId}/MiningMethods
    Site capability: methods enabled for this Site.

GET /Sites/{siteId}/MiningMethods/Available
    Effective current-user result:
    - Administrator: all enabled Site methods.
    - Regular user: enabled Site methods intersected with user scope.
```

The exact route names can change, but the three responsibilities should not be combined accidentally.

The preferred page-loading endpoint is the effective current-user endpoint. Backend should read the authenticated identity and role, apply the administrator bypass, and return the final Site-limited result. This avoids making the frontend reproduce authorization rules.

The current frontend role branch should be removed after that endpoint exists.

### Administrator rule

Administrators should bypass `UserSiteMiningMethodScope`, but they must not bypass Site enablement:

```csharp
if (user.IsInRole(Roles.Administrator))
{
    return enabledSiteMethods;
}

return enabledSiteMethods.Intersect(assignedUserMethods);
```

This check belongs on the backend. Frontend role checking controls presentation only and must not be treated as authorization.

### Identity rule

The web token's `sub` claim is a Keycloak ID, while `UserSiteMiningMethodScope.UserId` currently references the application user (`AspNetUsers.Id`). Identity translation must remain centralized on the backend through `KcUserMapping`; the frontend should not rewrite token identity.

### Future Site assignment and Site-list rule

Site access and Mining Method scope are currently maintained separately:

```text
site_id / drill_plan_site_id claim
    Grants access to the Site.

UserSiteMiningMethodScope
    Grants access to specific Mining Methods within that Site.
```

Assigning a regular user to a Site should eventually update both parts as one business operation:

1. Add the Site access assignment that produces the appropriate `site_id` claim in the user's
   token.
2. Insert one `UserSiteMiningMethodScope` row for every Mining Method selected for that user at
   that Site.
3. Validate that each selected method is enabled for the Site.
4. Remove or update both the Site access assignment and its method-scope rows when access changes.

The assignment UI must therefore collect Mining Methods, or explicitly apply a documented
"all enabled Site methods" default. Merely inserting a SiteId into the scope table would be
insufficient because the table requires one row per `(UserId, SiteId, MiningMethod)` combination.

The backend Site-list endpoint used by CORE Web should then return effective Sites for a regular
user using this rule:

```text
Regular-user visible Sites =
    active Sites authorized by site_id or drill_plan_site_id claims
    INTERSECT Sites having at least one UserSiteMiningMethodScope row for the user
    INTERSECT Sites where that scoped Mining Method is enabled
```

This is an intersection, not a replacement for Site authorization. A scope row alone must not
grant access to a Site, and a Site claim without any effective method should not expose a Site page
that can only display "No mining methods are available".

Administrators should continue to bypass `UserSiteMiningMethodScope`. Their Site list should use
active Sites and the agreed administrator Site-access rule, while the methods shown inside each
Site should be all methods enabled for that Site.

This filtering belongs on the backend. CORE Web should receive the final Site list and must not
join token claims, `KcUserMapping`, user scope rows, and Site enablement itself. Because the existing
`GET /Sites` endpoint is also used by non-web clients, engineering should confirm its consumers
before changing its semantics. A dedicated effective-current-user Site endpoint may be safer if
the existing route must preserve tablet or integration behavior.

Before enforcing the intersection, existing assignments must be backfilled. Otherwise users who
currently have valid Site claims but no scope rows would suddenly lose those Sites from the list.
Users must also sign in again after claim assignments change so their token contains the updated
Site claims.

### Migration path

1. Create `SiteMiningMethod` with the composite key and `Enabled` state.
2. Backfill Production rows from `SiteFeatures.FeatureType.Production`.
3. Backfill Development rows from `SiteFeatures.FeatureType.Development`.
4. Treat a missing legacy feature row and `Enabled = false` as disabled during migration.
5. Do not enable Cut and Fill globally by assumption. Support/Product must explicitly decide which Sites receive it.
6. Make Site Create/Edit write the new relationship.
7. Update assigned/effective method services to intersect user scope with enabled Site methods.
8. Update Rig creation and sync validation to use the Site relationship.
9. Keep legacy Production/Development SiteFeatures during a compatibility period.
10. Remove the frontend legacy-feature filter only after all Site-aware backend consumers are cut over.

### Required product/engineering answers

- When a method is disabled, should historical pages remain visible in read-only mode? Current legacy behavior hides the portal entry even though data remains in the database.
- Should a Site be allowed to have zero enabled methods? Existing behavior allows both legacy methods to be disabled, while the newer PRD suggests at least one.
- Which existing Sites should receive Cut and Fill during migration? Defaulting every Site to Cut and Fill would be incorrect.
- Are user methods global and intersected with Sites, or genuinely assigned per user per Site? The current table is per-site, while an earlier design discussion recommended global user methods.
- Should display label and order come from a method catalog/provider, or remain frontend/localization concerns? The enum integer alone is not sufficient UI metadata.

### Acceptance criteria for this decision

This decision is complete when:

- A database query can return the methods enabled for one Site without inspecting users or Rigs.
- An administrator receives all and only methods enabled for that Site.
- A regular user receives the intersection of Site methods and user methods.
- Cut and Fill can be enabled for one Site without appearing for every Site.
- Site Edit and Rig creation validate against the same Site-method source.
- Adding a future method does not require another `FeatureType.{MethodName}` conditional across frontend pages.

## Decision Point 2 - How should a table select its Mining Method provider?

### Context

Provider selection depends on the level that owns the table:

```text
Plan Summary = Site-level table
Shift History = Rig-level table
```

This difference changes how many providers should run for one request.

### Plan Summary rule

A Site can contain several Mining Methods, so the Plan Summary service can call one provider for
each method available to the current user at that Site:

```text
Site
|- Default Development Plan Summary provider
|- Default Production Plan Summary provider
`- Cut and Fill Plan Summary provider
```

The API returns one `TableDataResult` per available method in `TableDataResponse.Results`. The
frontend selects the result matching the active Mining Method tab.

This is appropriate because each result represents different Site-level content. However, the
underlying records must eventually identify their Mining Method. For example, Production and Cut
and Fill currently both query the same `Drive` table without a Mining Method filter, so they
temporarily return the same drives.

### Shift History rule

Shift History belongs to one Rig. A Rig has exactly one `MiningMethod`, so the service should not
try every provider allowed for the user:

```text
RigId
-> load Rig
-> verify that the Rig belongs to the requested Site
-> verify that the user may access the Rig's Mining Method
-> select the provider from Rig.MiningMethod
-> query that provider using RigId
-> return one TableDataResult
```

The user's allowed Mining Methods are an authorization check. They do not decide which Shift
History provider to call. The Rig's `MiningMethod` is the provider-selection source of truth.

Calling all allowed providers would be incorrect because:

- Production, Development, and Cut and Fill providers could all query the same `Shifts` table using
  the same `RigId`.
- It would perform unnecessary database queries.
- An empty result cannot indicate the wrong provider because the correct Rig may simply have no
  shifts yet.
- A new Rig with no shifts must still receive the correct method-specific columns.
- One unfinished or failing provider could prevent the correct provider result from being returned.

`TableDataResponse.Results` can remain a collection for contract consistency, but a Rig-level Shift
History response should contain exactly one result.

### Current implementation and proposal

`ShiftHistoryOverviewService.GetDataAsync` currently loops through every Mining Method returned by
`IUserMiningMethodScopeService`. This follows the Plan Summary pattern and should be changed before
the new Shift History endpoint is connected.

A proposed Rig-level replacement has been added as fully commented code in:

```text
Minnovare.Core.Services/TableData/ShiftHistoryOverview/IShiftHistoryOverviewService.cs
```

It is intentionally inactive so it can be reviewed with the lead. Activating it requires:

1. Injecting `IUnitOfWork` into `ShiftHistoryOverviewService`.
2. Making `RigId` mandatory for Shift History requests.
3. Having the controller derive and validate the Rig's Site.
4. Passing the authenticated administrator state to the service, or completing authorization in
   the controller.
5. Checking regular-user scope against `rig.MiningMethod`.
6. Calling only `_providerFactory.GetProvider(rig.MiningMethod)`.

### Acceptance criteria for this decision

- A Shift History request requires one valid `RigId`.
- The Rig's `MiningMethod` selects exactly one provider.
- A regular user cannot load a Rig whose method is outside their effective Site scope.
- An administrator follows the agreed administrator Site-method rule.
- A Rig with no shifts still returns its correct column metadata and an empty row collection.
- No Shift History provider is called merely to test whether it can find data.
- The response contains exactly one Shift History `TableDataResult`.

## Decision Point 3 - Where should Cut and Fill plans be stored?

### Confirmed current mismatch

The current Cut and Fill upload and Plan Summary workflows do not use the same persisted model:

```text
Cut and Fill upload card
-> temporarily calls UploadDevelopmentHeadingPdf
-> POST /Development/AddDrive/{siteId}
-> inserts DevelopmentDrive
-> uploads DevelopmentDriveDrillInstructions

Cut and Fill Plan Summary provider
-> DriveRepository.GetForSiteAsync(siteId)
-> reads the regular Drive table
```

The upload can therefore report success while the new name does not appear in the Cut and Fill
Plan Summary. Refreshing the card cannot resolve this because the summary provider is querying a
different table. The uploaded record may instead appear in Development content because it was
created as a `DevelopmentDrive`.

Both paths are temporary bridges:

- The frontend reuses the Development PDF upload because no Cut and Fill upload API exists yet.
- The backend Cut and Fill Plan Summary provider reuses the regular `Drive` repository because no
  Cut and Fill plan persistence model exists yet.

### Required long-term decision

Upload, Plan Summary, archived summaries, edit/archive actions, rig assignment, shift details, and
reporting must all refer to the same Cut and Fill plan identity. Two reasonable storage designs are:

#### Option A - Dedicated Cut and Fill tables

Create a `CutAndFillDrive` entity plus related instruction and drilling-detail entities. Add a Cut
and Fill upload endpoint and make every Cut and Fill provider query those records.

Advantages:

- Clear separation from Production rings and Development headings.
- Cut and Fill-specific fields such as section, centre length, and perimeter length have explicit
  ownership.
- Existing Production and Development schemas require less disruption.

Costs:

- More entities, repositories, migrations, endpoints, validation, and action implementations.
- Shared behavior such as names, archive state, and timestamps may be duplicated unless a common
  abstraction is introduced carefully.

#### Option B - Unified plan entity with Mining Method ownership

Introduce or evolve a common plan/drive entity with an explicit `MiningMethod` discriminator, then
store method-specific details in related records. Every provider must filter by both `SiteId` and
`MiningMethod`.

Advantages:

- One common identity and lifecycle for Plan Summary actions.
- Adding another method may require fewer new top-level tables and endpoints.
- Production and Cut and Fill records cannot be confused when providers apply the discriminator.

Costs:

- Larger migration and a greater risk of coupling method-specific fields into one model.
- Existing `Drive` and `DevelopmentDrive` behavior must be reconciled carefully.

### Temporary behavior and guardrails

Until Product and backend engineering choose the storage model:

- Treat the Cut and Fill upload card as UI/workflow testing only.
- Do not interpret its success result as proof that Cut and Fill Plan Summary data was created.
- Do not insert the same logical plan into both `DevelopmentDrive` and `Drive` merely to make the
  UI refresh; that creates duplicate identities and ambiguous edit/archive behavior.
- Do not permanently make the Cut and Fill provider query `DevelopmentDrive`, because that would
  make Cut and Fill indistinguishable from Development.
- Keep comments identifying both temporary reuse paths so they are replaced together.

### Acceptance criteria for this decision

- A Cut and Fill upload creates a record owned explicitly by `MiningMethod.CutAndFill`.
- The Cut and Fill Plan Summary reads that same record without also returning Production or
  Development plans.
- Edit, archive, archived-summary, and upload actions share the same record identity.
- Method-specific Cut and Fill drilling details can be stored without fabricating Production or
  Development records.
- A successful upload becomes visible after refreshing the Cut and Fill Plan Summary.

## Follow-up Decision Points

These should be resolved after the Site binding is agreed:

1. **User assignment granularity:** global per user versus per user per Site.
2. **Disabled-method visibility:** hidden entirely versus historical read-only access.
3. **Method catalog metadata:** source of display labels, ordering, capabilities, and configuration schemas.
4. **Site Create/Edit contract:** whether methods and per-method configuration save with the Site payload or through dedicated endpoints.
5. **Rig validation:** whether a Rig's method can change during creation and whether it becomes immutable after save.
6. **Compatibility removal:** the release in which legacy Production/Development SiteFeatures and frontend filtering can be deleted.
7. **Cut and Fill plan persistence:** dedicated Cut and Fill entities versus a unified plan entity with an explicit Mining Method discriminator.

## Related

- [[2026-07-20 Site Drilling Method Enablement Feature Explained]]
- [[2026-07-07 PRD Front End 15-22 Analysis and Executable Plan]]
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]]
- [[2026-06-24 MiningMethod Merge Missing Migration]]
