---
date: 2026-05-19
source: Claude Code (VS Code)
project: core-web
tags: [debugging, system-status-dashboard, site-filter, blazor]
---

# System Status Dashboard — sites missing from table/picker

## Symptom
A site appeared in the site-picker popup but never showed in the dashboard table (local test).

## Root cause 1 — inconsistent API params
`SystemStatusDashboard.razor.cs` called `GetSitesAsync()` with **no parameters**, while SitesList passed `showArchiveStatus: ShowActiveOnly, siteType: All`. The `/sites/` endpoint defaults `siteType` differently when omitted, so site sets differed between pages. Fix: pass the same parameters.

## Root cause 2 — two gatekeepers decide table rows
A site must pass **both** to render a row:
1. **Compliance data**: `POST /Sites/GetSiteComplianceSummaries` (via `UpdateFilter()`) only returns sites with shift/compliance data in the selected date range.
2. **Rig status**: `SiteStatusList` built from `GET /Rigs/RigStatusSummaries`; razor skips the row if `FirstOrDefault(x => x.SiteId == ...)` is null — sites without rig status summary records never render.

## UX fix derived from the diagnosis
The picker was showing sites that could never pass the gatekeepers → after `LoadStatusData()`, trim `Sites` to those present in `SiteStatusList`, so popup and table always agree.

## Debugging pattern
When "X shows in one place but not another", diff the exact API calls + parameters each page makes before diving into rendering logic.
