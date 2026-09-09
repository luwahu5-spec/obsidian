---
date: 2026-03-27
source: Claude Code (VS Code)
project: core-web-api
tags: [deswik, k92, shift-integration, api-design, dto, sql]
---

# Deswik–K92 Shift Integration — API Design (Claude session)

## Context
Deswik (on behalf of shared customer K92) requested a data integration between **Deswik OPS** and **Minnovare CORE**, building on the earlier LiveMine / DigiPLOD integration. Deswik will only *call* our API and use returned data directly — **they compute nothing**, so all values (utilisation, summaries, names) must be resolved server-side.

## Data Deswik needs (shift-level)
- Rig utilisation (Operating / Idle / Down)
- Consumables, Driller attribution, Shift events and pauses
- Hole-by-hole drilling metrics
- QA/QC compliance (PreStartData)

## Design decisions reached
- A single-call endpoint returning the full structured shift + hole data (they likely make one call and want everything).
- Reuse existing endpoints where possible (`GET /Shifts/ByRig/{rigId}?from=&to=` should include all shift-related info) rather than adding both `/Full` and `/ByRig` variants.
- Keep `GET /Shifts/{id}/DrilledObjects` but new version filtered by id + date range.
- Lookup endpoints kept in play: `/Drillers/{id}`, `/ShiftPauseReasons/{id}`, `/ConsumableReasons/{id}`, `/ConsumableTypes/{id}`, `/Shifts/{shiftId}/PreStartData`.
- Used `/codex:rescue` mid-session for a second-opinion design review of the proposed API.

## SQL snippets worth keeping
Create a customer API key (ApiKey is a `Guid`):
```sql
INSERT INTO [dbo].[Customers] ([Name], [ApiKey]) VALUES ('Deswik', NEWID())
SELECT [Id], [Name], [ApiKey] FROM [dbo].[Customers] WHERE [Name] = 'Deswik'
```
Bool → SQL `bit`: `true = 1`, `false = 0` (e.g. `Archived = 1` for archived rigs).
Update rigs safely — always constrain by `Id`, not just `SiteId`:
```sql
UPDATE [minnovare].[dbo].[Rigs]
SET [Archived] = 1, [Modified] = GETUTCDATE()
WHERE [SiteId] = 114 AND [Id] = <id>
```

## Git workflow used
Branch created from MC-1834: `prod_core-sync-issue-release-2.22` (for release/2.22), stash replayed onto it → changes in `RigsController.cs` + `Minnovare.Core.Shared` submodule.

## Related
- [[2026-04-23 Deswik Shift Endpoint Implementation]]
- [[2026-06-05 k6 Load Testing Setup for Core Sync]]
