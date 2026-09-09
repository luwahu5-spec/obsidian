---
date: 2026-07-07
updated: 2026-07-07
source: Claude Code
project: both
tags: [domain-knowledge, development-drilling, drives]
---

# Drive "Last Updated" column — what it actually shows

> **Takeaway:** "Last Updated" on the Drive list (Site Detail → Development Drilling tab) is NOT a
> row-modified/audit timestamp. It's `MAX(DateCompleted)` across every drilling-activity submission
> recorded against that drive, computed fresh on every page load — falling back to the drive's
> `Created` date if no drilling submissions exist yet. `DateCompleted` itself is client-supplied
> (driller tablet's "end cut time"), with no server-side check that it isn't in the future or before
> `Created`.

## The business/physical reality being modelled
A development drive (heading/tunnel) accumulates drilling activity over many shifts. Each time a
driller/tablet submits a shift's drilling work against a drive (cut length, holes drilled, chainage,
cut start/end time), that becomes one `DevelopmentDriveDrillingDetails` row. There is no single
"drive was edited at X" concept in this model — "Last Updated" is really "when was drilling work on
this drive last recorded as completed."

## The configuration chain
- UI: `DevelopmentDriveDetailsComponent.razor:118` renders `DevelopmentDrive.LastUpdated`
  (`UserTimeService.ConvertToUserTimezoneDisplay(...)`).
- Code-behind: `DevelopmentDriveDetailsComponent.razor.cs:91` populates the list from
  `DevelopmentRepository.GetDrivesForSite(SiteId, ArchivedDrives)`.
- Repo call: `Repositories/DevelopmentRepository.cs:26` → `GET /Development/GetDrivesForSite?siteId={id}&archived={bool}`.
- API (core-web-api): `Controllers/DevelopmentController.cs:565` `GetDrivesForSite(siteId, archived)`.

## Where it takes effect (the actual formula)
`Controllers/DevelopmentController.cs:567-580` — **not a stored column, computed per request**:
```csharp
var drives = ... select new DevelopmentDriveDisplayModel { Id, DriveName, Created = d.Created };

var drillDetails = _context.DevelopmentDriveDrillingDetails
    .Include(dddd => dddd.Drive)
    .Where(dddd => drives.Select(d => d.Id).Contains(dddd.Drive.Id));

drives.ForEach(d => d.LastUpdated =
    drillDetails.Any(dd => dd.Drive.Id == d.Id)
        ? drillDetails.Where(dd => dd.Drive.Id == d.Id).Select(dd => dd.DateCompleted).Max()
        : d.Created.Value.ToUniversalTime());
```

`DateCompleted` (`Minnovare.Core.Shared/Models/DevelopmentDriveDrillingDetails.cs:20`, the "end cut
time") is written at `Controllers/DevelopmentController.cs:819`:
```csharp
drilledDetail.DateCompleted = drillingDetailCommandModel.DateCompleted;
```
straight from the incoming `DevelopmentDriveDrillingCommandModel.DateCompleted`
(`DevelopmentDriveDrillingDetails.cs:57`) — i.e. whatever the submitting driller device says, no
clamping to "not in the future" or "not before drive Created."

`DateCompleted` became a `DateTimeOffset` (stored as UTC) via migration
`20241122022136_ChangeCutDateCompeletedToDatetimeOffset.cs:21`
(`UPDATE ... SET DateCompleted = CAST(DateCompleted AS datetimeoffset) AT TIME ZONE 'UTC'`).
`DevelopmentDrive.LastUpdated` on the display model is typed `DateTimeOffset?` but is fed either
`DateCompleted` directly or `Created.Value.ToUniversalTime()` — both are UTC instants, consistent,
but the naming doesn't make the two different sources obvious (flagged as a dev comment in
`DevelopmentDrive.cs:16`).

## How it feeds the UI
Sorting/filtering by "Last Updated" (`DevelopmentDriveDetailsComponent.razor.cs:110`,
`DateTimeOffsetSpecification` at `:168`) operates on this computed value, so sort order and date-range
filters reflect drilling-submission recency, not drive-record edit recency.

## Design subtleties & edge cases
- **Drive never drilled yet** → Created == Last Updated (no `DevelopmentDriveDrillingDetails` rows
  exist), which can misleadingly look like "just touched" when it really means "never drilled."
- **One stray submission dominates** → since it's `MAX()` over potentially many rows per drive, a
  single future-dated or backdated cut submission (bad device clock, manual edit) will override the
  Last Updated for the *entire* drive, masking all other legitimate activity.
- **Not an audit trail** — renaming the drive, archiving it, or any admin action does NOT move this
  value at all. Only posting through `PostDevelopmentDriveDrillingDetails`
  (`DevelopmentController.cs:766`) changes it.
- **No server-side validation** on `DateCompleted` vs "now" or vs `StartCutTime`/`Created` — trust is
  entirely in the submitting client.

## Questions asked
- 2026-07-07 — "Check the logic behind the Last Updated column, Site Detail → Development Drilling
  tab → Drive section" → It's `MAX(DateCompleted)` over drilling submissions per drive, falling back
  to `Created`; not an audit timestamp. See formula above.

## Related
- [[2026-06-26 Outstanding Length Excludes Recalculated Holes]] — another case of a displayed value
  being a computed model-level property rather than a stored column.
