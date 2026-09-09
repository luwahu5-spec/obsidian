---
date: 2026-06-18
source: Codex (VS Code)
project: core-web-api
tags: [debugging, core-sync, sql-pool, ef-core, async-void, prod-issue]
---

# Core Sync timeouts — DB connection / context hygiene findings

## Investigation
Following the PROD core-sync timeouts (EF log showed `WHERE SyncId = ...` on `DeviceSyncLogs`, issued by `DeviceSyncLogService`), the config/runtime layer was audited.

## Findings
1. **No `Max Pool Size`** in the production connection string (`appsettings.json`) — ADO.NET defaults to pooling on, max ~100 per exact connection string per process. Massive connection use is a plausible multiplier when RDS CPU is already high.
2. **`DeviceSyncLogService.SetDeviceSyncLog` is `async void`** — its DB work can't be awaited, retried, or error-handled by callers.
3. **Manually created `ApplicationDbContext` in `SitesController`** without `using`/DI disposal — connection hygiene smell.
4. A **fire-and-forget import creates its own context manually** — another path multiplying connections outside DI.
5. EF registered with plain `AddDbContext` (no pooling/retry options) in `Startup.cs` ~94.

## Related
- [[2026-06-05 k6 Load Testing Setup for Core Sync]] (the SyncId lookup / index question)
- [[2026-06-29 EF Core Hole.Id Key Error 500 in Prod]] (EnableRetryOnFailure missing)
