---
date: 2026-06-16
source: Claude Code (VS Code)
project: core-web
tags: [load-testing, playwright, oidc, keycloak, blazor, web-client]
---

# Playwright Web Client Load Testing (Core Web)

## Setup
Playwright suite hitting the Blazor web client (`https://localhost:5005`) with multiple concurrent browser workers, capturing HTTP 4xx/5xx, JS console errors, network failures, Blazor render exceptions, and p50/p95 latency per page. All scripts/config live in `C:\Users\CULI\OneDrive - Hexagon\Desktop\Load Testing\webclient` (token/env patterns borrowed from the `browser-polling` folder).

## Critical auth lesson
Bearer tokens only work when hitting the **backend API directly**. For the web client you must perform a **real OIDC browser login**: hit `https://localhost:5005/Sites/List` → redirected to `https://cloak.minnovare.com/realms/corelocal/...` → submit credentials → redirected back via `/signin-oidc`.

- **Credentials are per-realm**: `corelocal` (local dev) vs `corestaging` (staging/cloud). A staging login on corelocal fails with a silent stay-on-Keycloak-page → generic `page.waitForURL: Timeout`. Improved script to detect and report "Login failed — check credentials". Known-good local account: `njardine@minnovare.com`.
- After `npx playwright install` is required if browsers were never downloaded (`browserType.launch: Executable doesn't exist`).

## Two script versions (kept side by side)
| | v2.22 `load-test.js` | v2.23 `load-test-v223.js` |
|---|---|---|
| Config | `.env.local` | `.env.v223` |
| Pages | 5 separate URLs (dev/prod site split) | 1 site URL + dynamic tab clicks |
| Site config | `PRODUCTION_SITE_ID` + `DEVELOPMENT_SITE_ID` | single `SITE_ID` |
| Tab discovery | none | reads `.site-tabs button` dynamically |
| Results | `results/` | `results-v223/` |

Reason: **core refactor 2.23 removes the Development/Production site split** — site landing page loses those links; SiteDetails gets a mining-method tab in the heading. 2.23 is local-only for a while; 2.22 still needs cloud testing, so scripts must not share config.

## Repo hygiene
`.claude/` (incl. `settings.local.json`, `sessions/`) added to `.gitignore` — Claude Code local settings must never be committed; load-testing files also stay out of the repo.

## Related
- [[2026-06-05 k6 Load Testing Setup for Core Sync]]
