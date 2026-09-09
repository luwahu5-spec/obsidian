---
date: 2026-07-09
updated: 2026-07-09
source: Claude Code
project: core-web / core-web-api
tags: [domain-knowledge, shift-history, plan-summary, shift-report, table-data-contracts, core-refactor-2.23, mining-methods]
---

# Generic Metadata Tables — current logic and metadata-driven target

> **Takeaway:** All three "method-varying" tables today decide their columns with frontend
> `SiteType.Production/Development` conditionals while calling method-agnostic APIs that return the
> full superset model — the backend never says what to display. The Shared 2.23 submodule already
> contains the complete target contract (`TableDataRequest` → `TableDataResult` with per-column
> metadata, row schemas drafted for all 3 methods including Cut and Fill) but **no provider/endpoint
> exists yet**. The one non-obvious thing: the contract as drafted cannot express three things the
> current UI actually does — totals rows, dynamic diameter column groups, and server-side paging —
> and it leaves label localization ownership undecided. Those are the gaps to negotiate with backend
> *before* the generic component is built.

## The business reality being modelled

Production and Development are different drilling methods with genuinely different physics, so their
tables measure different things: Production tracks drilled length *per hole diameter* (a rig drills
several ream sizes in one shift) and downhole totals; Development tracks *expected advance* of a cut
vs actual drilled length. Plan summaries differ the same way: a Production Drive has
completed/outstanding meters and progress %, a Development Heading only has created/last-updated.
Cut and Fill (3rd method, coming) adds centre/perimeter hole splits. The refactor goal: a 4th method
should cost **configuration only, no new component code** (PRD §22 success criterion).

## Where the method-conditional logic lives today

### 1. Shift history list — ONE page shared by both methods
- Route carries the method: `/Sites/Rigs/{RigId}/{RigSiteType}/Shifts` — `ShiftList.razor:1`; parsed
  into `_siteType` via `SetSiteType(RigSiteType)` at `ShiftList.razor.cs:162`.
- **14 `SiteType` conditionals in the markup**, all column visibility/labels:
  - Diameter column group (Production only) with a two-row header — one sub-column per diameter:
    `ShiftList.razor:163-172` (header), `:192-200` (sub-header row), `:292-312` (cells).
  - "Total" (Prod) vs "Total Expected Advance" (Dev) — same `<th>`, label swap: `ShiftList.razor:174`.
  - "Total Downhole" Prod-only `:175-178`; "Total Length Drilled" Dev-only `:179-182`.
  - "Drives" (Prod) vs "Heading" (Dev) label: `ShiftList.razor:187`.
  - The totals row repeats every branch again: `ShiftList.razor:361-412`.
- **Dataset is method-agnostic**: `POST /Shifts/{rigId}/GetByRig` (server-paged,
  `ShiftRepository.cs:87-90`) then poll `GetShiftSummary`/`GetDrillerShiftPage` by pageId with a
  10×1s retry loop (`ShiftList.razor.cs:346-414`). The API returns the full `ShiftSummary` superset
  for every method; the frontend picks what to show.
- Diameter sub-columns are computed **client-side per page**: rig/site ream sizes ∪ diameters found
  in `summary.DrillLengths`, sorted desc (`ShiftList.razor.cs:375-399`). Columns depend on the data.
- Fixed shell (method-independent): date-range picker, driller `FilterModal`, shift-type select,
  pagination, page size, CSV export (`ExportShiftList`, `ShiftList.razor.cs:187-216`).

### 2. Plan summary — TWO near-duplicate components, picked by SiteDetails
- `SiteDetails.razor:105-128`: `ActiveSiteType == Production` → `DriveDetailsComponent`, else
  `DevelopmentDriveDetailsComponent`. `ActiveSiteType` comes from the selected mining-method tab's
  `LegacySiteType` bridge (`SiteDetails.razor.cs:74-76`, `GetLegacySiteType` at `:319`).
- **Production** (`DriveDetailsComponent.razor`): 6 columns (Drive link / Created / Last Drilled /
  Completed / Outstanding / Progress bar) + actions Edit | Archive | Export CSV (+ Delete/Unarchive
  in archived mode). Data: `GET /drives/GetDrivesForSite` → `DriveDisplayObject`
  (`DriveRepository.cs:223-226`). Filter modal "Drive Filter" has **6 fields** (name, created,
  drilled, progress, completed, outstanding), each a `FilterInput` = operator select + typed input
  (`DriveDetailsComponent.razor:15-79`), applied client-side via the Specification pattern
  (`DriveDetailsComponent.razor.cs:245-285`).
- **Development** (`DevelopmentDriveDetailsComponent.razor`): 3 columns (Heading link / Created /
  Last Updated) + Edit | Archive. Data: `GET /Development/GetDrivesForSite` →
  `DevelopmentDriveDisplayModel` (`DevelopmentRepository.cs:24-27`). Filter modal "Heading Filter"
  has **3 fields** (`DevelopmentDriveDetailsComponent.razor:14-44`), same Specification apply at
  `.razor.cs:149-175`.
- Both duplicate: quick-search box, per-column `SortState` machine + Dynamic-LINQ `OrderBy` string
  (`DriveDetailsComponent.razor.cs:157-243` vs `DevelopmentDriveDetailsComponent.razor.cs:105-147`),
  Clear, Refresh, archive/unarchive flows. Everything is **in-memory** — full list loaded once, no
  server paging. This duplication is exactly what the filter-modal metadata (`IsFilterable` +
  `FilterOperators`) is meant to collapse.

### 3. Shift report page (the "history report") — fixed shell + method-specific detail table
- Route: `/Sites/Rigs/{RigId}/Shifts/{ShiftId}/{RigSiteType}` — `ShiftDetails.razor:1` (this is what
  the shift-history "Evening ↗" link opens).
- 9 `SiteType` branches: handover notes Prod-only (`:53`), left/right shift-info column swap
  (`:73-95`), "Total Drilled Length" vs "Total Expected Advance" (`:96`, `:109`), "Total Length
  Drilled" Dev-only (`:112`), Edit button Prod-only (`:609`).
- The genuinely method-specific part is the bottom table: Production renders a "Holes" table
  (`ShiftDetails.razor:378-464`), Development a "Drilling Details" table (`:465+`) — matching the
  architecture-plan rule: keep the shell fixed, make only the drilling-details section configurable.

## The target contract (already in Shared 2.23, no backend provider yet)

All under `Minnovare.Core.Shared/Contracts/`:
- `TableDataRequest` — SiteId, RigId?, PlanSummaryId?, TableType (string), MiningMethod, UserId
  (`TableDataRequest.cs`). **No paging/sort/filter fields.**
- `TableDataResult` — MiningMethod + TableType + `Columns` (IColumnMetaData) + `Rows`
  (`IReadOnlyCollection<object>`); `TableDataResponse` wraps multiple results
  (`TableData/TableDataResult.cs`).
- `ColumnMetaData` — Key, IsVisible, IsSortable, IsFilterable, Label, Index, Format, UnitSuffix,
  ColumnType (Text/Integer/Decimal/DateTime/Link/ProgressBar/TextActionList/ButtonActionList),
  LinkType + RouteTemplate, FilterOperators, Actions (`TableData/ColumnMetaData.cs`).
- `RowActionMetaData` — Route/Command/Download + ActionId + ParameterMap + confirmation fields
  (`TableData/RowActionMetaData.cs`); ActionIds enumerated in `TableActionIdentifiers.cs`
  (PlanSummary.Edit/Archive/Unarchive/Delete/ExportCsv).
- `QuickFilterMetaData` — drives the quick-search box (`TableData/QuickFilterMetaData.cs`).
- `TableTypes` — PlanSummary, ArchivedPlanSummary, ShiftHistoryOverview, PlanData, PlanDataDetail,
  EntityData, SubEntityData, ShiftReportDrillingDetails (`TableData/TableTypes.cs`).
- `IRowSchemas.cs` — row shapes drafted for **all three methods × all table types**, including
  `ICutAndFillShiftHistoryOverviewRowSchema`, `ICutAndFillPlanSummaryRowSchema`, and the archived
  variants. Row keys are contract-level: implementations must use `nameof()` against these
  interfaces, not magic strings (doc comment on `IDefaultProductionPlanSummaryRowSchema`).

Mapping current UI → contract: visible columns = column present + `IsVisible`; label swaps = per-
method `Label`; filter-modal fields = `IsFilterable` + `FilterOperators`; row links =
`ColumnType.Link` + `RouteTemplate`; Edit|Archive|Export = `Actions`; Progress bar =
`ColumnType.ProgressBar`; unit suffixes (m/ft) = `UnitSuffix` (backend knows the site's measure
system).

## Design subtleties & contract gaps (raise with backend while it's still unfinalized)

1. **Totals row not expressible.** Shift history (`ShiftList.razor:361-412`) and the shift report
   tables render aggregate rows with per-column rules (sum vs "-"). The contract has no
   totals/aggregation concept. Options: `IsSummable` flag on ColumnMetaData, or backend ships a
   separate totals row object per result.
2. **Dynamic column groups not expressible.** Production's Diameter group is a two-row header whose
   sub-columns depend on the page's data (`ShiftList.razor.cs:375-399`). A flat column list can't
   render `colspan` groups. Since columns already travel per-request with the rows, the backend
   *can* emit one column per diameter present — but the contract needs a `ColumnGroup`/parent-label
   field for the spanning header. Note `IDefaultProductionShiftHistoryOverviewRowSchema.Diameters`
   is a single `decimal` — it cannot carry per-diameter lengths as drafted.
3. **Server paging missing.** Shift history is server-paged today (`POST GetByRig` +
   PageViewDetails); `TableDataRequest` has no page/pageSize/sort. Either the contract grows paging,
   or ShiftHistoryOverview keeps its existing paging pipeline and only the *column metadata* becomes
   backend-driven. Plan summaries are in-memory today so they're safe either way.
4. **Localization ownership undecided.** `Label` comes from the backend, but today all labels go
   through frontend `Localizer` + 7 .resx files. Either the API localizes (culture on the request)
   or it returns resource *keys* the frontend localizes. Must be decided before G-WEB-1 — it changes
   every .resx workflow.
5. **DateTime semantics.** Rows are `object`; `Format` can't express timezone policy. Today the same
   page mixes conversions: shift dates use site TZ (`timeService.ConvertToDestinationTimeZone…`,
   `ShiftList.razor:283`) while plan-summary dates use *user* TZ
   (`UserTimeService.ConvertToUserTimezoneDisplay`, `DriveDetailsComponent.razor:186-189`).
   The contract needs a per-column (or per-table) timezone convention.
6. **The two shift-history schemas silently rename fields** — Production `TotalDownholeLength` vs
   Development `TotalExpectedAdvance`, both alongside `TotalLength`. That's correct (different
   physics) but means the generic renderer must key cells strictly by `ColumnMetaData.Key` into row
   dictionaries — never bind to a typed model.
7. **Command actions need a frontend handler map.** Export CSV opens a loading modal and downloads
   (`DriveDetailsComponent.razor.cs:287-297`); `RowActionType.Command` + `ActionId` identifies it,
   but the renderer needs an ActionId → delegate registry supplied by the hosting page.

## Questions asked
- 2026-07-09 — "Check current logic of shift history (prod + dev), plan summary + filter modal, and
  history report page as groundwork for the metadata-driven generic table refactor" → this note;
  gaps list above is the pre-backend-finalization deliverable.

## Related
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]] — §5 declared the unify-plan-summary /
  fixed-shell-shift-pages strategy this note details.
- [[2026-07-07 PRD Front End 15-22 Analysis and Executable Plan]] — G-API-1 / G-WEB-1..4 are the
  tickets that consume this analysis.
- [[2026-07-07 Development Drive Last Updated Feature Explained]] — the Last Updated column shown in
  the Development plan summary.
- [[2026-07-08 MC-1825 Assign Plans Requirements Organized and Execution Plan]] — assign-plans work
  that should build on the same method-keyed architecture.
