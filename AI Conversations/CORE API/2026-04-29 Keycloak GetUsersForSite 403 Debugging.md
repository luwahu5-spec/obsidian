---
date: 2026-04-29
source: Codex (VS Code)
project: core-web-api
tags: [keycloak, 403, users-controller, kc-client, aws-secrets, debugging]
---

# GetUsersForSite failing in cloud — Keycloak 403 Debugging

## Symptom
`UsersController.GetUsersForSite` worked locally but failed **immediately** for every site in the cloud env (`api.core.dev.minnovare.com` → 500). Root chain: `KcClient.GetKcUsersForSite` (KcClient.cs ~line 190) got **`403 Admin forbidden`** from Keycloak.

## Key diagnosis facts
- All environments use the **same Keycloak host** (`https://cloak.minnovare.com`) but **different realms**: `corestaging` worked, `coretest` returned 403 on `GET /admin/realms/coretest/users?q=site_id:114`.
- Same admin user + same admin-cli token style → difference is **coretest realm admin permission config**. Check master realm → Users → admin → Role mappings, compare `coretest-realm` vs `corestaging-realm` roles: `query-users`, `view-users`, `manage-users`.
- Keycloak version was **20.1.0** (ruled out a Keycloak 26 regression). Check version via `GET /admin/serverinfo` with admin token (`$serverInfo.systemInfo.version`) or Admin Console → Server Info.

## Recovered detail — the null-safety analysis that preceded the real find
- The truly dangerous line was the **nested indexer chain**: `kcUsers[userMappings[new Guid(minnovareUser.Id)]].Attributes.TryGetValue(...)` — `TryGetValue` only protects the *last* dictionary; the two indexers before it throw `KeyNotFoundException` if a mapping is stale. Non-empty collections don't prove every lookup inside a loop is valid.
- `kcUser.Attributes` is **not a SQL column** — it's Keycloak user attributes from the Admin API (`KcUser` model).
- Diagnostic pivot that cracked the case: *"fails for **every** site, cloud-only, almost immediately"* → that shape means config/auth/network (the `Login()` path), not bad data for one site. `KcClient.Login()` uses AWS Secrets Manager when available, builds the token URL against `realms/master`, and **throws immediately on any non-200**.
- The browser console error is just the client wrapper around the server's generic 500 page — the real exception only exists in the API server log at that timestamp.

## Things learned along the way
- **`KeyCloak:AdminId` is NOT the admin user id** — it's the Keycloak **realm role id** for the role named `Administrator`, used in `KcClient.cs` (~line 337) when mapping the Administrator role to a user.
- `D:\TeamCity\buildAgent\work\...` paths in cloud stack traces are the **build machine's** source paths baked into PDBs — irrelevant to the runtime server.
- Local dev **does reach AWS Secrets Manager**: `Environments.Secrets.KcAdminCredentials(_environment.EnvironmentName)` fetches the KC admin credentials secret even locally.
- Null-safety review of `GetUsersForSite`: `ToDictionary` on empty result is fine (empty dict); `_userManager.Users.Where(...).ToList()` with empty id list returns **empty list, not null**; `MinnovareUser` maps to the AspNet users table.

## Related
- [[2026-05-04 Keycloak SSO and Audience Mapper]]
