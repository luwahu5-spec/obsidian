---
date: 2026-09-03
updated: 2026-09-03
source: Claude Code
project: core-web-api
tags: [domain-knowledge, sync, tablet-app, sites-controller, shifts-controller, device-sync-log]
---

# Sync after Login — what each API endpoint actually returns

> **Takeaway:** When a driller picks a rig on the tablet, the app pulls the *whole operating context* for that rig in one burst: identity (site + rigs + drillers), the pick-lists that make the shift-entry screens work (pause reasons, consumable reasons, service tags/types), the pre-start form definition, the shift skeleton, and the drives assigned to the rig. Two of these calls are **not** read-only — `GenerateUpcomingShifts` is a POST that *creates* Shift rows, and `GetAssignedDrivesForRig` bumps the rig's status summary. Every call in the burst also writes a `DeviceSyncLog` row keyed by the `XDeviceID` header and the `syncId` query param, which is how one login sync is stitched back together in the DB.

## The physical reality being modelled
A tablet on a jumbo underground has no reliable network. At login it grabs everything it needs to run a full shift offline: who I am, which rig, which site rules apply, what drop-down options the operator may pick from, what pre-start checklist to show, which shifts exist, and which drives this rig is allowed to work on. Everything afterwards is offline entry plus later upload.

## The endpoints, in the order the diagram fires them

### 1. Upload local driller — `POST /drillers` OR `PUT /drillers/{drillerId}`
`DrillersController.PostDriller` (`Minnovare.Core.WebApi/Controllers/DrillersController.cs:167`) / `PutDriller` (:248).
Not a read. The tablet pushes a driller created offline. Server resolves `driller.Site` from `SiteId` before validating (:169, :251) so the missing `Site` navigation does not fail ModelState, then site-access-checks with `SiteAccessType.DrillPlanOnly` (POST) / full access (PUT).
- **POST returns 201 plus the persisted `Driller`** (with its server-assigned `Id`) — that is what the tablet needs to map its local record.
- **PUT returns 204 No Content**, and 400 if `{id}` does not equal `driller.Id`.

### 2. `GET /sites/{siteId}?siteType=Development`
`SitesController.GetAsync` (`SitesController.cs:97`). Returns one **`Site`**:
- drilling tolerances and geometry rules: `DipTolerance`, `DumpTolerance`, `DrilingTolerance`, `GridOffset`, `PositiveDirectionDip/Dump`, `ZeroOffsetDip/Dump`, `DipLimit`, `DumpLimit`, `SingleSided`, `DownholeInclination` (`Models/Site.cs:53-176`)
- display/units: `MeasureSystem`, `GradeFormat`, `OverrideStrings`, `DevOpOverrideStrings`, lat/long, timezone
- `SiteFeatures` — the feature flags, loaded explicitly at :107
- `ShiftConfigurations` — shift names/numbers/start-end times; **falls back to `Shifts.GetDefault()` when the site has none** (:109-112)
- `SitePressureUnitConfiguration` — only if an MWD pressure-unit FeatureConfiguration row exists (:137, `GetSitePressureUnitConfig` :153)
- `SiteConfig` — every `FeatureConfiguration` subclass whose `TargetObject == typeof(Site)`, resolved one query per config type (`Minnovare.Core.Database/Extensions.cs:57`)

**Gotcha: this endpoint has no `siteType` parameter.** The signature is `GetAsync(long id, Guid? syncId)` (:97) — `?siteType=Development` in the diagram is silently ignored by model binding. It only matters on `/sites/` (list), `/rigs` and `/drillers`. See [[2026-05-25 Sites API siteType Default and Drillers 403 Analysis]].

Also supports conditional GET: emits `Last-Modified` from `site.Modified` and returns **304 Not Modified** if the tablet's `If-Modified-Since` is current (:120-133).

### 3. `GET /sites/{siteId}/drillers?showDriller=2&siteType=Development`
`SitesController.Drillers` (`SitesController.cs:431`). Returns `List<Driller>` for the site: `GivenName`, `FamilyName`, `Archived`, `DrillerType`, `LastTrainedDate` (`Models/Driller.cs:14-65`).
- `showDriller=2` is `ShowArchiveStatus.ShowAll` (`Shared/Definitions/Globals.cs:29-37`), so **archived drillers are included**. Deliberate: an archived driller must still resolve on historical shift data held on the tablet.
- `siteType=Development` filters on `x.DrillerType == siteType` (:452) — a hard equality, so a driller registered as Production never appears in a DevOp sync.

### 4. `GET /sites/{siteId}/rigs?showArchivedRigs=2&siteType=Development`
`SitesController.Rigs` (`SitesController.cs:348`). Returns `List<Rig>`: `Name`, `RigType`, `OffsetType`, `ReamSizes`, `BlendedDump`, `MiningMethod`, `Archived` (`Models/Rig.cs:45-132`).
- `showArchivedRigs=2` is ShowAll, so archived rigs are included (:364).
- `siteType` maps through `RigTypeClassification.DevelopmentRigs` / `ProductionRigs` (:370-375) — Development returns the jumbos only.
- Per rig, **conditionally enriched by site feature flag** (:379-390): `RigIpConfiguration` only when the site has `FeatureType.Teleremote`; `RigReamConfiguration` only when it has `FeatureType.MWD`. A missing IP config on the tablet is usually a missing feature flag, not missing data.

### 5. `GET /ConsumableTypeReasons/{siteId}/GetBySite`
`ConsumableTypeReasonsController.GetConsumableReasonsBySite` (:41). Returns `List<ConsumableTypeReason>` — each is a **type-to-reason pairing** (`ConsumableType` plus `ConsumableReason`, both eagerly included, `Models/ConsumableTypeReason.cs:7-11`), sourced from `SiteConsumableTypeReasons` where `!Archived` (:52). This drives the cascading "consumable then why it was consumed" drop-down (e.g. Drill Bit then Worn / Broken).

### 6. `GET /ShiftPauseReasons/{siteId}/GetBySite`
`ShiftPauseReasonsController.GetShiftPauseReasonsBySite` (:47), delegating to `_unitOfWork.ShiftPauseReasonRepository.GetForSiteAsync(id)` (:58). Returns `List<ShiftPauseReason>` — id plus `Name` only. The pick-list for "why did drilling stop" (Ground Support, Meal Break, Services).

### 7. `GET /DynamicForms/GetByRig/{rigId}`
`DynamicFormsController.GetByRig` (:129). Returns **one `DynamicForm`** — the pre-start checklist definition, fully expanded: `Title`, `ItemGroups` then `Items` then `ItemTemplate` then `Validation` then `Validations` (:148-151, `Models/DynamicUi/DynamicForm.cs:62-87`), each item carrying `ItemType`, `Options`, `DefaultValue`, and each group carrying `AllowBypass` / `BypassMessage`.
- Resolution: `f.Active && (f.Rig == rig || f.Site == rig.Site)` (:151) — a **rig-specific form wins, otherwise the site form**; `SingleOrDefaultAsync` means having both an active rig form and an active site form throws.
- 404 when no active form exists — normal for sites that do not run pre-starts.
- See [[2026-06-19 PreStartData Payload and Dynamic Form IDs]] for how the answers come back.

### 8. `GET /ServiceTags/GetBySite/{siteId}` and `GET /ServiceTypes/GetBySite/{siteId}`
`ServiceTagsController.GetBySite` (:41) and `ServiceTypesController.GetBySite` (:40). Both return a flat list of `{ Id, Name }` from `SiteServiceTag` / `SiteServiceType` where `!Archived`. Pick-lists for rig servicing records. Note the route shape differs from the two above — `GetBySite/{id}` here, `{id}/GetBySite` there.

### 9. `POST /Shifts/{rigId}/{endDate}/GenerateUpcomingShifts`
`ShiftsController.GenerateUpcomingShifts` (`ShiftsController.cs:2036`, route attribute :2030 — **`[HttpPost]`, not GET**). Returns `List<Shift>` = newly created plus already-existing generated shifts, ordered by `StartTime`.
**This endpoint writes.** It materialises the shift skeleton so the tablet can attach work to a shift offline:
- shift templates from `ShiftConfigurations` for the site, falling back to `Shifts.GetDefault(site)` (:2067-2071)
- `NextShiftOffset` from the `DEFAULTNEXTSHIFTOFFSET` global FeatureConfiguration, else `Shifts.NextShiftOffset` (:2063-2064)
- loops **from yesterday** (`DateTime.Today.AddDays(-1)`, :2074) to `endDate`, building `Shift { StartTime, EndTime, Rig, Name, Number, IsGenerated = true, IsSynced = false }` with the **site's own UTC offset** per day (`rig.Site.TimeZoneInstance.GetUtcOffset(day)`, :2078) — not the server's, not the tablet's
- de-dupes against existing generated shifts in a plus/minus one day window (:2099-2100), then persists via `AddMultipleShifts` (:2103)
- 400 if `endDate <= today` (:2058); the `while (day != endDate)` loop is an exact date equality test, so a non-midnight `endDate` would not terminate cleanly.

### 10. `GET /Development/GetAssignedDrivesForRig/{rigId}`
`DevelopmentController.GetAssignedDrivesForRig` (:679). Returns `List<DevelopmentDrive>` — `Id`, `DriveName`, `Site` (`Models/DevelopmentDrive.cs:5-11`) — from `DevelopmentDriveRigAssignments` for that rig where the drive is **not archived** (:700). This is the list of headings the jumbo may drill against.
**Side effect:** `_unitOfWork.RigStatusSummaryRepository.UpdateDetails(rig, DateTimeOffset.UtcNow)` (:696) — this call doubles as the rig's "last seen" heartbeat. Notably it does **no** `VerifyAccess` site check, unlike its `AssignDrivesToRig` sibling (:715).

## The thread that ties the burst together — DeviceSyncLog
Every endpoint above calls `_deviceSyncLogService.SetDeviceSyncLog(...)` with `DeviceId = Request.Headers["XDeviceID"]`, `UserAgent`, the `Rig` where known, and `SyncId = syncId.GetValueOrDefault()` from the `?syncId=` query param. One login sync equals one `syncId` GUID across roughly ten rows. That is the handle for "show me everything device X pulled during that sync".

## Design subtleties and edge cases
- **`showDriller=2` / `showArchivedRigs=2` mean ShowAll, not "archived only"** — 1 is `ShowArchivedOnly`. An off-by-one here silently gives the tablet only archived records.
- **`siteType` on `/sites/{id}` is dead** — bound to nothing; only the list/rigs/drillers routes honour it.
- **Two writes hide inside a "sync/read" flow** — `GenerateUpcomingShifts` (POST, creates rows) and `GetAssignedDrivesForRig` (updates RigStatusSummary). A retry loop on either is not free.
- **Shift times are built in site timezone, per day**, so DST transitions produce correctly offset shifts; generating on the server clock would be wrong.
- **Feature flags gate payload fields, not endpoints** — Teleremote/MWD absence makes rig config fields null rather than erroring.
- **DynamicForm is `SingleOrDefault`** — two active forms (one rig-level, one site-level, both `Active`) throws rather than preferring the rig one.

## Questions asked
- 2026-09-03 — "Please tell me what information comes back from these endpoints" (Code flow: Sync after Login diagram) — full endpoint-by-endpoint breakdown above; key surprises: `siteType` ignored on `/sites/{id}`, `showX=2` means ShowAll, and two of the calls mutate state.

## Related
- [[2026-05-25 Sites API siteType Default and Drillers 403 Analysis]]
- [[2026-06-19 PreStartData Payload and Dynamic Form IDs]]
- [[2026-05-05 How Authorization Flows from Core Web to Core API]]
