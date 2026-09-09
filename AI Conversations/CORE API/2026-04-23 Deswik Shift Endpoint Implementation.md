---
date: 2026-04-23
source: Codex (VS Code)
project: core-web-api
tags: [deswik, k92, shift-integration, dto, testing, sql]
---

# Deswik Shift Endpoint — Implementation (Codex session)

## Context
Implementation follow-up to [[2026-03-27 Deswik K92 Shift Integration API Design]]. New endpoint on the Shifts controller returning structured shift + hole data for Deswik in **one call**, only for **finalised shifts**.

## Key decisions
- **ShiftSummary mandatory** for readiness; **RigComplianceSummary included** (Deswik needs compliance data too).
- Flat DTO with **empty arrays instead of nulls**.
- **Deterministic UTC-based latest-shift selection**.
- Reused existing holes-plus-reams model rather than inventing a `DrilledObjects[]` shape.
- **DTO consolidation**: pushed back on 7–8 DTOs → one DTO embedding actual content (not just IDs). Lists like `Consumables`, `DrillerShifts` must be *unpacked* — Deswik can't chase IDs via `GET /ShiftPauseReasons/{id}` etc.
- `DowntimeNotes` / `ServiceNotes` / `ConsumableNotes` replaced by `HandOverNotes` from the shift entity.
- **Placement rule**: entity/model classes live in `Minnovare.Core.Shared` (Web needs them too); **DTOs stay in the API project only** — never in Shared.

## Testing patterns added
Endpoint validation tests for the shift-data controller:
- `400 BadRequest` for missing customer id, missing timestamp, empty rig array (validated **before** the customer API-key check).
- Mixed-rig case: one authorised rig succeeds, unauthorised rig returns a warning item.

## SQL worth keeping
Customer↔Site link is `Sites.CustomerId`:
```sql
UPDATE [MinnovareProd].[dbo].[Sites] SET [CustomerId] = 4 WHERE [Id] IN (36, 47);
SELECT [Id], [Name], [CustomerId] FROM [MinnovareProd].[dbo].[Sites] WHERE [Id] IN (36, 47);
```
Assigning sites to a customer lets that customer's API key access rigs belonging to those sites.
