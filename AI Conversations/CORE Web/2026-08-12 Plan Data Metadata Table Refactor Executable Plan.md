---
date: 2026-08-12
updated: 2026-08-13
project: [core-web, core-web-api]
tags: [planning, implementation, plan-data, metadata-table, compact-list, rig-details, mining-methods]
status: revised-plan-pending-implementation
priority: high
---

# Plan Data Compact List Reuse - Revised Executable Plan

> **Target outcome:** Keep the unified metadata-driven Plan Data route and backend providers, but
> replace its conventional column grid with the compact, scrollable item-row presentation already
> established in `RigDetails.razor`. Extract and reuse the visual row without importing Rig Details'
> ring-assignment/unassignment workflow into Plan Data. Preserve selected-row detail content and
> method-specific supporting documents. Plan Data Detail API work remains deferred.

## 1. Scope

### In scope

- Navigate from a Plan Summary row to one mining-method-aware Plan Data page.
- Render Production, Development, and Cut and Fill Plan Data through a reusable compact metadata
  list matching the established Rig Details visual structure.
- Complete the Development and Cut and Fill Plan Data backend providers.
- Make the Plan Data request support the different identifier types used by each method.
- Reuse the existing stable-key row-selection contract in the compact list.
- Extract the status strip, passive comment indicator, title, stacked metrics, progress pill,
  selected state, and optional trailing controls into a presentation-only shared row component.
- Keep `MetaDataTable` available for screens that genuinely require a conventional column grid.
- Preserve the existing selected-row detail content below or beside the table.
- Render method-specific status colors and a passive comment icon without hardcoding mining-method
  rules in the generic table.
- Preserve the Development supporting-document section and make it reusable for Cut and Fill when
  its document ownership is confirmed.
- Keep legacy URLs working during the transition.

### Out of scope

- Replacing the existing selected-row detail panels with the unfinished Plan Data Detail API.
- Implementing the `PlanDataDetail` providers or a new Plan Data Detail controller endpoint.
- Changing Rig Details assignment, unassignment, filtering, or ring-loading behavior.
- Giving the Plan Data checkbox Rig Details' "select for unassignment" meaning.
- Finalizing how Cut and Fill plans and supporting documents are persisted.
- Introducing frontend mining-method calculations or method-specific status mapping.

## 2. Confirmed Current State

### Current pages

| Mining method | Current Plan Data page | Current identifier |
|---|---|---|
| Production | `Pages/Blazor/PlanData/PlanDataDetails.razor` | `DriveId` (`long`) |
| Development | `Pages/Blazor/PlanData/PlanDataDetails.razor` | `HeadingId` (`Guid`) |
| Cut and Fill | `Pages/Blazor/PlanData/PlanDataDetails.razor` | Temporary `DevelopmentDrive` identifier (`Guid`) |

`RigDetails.razor` is not part of the Plan Data navigation chain, but its assigned-ring list is the
approved visual reference. The reusable boundary is its compact row presentation, not its parent
assignment workflow.

### Current backend support

- `TableDataController.GetPlanData` and `PlanDataService` exist.
- Production, Development, and Cut and Fill Plan Data providers are implemented.
- All three profiles return routes, status options, comment flags, filters, and method-specific
  visible metrics.
- Cut and Fill still uses the documented temporary Development storage bridge.
- `PlanDataDetail` profiles exist, but their providers remain unimplemented and no complete page
  rendering chain depends on them yet.

### Current frontend support

The unified Plan Data page currently renders through `MetaDataTable`, which supports:

- metadata-owned visible columns and ordering;
- text, number, date, link, progress, grouped, hover, and row-action rendering;
- route-token resolution through `TableRouteResolver`/`TableRowAccessor`;
- sorting, quick filters, metadata filters, totals, and row-leading content.

It now also exposes stable-key row selection and link/action event isolation. The remaining problem
is presentation: a conventional header-and-column grid produces the wrong Plan Data UI. Rig Details
uses a fixed-height scrollable list with a status strip, stacked metrics, compact progress, and
optional trailing controls.

## 3. Decisions

### 3.1 Use one canonical Plan Data page

Recommended route:

```text
/Sites/{SiteId}/PlanData/{MiningMethodKey}/{PlanId}
```

Examples:

```text
/Sites/99/PlanData/DefaultProduction/123
/Sites/99/PlanData/DefaultDevelopment/8a09c8d8-...-...
/Sites/99/PlanData/CutAndFill/456
```

The route carries `PlanId` as text because the underlying identifier is not the same type for every
method. The page passes the mining method and identifier to the API; it does not infer a method from
legacy `SiteType`.

Legacy routes should remain as thin redirect/wrapper pages until bookmarks and external links have
been migrated.

### 3.2 Distinguish the two route levels

There are two independent metadata routes:

1. **Plan Summary name link -> Plan Data page**
   - Production: unified page with `DriveId`.
   - Development: unified page with `HeadingId`.
   - Cut and Fill: unified page with the Cut and Fill plan identifier.

2. **Plan Data item link -> existing item detail route**
   - Production ring link keeps the existing ring route.
   - Development cut link keeps the existing cut route.
   - Cut and Fill section link keeps its existing section route when that route is confirmed.

Changing Plan Summary routes must not accidentally replace the item routes inside the Plan Data
table.

### 3.3 Keep stable-key row selection, but move it into the compact list

The compact list needs row selection because Production loads additional hole details when a ring is
selected. Development and Cut and Fill can use the same selection seam for their existing detail
content. Selection must remain based on the backend row key rather than row-object reference.

This does **not** require the unfinished `PlanDataDetail` providers. For this iteration:

```text
MetaDataCompactList row selected
    -> PlanDataCard/Page receives selected row
    -> method host extracts the configured selection key
    -> existing repository/component loads the existing detail panel
```

Later, only the last step changes to call the Plan Data Detail endpoint.

### 3.4 Backend owns status meaning; frontend owns semantic styling

The status values differ by mining method:

- Production: Not Started, Drilling, Done, Recalculated.
- Development: Completed, Not Completed.
- Cut and Fill: Completed, Not Completed.

The generic frontend must not translate these values through a mining-method switch. Add a small
metadata contract such as:

```csharp
public class StatusOptionMetaData
{
    public string Value { get; set; }
    public string Label { get; set; }
    public TableStatusTone Tone { get; set; }
}

public enum TableStatusTone
{
    Neutral,
    Warning,
    Success,
    Danger
}
```

The backend profile supplies the available value/label/tone options. Rows supply the stable status
value. CORE Web maps only `TableStatusTone` to approved CSS classes.

The exact Production rule for `Recalculated` must be confirmed before implementation because the
shared `DrillStatus` enum does not currently express that state directly.

### 3.5 Comments use a passive flag, not an action column

The Plan Data comment indicator is informational. It is not a hover tooltip and does not execute a
command. Each row should expose:

```csharp
bool HasComments
```

`CompactDrillingListItem` renders a comment icon only when `HasComments` is true. The icon has no
click handler. A leading status strip derived from the semantic status tone supplies the colour.

This avoids introducing mining-method checks or backend-provided CSS/icon class names.

### 3.6 Supporting documents stay outside the metadata table

Development currently has a supporting PDF/document area that Production does not have. This is
page composition, not Plan Data row metadata.

Extract the existing document logic into a reusable component such as:

```text
PlanInstructionDocuments
```

- Production: do not render it.
- Development: render it using the existing behavior.
- Cut and Fill: render it only after its document storage/API ownership is confirmed.

Until the backend exposes capabilities for this, keep the temporary method-to-document behavior in
one frontend configuration point and mark it for replacement.

### 3.7 Extract presentation, not Rig Details workflow

The shared component must not accept `Ring`, `Drive`, or mining-method-specific domain models. Use a
small presentation contract such as:

```csharp
public sealed class CompactDrillingListItemModel
{
    public object Key { get; init; }
    public string Title { get; init; }
    public string Route { get; init; }
    public TableStatusTone StatusTone { get; init; }
    public bool HasComments { get; init; }
    public IReadOnlyCollection<CompactMetric> Metrics { get; init; }
    public int? Progress { get; init; }
}
```

`CompactDrillingListItem` owns only the visual structure and exposes optional trailing content.

- Rig Details supplies its existing checkbox and external-link control through trailing content.
- Plan Data supplies row selection and an external-link control.
- The Rig Details checkbox continues to mean "select for unassignment" only inside Rig Details.
- Edit/Delete actions remain optional metadata actions; they do not define the compact layout.

### 3.8 Adapt metadata to the compact row without mining-method switches

Add `MetaDataCompactList` as the metadata adapter:

- the first visible link/text identity column becomes the title;
- remaining visible numeric columns become labelled metrics in metadata order;
- a `ProgressBar` column becomes the compact percentage pill;
- the hidden `StatusKey` column and its status options resolve the semantic tone;
- `HasComments` controls the passive comment icon;
- existing route metadata supplies the item destination;
- existing `TableRowAccessor`, `TableRouteResolver`, and formatting behavior are reused.

If the desired label, unit, order, or route is missing, correct the backend profile metadata. Do not
add Production/Development/Cut-and-Fill calculations or label switches to CORE Web.

## 4. Required Contract Changes

### 4.1 Replace the long-only Plan Summary identifier for Plan Data requests

Current `ITableDataRequest.PlanSummaryId` is `long?`, but Development uses a `Guid`. Do not add one
identifier property per mining method. Introduce a Plan Data-specific request contract:

```csharp
public interface IPlanDataTableDataRequest : ITableDataRequest
{
    string PlanId { get; set; }
}
```

Provider parsing:

| Mining method | Parse `PlanId` as |
|---|---|
| DefaultProduction | `long` |
| DefaultDevelopment | `Guid` |
| CutAndFill | its authoritative persisted identifier type |

Each provider must validate both the identifier format and that the selected plan belongs to
`request.SiteId`.

### 4.2 Add shared Plan Data interaction row values

Introduce a small common schema for presentation interactions, for example:

```csharp
public interface IPlanDataInteractionRowSchema
{
    string StatusKey { get; }
    bool HasComments { get; }
}
```

Method-specific row schemas retain their domain-specific IDs and displayed values.

### 4.3 Reuse the generic selection contract in `MetaDataCompactList`

Required parameters:

```csharp
[Parameter] public EventCallback<object> OnRowSelected { get; set; }
[Parameter] public string SelectionKey { get; set; }
[Parameter] public object SelectedKey { get; set; }
```

Rules:

- Compare rows by `SelectionKey` through `TableRowAccessor`, not object reference.
- Apply selected-row styling without changing row dimensions.
- Stop propagation from links and row actions so Edit/Delete/navigation does not also select.
- Keep selection optional so the component can also render read-only compact lists.

Selection keys:

| Mining method | Selection key |
|---|---|
| Production | `RingId` |
| Development | `CutId` |
| CutAndFill | `SectionId` |

## 5. Target Rendering Chain

```text
User clicks a Plan Summary name
    -> backend-owned Plan Summary RouteTemplate
    -> unified PlanData page receives SiteId + MiningMethod + PlanId
    -> PlanDataRepository calls GET api/Table/PlanData
    -> TableDataController builds IPlanDataTableDataRequest
    -> PlanDataService authorizes mining-method access
    -> PlanDataProviderFactory selects exactly one provider
    -> provider validates PlanId + SiteId ownership
    -> provider returns columns, rows, filters, totals, and table metadata
    -> PlanDataCard renders MetaDataCompactList
    -> MetaDataCompactList maps backend metadata to CompactDrillingListItem
    -> optional row selection loads the existing detail panel
    -> optional method-specific supporting documents render outside the table
```

The page displays what the selected provider returns. It must not attempt multiple providers or infer
the provider from row data.

## 6. Implementation Plan

### Phase 1 - Confirm contracts and routes (completed baseline)

**Goal:** Remove the ID and navigation ambiguity before component work begins.

- [x] Confirm the canonical Plan Data route and legacy redirect period.
- [x] Use the temporary Development storage identifier for Cut and Fill until authoritative storage exists.
- [x] Add `IPlanDataTableDataRequest.PlanId` as a string.
- [x] Update `TableDataController.GetPlanData` to accept `planId` as text.
- [x] Update `PlanDataService` and provider interface to use the specialized request.
- [x] Use `TableTypes.PlanData` instead of duplicated table-type string literals.
- [x] Align Plan Data authorization with the current regular-user/admin mining-method policy.
- [x] Update Plan Summary profile routes to the unified Plan Data page.

**Done when:** A Production long ID and Development GUID can reach the correct provider without
legacy `SiteType` conversion.

### Phase 2 - Complete backend Plan Data providers (completed baseline)

**Goal:** Return usable table responses for all three mining methods.

- [x] Keep Production behavior while adding site ownership validation.
- [x] Implement `DefaultDevelopmentPlanDataProvider` from the existing Development data source.
- [x] Implement the documented temporary Cut and Fill Development-storage bridge.
- [x] Normalize row keys with `nameof()` against the shared row schemas.
- [x] Return quick filters and metadata filters in `TableDataResult`.
- [x] Return stable status values, status options, and `HasComments`.
- [x] Correct inconsistent profile routes, labels, key casing, and action metadata.
- [x] Document the temporary Production `Recalculated` derivation.

**Done when:** Each provider returns only its method's rows and can reject a plan from another site.

### Phase 3 - Extract the reusable compact item row

**Goal:** Reproduce the approved Rig Details row visually while keeping behavior caller-owned.

- [x] Add `CompactDrillingListItem.razor` and isolated CSS under the shared Blazor components.
- [x] Reproduce the 40px status strip, passive comment icon, title, stacked metric labels/values,
      compact percentage pill, selected background, and row spacing from Rig Details.
- [x] Expose optional route, selection callback, and trailing-content fragment.
- [x] Keep links and trailing controls from bubbling into row selection.



### Phase 4 - Add the compact metadata adapter and switch Plan Data

**Goal:** Render backend-owned Plan Data metadata through the approved compact row structure.

- [x] Add `MetaDataCompactList.razor` to map `TableDataResult` into compact item models.
- [x] Reuse `TableRowAccessor`, `TableRouteResolver`, status options, and shared value formatting.
- [x] Switch `PlanDataDetails.razor` from `MetaDataTable` to `MetaDataCompactList`.
- [x] Replace the single `All` select with the established Rig Details comments-only checkbox and
      multi-select drilling-status `FilterModal`.
- [x] Preserve the status legend, loading/error handling, breadcrumbs, and
      stable-key selected-row behavior.
- [x] Preserve Production's existing selected-ring hole panel.
- [x] Preserve Development/Cut and Fill embedded supporting documents.
- [x] Use a fixed-height scrollable left list and the established `col-lg-5` width for all methods.
- [x] Render an external-link control for the item route; do not render an unassignment checkbox.
- [x] Correct missing labels, units, metric order, or routes in backend profile metadata rather than
      adding mining-method-specific display calculations in CORE Web.

**Done when:** Production visually matches the compact Rig Details list, all three methods use the
same compact renderer, and Production's selected-ring detail interaction has not regressed.

### Phase 5 - Compatibility, cleanup, and verification

**Goal:** Keep the new compact renderer narrow, tested, and compatible with existing routes.

- [x] Convert legacy Production and Development Plan Data routes to redirects/wrappers.
- [ ] Remove only the superseded Plan Data grid composition after compact-list parity is verified.
- [ ] Keep `MetaDataTable` unchanged for existing grid consumers.
- [ ] Keep existing detail repositories/components still used by the selection bridge.
- [x] Add focused component tests and run the Core Web build.
- [ ] Manually verify the scroll area, row selection, item link, Production hole panel, and responsive
      behavior at the widths represented by the approved screenshot.
- [ ] Update the implementation record with the final component boundary and verification results.

**Done when:** There is one compact Plan Data list renderer, Rig Details consumes the shared visual
row, `MetaDataTable` consumers are unaffected, and no legacy URL is broken.

## 7. Suggested Code Areas

### CORE Web

| Area | File/module | Expected change |
|---|---|---|
| Shared compact row | `Pages/Blazor/Shared/CompactDrillingListItem.*` | Extracted Rig Details visual row with optional trailing content |
| Metadata compact adapter | `Pages/Blazor/Shared/TableData/MetaDataCompactList.*` | Map metadata columns/rows into compact item presentation |
| Existing grid | `Pages/Blazor/Shared/TableData/MetaDataTable.*` | Remain the conventional grid renderer; avoid compact-mode branches |
| Route resolution | `Pages/Blazor/Shared/TableData/TableRouteResolver.cs` | Reuse existing token resolution; no method-specific routes |
| Rig Details | `Pages/Blazor/Rigs/RigDetails.*` | Consume the shared row while retaining assignment behavior |
| Unified page | `Pages/Blazor/PlanData/PlanDataDetails.*` | Replace grid with compact adapter; preserve filters/details/documents |
| Production legacy page | `Pages/Blazor/Drives/DriveDetails.*` | Redirect/wrapper or existing-detail bridge |
| Development legacy page | `Pages/Blazor/Development/Drives/DevelopmentDriveDetails.*` | Redirect/wrapper and document extraction |
| API client | `Repositories/` or `Pages/Blazor/PlanData/` provider | Load `TableDataResponse` from Plan Data endpoint |

### CORE API

| Area | File/module | Expected change |
|---|---|---|
| Controller | `Controllers/TableDataController.cs` | String `planId`, specialized request |
| Request contract | `Contracts/TableDataRequest.cs` | Add Plan Data-specific identifier contract |
| Service/factory | `Services/TableData/PlanData/` | Authorization and one-provider dispatch |
| Providers | `Services/TableData/PlanData/Providers/` | Implement Development/Cut and Fill, validate ownership |
| Profiles | `Services/TableData/PlanData/Profiles/` | Routes, statuses, comments, filters, actions |
| Row schemas | `Contracts/TableData/IRowSchemas.cs` | Interaction values and corrected keys/types |

## 8. Verification Plan

### Shared compact-list tests

- [ ] `CompactDrillingListItem` renders title, ordered metrics, optional progress, status tone, and
      passive comment indicator.
- [ ] Optional trailing content renders without changing the base row dimensions.
- [ ] Row selection emits the configured stable key and survives data reloads.
- [ ] Clicking the item link or trailing control does not also select the row.
- [ ] Unknown status values use neutral styling.
- [ ] Zero progress renders an empty track; nonzero and 100% values use the established progress
      presentation.
- [ ] `MetaDataCompactList` maps title, metrics, progress, route, status, and comments from metadata
      without mining-method switches.
- [ ] Existing `MetaDataTable` tests remain unchanged and passing.

### Backend provider tests

- [ ] Production parses a long ID and returns the expected rings.
- [ ] Development parses a GUID and returns the expected cuts.
- [ ] Cut and Fill parses its authoritative ID and returns the expected sections.
- [ ] Invalid ID formats return a controlled validation response.
- [ ] A plan belonging to another site is rejected.
- [ ] Quick filters, status options, row values, and action routes are present.
- [ ] Regular-user and administrator mining-method authorization is consistent with Plan Summary.

### End-to-end checks

- [ ] Plan Summary links open the correct method and plan.
- [ ] Production displays rings and preserves the existing selected-ring details.
- [ ] Production ring rows match the approved Rig Details compact layout rather than a header grid.
- [ ] Development displays cuts and preserves the supporting-document section.
- [ ] Cut and Fill displays sections and its supported supplementary content.
- [ ] Status labels/colors and comment icons match product requirements.
- [ ] Rig Details assignment/unassignment behavior is unchanged after consuming the shared row.
- [ ] Plan Data does not display or inherit Rig Details' unassignment checkbox behavior.
- [ ] Legacy URLs still navigate successfully.
- [ ] Browser back/forward behavior does not lose the selected method or plan.
- [ ] Release build and focused tests pass with warnings treated as errors.

## 9. Risks and Open Decisions

| Risk/decision | Why it matters | Required action |
|---|---|---|
| Mixed plan identifier types | Current long-only request cannot represent Development | Approve string `PlanId` contract |
| Cut and Fill persistence | Current data may still reuse Production/Development storage | Confirm authoritative entity and ID before provider implementation |
| Production `Recalculated` | Not represented directly by current `DrillStatus` enum | Confirm derivation and precedence with the lead/product team |
| Status metadata contract | Required for method-independent colors and filters | Agree semantic tone/options contract |
| Supporting document ownership | Development has working behavior; Cut and Fill ownership is unclear | Confirm API/storage source before enabling Cut and Fill documents |
| Administrator authorization | Plan Data must match tabs and Plan Summary | Align `PlanDataService` with the chosen admin policy |
| Existing detail behavior | A full rewrite risks Production regression | Keep `OnRowSelected` bridge; defer PlanDataDetail API |
| Checkbox semantics | Rig Details uses the checkbox for unassignment, while Plan Data uses row selection | Keep checkbox in Rig Details trailing content only |
| Over-generalizing `MetaDataTable` | A compact mode would add competing layout rules to an established grid | Use a sibling `MetaDataCompactList` adapter |
| Frontend method switches | Hardcoded field mappings would undermine backend-owned presentation | Map by metadata roles/types/order and correct profiles when metadata is insufficient |
| Shared CSS drift | Copying Rig Details CSS would create two visually similar implementations | Keep row CSS isolated with the shared compact component |
| Legacy route consumers | Bookmarks and links may still target old pages | Keep redirects/wrappers during migration |

## 10. Definition of Done

- [ ] One mining-method-aware Plan Data page renders all supported methods.
- [ ] Plan Summary route metadata navigates to that page for Production, Development, and Cut and Fill.
- [ ] Development GUID identifiers work without conversion to a numeric ID.
- [ ] All three Plan Data providers return real metadata and rows; no provider throws
      `NotImplementedException`.
- [ ] Plan Data renders through one metadata-backed compact-list adapter with no mining-method
      presentation branches.
- [ ] Rig Details and Plan Data reuse the same compact visual row component.
- [ ] The shared component contains no `Ring`, `Drive`, assignment, or unassignment logic.
- [ ] Production rows show the status strip, comment indicator, item name, Completed/Outstanding
      values, compact progress, selected state, and item route in the approved layout.
- [ ] Development and Cut and Fill use the same row structure with metrics supplied by their profiles.
- [ ] Existing `MetaDataTable` grid consumers remain visually and behaviorally unchanged.
- [ ] Production's current selected-ring detail behavior still works.
- [ ] Status labels, colors, filters, and comment indicators are metadata-driven.
- [ ] Development supporting documents remain available.
- [ ] Plan Data Detail API remains explicitly deferred and is not required for this release.
- [ ] Old Plan Data URLs remain usable during the transition.
- [ ] Focused tests and the Release build pass.

## 11. Deferred Follow-up

After this Plan Data migration is stable, implement the separate Plan Data Detail metadata chain:

```text
selected Plan Data row
    -> PlanDataDetail endpoint
    -> method provider
    -> metadata-driven detail table/panel
```

That later task can replace the existing selection bridge without changing the unified Plan Data
route or the stable-key selection contract used by `MetaDataCompactList`.

## 12. Implementation Record - 2026-08-13

### Completed

- Added `IPlanDataTableDataRequest`/`PlanDataTableDataRequest` with a string `PlanId`.
- Updated `GET api/Table/PlanData` to accept `siteId`, `planId`, and `miningMethod` and to pass
  the authenticated administrator state into `PlanDataService`.
- Aligned Plan Data authorization with the mining-method tabs: administrators temporarily use the
  complete method catalog; regular users use `UserSiteMiningMethodScope`.
- Updated `IPlanDataProvider` and `PlanDataService` to dispatch exactly one provider.
- Completed Production, Development, and Cut and Fill providers with PlanId parsing and site
  ownership validation.
- Kept Cut and Fill on the documented temporary `DevelopmentDrive`/
  `DevelopmentDriveDrillingDetails` bridge. The current model has no centre/perimeter split, so the
  total is temporarily returned as centre length and perimeter remains zero.
- Added `IPlanDataInteractionRowSchema` with `StatusKey` and `HasComments`.
- Added backend-owned status options using semantic `TableStatusTone` values.
- Updated all Plan Data profiles with corrected row keys, routes, quick status filters, status
  options, and existing Edit/Delete routes where available.
- Updated Plan Summary routes to the canonical page:
  `/Sites/{SiteId}/PlanData/{MiningMethod}/{PlanId}`.
- Added optional stable-key row selection to `MetaDataTable`; link and action clicks stop event
  propagation so they do not also select the row.
- Added metadata-driven leading status colors and passive comment indicators.
- Extended `MetaDataQuickFilter` to render a select when the API supplies status options while
  preserving the existing text-input behavior.
- Added `TableDataRepository.GetPlanDataAsync`.
- Added the unified `PlanDataDetails` route/page. Production preserves selected-ring hole details;
  Development and Cut and Fill compose the existing PDF document behavior beneath the table.
- Made `DevelopmentDriveDetails` embeddable so the unified page reuses its documents without the
  duplicate legacy drilling table or header.
- Converted legacy Production and Development Plan Data routes into compatibility redirects.
- Added focused source tests for row selection and link click propagation.

### Deliberately deferred

- The Plan Data Detail metadata endpoint/providers remain deferred. Production still uses its
  existing expanded Drive object for selected-ring holes.
- Cut and Fill needs an authoritative plan/section entity before centre and perimeter length can be
  independently populated.
- Cut and Fill documents temporarily reuse Development document ownership.
- Status text is backend-owned, but localization of metadata labels remains a separate concern.

### Verification status

Builds and tests were intentionally not run on 2026-08-13 at Allen's request. The implementation
received a source-only consistency and `git diff --check` review. Run the Release build and focused
tests before commit/PR.

## 13. Plan Revision - Compact Rig Details UI Reuse (2026-08-13)

The first unified-page implementation produced a conventional metadata grid. Product feedback
confirmed that the required Plan Data list is the compact assigned-ring presentation in
`Rigs/RigDetails.razor`.

The route, backend providers, metadata contracts, filters, selection bridge, detail panel, document
bridge, and compatibility redirects remain valid baseline work. The pending revision is deliberately
limited to the list presentation boundary described in Phases 3-5:

```text
RigDetails domain rows --------------------+
                                            -> CompactDrillingListItem
TableDataResult -> MetaDataCompactList -----+
```

Rig Details remains the owner of checkbox/unassignment behavior. Plan Data remains the owner of
selected-item detail behavior. Both reuse the same status/comment/title/metrics/progress row UI.

No implementation changes were made as part of this planning revision.

### Implementation completion update - 2026-08-13

- Added the presentation-only `CompactDrillingListItem` with isolated styling.
- Added `MetaDataCompactList`, using existing metadata access, route resolution, status options, and
  shared value formatting.
- Migrated both Rig Details ring rows and Plan Data rows to the shared visual component.
- Kept Rig Details checkbox/unassignment behavior in its parent component only.
- Kept Development and Cut and Fill compact lists at the same `col-lg-5` width as Production.
- Replaced the Plan Data `All` select with the Rig Details comments-only checkbox and drilling-status
  `FilterModal` presentation.
- Removed repository-local isolated build products after verification.
- Verification: Core Web build passed with 0 warnings/errors; focused bUnit tests passed 17/17; API
  services build passed with 0 errors and four unrelated existing warnings.
