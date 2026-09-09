---
date: 2026-08-06
tags: [core-web, archived-plan-summary, metadata-table, mining-method]
status: implemented
---

# Archived Plan Summary Metadata Table Implementation

## Outcome

CORE Web now has one shared mining-method-aware Archived Plan Summary host and card instead of
separate Production and Development table implementations. Small route wrappers preserve each
product's established page title and URL while passing a strongly typed `MiningMethod` into the
shared host. The host consumes the backend-owned `ArchivedPlanSummary` response and uses the
shared `MetaDataTable` for columns, rows, links, sorting, quick filters, metadata filters, progress,
and row actions.

The routes are:

```text
/{siteId}/ArchivedDrives           -> DefaultProduction
/{siteId}/ArchivedHeadings         -> DefaultDevelopment
/{siteId}/ArchivedCutAndFillDrives -> CutAndFill
```

## Goal And Scope

### Goal

Replace the legacy Production and Development archived-table implementations with one
metadata-driven rendering chain, while preserving the dedicated page URL, page title, breadcrumb,
site time, table title, and method-specific behavior users already expect.

### In Scope

- Dedicated archived pages for Production, Development, and Cut and Fill.
- Backend-owned column, row, sort, quick-filter, metadata-filter, progress, and action metadata.
- One shared archived host and one shared archived card.
- Existing unarchive, delete, and Production CSV export behavior.
- Method-specific empty states without falling back to another mining method.
- Existing `BreadcrumbHeader` layout and navigation back to Site Details.
- Release compilation and focused repository verification.

### Out Of Scope

- A new generic backend action-execution endpoint.
- Authentic Cut and Fill plan ownership while Cut and Fill still shares legacy Drive storage.
- Replacing administrator catalog access with explicit Site-to-MiningMethod binding.
- Server-side archived-table filtering and paging.
- Refactoring active Plan Summary or Shift History behavior unrelated to archive navigation.

## Complete Implementation Plan

### Phase 1 - Confirm Contracts And API Boundary

**Purpose:** establish the backend response as the source of truth for archived table structure.

- [x] Use `TableTypes.ArchivedPlanSummary` to validate the response type.
- [x] Deserialize `TableDataResponse` and its per-method `TableDataResult` values through
  `TableDataRepository`.
- [x] Preserve `MiningMethod` on each result so the routed page can select the exact provider
  response it owns.
- [x] Reuse backend `Columns`, `QuickFilters`, `Rows`, totals, and paging information.
- [x] Treat a missing method result as an empty method-specific page, not permission to display a
  different method.

**Done when:** the frontend does not define archived columns or infer which provider result belongs
to the page.

### Phase 2 - Build The Shared Archived Rendering Layer

**Purpose:** share rendering without merging distinct pages into one tabbed/query-string page.

- [x] Add `ArchivedPlanSummary` as a non-routable host component.
- [x] Pass `SiteId`, `MiningMethod`, `HeaderTitle`, and `TableTitle` from each route wrapper.
- [x] Load the site and archived API response once during initial rendering.
- [x] Select the `TableDataResult` whose `MiningMethod` exactly matches the wrapper parameter.
- [x] Add `ArchivedPlanSummaryCard` for quick search, metadata filters, sorting, table rendering,
  confirmation dialogs, mutation actions, and refresh.
- [x] Render rows through the shared `MetaDataTable` instead of method-specific Razor markup.

**Done when:** Production, Development, and Cut and Fill use the same host/card/table chain, while
the host itself owns no route and renders no mining-method tabs.

### Phase 3 - Restore Dedicated Route Ownership

**Purpose:** preserve the established navigation model and avoid query-string enum binding.

- [x] Keep `/{siteId}/ArchivedDrives` as the Production page.
- [x] Keep `/{siteId}/ArchivedHeadings` as the Development page.
- [x] Add `/{siteId}/ArchivedCutAndFillDrives` for Cut and Fill.
- [x] Make each wrapper pass a strongly typed, fixed `MiningMethod` to the shared host.
- [x] Remove `/ArchivedPlans?miningMethod=...` navigation.
- [x] Remove `[SupplyParameterFromQuery] MiningMethod?` from the archived rendering path.
- [x] Update active Plan Summary definitions so each method's archive link targets its dedicated
  route.

**Done when:** archive navigation causes a real page transition, URLs remain bookmarkable, and no
nullable enum is parsed from a query string.

### Phase 4 - Preserve Method-Specific Commands

**Purpose:** retain existing behavior while the backend metadata/action contract is still being
completed.

- [x] Execute Production and Cut and Fill unarchive through `DriveRepository.ArchiveAsync`.
- [x] Execute Development unarchive through `DevelopmentRepository.ArchiveDriveAsync`.
- [x] Retain Production CSV export through `DriveRepository.ExportDriveAsCSV` and `ExportFile`.
- [x] Require confirmation when action metadata requests it.
- [x] Refresh the archived API result after a successful mutation.
- [x] Temporarily suppress unsupported Delete actions according to role and mining method.
- [x] Keep identifier extraction isolated in the card until backend-owned action execution exists.

**Done when:** action labels/order remain metadata-driven and legacy repository branches are
limited to execution compatibility.

### Phase 5 - Match The Existing Archived Page Layout

**Purpose:** preserve the original page header and table composition after moving to shared
components.

- [x] Reuse `BreadcrumbHeader` rather than reproducing its markup.
- [x] Show the back-chevron and site-name link to `/{siteId}`.
- [x] Show `Archived Drives` or `Archived Headings` as the page heading.
- [x] Keep the site time aligned at the right side of the constrained header container.
- [x] Pass `Drives` or `Headings` into the shared card as its table heading.
- [x] Keep the table scroll container, quick filter, Filter, and Clear controls.
- [x] Show a warning when the selected mining method has no archived result.

**Done when:** the dedicated page matches the original header spacing and navigation while using
the generic metadata table below it.

### Phase 6 - Verification And Regression Protection

- [x] Search CORE Web for stale `ArchivedPlans`, `miningMethod=` archive links, and archived
  `[SupplyParameterFromQuery]` usage.
- [x] Build `Minnovare.Core.Web` in Release configuration.
- [x] Run `git diff --check`.
- [x] Verify that no implementation files were staged automatically.
- [ ] Manually verify all three archive links from Site Details in the target environment.
- [ ] Verify Production unarchive, delete, and CSV export with an administrator.
- [ ] Verify Development unarchive and confirm unsupported Delete is absent.
- [ ] Verify Cut and Fill route, empty/data state, and drive-backed temporary actions.
- [ ] Verify regular-user action visibility.

**Done when:** automated checks pass and each route/action scenario is manually confirmed with the
appropriate role and data.

## File Ownership Map

| Area | File | Responsibility |
|---|---|---|
| Production route | `Pages/Blazor/ArchivedDrives.razor` | Fixed Production method, route, and localized titles |
| Development route | `Pages/Blazor/Development/Drives/ArchiveHeading.razor(.cs)` | Fixed Development method, route, and localized titles |
| Cut and Fill route | `Pages/Blazor/ArchivedCutAndFillDrives.razor` | Fixed Cut and Fill method and dedicated route |
| Shared host | `Sites/PlanSummary/Archived/ArchivedPlanSummary.razor(.cs)` | Site/API loading, exact method-result selection, page header, load/empty/error states |
| Shared card | `Sites/PlanSummary/Archived/ArchivedPlanSummaryCard.razor(.cs)` | Search, filters, actions, confirmation, CSV, refresh |
| Shared table | `Shared/TableData/MetaDataTable.razor(.cs)` | Metadata-driven headers, cells, groups, sorting, links, progress, and row actions |
| API repository | `Repositories/TableDataRepository.cs` | Calls and parses the archived TableData endpoint |
| Active-card bridge | `Sites/PlanSummary/ApiPlanSummaryTableDataProvider.cs` | Supplies each method's dedicated archive URL until routes come from backend metadata |

## Runtime State Handling

| State | Expected UI |
|---|---|
| Site/API loading | Loading indicator |
| Site request failure | Standard `BlazorErrorBox` |
| Wrong `TableType` | Archived Plan Summary contract error |
| Selected method absent from response | Method-specific warning; never fall back to another method |
| Valid method result with no rows | Metadata table headers and empty body |
| Action failure | Error box inside the archived card |
| Successful mutation | Reload archived response and keep the same dedicated page |

## Definition Of Done

- [x] Archived navigation opens a new dedicated page rather than changing Site Details content.
- [x] No archive page uses a mining-method query string.
- [x] Production and Development legacy URLs remain valid.
- [x] Cut and Fill has a dedicated archive URL.
- [x] Header uses the standard breadcrumb layout and site-time placement.
- [x] Archived columns and rows come from backend metadata.
- [x] Shared table features remain available.
- [x] Existing supported archive actions remain functional through the temporary bridge.
- [x] Release build succeeds with zero warnings and zero errors.
- [ ] Manual role/action/route verification is completed in the deployment environment.

## Rendering Chain

1. An active `PlanSummaryCard` links to the dedicated archive route for its mining method.
2. The route wrapper passes `SiteId`, `MiningMethod`, `HeaderTitle`, and `TableTitle` into the
   shared `ArchivedPlanSummary` host.
3. No mining-method query string is parsed; the wrapper supplies the enum as a normal component
   parameter.
4. `TableDataRepository.GetArchivedPlanSummaryAsync(siteId)` calls:

   ```text
   GET /api/Table/ArchivedPlanSummary?siteId={siteId}
   ```

5. `TableDataRepository` reuses its generic table-response parser to materialize columns, quick
   filters, rows, totals and page information.
6. The host selects only the `TableDataResult` whose `MiningMethod` matches the wrapper. It never
   falls back to another result under a method-specific page title.
7. `ArchivedPlanSummaryCard` receives that selected result and the localized table title.
8. `MetaDataQuickFilter`, `MetaDataTableFilterModal`, `RowFilterEvaluator`, and `MetaDataTable`
   provide the table UI without mining-method-specific column markup.

## Components Added

```text
Pages/Blazor/Sites/PlanSummary/Archived/
  ArchivedPlanSummary.razor
  ArchivedPlanSummary.razor.cs
  ArchivedPlanSummary.razor.css
  ArchivedPlanSummaryCard.razor
  ArchivedPlanSummaryCard.razor.cs
  ArchivedPlanSummaryCard.razor.css
```

The host owns site/API loading and method-result selection. The card owns client-side filtering
and command execution around the method-blind metadata table. The three route wrappers contain
no table logic.

## Row Actions

The backend supplies `RowActionMetaData`; the frontend does not decide which action labels to
render. Commands currently use a temporary compatibility bridge because the legacy mutation
endpoints are not generic:

| Mining method | Unarchive operation | Identifier |
|---|---|---|
| Production | `DriveRepository.ArchiveAsync(id, false)` | `DriveId` (`long`) |
| Development | `DevelopmentRepository.ArchiveDriveAsync(id, false)` | `HeadingId` (`Guid`) |
| Cut and Fill | `DriveRepository.ArchiveAsync(id, false)` | `DriveId` (`long`) |

Production CSV export continues through `DriveRepository.ExportDriveAsCSV` and the hidden
`ExportFile` component.

The API currently advertises Delete more broadly than the legacy UI supports. CORE Web therefore
temporarily hides Delete from regular users and from Development. Administrators retain Delete
for drive-backed Production and Cut and Fill rows. This frontend filtering should be removed once
backend action metadata includes authorization and Development has a supported delete operation.

Long term, command metadata should target one backend-owned table-action endpoint. That will let
CORE Web remove the mining-method branches and identifier knowledge from the card.

## Route Ownership

The existing Production and Development URLs now render the shared metadata host directly, so
bookmarks remain valid without redirects. Cut and Fill receives its first dedicated archive route:

```text
/{siteId}/ArchivedDrives
/{siteId}/ArchivedHeadings
/{siteId}/ArchivedCutAndFillDrives
```

This removes the legacy `DriveDetailsComponent` and `DevelopmentDriveDetailsComponent` from the
archived rendering path while keeping method-specific page headings and empty-state wording.

## Query-String Failure And Fix

The first implementation used:

```csharp
[SupplyParameterFromQuery]
MiningMethod? RequestedMiningMethod
```

At runtime Blazor raised:

```text
Querystring values cannot be parsed as type Nullable<MiningMethod>.
```

The query-string route was removed. Dedicated route wrappers now pass `MiningMethod` as a normal
component parameter, so Blazor performs no query-string enum conversion and cannot fail before
the shared host loads. This also prevents an invalid query value from showing one method's data
under another method's title.

## Temporary Limitations

- The API returns a `MiningMethod` enum but no page/table title. The small route wrappers
  temporarily own localized headings such as `Archived Drives`, `Drives`, and `Headings`.
- Administrators currently receive the full mining-method catalog until explicit
  Site-to-MiningMethod binding is implemented.
- Cut and Fill still uses the legacy Drive data source, so its table is suitable for UI testing but
  cannot represent authentic Cut and Fill ownership yet.
- Archived data currently uses client-side filtering, sorting and a scroll container. Server-side
  paging can be added when the archived endpoint supplies paging behavior.

## Verification

- The current dedicated-route implementation passed a complete Release build with zero warnings
  and zero errors.
- Focused Release test passed:

  ```text
  TableDataRepositoryTests.GetArchivedPlanSummaryAsync_UsesArchivedEndpointAndParsesResponse
  ```

- Debug test-project build was blocked only because the running CORE Web process held
  `bin/Debug/net8.0/Minnovare.Core.Web.exe`; Release verification used a separate output folder.

## Follow-Up Decisions

1. Add backend-owned table titles so route wrappers do not own presentation labels.
2. Make action visibility role-aware in the API response.
3. Decide whether Development supports permanent deletion.
4. Add one generic table-action endpoint and remove the frontend mutation bridge.
5. Bind archived drives/headings to `MiningMethod`, especially for Cut and Fill.

## Related

- [[2026-07-09 Generic MetaDataTable Component Executable Plan]]
- [[2026-07-09 Generic Metadata Tables Feature Explained]]
- [[2026-07-22 Mining Method Architecture Decision Points]]
