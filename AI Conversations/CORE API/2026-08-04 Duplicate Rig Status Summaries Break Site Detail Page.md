---
date: 2026-08-04
source: Codex (VS Code)
project: core-web-api, core-web
tags: [debugging, rig-status-summary, duplicate-data, ef-core, site-detail, concurrency]
---

# Duplicate `RigStatusSummary` rows break site detail loading

## Executive summary

Opening site ID `143` produced this API exception:

```text
Sequence contains more than one element.
at RigStatusSummaryRepository.GetForRigAsync(Int64 rigId)
at RigsController.RigStatusSummaries(Int64 rigId)
```

The site lookup itself was not the failing query. CORE Web first retrieved the site's rigs through `GET /Sites/143/Rigs`. Each rig card then made a separate request to `GET /Rigs/RigStatusSummaries/{rigId}` to obtain its calibration status.

`RigStatusSummaryRepository.GetForRigAsync` uses `SingleOrDefaultAsync`. That operation accepts zero or one matching status row. The exception proves that the database returned at least two `RigStatusSummaries` rows for the requested rig.

The immediate data problem was therefore:

```text
One rig under site 143 had more than one RigStatusSummaries row.
```

This table is conceptually a current-status snapshot and should contain only one row per rig. However, the database has only a non-unique index on `RigId`, so it does not enforce that rule.

## Request chain

```text
Open site detail for site 143
        |
        v
GET /Sites/143/Rigs
        |
        +-- succeeds and returns the site's rigs
        |
        v
RigCardComponent.GetCalibrationDue()
        |
        v
GET /Rigs/RigStatusSummaries/{rigId}
        |
        v
RigStatusSummaryRepository.GetForRigAsync(rigId)
        |
        v
SingleOrDefaultAsync finds multiple rows
        |
        v
HTTP 500: Sequence contains more than one element
```

The preceding log entry from `SitesController.Rigs` identifies the page activity, but the stack trace identifies `RigsController.RigStatusSummaries` as the actual failing endpoint.

## Confirmed root condition

The repository query is:

```csharp
return await _dbSet
    .Include(rss => rss.Rig)
    .Include(rss => rss.Site)
    .SingleOrDefaultAsync(rss => rss.Rig.Id == rigId);
```

`SingleOrDefaultAsync` behaves as follows:

| Matching rows | Result |
|---:|---|
| 0 | Returns `null` |
| 1 | Returns the status row |
| 2 or more | Throws `Sequence contains more than one element` |

The error is not caused by a duplicate site or by the number of rigs at the site. It specifically means there are multiple status rows whose `RigId` points to the same rig.

## Diagnostic SQL

Find every affected rig under site 143:

```sql
SELECT
    rss.RigId,
    r.Name AS RigName,
    COUNT(*) AS SummaryCount
FROM dbo.RigStatusSummaries AS rss
INNER JOIN dbo.Rigs AS r
    ON r.Id = rss.RigId
WHERE r.SiteId = 143
GROUP BY rss.RigId, r.Name
HAVING COUNT(*) > 1;
```

Inspect all candidate rows before changing any data:

```sql
SELECT
    rss.*,
    r.Name AS RigName
FROM dbo.RigStatusSummaries AS rss
INNER JOIN dbo.Rigs AS r
    ON r.Id = rss.RigId
WHERE r.SiteId = 143
ORDER BY rss.RigId, rss.Id;
```

Check whether the problem exists elsewhere in the database:

```sql
SELECT
    RigId,
    COUNT(*) AS SummaryCount
FROM dbo.RigStatusSummaries
GROUP BY RigId
HAVING COUNT(*) > 1;
```

## Why the database permits duplicates

The EF relationship is configured as a required many-to-one relationship:

```csharp
builder.Entity<RigStatusSummary>()
    .HasOne(r => r.Rig)
    .WithMany()
    .IsRequired();
```

The original migration created this index:

```csharp
migrationBuilder.CreateIndex(
    name: "IX_RigStatusSummaries_RigId",
    table: "RigStatusSummaries",
    column: "RigId");
```

It is not marked `unique: true`. The application expects one status row per rig, but SQL Server is allowed to store any number of them.

## How duplicate rows could be created

The stack trace proves the duplicate data condition, but it does not prove when or which operation created the extra row.

The strongest code-level mechanism is a race during first-time status creation. `RigStatusSummaryRepository.UpdateDetails` performs a check followed by an insert:

```csharp
var rigSummary = _dbSet.FirstOrDefault(rs => rs.Rig.Id == rig.Id);

if (rigSummary == null)
{
    rigSummary = new RigStatusSummary
    {
        Rig = rig,
        Site = rig.Site
    };

    _dbSet.Add(rigSummary);
}
```

Two concurrent requests can both observe that no row exists and then both insert one. Because there is no unique constraint on `RigId`, both inserts can succeed.

Other possible origins include a historical import, a manual database operation, or a data-rebuild utility. Database audit information would be needed to distinguish these possibilities after the fact.

## Safe recovery

1. Run the diagnostic queries and identify every duplicated `RigId`.
2. Compare the rows' `LastSyncDate`, `LastCalibrationUpdate`, `CalibrationState`, and `SensorErrorCount`.
3. Decide which values represent the authoritative current state. The row with the highest `Id` is not automatically correct because different requests update different status fields.
4. Back up the candidate rows.
5. Merge any required current values into one retained row and delete only the confirmed obsolete rows.
6. Re-run the duplicate query and verify it returns no results.
7. Add a unique database index on `RigStatusSummaries.RigId` so the invariant is enforced permanently.

Do not change `SingleOrDefaultAsync` to `FirstOrDefaultAsync` as the fix. That would hide the corruption and make the selected status row dependent on unspecified database ordering.

## Permanent prevention

The model and migration should enforce uniqueness, for example with a unique index on the shadow foreign-key property:

```csharp
builder.Entity<RigStatusSummary>()
    .HasIndex("RigId")
    .IsUnique();
```

Existing duplicates must be cleaned before applying that migration. Otherwise, SQL Server will reject creation of the unique index.

The first-insert path should also handle concurrent creation deliberately. The unique constraint is the final authority; application logic can then catch a duplicate-key race and reload the row, or use a transaction/upsert strategy.

Recommended regression coverage:

- A rig with no status row returns `null` without error.
- A rig with one status row returns that row.
- The database rejects a second status row for the same rig.
- Concurrent first-time updates result in exactly one status row.
- Site detail still loads when a rig has no calibration status yet.

## Relevant code

- `Minnovare.Core.Database/Repositories/RigStatusSummaryRepository.cs`
- `Minnovare.Core.Database/ApplicationDbContext.cs`
- `Minnovare.Core.Database/Migrations/20220531015108_MC-524_Rig_And_Calibration_Status.cs`
- `Minnovare.Core.WebApi/Controllers/RigsController.cs`
- `Minnovare.Core.Web/Repositories/RigRepository.cs`
- `Minnovare.Core.Web/Pages/Blazor/Shared/RigCardComponent.razor.cs`

## Related notes

- [[2026-07-27 Stale Calibration Date and Rig Status Lost Update]] — explains why `RigStatusSummary` is a single current snapshot and documents a separate full-row lost-update risk in the same repository.

