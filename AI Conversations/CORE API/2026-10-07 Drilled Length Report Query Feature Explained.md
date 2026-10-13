---
date: 2026-10-07
updated: 2026-10-07
source: Claude Code
project: core-web-api
tags: [domain-knowledge, reports, drilled-length, ef-core, fromsql]
---

# Drilled Length report query — how it's fetched, and why FromSqlRaw is still there

> **Takeaway:** The report is "total metres drilled per rig and driller at a site, optionally filtered by rig, driller and date range".
> - `ReportRepository.GetDrilledLengthReportAsync` gets it by calling the stored procedure `DrilledLengthReport` with `FromSqlRaw`.
> - EF LINQ can't call a stored procedure, which is why that line remains.
> - **But the procedure is only a filtered GROUP BY/SUM over the `DrilledObjects` view, so it can be rewritten in LINQ.** That needs a new keyless entity shaped like the view, because the existing `LengthReport` entity is shaped like the procedure's output.

## LINQ rewrite: built, verified, then reverted (2026-10-08)

> **Status: reverted on request ("not now, keep the procedure").** The code still calls the
> procedure through `FromSqlRaw`, with a comment explaining why. Below is the verified design,
> kept for when this is picked up again.

The rewrite made `ReportRepository.GetDrilledLengthReportAsync` stop calling the procedure:
- It rebuilds the view's logic from the **existing** entities. No new entity and no migration were needed: `Holes`/`Reams` → `DrilledByRig`, `DrilledByDriller`, `Ring.Drive.Site` (reams via `Hole.Ring.Drive.Site`).
- **Holes:** `DrillStatus == Done` (=2), `ActualLength` and `DateDrilled` not null. **Reams:** `ActualLength` and `DateDrilled` not null. Imperial sites (`MeasureSystem != Metric`) multiply by 0.3048.
- **Two grouped queries** (holes, reams), each with filters → `GroupBy` → `Sum` before `ToListAsync`, so SQL does the totalling. The two small results are merged in C#: null rig/driller → 0; DrillerName `"Given Family"`, or null with no driller.
- **Verified against the procedure** on the local prod copy, 12 cases: sites 8, 36, 47, 18, 68 (imperial), 11, 1 (many null rig/driller), plus rig, driller and both filters, and date ranges in +08/+00/-05.
  - Identical rows, ids and names.
  - Lengths differ only by float rounding on imperial sites (site 8: 0.022 m over 1.87M m). The procedure multiplies in `real`, LINQ in `double`.
- **Leftovers, left on purpose:**
  - The `DrilledLengthReport` procedure and `DrilledObjects` view stay in the database. The view is also used by another procedure (`Initial.cs:668`).
  - The `LengthReports` DbSet/`ToView` mapping stays. Removing it changes the model snapshot and needs a migration, and the `LengthReport` class is still the controller's response DTO.

## The business reality

- `DrilledObjects` (view, `20190917023952_Initial.cs:542-569`) is a list of every drilled piece of work.
- It UNIONs completed holes (`DrillStatus = 2`) and reams that have a length and a drilled date.
- Length is converted to metres when the site uses imperial units (`MeasureSystem != 0` → ×0.3048).
- Rig and driller come from FULL JOINs, so either can be NULL (drilled by an unknown rig or driller).
- The view has never been altered since it was created (no later `ALTER VIEW` in migrations).

## The configuration chain

| Piece | Where | What |
|---|---|---|
| Stored procedure | `20190924070704_CalibrationAndSyncLogs.cs:74-100` | `WHERE SiteId = @siteId` plus optional rig, driller, `@fromDate`, `@toDate`; `GROUP BY RigId, RigName, DrillerId, GivenName, FamilyName`; `SUM(Length)`; NULL RigId/DrillerId → 0; DrillerName = `CONCAT(GivenName,' ',FamilyName)`, or NULL when there's no driller |
| EF entity | `LengthReport.cs` (RigId, RigName, DrillerId, DrillerName, Length), mapped `HasNoKey().ToView("DrilledObjects")` (`ApplicationDbContext.cs:267`) | **Shaped like the procedure's output, not the view.** `DrillerName` doesn't exist in the view. The `ToView` mapping only exists so EF has an entity to run `FromSqlRaw` against |
| Repository | `ReportRepository.cs:33` | `_context.LengthReports.FromSqlRaw("DrilledLengthReport @siteId, …")` |
| Caller | `SitesController.GetDrilledLengthReport` → `_unitOfWork.ReportRepository` | Replaces null results with "Unknown" names |

## Design subtleties & edge cases

- **What the lead objected to.** The objection was `FromSql` on the **generic `IRepository`**: any controller could run raw SQL and bypass the Unit of Work. That method was removed. The one remaining call is wrapped in a dedicated `IReportRepository` behind `IUnitOfWork`, so controllers can't write SQL.
- **Why `_context.LengthReports` can't just be queried with LINQ.** EF would `SELECT DrillerName …` from the view, and that column doesn't exist. A LINQ rewrite needs a new keyless entity matching the view columns (`SiteId`, `DateDrilled`, nullable `RigId`/`DrillerId`, `RigName`, `GivenName`, `FamilyName`, `Length`). Then `Where` → `GroupBy` → `Sum`, translated to SQL.
- **Behaviour to preserve in a rewrite:**
  - NULL rig/driller → id 0.
  - `CONCAT` treats NULL names as empty strings.
  - DrillerName is NULL when there's no driller.
  - Dates are compared as `datetimeoffset` (UTC).
  - Grouping includes the names, so two drillers with the same id but different names would split. They can't in practice, but the SQL does it.
- A new entity changes the model snapshot. It needs its own migration (empty `Up`/`Down`, because views aren't created by EF).
- After a rewrite the stored procedure is unused. Drop it in a later migration only after it's confirmed nothing else (Lambdas, reports) calls it.

## Plain-language picture

- **View `DrilledObjects`** = raw rows, one per hole or ream: site, date, rig, driller first/last name, length.
- **Procedure `DrilledLengthReport`** = a program in SQL Server that filters those rows and **adds up length per rig and driller**.
- **Class `LengthReport`** = the shape of the **totals** (RigId, RigName, DrillerId, DrillerName, Length).
- LINQ can read tables and views but **can't call procedures**, so running the procedure needs `FromSqlRaw`.
- Reading the view with LINQ through `LengthReport` fails: it would ask the view for `DrillerName` (doesn't exist), and it lacks `SiteId` and `DateDrilled`. It's like reading raw receipts through a form designed for the monthly summary.
- **Fix:** add a class shaped like the view, then filter, group and sum in LINQ. EF turns that into one SQL query.

## Questions asked
- 2026-10-07 — "Lead wants FromSql gone. Why does GetDrilledLengthReportAsync still use FromSqlRaw, can't we use the context?" → The lead's concern (raw SQL on the generic repository) is resolved. The remaining call runs a stored procedure, which LINQ can't. It can be replaced by LINQ over the view, with a new view-shaped keyless entity. See *Design subtleties*.
- 2026-10-07 — "I still don't get it, explain again" → Added *Plain-language picture* (view = raw rows, procedure = totals, class = totals shape).
- 2026-10-07 — "Can we `_context.LengthReports.Where(...)` and build the report in code?" → No.
  - EF would SELECT `DrillerName` from the view, which fails before any rows arrive, and `LengthReport` has no `SiteId`/`DateDrilled` to filter on.
  - It works with a view-shaped class (`DrilledObject`). Put `GroupBy`/`Sum` **before** `ToListAsync()` so the database still aggregates; summing in memory would pull every hole and ream for the site.
  - `LengthReport` is also the API response DTO (`SitesController.cs:768`), so leave it unchanged.
- 2026-10-08 — "Check which tables the procedure uses; if LINQ joins need no new entity, drop raw SQL" → All 7 tables (Holes, Reams, Rigs, Drillers, Rings, Drives, Sites) are existing entities with navigations, so it's done in LINQ. See *LINQ rewrite*.
- 2026-10-08 — "Let's not do that now, revert, keep the procedure" → Reverted. `ReportRepository` and `IReportRepository` are back to the procedure; only the explanatory comment above `FromSqlRaw` remains as a change.
- 2026-10-08 — "Online advice: `SqlQuery<T>` with interpolation for functions, `ExecuteSqlAsync` with `CALL` for procedures. Does it apply?" → Partly.
  - The advice is PostgreSQL-specific. SQL Server uses `EXEC`, not `CALL`.
  - `ExecuteSqlAsync` only returns rows affected, so it can't return our report.
  - `Database.SqlQuery<DrilledLengthReportRow>($"EXEC DrilledLengthReport {siteId}, …")` **does** work (EF Core 8.0.23 supports unmapped `T`). It's parameterized, null-safe, maps straight to the DTO, and drops the SqlParameters and the `LengthReport`/`ToView` dependency.
  - It's still raw SQL and can't be composed over (no `.Where` after `EXEC`). A tidier call, not an elimination.

## Related
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]]
