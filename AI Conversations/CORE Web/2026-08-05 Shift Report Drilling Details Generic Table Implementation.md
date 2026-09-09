---
date: 2026-08-05
tags: [core-web, shift-report, metadata-table, implementation]
status: implemented
---

# Shift Report Drilling Details Generic Table Implementation

## Outcome

The method-specific drilling-details table at the bottom of the Shift Report now renders through the shared `MetaDataTable` component. CORE Web sends only `siteId`, `rigId`, and `shiftId`; CORE API reads the rig's persisted `MiningMethod`, selects one provider, and returns the appropriate columns, actions, routes, and rows.

This change replaces the hardcoded Production `Holes` table and Development `Drilling Details` table in `ShiftDetails.razor`. It does not refactor the rest of the Shift Report page.

## Scope Implemented

- Added a frontend repository call for `GET /api/Table/ShiftReportDrillingDetails`.
- Added a transport-independent Shift Report drilling-details provider interface.
- Added the API-backed provider implementation.
- Added `ShiftReportDrillingDetailsCard` as the feature-level wrapper around `MetaDataTable`.
- Registered the provider in dependency injection.
- Replaced both hardcoded method-specific tables in `ShiftDetails.razor` with the new card.
- Preserved the existing Targets Set modal through a temporary callback bridge.
- Updated `TextActionList` so a command attached to a data column displays the row value, such as the Targets Set count, rather than the generic action label.
- Added tests for data-column text actions and ordinary action-column fallback labels.
- Added backend-owned totals rows for Production, Development, and Cut and Fill, and forwarded `TotalsRow` through the Shift Report card.

## Rendering Chain

```text
ShiftDetails.razor
    -> ShiftReportDrillingDetailsCard
        -> IShiftReportDrillingDetailsTableDataProvider
            -> ApiShiftReportDrillingDetailsTableDataProvider
                -> TableDataRepository.GetShiftReportDrillingDetailsAsync
                    -> GET /api/Table/ShiftReportDrillingDetails
                        -> controller validates Site, Rig, Shift, and access
                        -> controller copies Rig.MiningMethod into TableDataRequest
                        -> ShiftReportDrillingDetailsService selects exactly one provider
                        -> provider returns TableDataResult
                <- API response is materialized into ColumnMetaData and JObject rows
        -> MetaDataTable renders metadata without mining-method conditionals
```

The frontend does not infer the mining method from legacy `SiteType`, rig type, route text, or row shape.

## Frontend Files

### New

- `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftReport/IShiftReportDrillingDetailsTableDataProvider.cs`
- `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftReport/ApiShiftReportDrillingDetailsTableDataProvider.cs`
- `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftReport/ShiftReportDrillingDetailsCard.razor`
- `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftReport/ShiftReportDrillingDetailsCard.razor.cs`
- `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftReport/ShiftReportDrillingDetailsCard.razor.css`

### Updated

- `Minnovare.Core.Web/Repositories/TableDataRepository.cs`
- `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftDetails.razor`
- `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftDetails.razor.cs`
- `Minnovare.Core.Web/Pages/Blazor/Shared/TableData/MetaDataTable.razor`
- `Minnovare.Core.Web/Pages/Blazor/Shared/TableData/MetaDataTable.razor.cs`
- `Minnovare.Core.Web/Startup.cs`
- `Minnovare.Core.Web.Test/Blazor/Shared/TableData/MetaDataTableTests.cs`

## Targets Set Command Flow

Development and Cut and Fill metadata define `Targets Set` as `ColumnType.TextActionList` with action identifier `ShiftReport.ViewTargetsSet`.

1. `MetaDataTable` displays the row's `TargetsSet` value, for example `3`.
2. Clicking the value emits a `TableRowActionRequest`.
3. `ShiftReportDrillingDetailsCard` verifies the action identifier.
4. It resolves `DrillingDetailId` using the action's `ParameterMap`.
5. It invokes `OnViewTargetsSet` on `ShiftDetails`.
6. `ShiftDetails` finds the matching legacy Development drilling-detail record and opens the existing modal.

The modal bridge is temporary because the table API currently returns only the target count, not target angle/time records. A dedicated target-details endpoint should eventually remove the duplicate legacy Development request.

## Compatibility Bridges Still Present

### Legacy detail loaders

`LoadHolesForShift` and `LoadDrivesForShift` remain active even though their Razor tables were removed. They still support:

- per-driller drilled-length calculations in the report shell;
- the existing Development Targets Set modal;
- report behavior outside the metadata table.

This temporarily means the page loads both legacy report data and the new table-data response.

### Legacy SiteType route

The route remains:

```text
/Sites/Rigs/{RigId}/Shifts/{ShiftId}/{RigSiteType}
```

`ShiftDetails.OnInitializedAsync` still gates the entire report shell through `SiteType.Production` or `SiteType.Development`. The new drilling-details card itself does not use `SiteType`, but a Cut and Fill route cannot initialize the surrounding report shell until that page-level gate is replaced by rig `MiningMethod` or a mining-method-neutral report endpoint.

## Current Mining-Method Behavior

### Default Production

- API returns Production hole/ream rows and Production column metadata.
- Drive, Ring, Hole, comments, numeric fields, and route actions render generically.
- The provider returns one overall totals row for `DRILLED LENGTH Planned` and `DRILLED LENGTH Actual`.
- The legacy table inserted a subtotal after each Drive group; that grouped-subtotal behavior is not represented by the current contract.

### Default Development

- API returns Development drilling-detail rows and Development metadata.
- Targets Set opens the existing modal through the compatibility bridge.
- The provider totals `Expected Advance`, `Drilled Holes`, and `DRILLED LENGTH`.
- Grade remains empty because its required persisted source and format conversion are not yet available in the provider.

### Cut and Fill

- API returns the complete Cut and Fill column contract with an empty row set and a zero totals row for Centre, Perimeter, and Total drilled length.
- The provider deliberately does not reuse Production or Development operational rows.
- Authentic rows require a persisted Cut and Fill drilling-detail read model containing centre/perimeter lengths and section state.
- The surrounding Shift Report page must also stop depending on legacy `SiteType` before a Cut and Fill report route can render end to end.

## Remaining Decisions and Risks

| Area | Current behavior | Required follow-up |
|---|---|---|
| Production grouped subtotals | One overall totals row is implemented; legacy per-drive subtotal rows were removed with the hardcoded table | Add a grouping/subtotal contract if product still requires a subtotal after every Drive group |
| Date/time ownership | `MetaDataTable` formats timestamps with `UserTimeService` | Confirm whether Shift Report detail times must use user timezone or site timezone |
| Targets Set details | Modal reads the legacy Development collection | Add a target-details API by `DrillingDetailId` |
| Cut and Fill rows | Metadata only, no rows | Add a dedicated persisted model and provider query |
| Cut and Fill page shell | Blocked by legacy `SiteType` parsing | Refactor ShiftDetails initialization and breadcrumbs around `MiningMethod` |
| Legacy duplicate requests | Required for calculations and modal data | Move report-shell calculations and target details to API-owned contracts |
| Localization | Backend currently supplies English labels | Decide whether API returns localization keys or localized labels |
| Authorization-sensitive actions | Frontend renders actions returned by API | Backend profiles/services should omit actions the current user cannot perform |

## Verification

Completed on 2026-08-05:

```text
dotnet build Minnovare.Core.Web.Test/Minnovare.Core.Web.Test.csproj
  --no-restore --configuration Release /warnaserror

Result: Build succeeded, 0 warnings, 0 errors.
```

```text
dotnet test Minnovare.Core.Web.Test/Minnovare.Core.Web.Test.csproj
  --no-build --no-restore --configuration Release
  --filter FullyQualifiedName~MetaDataTableTests

Result: 11 passed, 0 failed.
```

The npm audit executed by the existing Release build reported 21 dependency vulnerabilities. They pre-exist this feature and were not changed as part of this implementation.

## Recommended Cleanup Order

1. Manually verify Production and Development report tables against known shifts.
2. Decide and implement Production per-drive subtotal metadata.
3. Confirm site-timezone versus user-timezone behavior.
4. Add a target-details endpoint and remove the Development modal bridge.
5. Move per-driller report calculations to an API-owned report summary contract.
6. Remove the legacy hole and Development drilling-detail loaders.
7. Refactor the Shift Report route and shell from `SiteType` to `MiningMethod`.
8. Add the Cut and Fill persisted read model and return real rows.

## Related

- [[2026-07-09 Generic MetaDataTable Component Executable Plan]]
- [[2026-07-09 Generic Metadata Tables Feature Explained]]
- [[2026-07-22 Mining Method Architecture Decision Points]]

