---
date: 2026-07-20
updated: 2026-07-20
source: Claude Code
project: both
tags: [domain-knowledge, site-features, drilling-methods, sync, mc-1851]
---

# Site Drilling Method Enablement (Production/Development SiteFeatures) — what the feature actually does

> **Takeaway:** A site's "drilling methods" today are just `SiteFeatures` rows with an `Enabled` bit — and **Edit Site can already disable them, with no confirmation and no minimum-one validation**. Disabling deletes nothing, but it hides the whole product portal in CORE Web (historical data becomes unreachable in the UI) and drops the site from the tablet app's site list (sync stops). So MC-1851's PRD line "historical operational data must remain available within CORE Web" is **NOT current behaviour** — today it becomes invisible; keeping it visible is a new requirement, not keep-same.

## The business/physical reality being modelled
A mine site does certain *types* of drilling: Production (blast rings) and/or Development (tunnel headings); Cut & Fill is being added by PRD#19. In code this is not a Site→DM→Rig hierarchy: the Site has a set of enabled method flags, and each Rig carries exactly one `MiningMethod` value (`Rig.cs:132`; enum `DefaultProduction/DefaultDevelopment/CutAndFill` in `Minnovare.Core.Shared/Models/MiningMethod.cs`). The two tablet apps (PRODOP / DEVOP) each serve one method.

## The configuration chain
- **Storage:** one `SiteFeatures` row per feature with an `Enabled` bool. `Site.HasFeature(f)` = `SiteFeatures.Any(x => x.Enabled && x.Feature == f)` — `Minnovare.Core.Shared/Models/Site.cs:291-293`.
- **Create Site (web):** `SetSiteFeaturesCreate` — `core-web SiteCreateEdit.razor.cs:353-368`. Quirk: the sub-feature rows (ChargePlan, SurveyModule, Teleremote, MWD) are only created **inside the `if (_productionEnabled)` block** — a Development-only site gets no rows for them at all.
- **Edit Site (web):** `SetSiteFeaturesEdit` — `SiteCreateEdit.razor.cs:370-388`. Production/Development are plain Yes/No dropdowns (`SiteCreateEdit.razor:90-94`), editable in Edit mode; switching to No flips `Enabled = false`. **No "are you sure" prompt, no at-least-one check — you can disable both today.**
- **API save:** PUT in `core-web-api SitesController.cs:277-298` only maps the production field block (tolerances, ream sizes…) if `HasFeature(Production)`, and the dev block (drilling tolerance, grade format) if `HasFeature(Development)`.

## Where it takes effect (what disabling actually does)
1. **CORE Web portal entry blocked** — `ProductSelector.razor.cs:125-151`: `GoToProduction/GoToDevelopment` only navigate to `/{SiteId}` or `/{SiteId}/Development` when the feature is enabled; otherwise they show the "add this product" promo panel. Net effect: **historical data for a disabled method is still in the DB but unreachable through the UI.**
2. **Edit Site tabs hidden** — Production/Development settings tabs use `Hidden="!_productionEnabled"` (`SiteCreateEdit.razor:204`), so the method's config becomes uneditable.
3. **Tablet sync stops** — the app's site list `GET /Sites?siteType=…` filters with `HasFeature` per `SiteType` (`SitesController.cs:62-93`; default `siteType=Production`). A site with the method disabled disappears from that app's list, so no further sync for that method. (`GET /Sites/{id}` itself is not feature-gated — `SitesController.cs:95-97` — the gate is the list.)
4. **Nothing is deleted** — disabling only flips the `Enabled` bit. Data removal happens only via the separate purger lambdas (LambdaPurger, MwdDataPurger), which run on their own schedules and do not look at feature flags. There is **no retention period tied to enable/disable**.

## How it feeds MC-1851 (D2.1)
- "No more updates are CORE Synced for this DM" → **already current behaviour** (effect 3).
- "Historical data still viewable in CORE — for how long?" → duration is a non-issue (nothing is ever deleted by disabling), **but visibility is the real gap**: today disabling hides the portal entrance (effect 1), so the PRD's "must remain available within CORE Web" requires *new* behaviour — keep the method's pages readable (e.g. read-only) after disable.
- Dale's "keep same functionality" and the PRD sentence therefore conflict on exactly this one point — worth an explicit decision.

## Design subtleties & edge cases
- No minimum-one-method validation exists today; MC-1851 B4.1 (must select ≥1 to save) is new.
- Development-only sites have no ChargePlan/SurveyModule/Teleremote/MWD `SiteFeatures` rows (create-path quirk above) — any migration to a new Site↔MiningMethod model must handle absent-row vs Enabled=false the same way `HasFeature` does (both count as off).
- Cut & Fill exists in the `MiningMethod` enum but has **no `FeatureType`** — site-level enablement model must be extended (new FeatureType vs Site↔MiningMethod link + migration of existing Production/Development rows, per B5.1).

## Questions asked
- 2026-07-20 — "About MC-1851 D2.1, what is the current behavior when a drilling method is disabled?" → Disable already exists in Edit Site, no confirmation; sync stops via site-list filtering; data hidden in UI but never deleted; see *Where it takes effect*.

## Related
- [[2026-07-07 PRD Front End 15-22 Analysis and Executable Plan]] — parent PRD analysis (per-method config decisions)
- [[2026-05-25 Sites API siteType Default and Drillers 403 Analysis]] — the `siteType=Production` default on GET /Sites
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]] — one-site-page + method tabs direction this feeds into
