---
date: 2026-06-12
source: Claude Code (VS Code)
project: core-web
tags: [blazor, error-ui, circuit, concepts, debugging]
---

# When does the "Error occurred. Reload page" banner appear?

## Two different banners in `_Host.cshtml`
- `components-reconnect-modal` (~line 40) — "Could not reconnect to server" → SignalR/circuit connectivity loss.
- `blazor-error-ui` (~line 43) — "Error occurred. Reload page…" → shown by `blazor.server.js` when an **unhandled exception crashes server-side component code**. Backend can be perfectly fine; this is a frontend-component crash.

## Common crash circumstances in this codebase
1. **Null reference before data loads** — razor touches `_shift.Name` before `OnInitializedAsync` completes; guard with `@if (_shift != null)`.
2. **Unhandled exception in `OnInitializedAsync`** — repository throws (not just returns null).
3. **Unhandled exception in event handlers** — wrap and surface via `NotifierService` toast.
4. **Token expiry mid-session** — 401 bubbling up as exception instead of clean redirect (handle in `ApiService`).
5. **`IJSRuntime` interop failure** — wrap interop calls in try/catch.

All are resolvable — the fix family is: null guards, try/catch + error flag/toast, clean 401 handling.

## Bonus layout fact
Page width differences between similar pages: `<div class="container">` (fixed ~1320px max) vs `container-fluid`; ShiftList "fixed" it with an inline `.container { max-width: 100%; }` style hack — `container-fluid` is the clean version.
