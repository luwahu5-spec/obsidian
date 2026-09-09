---
date: 2026-07-15
tags: [planning, core-web, metadata-table, api-integration, cleanup]
status: planning
---

# Generic Metadata Table API Finalization Cleanup Plan

## Purpose

This document records which frontend pieces should remain, be replaced, or be removed once the Table Data API is finalized for Plan Summary and Shift History.

The central ownership rule is:

> The API decides which columns, groups, labels, values, sorting options, filters, and row actions apply to a mining method. The frontend renders that contract and owns only browser interaction and page layout.

This is a cleanup plan, not an instruction to remove the temporary adapters before the API reaches feature parity.

## Shared Generic Table Foundation

| Item | Final decision | Reason |
|---|---|---|
| `MetaDataTable` | Keep | It is the mining-method-blind renderer for `TableDataResult.Columns` and `Rows`. |
| `MetaDataTableColumn` | Keep only for web presentation properties | CSS classes, empty text, link icons, and other browser-only options do not belong in the shared transport contract. Reassess properties that the API later owns. |
| `MetaDataTableFilterModal` | Keep | It builds controls from `IsFilterable` and `FilterOperators` instead of feature-specific fields. |
| `TableRowAccessor` | Keep | It isolates rendering from dictionary, `JObject`, and typed-row representations. It can be simplified when the final deserialization shape is fixed. |
| `TableRowActionRequest` | Keep | It carries a metadata-defined action and selected row to the feature component. |
| `TableFilterCondition` | Keep or replace with the final API request contract | The UI still needs filter state, but server-paged tables should submit it to the API instead of filtering one loaded page. |
| `RowFilterEvaluator` | Remove for server-paged API tables | Client filtering only the loaded page gives incomplete results. It can remain only for explicitly client-owned tables. |
| `GroupKey` / `GroupLabel` rendering | Keep | The same renderer can show any grouped columns without knowing the mining method. |
| `ColumnType.GroupedColumn` / `ChildColumnTemplate` expansion | Add only after the API row-value shape is agreed | The shared contract identifies a grouped-column concept but does not yet define whether dynamic children arrive as a dictionary, list, or expanded columns. |

## Expected Grouped-Column Contract

The preferred API response is a collection of concrete child columns. Columns belonging to the same group share `GroupKey` and `GroupLabel`:

```json
{
  "key": "Diameter_127",
  "label": "127mm",
  "columnType": "Decimal",
  "groupKey": "Diameter",
  "groupLabel": "Diameter"
}
```

With this response shape, the existing `MetaDataTable` grouped-header renderer works without mining-method checks.

If the API instead returns one parent column with `ColumnType.GroupedColumn` and `ChildColumnTemplate`, the API team must first define the corresponding grouped values in each row. The frontend should not guess that wire format.

## Plan Summary

### Keep

| Item                                                  | Reason                                                                                                                                                                  |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PlanSummaryCard`                                     | It owns page-level controls, loading state, confirmation modals, navigation, and dispatching user commands.                                                             |
| `MetaDataTable` integration                           | It is the reusable rendering layer and already supports metadata columns, sorting, filtering, links, progress, and row actions.                                         |
| Plan Summary CSS and layout                           | These are frontend presentation responsibilities.                                                                                                                       |
| `IPlanSummaryTableDataProvider` boundary, temporarily | Keeping an interface allows the temporary adapter to be replaced without changing `PlanSummaryCard`. It may later become a generic `ITableDataProvider`.                |
| `PlanSummaryTableDefinition`, conditionally           | Keep only properties that remain frontend-owned, such as page title or archived-page navigation. API-owned quick filters and actions should move into the API contract. |
| `PlanSummaryTableDataView`, conditionally             | Keep as a composition object only while the page combines API table data with frontend-owned definition data.                                                           |

### Replace

| Current item                                                | Replacement                               | Reason                                                                                                                                 |
| ----------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `FrontendPlanSummaryTableDataProvider`                      | API-backed table-data repository/provider | The temporary provider calls legacy endpoints and manufactures metadata, rows, labels, actions, and calculated values in the frontend. |
| Hardcoded Production and Development column definitions     | `TableDataResult.Columns` from the API    | Adding a mining method must not require another frontend column switch.                                                                |
| Frontend-created row dictionaries                           | `TableDataResult.Rows` from the API       | Row keys and values must match the backend metadata contract.                                                                          |
| Client-side sort/filter for server-paged data               | API query parameters and returned page    | Sorting or filtering one page does not represent the complete dataset.                                                                 |
| Feature-specific export routing, where metadata supports it | API-provided row/table actions            | Export availability and endpoints vary by mining method.                                                                               |

### Remove After API Parity

| Item | Reason |
|---|---|
| `FrontendPlanSummaryTableDataProvider` | Its only purpose is to emulate the future API. |
| Frontend mining-method checks that choose Production or Development schemas | The returned table provider is the source of truth. |
| Frontend calculations used only to populate display rows | The frontend should display API values. |
| Legacy `DriveDetailsComponent` and `DevelopmentDriveDetailsComponent` usages | Remove only after active and archived workflows have generic-table parity. |
| Duplicate legacy filtering, sorting, and action markup | `MetaDataTable` and API metadata replace it. |

### Plan Summary Removal Prerequisites

- The API returns table metadata and rows for every supported mining method.
- Quick search, metadata filters, sorting, paging, archive, edit, add, and export have confirmed ownership.
- Active and archived Plan Summary pages have parity.
- Empty, loading, failure, and authorization states are defined.
- Localization strategy for API-provided labels is agreed.
- Integration and regression tests pass before legacy components are deleted.

## Shift History

Shift History generic-table integration is deferred because it is outside the current sprint. The generic grouped-column foundation remains available in `MetaDataTable`.

### Keep

| Item | Reason |
|---|---|
| Generic `MetaDataTable` grouped-header support | Shift History is one consumer, but grouped headers are a reusable metadata capability. |
| Shared `GroupKey`, `GroupLabel`, `ColumnType.GroupedColumn`, and `ChildColumnTemplate` contract | These are backend/frontend coordination points, although the dynamic child-value shape still needs agreement. |
| Generic grouped-column tests | They protect rendering independently from Shift History. |
| `ShiftHistoryCard`, when the feature resumes | It can own Shift History-specific icons, popovers, commands, and page composition around `MetaDataTable`. |
| Existing filter, paging, polling, and export UX, until API replacements exist | These are live behaviors and must not be lost during migration. |

### Replace When Work Resumes

| Current/deferred item | Replacement | Reason |
|---|---|---|
| `FrontendShiftHistoryTableDataProvider` | API-backed table-data provider | The temporary provider still selects schemas and calculates grouped diameter columns in the frontend. |
| `ShiftHistoryTableDataRequest` | Final API request including rig, mining method, filters, sort, and paging | The current request contains loaded rows and configuration because it adapts the legacy endpoint. |
| `ShiftHistoryTableDataView` and `ShiftHistoryTableDefinition` | API result plus minimal frontend page state | Totals and footer/paging data should follow the final API response contract. |
| `ShiftHistoryTableRowKeys` | Keys returned by API metadata and rows | Display keys must not be predefined per mining method in the frontend. Keep only interaction keys if they are not represented by action metadata. |
| Frontend diameter calculation | API-provided grouped columns and row values | Rig/site ream-size fallback and loaded-shift diameter union are domain/data-provider responsibilities. |
| `_siteType` Production/Development display branches | Returned metadata | Future mining methods must work without adding Razor conditionals. |

### Remove After API Parity

| Item | Reason |
|---|---|
| Handwritten Shift History table markup | `MetaDataTable` renders the returned schema. |
| `FrontendShiftHistoryTableDataProvider` | It is a temporary API emulator. |
| Temporary Shift History request/view/definition/row-key classes that duplicate the final API contract | Duplicate contracts drift and create mapping work. |
| Frontend mining-method checks for grouped columns | The presence of `GroupKey` in returned metadata determines whether a group is shown. |
| Frontend rig/site diameter fallback and page-diameter aggregation | The API should return authoritative columns and totals. |

### Shift History Removal Prerequisites

- The Shift History API returns method-specific `TableDataResult` metadata and rows.
- Production grouped diameters and Development non-grouped layouts are represented entirely by metadata.
- The API contract defines grouped child values or returns concrete grouped child columns.
- Server-side paging, sorting, filtering, totals, and export behavior are agreed.
- Prestart, handover-note, overlapping-shift, and navigation interactions are represented either by metadata actions or a documented frontend extension.
- The new card matches the legacy table layout and behavior before the handwritten table is removed.

## Recommended API Integration Sequence

1. Confirm the final `TableDataResult` JSON, including grouped columns, actions, totals, paging, and localization.
2. Add an API-backed provider behind the existing feature interface.
3. Run the temporary and API providers against the same site and compare columns, rows, totals, and actions.
4. Move sorting, filtering, and paging to the API when datasets are server-paged.
5. Switch the feature card to the API provider.
6. Complete UI and workflow regression testing.
7. Remove the temporary frontend provider and duplicated contracts.
8. Remove legacy feature-specific tables only after all active and archived workflows have parity.

## Related

- [[2026-07-09 Generic MetaDataTable Component Executable Plan]]
- [[2026-07-10 MC-1833 Shift History Group Columns Executable Plan]]
- [[2026-07-09 Generic Metadata Tables Feature Explained]]
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]]
