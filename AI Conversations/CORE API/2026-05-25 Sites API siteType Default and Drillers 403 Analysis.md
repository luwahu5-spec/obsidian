---
date: 2026-05-25
source: Codex (VS Code)
project: core-web-api
tags: [debugging, sites-controller, site-authorization, 403, api-defaults]
---

# `/sites/` siteType default + when `/Sites/{id}/Drillers` returns 403

## The `/sites/` default that keeps biting
`SitesController.Get` (~64):
```csharp
public IEnumerable<Site> Get(
    [FromQuery] ShowArchiveStatus showArchiveStatus = ShowArchiveStatus.ShowActiveOnly,
    [FromQuery] SiteType siteType = SiteType.Production)   // ← default!
```
`siteType` switch: `Production` → only sites with `FeatureType.Production`; `Development` → only Development feature; `All` → no feature filter. So a bare `/sites/` call **silently drops Development-only sites** — the root cause behind "site shows in Sites/List but not in SystemStatusDashboard popup" (see [[2026-05-19 System Status Dashboard Missing Sites Debugging]]).

## 403 analysis for `/sites/{siteId}/drillers`
`SiteAuthorizationHandler` (`Minnovare.Core.Shared/Security/SiteAuthorizationHandler.cs`):
```csharp
if (context.User.IsInRole(Roles.Administrator)) { context.Succeed(requirement); }
```
- **Administrator role bypasses site assignment entirely** — no `site_id`/`drill_plan_site_id` claim needed; inactive site doesn't matter for this check.
- Non-admin: 403 when the token's site claims don't include the requested site (e.g. access to site 10, requesting site 20).
- An "administrator token" can still 403 if Core doesn't actually see the `Administrator` role in the token (check realm_access roles claim).
