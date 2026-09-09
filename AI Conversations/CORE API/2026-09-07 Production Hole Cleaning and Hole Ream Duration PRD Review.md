---
title: Production Hole Cleaning and Hole Ream Duration PRD Review
date: 2026-09-07
tags: [core-api, core-web, prd, production-optimiser, hole-cleaning, drilling-duration, synchronization]
status: engineering-review-decisions-required
repositories: [core-web, core-web-api]
reviewed_branch: MC-1842
---

# Production Hole Cleaning and Hole/Ream Duration PRD Review

Both PRDs are feasible. Hole Cleaning introduces a new activity and retained Service records; Hole and Ream Duration exposes elapsed timing on existing drilling records. The critical engineering work is preserving timestamp meaning and records across synchronization, rather than adding display columns alone.

This note consolidates the PRD explanations and source reviews from this conversation. Findings describe the local `MC-1842` checkouts reviewed during the session, not verified production deployments. No application code was changed and no builds, tests, database queries, or device tests were run. Production Optimizer app source was unavailable, so app state-machine and RD-Link behavior still require verification. Screenshots referenced in the pasted PRDs were not supplied.

## 1. What the two PRDs do

| Area | Hole Cleaning | Hole and Ream Duration |
|---|---|---|
| Purpose | Record cleaning against a completed production hole | Show elapsed time for each completed hole or ream pass |
| Operator workflow | New Hole Clean tab using alignment, Collar, depth entry, completion | Existing Collar and completion actions |
| Timing | Cleaning Collar to cleaning completion | Latest pass-specific Collar to applicable completion |
| History | Latest activity in tab; every completed clean retained as a separate Service | Latest timestamps overwrite earlier values on the same record |
| Reporting | Services rows, Shift History totals, Digi-PLOD export | Hole/ream tabs and Shift Drilling Details CSV |
| Services created | One per completed clean | None |
| Incomplete activity | No Service created | No completed drilling record reported |

### Hole Cleaning, in plain language

The operator selects an already completed hole, opens HOLE CLEAN, realigns, presses COLLAR, enters cleaned depth and comments, and completes the clean. Each completion creates an individual Hole Cleaning Service with source Drive/Ring/Hole, depth, start, end, and comments. Repeated cleans must create additional Service lines even though the tab retains only the latest activity. Blue status means cleaning has previously been completed and remains visible during a later incomplete clean.

The Service is attributed to the shift and driller at completion, even when Collar occurred in an earlier shift. The PRD requests aggregation into existing Shift History service fields and individual Digi-PLOD Services rows, without duplicate manual entry.

### Hole and Ream Duration, in plain language

If the operator presses COLLAR at 10:00 and HOLE COMPLETE at 11:24, CORE shows Time Collared 10:00, Duration 1h 24m, and Time Completed 11:24. CSV duration is 1.4 hours. A 30-minute break during that interval remains included: this is elapsed time, not active drilling time.

Each ream pass uses its own timestamps. Non-final reams use REAM COMPLETE; final reams use HOLE COMPLETE according to the PRD. The app must verify which record each action actually updates.

A hole collared in Shift A and completed in Shift B remains reported in Shift B, with duration spanning both. Repeating Collar and completion replaces the old timestamps. Missing Collar means blank Collar and Duration, with no estimation.

Required output is on the applicable Hole page tab and in the Shift Drilling Details CSV. The CSV replaces `Date` with `Time Collared, Duration, Time Completed` after Driller. An on-screen shift drilling Duration column is optional. Main Shift History changes are out of scope for this PRD.

## 2. Hole Cleaning: confirmed concerns

### Cross-shift Services are currently rejected

`core-web-api/Minnovare.Core.WebApi/Controllers/ShiftsController.cs:1842` validates generated-shift child times. A Service starting before its parent shift produces HTTP 422. This directly conflicts with the permitted earlier-shift Collar and completion-shift ownership.

Retaining the PRD's full-duration completion-shift rule requires an explicit API exception/design and review of downstream time calculations. Merely relaxing validation is insufficient: service time could exceed the completion shift's length. Shift-edit logic also clamps Service timestamps to driller-shift boundaries (`ShiftsController.cs:2174`), potentially changing recorded activity times.

### Uploaded Service collections can remove retained records

`ShiftsController.cs:1894` reconciles incoming Services, and `RemoveUnlinkedItems` at line 1921 physically removes non-archived rows omitted from the corresponding uploaded collection. An older or stale client uploading the same applicable shift/driller-shift collection could omit and remove generated records. This is a conditional sync risk, not proof that every older-client upload deletes them.

The requirement that older versions preserve Hole Clean records needs explicit merge/ownership rules. Whole-property updates also require care so omitted fields do not clear new data.

### Exactly one Service per completion needs durable identity

Existing Service GUID reuse supports updating the same record on retry (`ShiftsController.cs:1941`). A durable completion identity, potentially the Service GUID itself if guaranteed stable, must survive local completion, crashes, and retry. Creating a fresh GUID on retry can produce a duplicate. Each genuinely new clean needs a distinct identity.

The app should persist completion and its generated Service atomically or use a recoverable pending-sync mechanism. Tab overwrite must never discard earlier pending Services.

### Totals do not always equal the sum of row durations

`Minnovare.Core.Services/ShiftSummaryService.cs:123` sums Service depths and lists service types. `Minnovare.Core.Shared/Minnovare.Core.Shared/Models/Shift.cs:102` merges overlapping Service intervals before totaling time. Consequently, overlapping rows can have a summed duration larger than Services Time.

The PRD must distinguish summed activity hours from non-overlapping rig service time and preserve contributions from existing Service types. It should not silently redefine existing totals as Hole Cleaning-only totals.

### Existing data fields help, but global availability needs design

`ShiftService.cs:10` already contains timestamps, depth, comments, nullable Drive/Ring/Hole IDs, and display names. Migrations exist for these location fields; deployed schema state was not checked. Sync resolves names from IDs at `ShiftsController.cs:1757`. That resolution alone does not prove source hierarchy/site integrity or define historical naming behavior.

`ServiceTypesController.cs:38` returns site-associated Service types. No literal Hole Cleaning definition was found in source; that does not establish whether it exists in live data. Global feature availability needs a stable Service-type mapping and a decision about site associations, rename/archive behavior, and manual Hole Cleaning availability.

### Reporting and edits

Web's Services table (`core-web/Minnovare.Core.Web/Pages/Blazor/Rigs/ShiftDetails.razor:267`) requires Duration, Drive, Ring, and Hole. API CSV generation at `ShiftsController.cs:1173` needs matching fields. Shift History's table provider and legacy export are separate paths and must reconcile.

The reviewed Web UI displays Service rows read-only; existing shift-level editing is not proof of an existing individual Service edit/validation workflow. Clarify whether generated Services can be edited/deleted, what happens to source activity values, and whether original observations remain auditable.

## 3. Hole/Ream Duration: confirmed concerns

### Reuse existing timestamps, with a clear calculation rule

Both `Hole.cs:105` and `Ream.cs:31` already contain nullable `DateCollared` and `DateDrilled`. No persisted Duration field was found. Prefer deriving Duration from these instants so timestamp changes cannot leave a stale third value. Local app calculation can still satisfy offline behavior without making a stored duration authoritative.

Specify handling for missing values, completion before Collar, device-clock changes, very long intervals, rounding, and decimal export culture. Do not invent timestamps or silently turn invalid intervals into zero.

### Latest-value overwrite is inconsistent across paths

- `RigsController.cs:743`: POST DrillPlan skips updates when the existing hole is Done and has a Collar timestamp; the nested ream update is inside this guard.
- `RigsController.cs:1385`: PUT DrillerDrillPlan accepts relevant state transitions or a strictly newer incoming Modified timestamp.
- `RigsController.cs:1469`: ream updates similarly depend on state/Modified.
- `Hole.cs:333`: its shared Update method omits DateCollared, while `Ream.cs:90` includes it. A separate DrivesController path uses Hole.Update; this is not the same as the PUT sync path, which uses CurrentValues.SetValues.

Map supported clients to the actual routes before deciding which guards to change. Every repeated Collar/completion must advance the appropriate record's Modified value. Preserve timestamp pairs from the same activity when handling delayed or older payloads.

### A completion timestamp may currently be a fallback

For an existing hole marked Done without DateDrilled, `RigsController.cs:1401` substitutes DateCollared or Modified. Therefore, a populated DateDrilled is not universally proof of the exact completion-button time. Calculating Duration blindly can show an artificial zero or inferred elapsed interval.

New app versions must send the actual completion time. Define how legacy/fallback values qualify for duration reporting without removing existing compatibility behavior indiscriminately. Historical backfill being out of scope does not resolve the trustworthiness of already populated values.

### Completion-shift attribution already fits

`HolesController.cs:144` selects by DateDrilled and rig; holes additionally require Done. Collar need not fall in the shift. The Service cross-shift rejection from Hole Cleaning does not apply to this drilling report selection.

Current predicates include both shift endpoints. An exact boundary timestamp can qualify for adjacent shifts with shared boundaries; this is a pre-existing edge case to test, not a reason to silently change attribution under this PRD.

### UI and timezone work must be explicit

`HoleEdit.razor:416` already displays hole/ream dates, but uses date-only format `d`, shows completion before Collar, and has no Duration. Add full date/time, labels, ordering, and localization. Preserve existing incomplete-record visibility unless product explicitly wants it removed; excluding new incomplete-duration reporting does not necessarily require hiding an existing Collar date.

The production CSV uses requesting-user timezone (`HolesController.cs:153`); the Hole page also uses user conversion, while the newer shift table receives site timezone. Agree which policy applies to each surface. Changing everything to site timezone would be a product change, not automatically required by this PRD.

`UserTimeService.cs:34` converts DateTimeOffset.DateTime as UTC. That assumes a zero offset; non-zero incoming offsets need verification. Compute elapsed time using DateTimeOffset instants, then format timestamps. Include DST and offset tests, and define rounding tolerance for UI/CSV reconciliation.

### Export changes affect another export

`Minnovare.Core.WebApi/Data/ShiftDetailReportHelper.cs:12` defines the Date column and hole/ream row layout. `DrivesController.cs:751` also uses this helper for Drive CSV. Changing the helper therefore has wider scope than Shift Drilling Details alone.

Choose a versioned export, a transition strategy, or a deliberately separate schema. Customer spreadsheet/import dependencies were not inventoried during this review. Keep underlying API property names stable unless a separate contract change is approved.

### App workflow remains unverified

Confirm final HOLE COMPLETE updates the correct final-Ream and parent-Hole fields, and that each reported pass has its own start/end pair. Verify Smart Collar/recalculated relationships, added holes, parallel/horizontal modes, and RD-Link in app source and on representative devices. Clarify reopening: starting a new Collar must not pair it with the previous completion while the new attempt is incomplete.

## 4. Current implementation boundaries

The visible shift drilling table now uses `ShiftReportDrillingDetailsCard` and the API metadata provider (`ShiftDetails.razor:377`). Legacy `_shiftHoles` loading remains for other page calculations; changing that list alone will not add a visible Duration column. If the optional column is included, change the production provider/profile/shared row contract.

CSV remains separate: Web's `ShiftRepository.GenerateReport` downloads API CSV and packages the ZIP. Update the API report helper and test it independently of the metadata table. Shared model changes must be coordinated across the repositories that consume them.

## 5. Decisions and acceptance checks before implementation

| Decision | Applies to |
|---|---|
| Preserve full activity duration on completion shift and define operational totals | Hole Cleaning |
| Stable completion identity and preservation of omitted records | Hole Cleaning |
| Manual Services, global Service mapping, edits/deletes, audit ownership | Hole Cleaning |
| Consistent overwrite behavior across supported sync routes | Both |
| Timestamp provenance, missing/invalid handling, offset policy and rounding | Both |
| Reopen/re-Collar and final-Ream timestamp pairing | Hole/Ream Duration |
| Versioned CSV versus in-place change, including Drive CSV impact | Hole/Ream Duration |
| Include optional on-screen shift Duration column | Hole/Ream Duration |

Acceptance tests should cover first completion, repeated completion, incomplete restart, delayed and repeated sync, crash recovery, older-client payloads, cross-shift and exact-boundary completion, missing/invalid timestamps, non-zero offsets, DST, and durations over 24 hours. For cleaning, additionally test overlapping Services, shift edits, pending repeated cleans, source-hole linkage and manual duplicates. Reconcile Web values with each applicable export.

No delivery estimate was established. App source review and these decisions are prerequisites for a reliable estimate.

## 6. Examples to use in product and engineering review

| Scenario                                                   | Expected outcome / decision needed                                                                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Hole collared 10:00, completed 11:24, including a break    | Duration 1h 24m; CSV 1.4 hours. No downtime subtraction.                                                                                   |
| Collar 17:30, completion 19:00 after shift change          | Drilling record appears in the completion shift with 1h 30m elapsed. Cleaning needs API changes to permit the equivalent Service.          |
| Same hole cleaned twice for 20 minutes each                | Two retained Service rows, 40 minutes summed activity duration; latest clean shown in tab.                                                 |
| Cleaning Service overlaps another Service for 10 minutes   | Summed row durations and merged service time differ by the overlap; product must define reconciliation.                                    |
| Completed hole re-collared but not completed again         | Define whether old completed values remain visible or the record becomes incomplete. Never subtract the new Collar from an old completion. |
| Service upload succeeds but response is lost               | Retry must reuse the durable identity and retain one row.                                                                                  |
| Older payload omits a previously uploaded cleaning Service | The generated Service must survive; current collection reconciliation needs adjustment.                                                    |
| Completed historical hole has no Collar                    | Show completion if available; Collar and Duration blank.                                                                                   |
| Completion earlier than Collar                             | Flag/leave duration unavailable according to agreed policy; do not display a misleading normal duration.                                   |
| Activity lasts 26h 10m                                     | Display total elapsed hours or an explicit day format; do not use TimeSpan.Hours alone, which omits whole days.                            |

## 7. Suggested wording to tighten the PRDs

The following wording is proposed for review, not an approved product decision.

**Timestamp authority and duration:** "The recorded Collar and completion instants are authoritative. CORE derives elapsed Duration from the applicable timestamp pair. Any locally calculated Duration must agree with that pair. Timezone conversion is applied for display only. The export column identifies the unit as Duration (hours), with an agreed precision and rounding rule."

**Valid pairs:** "Duration is available only for a completed activity with a valid Collar and completion pair belonging to that same attempt. Missing or invalid timestamps produce an unavailable Duration; values are not estimated. Legacy completion fallback handling is documented separately."

**Repeated drilling:** "A subsequent completed attempt replaces the previous timing for that hole or ream. The behavior while a subsequent attempt is incomplete must be defined, and a new Collar must never be paired with the previous attempt's completion."

**Repeated cleaning and retries:** "Every distinct completed clean has a durable identity. Retrying the same completion or synchronization reuses that identity and does not create an additional Service. Starting a distinct clean creates a distinct identity. Previous completed and pending-upload Service records survive reuse of the Hole Clean tab."

**Backward compatibility:** "A supported older client or stale upload must not remove generated Hole Cleaning Services merely because the records are absent from its payload. The API applies explicit ownership and merge rules while preserving intentional authorized edits/deletions."

**Cross-shift cleaning:** "The full recorded activity is assigned to the completion shift and completed-record driller, including an earlier Collar timestamp. The distinction between summed activity duration and operational shift service time is explicitly documented and tested." This preserves the current PRD direction; splitting time would require a product change.

**Export compatibility:** "Identify all consumers of the shared production CSV schema, including Drive export. Publish the approved schema transition/version and maintain field alignment between headers and rows for holes and reams."

## 8. Confidence and next engineering steps

Confirmed from reviewed source: timestamp fields exist; Service location fields exist; generated-shift Services are constrained to shift boundaries; absent Service collection items can be removed; service intervals are merged for totals; upload paths differ on completed-hole updates; the CSV helper is reused; Hole page timing is date-only; the visible shift drilling table is now metadata-driven.

Not established by this session: production schema/deployment status, actual older-client payload behavior, which routes every supported app version uses, exact final-Ream button effects, device timestamp normalization, actual customer CSV dependencies, and full RD-Link behavior. These must not be presented as verified app defects or deployment facts.

Recommended sequence:

1. Review app action handlers and local persistence for every supported pass type, especially final Ream, Smart Collar, and re-Collar.
2. Map supported app versions and rig workflows to POST/PUT sync contracts and identify what a downgrade/stale retry sends.
3. Agree timestamp validity, cross-shift cleaning totals, generated-Service ownership, and export transition rules.
4. Implement shared calculation and API persistence/merge behavior, then the required Web and CSV changes.
5. Run focused workflow/sync tests and compare the same records across Hole tabs, Services rows, shift totals, and exports.

Related corrections to earlier discussion: existing timestamp/location fields reduce schema work but do not prove the feature is already supported end to end. No source literal for Hole Cleaning is not evidence that the live Service type is missing. An optional shift table Duration column belongs to the current metadata provider, even though legacy hole-loading code remains in the page.

## Related knowledge

- [[2026-05-21 Drill Plan Upload NRE and Duplicate Ring Debugging]]
- [[2026-07-14 User Timezone Feature Explained]]
- [[2026-07-07 Reamer Hierarchy Feature Explained]]
- [[2026-07-09 Generic Metadata Tables Feature Explained]]
- [[2026-03-27 Deswik K92 Shift Integration API Design]]

PRD references supplied in the conversation: Hole Cleaning, PO-I-19; Hole and Ream Duration, PO-I-80. This review used the pasted requirements rather than accessing the Aha pages.
