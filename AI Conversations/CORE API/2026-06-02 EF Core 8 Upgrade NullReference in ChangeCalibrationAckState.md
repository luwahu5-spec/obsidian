---
date: 2026-06-02
source: Codex (VS Code)
project: core-web-api
tags: [debugging, ef-core-8, upgrade, nre, navigation-properties, rigs-controller]
---

# EF Core 6→8 upgrade broke `ChangeCalibrationAckState` (NRE on `rcs.Rig.Id`)

## Symptom
`RigsController.ChangeCalibrationAckState` (~line 2353) threw `System.NullReferenceException: RigCalibrationSummary.Rig.get returned null`. **Staging (EF Core 6) was fine; the EF Core 8 upgrade branch crashed.**

## Root cause
Old pattern materialized everything first, then touched navigation properties in memory:
```csharp
var calibrationSummary = _unitOfWork.Repository<RigCalibrationSummary>()
    .GetAllAsync().Result
    .OrderBy(rcs => rcs.Id)
    .LastOrDefault(rcs => rcs.Rig.Id == rigId);   // rcs.Rig is null in memory
```
After `.Result`/materialization, the predicate runs as **LINQ-to-Objects** — `rcs.Rig` is only non-null if EF happened to fix it up from already-tracked entities. EF Core 8's changed tracking/lazy behavior stopped papering over it.

## Fix
Query with the FK translated in SQL instead:
```csharp
_context.RigCalibrationSummaries.Where(rcs => rcs.Rig.Id == rigId)...
```
(predicate stays an EF **expression** → translated to SQL, no in-memory navigation access). Fixed on branch MC-1834 (created on top of MC-1801).

## Rule of thumb (project-wide sweep result)
- **Safe**: `repository.Find(rss => rss.Rig.Id == rigId)` — predicate stays an expression until `ToList()`.
- **Risky**: `GetAllAsync().Result ... .LastOrDefault(x => x.Nav.Prop)` — materialize-then-navigate.
- Closest other risk found: `DevelopmentController.GetDrivesForSite` (~580): `drillDetails.Any(dd => dd.Drive.Id == d.Id)` after materialization.

## Lesson
When an ORM upgrade "randomly" breaks an endpoint, look for **in-memory navigation property access after materialization** — behavior differences between EF versions surface exactly there.
