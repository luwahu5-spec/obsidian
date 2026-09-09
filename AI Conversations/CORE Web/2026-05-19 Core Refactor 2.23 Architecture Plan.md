---
date: 2026-05-19
source: Codex (VS Code)
project: core-web
tags: [planning, architecture, core-refactor-2.23, drilling-methods, site-details, adapter-pattern]
---

# Core Refactor 2.23 — removing the Production/Development split (architecture plan)

## The problem
The site page was "two pages in disguise": routed by `/{SiteId}/{ProductCategory}`, then branching everywhere on `ProductCategoryType == Production/Development` — SiteDetails.razor (~74, ~190), `UploadFiles()` in SiteDetails.razor.cs (~173), two hardcoded plan-summary components, shift pages routed by `RigSiteType`. **Every new drilling method (e.g. Cut and Fill) would multiply the branches.**

## Target design — one site workspace with drilling-method tabs
Site Details becomes **one page with dynamic method tabs** (`[Production] [Development] [Cut and Fill]`); the selected method drives rigs, plan summary, upload, and method-specific actions. Repo already has a reusable `TabControl`/`TabPage`.

## Key architectural decisions

### 1. Config-driven upload module (backend has no provider yet)
Build the frontend **as if the backend will eventually own upload configs**: one central upload module; drilling-method knowledge isolated in one mapping/config provider; pages only render configs.
- `SiteUploadConfig` = the whole upload card (title, subheading, accepted file types, single/multiple, upload function). One class is enough at current scope — no `UploadFieldConfig` sub-model needed yet.
- Became `SiteUploadConfig.cs` / `SiteUploadConfigProvider.cs` / `SiteUploadCard.razor` (MC-1822).

### 2. Frontend adapter layer for drilling methods (`Shared/Drilling`)
Don't depend on the backend's final drilling-behavior metadata shape (unknown at the time). Build an interpretation layer:
```
Backend model (unknown/future) → IDrillingMethodAdapter (translator) → stable frontend model → pages
```
- Temporary source: today's `SiteFeatures` + `FeatureType.Production/Development` (rig method inferred from `RigTypeClassification`); backend metadata later replaces adapter internals **without touching consuming pages**.
- Files: `DrillingMethodAdapter`, `DrillingMethodDefinition`, `DrillingMethodKeys`, `IDrillingMethodAdapter`, `SiteDrillingMethod` — registered in DI (landed on branch MC-1773).

### 3. Frontend must not know method names (final principle)
Frontend renders **whatever the backend returns** — never `if (method == Production)`. Generic shape:
```csharp
public class SiteDrillingMethod { string Key; string Label; bool Enabled; int? DisplayOrder; }
```
Render rule: `Where(Enabled).OrderBy(DisplayOrder ?? int.MaxValue).ThenBy(Label)`. DisplayOrder uses gapped values (10/20/30) so items can be inserted later; it's UI ordering metadata, not business logic. **The adapter is a translator, not a method registry.**

### 4. Entity relationship
Keep `Site` as the parent/context object (name, GUID, Rigs, Drives, Drillers, reason lists all stay); drilling methods become a capability/config **collection inside Site** (`Site → SiteDrillingMethods`).

### 5. Plan summary + shift pages
- Plan summary: production component (6 fixed columns/filters, `DriveRepository.GetDrivesForSite`) and development component (3 columns, `DevelopmentRepository.GetDrivesForSite`) duplicate sort/search/archive/refresh logic → unify into one config-driven table.
- Shift pages are more complex (paging, filters, CSV export, totals) — keep a **fixed shell**, make only the genuinely method-specific parts (final drilling-details section) configurable.

## Ripple effects recorded elsewhere
- Load-testing scripts needed a v2.23 variant (tab clicks instead of dev/prod URLs): [[2026-06-16 Playwright Web Client Load Testing]]
- Backend side: `MiningMethod` enum + `UserSiteMiningMethodScope` migration: [[2026-06-24 MiningMethod Merge Missing Migration]]
- Implementation sequencing (Users first): [[2026-05-19 Site Feature Implementation Order Plan]]
- Stub `StubUserMiningMethodScopeService` (hardcoded, current dev) and the real DB-backed `UserMiningMethodScopeService` are deliberately kept side by side until the backend scope service is complete
