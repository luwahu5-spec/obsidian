---
date: 2026-06-25
source: Codex (VS Code)
project: core-web-api
tags: [debugging, core-sync, sql-timeout, shifts-controller, ef-core]
---

# PostShift SQL timeout — insert vs update, and why timeouts look "random"

## Reading the EF command log
The prod log showed a command taking **53,816 ms with `CommandTimeout='30'`** — the batch included `UPDATE [Shifts] SET ... WHERE [Id] = @p40` plus `INSERT INTO [DeviceSyncLogs] ...`.

## PostShift does NOT always insert
`ShiftsController.PostShift` (~1866) calls `ProcessLinkedItem(shift, itemId => _context.Shifts.Find(itemId), model => _context.Shifts.Add(...))` (~1941): if `Find(shift.Id)` returns null → INSERT, otherwise the tracked entity is updated → UPDATE. So the logged request was **updating an existing shift**, while sync-log tables got inserts in the same batch.

## Why the timeout is intermittent
Short-lived locks, sync bursts hitting the same rows, or RDS CPU pressure make SQL timeouts appear "random" — they clear when load drops and return during bursts. **Disappearing ≠ fixed**; treat as a recurring blocking/performance risk, especially around
`_context.DeviceSyncLogs.FirstOrDefault(x => x.SyncId == ...)` (see [[2026-06-18 Core Sync DB Connection Pool Findings]]).

## Related
- [[2026-06-05 k6 Load Testing Setup for Core Sync]]
