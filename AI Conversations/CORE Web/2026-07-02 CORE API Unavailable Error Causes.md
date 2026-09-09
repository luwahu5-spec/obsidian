---
date: 2026-07-02
source: Claude Code (VS Code)
project: core-web
tags: [debugging, api-service, connection, ports]
---

# "CORE API Unavailable. Please try again later." — what actually throws it

Thrown by `ApiService.cs` (Repositories/) in three cases:
1. **SocketException** (~60) — connection refused: nothing listening on the configured address.
2. **TaskCanceledException** (~296) — HTTP request timed out.
3. Explicit task cancellation (~52).

## Checklist when "API is running but frontend says unavailable"
- Compare `appsettings.Development.json` → `CoreAPI.BaseUrl` (e.g. `http://localhost:5001`) with the port the API *actually* bound (a .NET 8 API defaults to `http://localhost:5000` / `https://localhost:7001` unless configured).
- API still warming up when the first request hit → transient.
- Also note: PostHole duplicate-name returns `Conflict(...)` = a normal **409 response helper**, not an exception — if testers report a "crash" on duplicates, check whether the precheck is bypassed by casing/whitespace or a race into the DB unique index (2026-05-28 session).
