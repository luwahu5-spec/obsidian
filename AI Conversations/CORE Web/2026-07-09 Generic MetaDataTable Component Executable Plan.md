---
date: 2026-07-09
source: Claude Code
project: core-web
tags: [planning, core-refactor-2.23, table-data, metadata-tables, plan-summary, mining-methods]
---

> **Current status (2026-07-21):** The frontend-first sections below are retained as implementation
> history. Plan Summary now consumes the real `GET /api/Table/PlanSummary?siteId={siteId}` endpoint.
> `FrontendPlanSummaryTableDataProvider` has been removed and DI now uses
> `ApiPlanSummaryTableDataProvider`. See **2026-07-21 implementation update** at the end of this note
> for the live rendering chain, temporary compatibility bridges, verification, and grouped-column status.

# Generic MetaDataTable — executable plan (frontend-only, testable on day one)

> **Original frontend-first takeaway (historical):** Backend provides nothing yet, but the Shared 2.23 contracts are enough to build the
> whole thing frontend-first: a `FrontendPlanSummaryTableDataProvider` fabricates `TableDataResult` (per-method
> column metadata + row dictionaries) from today's endpoints, a generic `MetaDataTable` renders it,
> and SiteDetails' Production/Development plan-summary split becomes one component. Because the
> stubbed `/MiningMethods/Assigned` already returns all three methods and the method tabs already
> switch on one page, **clicking tabs demonstrates different columns per drilling method with zero
> backend work**. When the real endpoint ships, only provider internals change — the exact pattern
> already proven by `SiteUploadConfigProvider`.

Target slice: **PlanSummary table on SiteDetails** (simplest: in-memory, no totals row, no dynamic
column groups, no server paging — all the contract gaps live in the shift tables, phase 2+).
Assumed base branch: `core-web@MC-1773` (tabs + contracts live there) — confirm ticket/branch name.

## Step 1 — Keep the `MiningMethod` enum alive through the tab layer
Today the enum is parsed then discarded for a display label (`MiningMethodRepository.cs:58-79`);
`TableDataRequest.MiningMethod` needs the enum.
- Return `AssignedMiningMethod { MiningMethod Method; string Label; }` from
  `GetAssignedMiningMethods` instead of `string`.
- Add `MiningMethod Method` to `MiningMethodTab` (`SiteDetails.razor.cs:461`) and populate in
  `CreateMiningMethodTab` (`:299`). `GetLegacySiteType` keys off the enum instead of label strings.

## Step 2 — Provider seam (new `Minnovare.Core.Web/TableData/`)
- `IPlanSummaryTableDataProvider`:
  - `Task<PlanSummaryTableDataView> GetPlanSummaryAsync(TableDataRequest request)`
  - Row commands are emitted by `MetaDataTable` and handled by `PlanSummaryCard`; the interim
    provider is responsible only for loading metadata and rows.
- `FrontendPlanSummaryTableDataProvider` (interim): switch on `(TableType, MiningMethod)`:
  - PlanSummary + DefaultProduction → `DriveRepository.GetDrivesForSite` → rows as
    `Dictionary<string,object>` keyed with `nameof(IDefaultProductionPlanSummaryRowSchema.…)`
    (contract doc-comment mandates nameof, no magic strings) + Production column list.
  - PlanSummary + DefaultDevelopment → `DevelopmentRepository.GetDrivesForSite` + 3-column list.
  - PlanSummary + CutAndFill → column config + empty rows (**the "method 4 = config only" demo**).
- `PlanSummaryColumnConfigs.cs`: `ColumnMetaData` lists incl. `ColumnType.Link` + RouteTemplate
  (`/Sites/Drives/{DriveId}` vs `/Sites/Development/Drive/Details/{HeadingId}`),
  `ProgressBar`, `TextActionList` with `TableActionIdentifiers.PlanSummary.*` actions,
  `IsFilterable` + `FilterOperators` (this is what makes the filter modal differ per method),
  `UnitSuffix` from site measure system.
- Chrome that backend will never own stays here too: card title (Drives/Heading), Add-button route,
  quick-search placeholder, archived-list route → small `PlanSummaryTableDefinition` next to the columns.
- Labels localized in the provider via `IStringLocalizer` + new resx ×7 (copy existing translations
  from DriveDetailsComponent/DevelopmentDriveDetailsComponent resx).
- TokenStore: provider receives it per call (repos are per-request token-bound).
- DI: `services.AddScoped<IPlanSummaryTableDataProvider, FrontendPlanSummaryTableDataProvider>()` next to
  `SiteUploadConfigProvider` (`Startup.cs:82`).

## Step 3 — Generic renderer `Pages/Blazor/Shared/TableData/MetaDataTable.razor(.cs)`
- Parameters: `TableDataResult Data`, `Func<string, object, Task> OnAction` (ActionId + row),
  optional empty-state fragment.
- `TableRowAccessor` helper: value-by-key from `IDictionary<string,object>` / `JObject` /
  `JsonElement` / POCO-reflection fallback — rows survive the later switch to real JSON responses.
- Render rules: `Columns.Where(IsVisible).OrderBy(Index)`; **columns absent or invisible are not
  rendered at all** (PRD: no empty "-" columns). Header sort chevrons when `IsSortable`
  (single-active SortState, client-side sort through the accessor — replaces the six copy-pasted
  `Change*Sort` methods). Cell switch on `ColumnType`: Text / Integer / Decimal (`Format` +
  `UnitSuffix`) / DateTime (`UserTimeService.ConvertToUserTimezoneDisplay` — matches current
  plan-summary behavior) / Link (RouteTemplate `{Token}` substitution from row values) /
  ProgressBar (existing `mn-progress` markup incl. the `Math.Max(progress,25)` display rule) /
  TextActionList (`link` anchors joined by `|`; Route → href, Command → `OnAction`,
  `RequiresConfirmation` → GenericModal confirm first — replaces ArchiveDriveModal/DeleteDriveModal
  duplication).

## Step 4 — Generic filter modal `MetaDataTableFilterModal.razor` + `RowFilterEvaluator`
- Generated from `Columns.Where(IsFilterable)`: one `FilterInput`-style row per column — operator
  select restricted to the column's `FilterOperators`, input type from ColumnType
  (Text→text, DateTime→date, Integer/Decimal→number).
- ⚠️ Two `FilterOperator` enums exist: `Shared.Filters.FilterOperator` (used by `FilterInput`) and
  `Shared.Contracts.TableData.FilterOperator` (the contract, with date operators folded in). Render
  the operator select from the contract enum directly rather than mapping back and forth.
- Output: `List<TableFilterCondition { Key, FilterOperator, RawValue }>`; `RowFilterEvaluator`
  applies them over row dictionaries (the existing `Specification<T>` classes are typed expression
  trees — unusable on dictionary rows, don't try).
- Quick-search box driven by `QuickFilterMetaData` from the provider.

## Step 5 — Host card `PlanSummaryCard.razor(.cs)`
- Parameters: `SiteId`, `MiningMethod`, `ArchivedDrives`, `CardFooter`.
- Composes: chrome (title/search/Filter/Clear/Add from PlanSummaryTableDefinition) + `MetaDataTable` +
  `MetaDataTableFilterModal`; loads via `IPlanSummaryTableDataProvider` on first render with the
  `TryRunOrRefresh` + inline `BlazorErrorBox` pattern (`DriveDetailsComponent.razor.cs:107-141`);
  routes `OnAction` to `provider.ExecuteActionAsync` (loading modal around ExportCsv);
  public `RefreshAsync()` preserved for the upload-success callback.

## Step 6 — Wire into SiteDetails
- Replace the Production/Development component fork (`SiteDetails.razor:105-128`) with one
  `<PlanSummaryCard SiteId="SiteId" MiningMethod="_selectedMiningMethodTab.Method">`; archived link
  in CardFooter comes from PlanSummaryTableDefinition.
- `RefreshActivePlanSummary` (`SiteDetails.razor:103`) now calls the single card's `RefreshAsync`.
- **Do not delete** `DriveDetailsComponent`/`DevelopmentDriveDetailsComponent` yet —
  `ArchivedDrives.razor` / `ArchiveHeading.razor` still use them (`ArchivedDrives=true`). Migrating
  those = ArchivedPlanSummary table type, a follow-up ticket (schemas already exist).

## Step 7 — Tests + manual verification script
- bUnit: MetaDataTable (hidden column NOT in DOM, Index ordering, Format/UnitSuffix, link routing,
  progress bar, action dispatch + confirmation), filter modal field generation per method,
  RowFilterEvaluator operator matrix, PlanSummaryCard ×2 methods with a fake provider.
  New `[Inject]`s on shared components → run the affected bUnit projects (house rule).
- Manual (the "see it work" script): run core-web + core-web-api locally → open a site with both
  Production and Development features → **click the method tabs: Production shows 6 columns +
  6-field filter modal, Development shows 3 columns + 3-field filter modal, all from metadata** →
  verify sort, quick search, filter, Archive (confirm modal), Export CSV, Add Drive route.
- Cut and Fill caveat: `FilterMiningMethodTabsByLegacySiteFeatures` (`SiteDetails.razor.cs:273`)
  hides the Cut and Fill tab because it has no legacy SiteType. To demo method-4-by-config, add a
  temporary Development-environment-only bypass (allow tabs with null LegacySiteType) — or accept
  Production↔Development as the demo until the bridge dies (G-WEB-4).

## Earliest visible result (vertical slice order)
1 → 2 (Production config only) → 3 → 6 = generic table rendering Production from metadata (~2 days).
Add the Development config = **tab-switching column demo works**. Then 4 (filter modal), 5 polish,
7 tests.

## Estimates (PRD scale: 1 pt = 4–12h)
| Step | Effort |
|---|---|
| 1 enum plumbing | 0.5 day |
| 2 provider + configs + resx | 1 day |
| 3 MetaDataTable | 1–1.5 days |
| 4 filter modal + evaluator | 1 day |
| 5+6 card + SiteDetails wiring | 1 day |
| 7 bUnit + manual pass | 1 day |
| **Total** | **~5–6 days (≈ 5 pts)** |

## Phase 2+ (outline only — blocked on contract gaps)
- **ShiftHistoryOverview**: needs totals-row + column-group + paging additions to the contract
  (gaps #1–3 in the feature note) — raise with backend now; column metadata can still go generic
  ahead of that with the fixed shell kept.
- **ShiftReportDrillingDetails**: swap the Holes / Drilling Details fork in `ShiftDetails.razor`.
- **G-WEB-4**: delete the two legacy components + bridge once archived pages are migrated.

## Related
- [[2026-07-09 Generic Metadata Tables Feature Explained]] — current-state map + the
  7 contract gaps this plan routes around.
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]] — adapter/provider principle this follows.
- [[2026-07-07 PRD Front End 15-22 Analysis and Executable Plan]] — this is G-WEB-1/G-WEB-2 (partial,
  frontend-provider flavor per Q9).

---



## Claude review of the implementation (2026-07-10) — concerns for the next increment

Overall verdict: **approve** — the plan deviation (isolating from the untested tab pipeline) and the naming boundary were the right calls; MC-1833 reused `MetaDataTable` with zero changes, proving the boundary. Concerns, by severity:

**Correctness (fix in MC-1833's S2 — same files touched):**
- C1: `BuildRoute` (`MetaDataTable.razor.cs:190-207`) substitutes tokens only for dictionary rows; `TableRowAccessor` supports JObject/POCO — real JSON responses will silently render unreplaced Link templates (`/Sites/Drives/{DriveId}`) and 404. Fix: resolve tokens via `TableRowAccessor`; bUnit case with a JObject row.
- C2: `GetProgress` (`:114-118`) uses `Convert.ToInt32` — one malformed progress value = unhandled exception = Blazor error banner. Fix: TryParse fallback to 0.

**Release-gating debts (Codex's own follow-up list — need tickets):**
- D1: bUnit coverage thin (sorting, quick search, filter combos, action dispatch, method switching untested). This component is becoming the rendering engine for every tabular page — its tests are infrastructure. Block parity release on it.
- D2: resx ×7 missing for all new strings (card, filter modal, operators, metadata labels).

**Architecture (track):**
- A1: `UnitSuffix` compatibility cast (`:104-112`) escalation has no owner — folded into MC-1833 S1 contract additions; verify it lands or the cast becomes permanent.
- A2: bridge count is now TWO (`SelectedPlanSummaryMiningMethod` conversion + `FilterMiningMethodTabsByLegacySiteFeatures`) plus the deferred enum-through-tab-layer work — G-WEB-4's ticket must name all three explicitly.
- A3: legacy `DriveDetailsComponent`/`DevelopmentDriveDetailsComponent` live on for archived pages → every plan-summary change has two implementations to sync until `ArchivedPlanSummary` migrates. Schedule it soon after MC-1833.

**Cosmetic:** idle sort icon `fa-align-left` (`:160`) — check against `SortableHeader` convention; `Rows` re-sorts every render (`:42-61`) — fine bounded by paging, don't feed it thousands of client rows.

## Original Frontend-First Plan Summary Rendering Chain (Historical)

### 1. Existing mining-method tab remains the entry point

`SiteDetails` continues to load, filter, order, and select mining-method tabs through the existing
tab implementation. The generic-table refactor does not change that API or tab state.

The selected tab still exposes its temporary `LegacySiteType`. At the Plan Summary boundary only,
`SelectedPlanSummaryMiningMethod` converts:

- `SiteType.Development` to `MiningMethod.DefaultDevelopment`
- `SiteType.Production` to `MiningMethod.DefaultProduction`

This isolates the new table from the untested tab pipeline.

### 2. SiteDetails creates one PlanSummaryCard

When the site has a supported selected method, `SiteDetails.razor` renders:

```razor
<PlanSummaryCard
    SiteId="SiteId"
    MiningMethod="SelectedPlanSummaryMiningMethod.Value" />
```

The former Production/Development component fork is gone from the active Site Details path.

### 3. PlanSummaryCard builds the request

On first render, and whenever SiteId or MiningMethod changes, `PlanSummaryCard.RefreshAsync` creates:

```csharp
new TableDataRequest
{
    SiteId = SiteId,
    MiningMethod = MiningMethod,
    TableType = TableTypes.PlanSummary,
    UserId = UserId
}
```

It passes that request to the injected `IPlanSummaryTableDataProvider`. The card owns loading and friendly
error states.

### 4. Dependency injection selects the interim provider

`Startup` currently registers:

```csharp
services.AddScoped<IPlanSummaryTableDataProvider, FrontendPlanSummaryTableDataProvider>();
```

Therefore, no real metadata API is called yet. `FrontendPlanSummaryTableDataProvider` is the frontend-owned
adapter that makes today's endpoints look like the future backend table-data response.

### 5. Provider chooses the method-specific legacy endpoint

`FrontendPlanSummaryTableDataProvider.GetPlanSummaryAsync` switches on `TableType + MiningMethod`.

Production:

```http
GET /drives/GetDrivesForSite?siteId={siteId}&archived=false
```

Development:

```http
GET /Development/GetDrivesForSite?siteId={siteId}&archived=false
```

The provider does not duplicate or calculate additional Drive/Heading records. It maps every object
returned by the existing endpoint into one generic table row.

### 6. Provider returns rows, column metadata, and frontend chrome

Each legacy model becomes a `Dictionary<string, object>`. Keys use `nameof(...)` against Shared row
schema interfaces so metadata keys and row keys stay aligned.

Production rows include DriveId, SiteId, DriveName, Created, LastDrilled, Completed, Outstanding,
and Progress.

Development rows include HeadingId, HeadingName, Created, and LastUpdated.

The returned `PlanSummaryTableDataView` contains:

- `Data` (`TableDataResult`)
  - MiningMethod
  - TableType
  - Columns
  - Rows
- `PlanSummaryTableDefinition`
  - card title
  - quick-filter definition
  - Production-only Add route
  - archived-list route

Production column metadata declares six data columns plus Edit, Archive, and Export CSV actions.
Development declares three data columns plus Edit and Archive actions.

### 7. PlanSummaryCard owns client-side table state

After loading, the card preserves the original provider rows in `_allRows`. The visible
`TableDataResult.Rows` collection is rebuilt from:

1. active metadata filter conditions;
2. quick name search.

Search/filter state is reset when the mining-method tab changes so Production conditions cannot
leak into Development keys. Normal refresh keeps the current view state.

The card renders controls according to `PlanSummaryTableDefinition` and column metadata:

- quick-search input only when `QuickFilter` exists;
- Filter only when at least one visible column is filterable;
- Clear for local search/filter/sort reset;
- Add Drive only when `AddRoute` exists;
- archived link from `ArchivedRoute`.

### 8. MetaDataTable renders without mining-method checks

`MetaDataTable` receives only `TableDataResult`. It does not know Production, Development, SiteType,
Drive, or Heading.

Rendering follows this sequence:

1. Keep columns where `IsVisible` is true.
2. Order columns by `Index`.
3. Apply the current generic sort key/direction to rows.
4. Read every cell through `TableRowAccessor` using `ColumnMetaData.Key`.
5. Render by `ColumnType`.

Supported rendering currently includes:

- Text / Integer / Decimal
- DateTime through `UserTimeService`
- Link through route-template token replacement
- ProgressBar with the existing minimum visible fill rule
- TextActionList ordered by each action's Index

The table itself is inside the restored fixed-height horizontal/vertical scroll container, and
headers remain sticky while rows scroll.

### 9. Search, filter, and sort interaction paths

Quick search:

```text
input event
-> PlanSummaryCard.QuickSearchChanged
-> ApplyClientFilters
-> name value read through QuickFilter.Key
-> TableDataResult.Rows replaced
-> MetaDataTable rerenders
```

Filter:

```text
Filter button
-> MetaDataTableFilterModal generated from filterable columns
-> operator + raw value returned as TableFilterCondition
-> RowFilterEvaluator checks each dictionary row
-> TableDataResult.Rows replaced
-> MetaDataTable rerenders
```

Sort:

```text
column sort icon
-> MetaDataTable.ChangeSort
-> ascending / descending / none state
-> rows ordered through TableRowAccessor
-> table rerenders
```

Clear resets all three without reloading the API.

### 10. Row-action paths

Route actions such as Edit are completed entirely by `MetaDataTable`:

```text
RowActionMetaData.RouteTemplate
-> replace {SiteId}/{DriveId}/{HeadingId} from row dictionary
-> render href
```

Command actions are emitted back to the card as `ActionId + row`.

Archive:

```text
Archive link
-> MetaDataTable emits PlanSummary.Archive
-> PlanSummaryCard opens ArchiveDriveModal
-> card reads DriveId or HeadingId from row
-> existing Production or Development repository archives it
-> RefreshAsync reloads provider data
```

Production Export CSV:

```text
Export CSV link
-> MetaDataTable emits PlanSummary.ExportCsv
-> PlanSummaryCard reads DriveId and DriveName
-> existing DriveRepository downloads bytes
-> existing ExportFile saves {DriveName}.csv
```

### 11. Upload refresh path

Both upload-card locations on `SiteDetails` already call `RefreshActivePlanSummary` after a
successful upload:

```text
upload succeeds
-> SiteDetails.RefreshActivePlanSummary
-> PlanSummaryCard.RefreshAsync
-> provider reloads current method
-> current table rerenders
```

### 12. Future backend replacement point

When the real Plan Summary table-data API is ready, DI can replace
`FrontendPlanSummaryTableDataProvider` with an API-backed implementation of
`IPlanSummaryTableDataProvider`.

`PlanSummaryCard`, `MetaDataTable`, filter evaluation, action dispatch, and SiteDetails composition
should remain stable. Legacy endpoint mapping and frontend column construction are the parts meant
to disappear.

---

## Naming Boundary: Generic Table Infrastructure vs Plan Summary

### Decision

The first implementation was built from the Plan Summary use case, but not every class created for
it belongs to Plan Summary. Rename only objects whose properties or behavior are genuinely specific
to that workflow.

### Renamed as Plan Summary-specific

| Previous name               | Current name                           | Reason                                                                          |
| --------------------------- | -------------------------------------- | ------------------------------------------------------------------------------- |
| `ITableDataProvider`        | `IPlanSummaryTableDataProvider`        | Its current method loads only Plan Summary                                      |
| `FrontendTableDataProvider` | `FrontendPlanSummaryTableDataProvider` | It calls Drive/Development plan-summary endpoints and rejects other table types |
| `TableDataView`             | `PlanSummaryTableDataView`             | It combines generic table data with Plan Summary page chrome                    |
| `TableDefinition`           | `PlanSummaryTableDefinition`           | Add Drive and Archived Drives/Headings routes are Plan Summary controls         |

The provider method is now explicitly named:

```csharp
Task<PlanSummaryTableDataView> GetPlanSummaryAsync(TableDataRequest request);
```

### Kept generic intentionally

| Type                                 | Why Shift History can reuse it                                                          |
| ------------------------------------ | --------------------------------------------------------------------------------------- |
| `MetaDataTable`                      | Renders visible ordered columns and rows without checking a mining method or table type |
| `TableRowAccessor`                   | Reads a value by metadata key from any dictionary/JObject row                           |
| `TableRowActionRequest`              | Carries generic `RowActionMetaData + row`; it contains no Drive/Heading fields          |
| `MetaDataTableFilterModal`           | Builds controls from any filterable column collection                                   |
| `TableFilterCondition`               | Describes a key/operator/value condition without a Plan Summary model                   |
| `RowFilterEvaluator`                 | Applies those conditions to any metadata row                                            |
| `TableDataRequest`                   | Already contains TableType, MiningMethod, SiteId, RigId, and PlanSummaryId              |
| `TableDataResult`                    | Contains only MiningMethod, TableType, Columns, and Rows                                |
| `IColumnMetaData` / `ColumnMetaData` | Describes generic labels, order, formatting, links, filters, and actions                |
| `RowActionMetaData`                  | Describes generic Route/Command/Download actions                                        |

Renaming these generic contracts to Plan Summary would force Shift History to duplicate the same
renderer/accessor/action infrastructure and would defeat the purpose of metadata-driven tables.

### Shift History-specific layer to add later

Shift History should introduce its own host and provider-side page model:

```text
ShiftHistoryCard
IShiftHistoryTableDataProvider
FrontendShiftHistoryTableDataProvider
ShiftHistoryTableDataView
ShiftHistoryTableDefinition
```

Those types will own Shift History concerns such as:

- RigId and selected mining method;
- date-range, driller, and shift-type filters;
- page number and page size;
- total result count;
- Export button;
- totals-row data;
- dynamic diameter column groups.

They will still pass the resulting generic `TableDataResult` into the same `MetaDataTable`.

```text
PlanSummaryCard
  -> PlanSummaryTableDataView
  -> TableDataResult
  -> MetaDataTable

ShiftHistoryCard
  -> ShiftHistoryTableDataView
  -> TableDataResult
  -> MetaDataTable
```

The two workflows therefore share the rendering language and row mechanics, while each keeps its
own loading, page chrome, paging, totals, and business actions.

---

## 2026-07-21 implementation update

### Current outcome

Plan Summary has moved from the temporary frontend-owned provider to the real backend table-data
endpoint:

```http
GET /api/Table/PlanSummary?siteId={siteId}
```

The API now owns the Plan Summary `TableDataResult` values for every allowed mining method:

- `MiningMethod`
- `TableType`
- column metadata
- dynamic row dictionaries
- sortable/filterable flags
- row action metadata

The generic component and the Plan Summary host remain frontend-owned because rendering, page
controls, confirmation modals, downloads, and navigation are UI responsibilities.

### Files introduced, replaced, and retained

| Area | Current implementation | Status |
|---|---|---|
| API access | `Repositories/TableDataRepository.cs` | Added; calls the real Plan Summary endpoint and materializes interface-typed columns |
| Plan Summary adapter | `Pages/Blazor/Sites/PlanSummary/ApiPlanSummaryTableDataProvider.cs` | Added; selects the active mining-method result and applies temporary compatibility mappings |
| Temporary provider | `FrontendPlanSummaryTableDataProvider.cs` | Removed |
| Provider seam | `IPlanSummaryTableDataProvider` | Retained; keeps the card independent of transport and API compatibility details |
| Feature host | `PlanSummaryCard.razor(.cs)` | Retained; owns quick search, metadata filters, Add/Archived controls, archive confirmation, and CSV download |
| Generic renderer | `MetaDataTable.razor(.cs)` | Retained and extended for real API metadata |
| Dynamic row access | `TableRowAccessor.cs` | Retained and made case-insensitive for API rows and route tokens |
| DI | `Startup.cs` | Now registers `TableDataRepository` and `ApiPlanSummaryTableDataProvider` |

### Live Plan Summary rendering chain

```text
selected mining-method tab
-> SiteDetails renders PlanSummaryCard
-> PlanSummaryCard creates TableDataRequest
-> IPlanSummaryTableDataProvider
-> ApiPlanSummaryTableDataProvider
-> TableDataRepository
-> GET /api/Table/PlanSummary?siteId={siteId}
-> select TableDataResult matching request.MiningMethod
-> apply temporary API compatibility mappings
-> PlanSummaryTableDataView
-> PlanSummaryCard controls and command handling
-> MetaDataTable renders API columns and rows
```

`TableDataRepository` parses columns as concrete `ColumnMetaData` values because the shared response
contract exposes `IColumnMetaData`. Rows remain `JObject` instances so keys are not coupled to a
frontend DTO for any particular mining method.

### Temporary frontend compatibility bridges

These are deliberate compromises and should be removed as the API contract is aligned:

1. **Quick filter and page chrome**
   `TableDataResult` does not currently serialize the backend profile's quick filters. The adapter
   temporarily supplies the quick name filter, card title, Production Add route, and Archived route
   through `PlanSummaryTableDefinition`.

2. **Production Edit route SiteId**
   The API route template refers to `SiteId`, but current Production rows do not include that key.
   The adapter temporarily adds the request SiteId to each dynamic row.

3. **Action identifiers**
   The API currently emits `Drive.Archive` and `Drive.ExportCSV`. The adapter maps them to the shared
   `PlanSummary.Archive` and `PlanSummary.ExportCsv` identifiers already handled by the card.

4. **Measurement units**
   The API currently supplies a hardcoded metre suffix. For Production length columns, the adapter
   temporarily applies the site's configured measurement unit.

5. **Command execution**
   Archive and CSV export still call the existing Drive/Development repositories. API metadata
   decides which commands are displayed; `PlanSummaryCard` owns their current frontend execution.

### Generic renderer updates made for the real API

- `ButtonActionList` now renders route buttons or dispatches command buttons using the same generic
  action callback as `TextActionList`.
- Route placeholders are resolved through `TableRowAccessor` and are case-insensitive. For example,
  API token `{driveId}` resolves row key `DriveId`.
- Dictionary, `JObject`, and POCO row lookups all have a case-insensitive fallback.
- Development archive IDs are parsed from dynamic JSON values with `Guid.TryParse`.
- Malformed progress values render as zero and valid values are clamped to 0-100.
- API timestamps are parsed with `DateParseHandling.DateTimeOffset`. This prevents JSON timestamps
  from becoming `DateTimeKind.Local`, which cannot be passed to `ConvertTimeFromUtc`.

### Quick filters

Backend profiles already define quick-filter metadata, but `TableDataResult` does not currently carry
it in the HTTP response. No second quick-filter API exists in the current controller. Until the shared
response contract exposes those values, `PlanSummaryTableDefinition.QuickFilter` remains the temporary
frontend source.

### Mining-method test stub

The updated backend `UserMiningMethodScopeService` queries `UserSiteMiningMethodScope`. The local test
database currently has no assignment rows, which caused every Site Details page to show:

```text
No mining methods are available for this site.
```

For UI/API integration testing, the backend scope service temporarily returns:

```text
DefaultProduction
DefaultDevelopment
```

This stub is placed in the shared scope service rather than only in `MiningMethodsController`, because
both `MiningMethods/Assigned` and the Plan Summary table service use that scope. Replace it with the
repository query when assignment maintenance and test data are ready.

### Grouped-column API status

The API grouped-column contract was added in commit `17c359c`:

- `ColumnType.GroupedColumn`
- `ColumnMetaData.ChildColumnTemplate`
- `ColumnMetaData.GroupKey`
- `ColumnMetaData.GroupLabel`
- `GroupedColumn` row-schema marker

For Default Production Shift History, the API profile declares one `Diameters` placeholder. The
`DefaultProductionShiftHistoryOverviewColumnBuilder` expands it into ordinary child columns such as:

```text
Diameters_89
Diameters_102
Diameters_152
```

Each resolved child column carries the same `GroupKey` and `GroupLabel`. This flattened response is
already compatible with `MetaDataTable`: the renderer groups adjacent metadata columns and produces a
two-row header without knowing about Shift History or mining methods.

The grouped-column work is **component-compatible but not end-to-end complete**:

| Part | Status |
|---|---|
| Shared grouped-column contract | Complete |
| API Production diameter column builder | Implemented |
| `MetaDataTable` grouped-header rendering | Implemented and covered by bUnit |
| Shift History API rows | Not implemented; Production provider currently returns an empty row collection |
| Shift History controller endpoint | Not exposed in `TableDataController` |
| Frontend Shift History API repository/provider | Not implemented |
| Current Shift History page | Still uses legacy `ShiftList` and `ShiftRepository` |
| Diameter source parity | Not complete; API currently reads only `site.ReamSizes`, while legacy behavior must be confirmed/preserved |

Therefore, grouped metadata returned in the resolved `GroupKey`/`GroupLabel` form will render correctly,
but the existing Shift History page does not consume it yet.

### Proposed `StringHover` column type

Product requires comment values to render as a compact speech-bubble icon rather than displaying the
complete string in the table cell. Hovering over the icon will reveal the underlying comment in a
tooltip. This remains metadata-driven and does not add Shift History or mining-method checks to
`MetaDataTable`.

The shared API contract must be updated before frontend implementation begins:

```csharp
public enum ColumnType
{
    // Existing values...
    GroupedColumn = 8,
    StringHover = 9
}

public interface IColumnMetaData
{
    string Icon { get; set; }
}
```

`ColumnMetaData` must also implement `Icon`. A backend profile can then describe a comment column as:

```csharp
new ColumnMetaData
{
    Key = nameof(IShiftReportRowSchema.Comments),
    Label = "Comments",
    ColumnType = ColumnType.StringHover,
    Icon = "Comment",
    IsVisible = true
}
```

Once the shared contract is available, CORE Web will:

1. Add a `ColumnType.StringHover` case to `MetaDataTable.razor`.
2. Read the underlying string through `TableRowAccessor`, as it does for other metadata cells.
3. Render the existing `Tooltip` component around a speech-bubble icon when text is present.
4. Map semantic API icon names through a frontend allowlist, for example `Comment` to
   `far fa-comment mn-micro-icon`. The API must not supply arbitrary CSS classes.
5. Render the agreed empty state when no comment exists. Product still needs to confirm whether this
   is a dash or a disabled grey comment icon.
6. Add bUnit coverage for populated comments, empty comments, icon mapping, and tooltip text.

No special filtering or sorting implementation is required. The filter modal already treats unknown
non-numeric column types as text, `RowFilterEvaluator` falls back to string operators, and sorting uses
the raw row value through `TableRowAccessor`. `StringHover` changes cell presentation only.

The frontend implementation is intentionally waiting for the shared `ColumnType.StringHover` and
`Icon` properties. Adding a frontend-only enum value or metadata property now would create a contract
that the API cannot serialize consistently.

### Verification completed

- `Minnovare.Core.Web` Verification build: **0 warnings, 0 errors**.
- Focused `MetaDataTableTests`: **8 passed, 0 failed**.
- Coverage includes visible-column ordering, grouped headers, no-group headers, `IsSortable`, JSON
  routes, case-insensitive route keys, `ButtonActionList`, and malformed progress handling.
- Shift History was intentionally left on its legacy page while its API endpoint and row providers
  remain incomplete.

### Next API-alignment work

1. Add quick filters and any required table-level definition metadata to the API response contract.
2. Align action identifiers with `TableActionIdentifiers` and include every route token in row data.
3. Return site-aware unit metadata from the API.
4. Populate `UserSiteMiningMethodScope` and remove the temporary scope stub.
5. Finish Shift History row mapping, expose its controller endpoint, and preserve the agreed diameter
   source/order behavior.
6. Add a Shift History API repository/feature host that passes the resolved `TableDataResult` into the
   existing `MetaDataTable`.
