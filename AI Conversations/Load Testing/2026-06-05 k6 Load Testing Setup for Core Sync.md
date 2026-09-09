---
date: 2026-06-05
source: Codex (VS Code)
project: core-web-api
tags: [load-testing, k6, core-sync, staging, prod-issue]
---

# k6 Load Testing Setup for Core Sync

## Context
A PROD issue on **core-syncs** was prioritized over PI planned work. Lead asked to set up a **load testing environment on Staging** to replicate the issue, so fixes can be *proven* rather than guessed ("otherwise we're just guessing and hoping we fix the issue").

## What "load testing environment" means
A safe non-production setup (Staging) where simulated traffic is sent at core-syncs to reproduce the PROD problem under pressure — watching logs, errors, CPU/memory, DB load, timeouts.

## The core-sync flow
- Tablet user clicks the **"Core sync" button** → triggers a sequence of API calls (mostly GET + some POST).
- Auth: **bearer token** — for load testing, call a separate token API instead of interactive login.
- Tool chosen: **k6** (installed locally first, plan was to move to Staging once familiar).
  - **VUs** = virtual users (concurrent simulated tablets); **duration** = how long the test runs.

## Key design decisions
- Tests made **flexible via env parameters**: `BASE_URL`, `TOKEN_URL`, `SITE_ID`, `PAYLOAD_SET`, `DATASET_ID`, `DATASET_NAMES` in `.env.local` — so different sites/problems can be targeted and re-run after fixes.
- Payload datasets organized as `payloads/<PAYLOAD_SET>/dataset_<DATASET_ID>/dataset-NNN/` (e.g. `DATASET_ID=dev` → `payloads/current/dataset_dev`, folders `dataset-001`..`dataset-050`).
- Scripts live in `C:\Users\CULI\OneDrive - Hexagon\Desktop\Load Testing\production-core-sync\` (plus a `browser-polling` variant with a `run-browser-polling.ps1` runner).

## Recovered design details (from mid-session)
- **Core sync is not one endpoint — it's a journey**: metadata reads → assigned development drives → drilling instructions per drive → uploads (targets, shifts, sensor states). The load test replays the whole journey with a **`syncId`** so the server's `DeviceSyncLogs` can correlate the calls.
- **Safety-first versioning**: first script shipped with `READ_ONLY=true` default (GET/metadata only); POST hooks filled in later once real tablet payloads and write permission existed.
- **Incident-replay pattern**: workflow script stays stable; payloads are swappable via `PAYLOAD_SET` folders — drop in the exact payloads copied from PROD logs for a specific failure and rerun the same flow.
- **Token flow**: script calls Keycloak itself (`/connect/token` on the base URL) and extracts `access_token`; `AUTH_TOKEN` env is only a quick-debug override. Keycloak expects the form key **`scope`** (`TOKEN_SCOPE=offline_access`) even though Postman calls it `scopes`.
- k6 quirks: `open()` only works during script **init**; installed via winget to `C:\Program Files\k6\k6.exe` (fresh terminal needed for PATH).
- First package lived in-repo at `load-tests/core-sync/` (later moved to the Desktop Load Testing folders to keep the repo clean).

## Root-cause finding (DeviceSyncLog)
- Timeout traced to a lookup by `SyncId` in [[DeviceSyncLogService]] (`Minnovare.Core.WebApi/Services/DeviceSyncLogService.cs`):
  `_context.DeviceSyncLogs.FirstOrDefault(x => x.SyncId == deviceSyncLog.SyncId)` — checked whether **SyncId is indexed**.
- "New tablets" in the issue notes means **newer tablet app versions that send `syncId`**, not brand-new physical tablets — existing tablets on new app versions also hit this path.

## Troubleshooting learned
- k6 error `tls: first record does not look like a TLS handshake` = k6 uses `https://` but server speaks plain HTTP (or wrong port). Fix `BASE_URL` scheme/port in `.env.local` (e.g. `http://localhost:5000` vs `https://localhost:5001`).

## Related
- [[2026-06-05 Load Testing Environment Basics]]
- [[2026-03-27 Deswik K92 Shift Integration API Design]]
