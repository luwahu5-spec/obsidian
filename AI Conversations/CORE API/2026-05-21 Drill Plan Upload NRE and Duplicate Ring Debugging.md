---
date: 2026-05-21
source: Codex (VS Code)
project: core-web-api
tags: [drill-plan, debugging, nre, rings, unit-of-work, rigs-controller]
---

# Drill Plan POST — NullReferenceException & Duplicate Ring Debugging

## Context
After a hotfix revert on **PUT DrillPlan**, uploads via **POST {id}/DrillPlan** started throwing `System.NullReferenceException` in `UnitOfWork.TrackDeletedEntities()` (`Minnovare.Core.Database/UnitOfWork.cs:90`, called from `SaveChangesAsync`). Same dataset worked before — suspicion was the revert also touched POST.

## Why POST deletes holes during an "upload" (important!)
`POST {id}/DrillPlan` is **not add/update-only** — it treats the uploaded plan as the **new full state** of the drive/ring:
```csharp
var holesToRemove = new List<Hole>(dbRing.Holes);   // assume all DB holes go
holesToRemove.Remove(dbHole);                       // keep matched ones
_unitOfWork.HoleRepository.RemoveRange(holesToRemove); // delete the rest
```
So any DB hole missing from the payload gets **deleted** — and the NRE came from deleting a hole whose Ring/Drive navigation wasn't loaded.

## Duplicate ring failure
SQL error `IX_Rings_DriveId_Name duplicate key (69897, R50)` — one drive cannot have two rings with the same name. Root cause: the **payload itself contained `R50` twice** under drive `HJ8732 500 13 V4` (plus a trailing comma making the JSON invalid as pasted).

Ring matching logic in `RigsController.cs` (~line 680):
```csharp
r.Id == ring.Id || (r.Name == ring.Name && r.DriveId == drive.Id)
```

## Which endpoint produced the log (recovered detail — important!)
`"Error processing Hole ..."` comes **only from `SaveHole`'s catch block**, and `SaveHole` is called only by **`PUT {id}/DrillerDrillPlan`** (the tablet/driller sync upload) — **not** by `POST {id}/DrillPlan` (which has its own inline hole/ream logic). When the tablet "uploads a drill plan", the failing path is the PUT, despite the name.

## Additional bugs found in the same investigation
- **Inverted POST ream logic** (~line 731): it *added* when `dbReam != null` and *updated* when `dbReam == null` — backwards; could duplicate an existing ream or NRE in `ApplyChanges`. Fixed, together with a nullable modified-date comparison.
- **Logging hides the real SQL error**: catch block logged only `ex.Message` ("See the inner exception for details") — the inner SQL exception was never written. Fix: log the inner exception.
- **Failed `SaveChanges` leaves poisoned tracking**: after a per-hole save failure, EF keeps the failed Added/Modified entries tracked; the later final `_unitOfWork.SaveChanges()` replays them and turns a recoverable per-hole error into the endpoint-level 500. Clean failed pending changes after a `SaveHole` failure.
- **DB constraints that produce these failures**: unique index on `Holes (RingId, Name)`, unique *filtered* index on `OriginalHoleId`, required `Reams.HoleId` + optional driller/rig FKs. Usual SQL causes: duplicate hole name in a ring, duplicate recalculated-hole link, stale tablet FK pointing at a missing row, or "new" entity arriving with a positive Id (explicit identity insert rejected — `Id = 0` means "let SQL generate it", safe for multiple new entities).

## Lessons
- When "same dataset worked before", check whether a revert touched shared code paths (PUT revert affecting POST).
- Check the payload for duplicates before suspecting the matching logic.
- Full-state-replacement endpoints need navigation properties loaded before RemoveRange.

## Related
- [[2026-06-05 k6 Load Testing Setup for Core Sync]]
