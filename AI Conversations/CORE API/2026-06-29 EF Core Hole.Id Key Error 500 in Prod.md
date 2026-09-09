---
date: 2026-06-29
source: Claude Code + Codex (VS Code) — parallel investigation, both sessions merged here
project: core-web-api
tags: [debugging, ef-core, drill-plan, rigs-controller, prod-issue, fk-violation]
---

# Prod 500: "The property 'Hole.Id' is part of a key and so cannot be modified"

## Symptom
Prod error log (recursive SaveHole → 3 repeated errors + unhandled exception):
`Error processing Hole 4-rev-rev (id 0). Reason The property 'Hole.Id' is part of a key and so cannot be modified...`
Postman replays locally returned **200 OK with errors in the response json** — never the prod 500.

## Root cause (the actual bug)
In RigsController hole-saving logic:
1. Line ~1367: `orgHole.RecalculatedHole = hole` — sets navigation (hole untracked → EF does not fix up FK yet).
2. Line ~1368: `_context.Entry(orgHole).Property("RecalculatedHoleId").CurrentValue = hole.Id` — ran **before** Add while `hole.Id == 0`, hard-coding a literal `0` FK. SQL Server has no row Id=0 → FK constraint violation at `SaveChanges()`.
3. Lines ~1515–1521 already set `RecalculatedHoleId` correctly *after* insert — so line 1368 was **redundant and harmful**. Fix: remove it and let EF's relationship fixup handle the temporary key.

## Why the prod 500 couldn't be reproduced locally (key insight)
On a clean local DB, when the inner `SaveChanges()` fails the transaction **rolls back** and `DetachFailedChanges()` cleans the context → the later SaveChanges passes → 200 OK.
In prod, **accumulated corruption from prior failed syncs** (bad `RecalculatedHoleId = 0` FK data committed over multiple requests) made the later save fail too → unhandled 500. You cannot recreate stateful-corruption bugs with a single payload against a clean DB.

Related evidence: deleting hole 65528 in prod DB hit `FK_Holes_Holes_OriginalHoleId` SAME TABLE REFERENCE constraint — self-referencing FKs (`OriginalHoleId`, `RecalculatedHoleId`) are the corruption vector.

## The Codex-side investigation (same issue, same day)
Codex worked the same prod logs (`Scotia(132), DRI033 (487)`) and added these pieces:

**Why 200 OK locally despite the error being logged** — the per-hole exception is *caught* (logged + `DetachFailedChanges()`), so Postman sees 200 with errors in the response JSON. The prod 500 happens because the **final `_context.SaveChanges()` after the loop is NOT inside a try/catch** — when poisoned/corrupt state survives to that point, it throws unhandled. That's the exact bridge between "error in log" and "500 to the tablet".

**The two log lines decoded** — `Hole 5-rev-rev (id 0)` (the *new* recalculated hole) and `Hole 5-rev (id 3566943)` (the *existing* original hole) both fail with "Hole.Id is part of a key" — EF trying to rewire an identifying FK on already-tracked entities.

**What the hotfix changed in SaveHole** (verified before continuing, to prove the new errors weren't caused by the hotfix itself):
1. Duplicate recalculated-hole protection — one original hole may have only one recalculated hole (avoids duplicate `OriginalHoleId` one-to-one conflicts).
2. Recalculated-hole attachment changed to attach to the **resolved DB hole** instead of the incoming request hole (the sensitive change; `hole.RecalculatedHole.OriginalHole = holeToSave`).

**Reproduction discipline** — the driving rule of the whole session: *"if we cannot recreate it, we don't know the fix is real — a hotfix release cycle is too slow to guess."* Reproduction attempts included re-uploading an exported drive CSV (hit `FK_Drives_Sites_SiteId` UPDATE conflict — a different bug), inserting `recalculatedHole` + `originalHoleId` into payloads, and testing whether `originalHoleId` must belong to the same drive/ring.

Also asked & answered in the Codex chat: `if (hole.Id <= 0)` distinguishes new holes; `hole.Id = 0` before Add prevents explicit identity inserts (new/smart-collared holes may arrive with a local id — reset before Add so EF doesn't attempt an explicit identity insert).

**Shipping rule from the same incident:** production ran `release/2.21` while work was based on `release/2.22` — hotfixes for PROD must branch off the release branch production actually runs, cherry-picking only the fix; never PR a later release branch into an earlier one.

## Bonus finding
`EnableRetryOnFailure` is **not configured** in `Startup.cs` (~94) — transient SQL failures surface immediately as unhandled exceptions. Candidate hardening:
```csharp
sql.EnableRetryOnFailure(maxRetryCount: 5, maxRetryDelay: TimeSpan.FromSeconds(10), errorNumbersToAdd: null);
```

## Related
- [[2026-05-21 Drill Plan Upload NRE and Duplicate Ring Debugging]]
- [[2026-06-18 Core Sync DB Connection Pool Findings]]
