---
date: 2026-07-10
updated: 2026-08-06
project: core-web / core-web-api
tags: [implementation, mc-1833, shift-history, metadata-table, grouped-columns]
status: implemented
---

# MC-1833 Shift History Overview Metadata Table Implementation

## Goal

Render Production, Development, and Cut and Fill Shift History Overview tables through one reusable `MetaDataTable`. The backend owns mining-method-specific columns, labels, routes, grouped columns, rows, totals, and paging metadata. The frontend owns the page shell and generic rendering.

## Current Architecture

```text
Rig card: View Shift History
  -> ShiftList page
  -> build date, driller, shift-type, and paging request
  -> GET /api/Table/ShiftHistoryOverview
  -> API loads the rig and uses Rig.MiningMethod
  -> ShiftHistoryOverviewService selects exactly one provider
  -> provider returns TableDataResult
  -> ApiShiftHistoryTableDataProvider validates and wraps the result
  -> ShiftHistoryCard adds Shift History interactions
  -> MetaDataTable renders metadata, rows, totals, routes, and footer
```

The frontend does not select a provider and does not build mining-method-specific columns.

## Rendering Chain

### 1. Rig card navigation

`RigCardComponent.razor` renders the **View Shift History** link:

```text
/Sites/Rigs/{RigId}/{RigSiteType}/Shifts
```

`RigSiteType` is still required by legacy page behavior such as the breadcrumb, driller lookup, and some route compatibility. It is not the source used by the API to select the Shift History provider.

### 2. ShiftList initialization

`ShiftList.razor` owns the page route. `ShiftList.razor.cs`:

- loads the rig and site;
- reads the persisted `Rig.MiningMethod`;
- initializes the default date range;
- loads drillers and feature settings;
- retains the date, driller, shift-type, page-size, paging, export, and modal controls.

The default date range starts on the first day of the current month. On the first day of a month it starts 14 days earlier.

### 3. Frontend API request

`ShiftList.UpdateFilter()` creates `ShiftHistoryOverviewTableDataRequest` with:

- `SiteId`;
- `RigId`;
- `MiningMethod` for response validation;
- date range;
- selected driller IDs;
- shift number;
- current page and page size.

`TableDataRepository.GetShiftHistoryOverviewAsync()` calls:

```text
GET /api/Table/ShiftHistoryOverview
```

The query contains site, rig, filters, page, and page size. The client does not send Mining Method as a query parameter because the API resolves it from the persisted rig.

### 4. API validation and provider selection

`TableDataController.GetShiftHistoryOverview()`:

1. validates the date and paging values;
2. loads the site and verifies user access;
3. loads the rig and verifies it belongs to the site;
4. reads `Rig.MiningMethod` into the internal request;
5. calls `ShiftHistoryOverviewService`.

For regular users, the service confirms that the rig's Mining Method is allowed by the user's site scope. Administrators bypass this user-level scope check.

`ShiftHistoryOverviewProviderFactory` selects exactly one provider from `Rig.MiningMethod`:

- `DefaultProductionShiftHistoryOverviewProvider`;
- `DefaultDevelopmentShiftHistoryOverviewProvider`;
- `CutAndFillShiftHistoryOverviewProvider`.

Trying every provider would be incorrect because one rig belongs to one Mining Method and all providers query the same Shift Summary source.

### 5. Data query and response

`ShiftHistoryOverviewQuery` reads `ShiftSummary` records for the selected rig. It applies:

- synced-shift requirement;
- date range;
- optional driller filter;
- optional shift-number filter;
- descending start-time order;
- server-side paging.

The selected provider returns one `TableDataResult` containing:

- `TableType` and `MiningMethod`;
- ordered `Columns`;
- `QuickFilters` supplied by the profile;
- current-page `Rows`;
- current-page `TotalsRow`;
- `PageInfo`.

The API profile owns labels, order, visibility, sortable flags, column types, route templates, grouped-column metadata, and row actions.

### 6. Frontend result validation

`ApiShiftHistoryTableDataProvider` expects one API result and checks:

- `TableType == ShiftHistoryOverview`;
- returned `MiningMethod` matches the rig loaded by the page.

It passes the backend data through and creates only frontend page text, currently the localized viewing-range footer.

### 7. ShiftHistoryCard

`ShiftHistoryCard` hosts `MetaDataTable` and adds interactions that are specific to Shift History:

- overlapping-shift warning icon;
- pre-start status icons;
- handover-notes command callback.

The warning and pre-start icons are rendered through `RowLeadingContent` because they are not normal metadata columns. They still use the backend Shift link route. `ShiftHistoryCard.GetShiftDetailsUrl()` finds the Shift `Link` column and resolves its `RouteTemplate` with `TableRouteResolver`.

The normal Shift-name link is rendered directly by `MetaDataTable` from the same backend route template.

### 8. MetaDataTable rendering

`MetaDataTable` is mining-method blind. It renders:

- visible columns ordered by `Index`;
- one-level grouped headers from `GroupKey` and `GroupLabel`;
- sortable icons only when `IsSortable` is true;
- links and route actions from backend route templates;
- text, decimal, integer, progress, hover-string, and action columns;
- duration values using the shared `duration-minutes` format token;
- totals using `TotalsRow`;
- the viewing-range footer.

`TableRouteResolver` is shared by `MetaDataTable` and Shift History's custom leading icons so both replace route tokens through `TableRowAccessor` in exactly the same way.

## Production Diameter Group

Production is the only current Shift History Overview with a true dynamic column group.

The backend determines the displayed diameter columns as follows:

```text
configured diameters = rig ream sizes, when present
                       otherwise site ream sizes

displayed diameters = configured diameters
                      union diameters observed across the complete filtered result set
```

The values are distinct and sorted descending. The API inspects the complete filtered result set, not only the current page, so grouped columns remain stable while paging.

For every diameter, the column builder creates metadata with:

- a generated row key;
- a label such as `127mm`;
- `ColumnType.Decimal`;
- the Production diameter `GroupKey` and `GroupLabel`.

Rows without a drilled value for a configured diameter still receive a zero value. This preserves configured columns even when the current page has no drilling at that diameter.

## Routes and Actions

### Shift details

The Shift link route comes from the backend profile. `MetaDataTable` resolves it for the Shift text link. `ShiftHistoryCard` reuses it for warning and pre-start icons.

### Handover notes

The API exposes Handover Notes as a command action with an `ActionId` and parameter mapping. `MetaDataTable` raises `OnAction`, `ShiftHistoryCard` extracts the mapped `ShiftId`, and `ShiftList` opens the existing modal.

### Export

Export remains a page-level Shift History operation and uses the existing export workflow. It is not currently a row action in the metadata table.

## Paging and Filtering Ownership

Server-owned:

- date, driller, and shift-number filtering;
- total item count;
- current page and page size;
- row paging;
- observed-diameter scan across the filtered result set.

Frontend-owned:

- date-range control;
- driller selector;
- shift-type selector;
- paging controls and page-size selector;
- applying changed controls by requesting the API again.

The API profiles return `QuickFilters`, but Shift History Overview currently uses its dedicated page filters rather than rendering `MetaDataQuickFilter`.

## Important Ownership Rules

1. Use persisted `Rig.MiningMethod` to select the provider.
2. Never try every provider for one rig.
3. Do not build mining-method columns in Blazor.
4. Do not hardcode Shift-detail row routes in the frontend.
5. Keep page-level navigation and page controls outside `MetaDataTable` until the table contract explicitly supplies them.
6. Keep generic rendering behavior in `MetaDataTable`; keep Shift-specific icons and commands in `ShiftHistoryCard`.

## Remaining Legacy Boundary

The rig-card URL and parts of `ShiftList` still carry `RigSiteType`/`SiteType`. This remains for legacy navigation, breadcrumbs, driller loading, and compatibility with existing pages. Provider selection is already Mining Method based.

The long-term cleanup is to remove this Site Type dependency after those page services and routes accept Mining Method directly. It must not be removed only from the URL while downstream code still depends on it.

## Verification Checklist

- Production, Development, and Cut and Fill each select one matching backend provider.
- Date, driller, and shift-type filters reload data from the API.
- Page and page-size controls use `PageInfo` and retrieve only the selected page.
- Production diameter columns remain unchanged between pages for the same filter.
- Rig ream sizes override site ream sizes; site sizes are the fallback.
- Shift links, warning icons, and pre-start icons resolve the backend route template.
- Handover Notes opens through the metadata command callback.
- Duration cells and totals display as hours and minutes.
- Empty results show the existing no-shifts message.
- Export continues to use the existing page-level workflow.

## Key Code Areas

| Area | Location |
|---|---|
| Rig navigation | `Minnovare.Core.Web/Pages/Blazor/Shared/RigCardComponent.razor` |
| Shift History page shell | `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftList.razor(.cs)` |
| Frontend API adapter | `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftHistory/ApiShiftHistoryTableDataProvider.cs` |
| Shift-specific host | `Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftHistory/ShiftHistoryCard.razor(.cs)` |
| Generic renderer | `Minnovare.Core.Web/Pages/Blazor/Shared/TableData/MetaDataTable.razor(.cs)` |
| Shared route resolver | `Minnovare.Core.Web/Pages/Blazor/Shared/TableData/TableRouteResolver.cs` |
| Web API client | `Minnovare.Core.Web/Repositories/TableDataRepository.cs` |
| API endpoint | `Minnovare.Core.WebApi/Controllers/TableDataController.cs` |
| API service/factory/query | `Minnovare.Core.Services/TableData/ShiftHistoryOverview/` |
| Mining-method profiles/providers | `Minnovare.Core.Services/TableData/ShiftHistoryOverview/Profiles/` and `Providers/` |

## Related

- [[2026-07-09 Generic MetaDataTable Component Executable Plan]]
- [[2026-07-09 Generic Metadata Tables Feature Explained]]
- [[2026-07-22 Mining Method Architecture Decision Points]]
