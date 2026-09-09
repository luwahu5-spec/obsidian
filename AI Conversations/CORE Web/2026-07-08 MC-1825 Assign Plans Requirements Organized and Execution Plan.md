---
date: 2026-07-08
updated: 2026-07-08
source: Claude Code (Jira MC-1825 read via browser)
project: core-web
tags: [planning, mc-1825, assign-plans, rig-details, drilling-methods, core-refactor-2.23]
---

# MC-1825 Assign Plans — requirements organized & execution plan

> **Takeaway:** the ticket is ONE workflow spec written three times (DEVOP / PRODOP / CUT&FILL). DEVOP and CUT&FILL are identical except the plan noun; PRODOP adds a two-level Drive→Rings tree and deletes legacy features. The real (hidden) functional change is **multi-rig selection + bulk assignment** — today's page is per-rig. Implement as ONE config-driven page or we create the exact SiteType branching the refactor (§22) exists to remove. Ticket: MC-1825, approved by DC 6-Jun-2026.

## What the ticket actually says (decomposed)

**One sentence:** replace the per-rig "Rig Details" page with a simplified **"Assign Plans"** page, identical across all 3 drilling methods: *select rig(s) → assign plan items → view/unassign assigned items* — and delete PRODOP's extra complexity (comment icons, "show only drives with comments", "filter by drilling status", View Holes, inline hole tables/edit).

### The shared pattern (applies to ALL three methods — write once)
1. **Entry (x-IA.01/.02):** rig card button renamed "Rig Details" → **Assign Plans**; page title = "Assign Plans" (was "Rig: {name}"); breadcrumb back to Selected Site page stays.
2. **Rig Selector (x-IA.1, x-IB):** new box LEFT of the Assign button, ~1.5–2× its width, default text = current rig's name. Clicking anywhere on it opens a modal: **all ACTIVE rigs of the site** (never archived, regardless of how many plans each has), A→Z order, multi-select, scroller for long lists, selected rig need not be visible when modal opens (DC 4/6/26).
3. **Selector result string (x-IC):** Apply → box shows `{n} rigs selected >` / `All rigs selected >` / `Select rigs >` (when 0 — and reopening after 0 defaults back to ALL selected; the darkened "0 selected" while modal open stays — DC 4/6/26).
4. **Instruction strings (x-IA.2):** two Setup lines (Single Rig / Multi-Rig) — exact copy provided in ticket ("easy Cut&Paste").
5. **Assign modal (x-IIB/IIC):** existing modal, retitled "Available {noun} (Unassigned)"; shows active items with NO or SOME rigs assigned; hides items fully assigned to the selection and archived items; **A→Z order (change from selection-order)**; taller window (≥⅔ viewport); button renamed **Apply**, disabled until something ticked; ticked boxes blue.
6. **Assigned list on page (x-IID):** title as H2 (bigger, smaller than H1) + note *"List below displays only {noun}(s) assigned to (all) rig(s) SELECTED above."*; items A→Z, none ticked by default; ticking enables **Unassign** button (disabled-UI otherwise); Undo msg1/msg2 behavior as current; click-to-toggle zones = tickbox + label ONLY (blank space right of label reserved for PRODOP chevrons).

### The per-method deltas (this is ALL that differs)
| | DEVOP (D) | PRODOP (P) | CUT&FILL (CF) |
|---|---|---|---|
| Plan noun / button | Headings / **Assign Headings** | Drives>Rings / **Assign Rings** | Drives / **Assign Drives** |
| Assigned list shape | flat list | **two-level tree**: bold Drives, chevron (32×32 zone, right-aligned to Unassign button edge), grey divider between drives only, rings normal font, drive-tick = all rings ticked, some-rings-ticked ≠ drive ticked | flat list |
| Assign modal internals | tick list | **unchanged** current Drive select → Rings multi-select with Ctrl-A (see [[2026-04-28 Assign Rings Ctrl-A Multi-Select Debugging]]) — only taller + Apply rename | tick list |
| Removals | — | comment icon/data, "show only drives with comments", "filter by drilling status", **View Holes** (string + functionality), inline edit | — |
| Current component | `Development/Rigs/DevelopmentRigDetails.razor` (Assign Headings, :25-122) | `Rigs/RigDetails.razor` (Assign Rings :20, comments filter :131, status filter :136) | none (new) |

### The hidden functional change (the ticket undersells this)
Today both pages are **per-rig** (route `/Sites/Rigs/{rigId}`). The new UI selects **multiple rigs** and then:
- **Availability** is computed against the *set* ("hide items already assigned to ALL selected rigs"),
- **Assigned list** is the *intersection* ("assigned to (all) rigs selected"),
- **Assign/Unassign** are *bulk* operations across all selected rigs.

That is not a UI update — it's new assignment semantics needing backend endpoints shaped like `(siteId, methodKey, rigIds[]) → available/assigned` and batch assign/unassign. "UI Updates" in the title is misleading.

## Execution plan

**A-1 — AssignPlans page shell + method config (foundation).** One page (route method-keyed via `MiningMethodKey`, NOT `RigSiteType`), one `AssignPlansConfig` per method (noun, button label, list renderer type flat|tree, modal type ticklist|drive-rings) delivered by a provider — same pattern as `SiteUploadConfigProvider` (MC-1822). Breadcrumb + title + instruction strings from config.
**A-2 — RigSelectorModal (new shared component).** Active rigs A→Z, multi-select, result-string states (n/All/Select), the 0-selected→reopen-all rule. bUnit-heavy (pure UI state machine).
**A-3 — DEVOP variant.** Refit `DevelopmentRigDetails` content into the shell: flat tick list, retitle modal, Apply rename, A→Z sort, H2+note, click-zones.
**A-4 — CUT&FILL variant = config entry only.** If CF needs any new component code, A-1 failed — this ticket is the refactor's litmus test in miniature.
**A-5 — PRODOP variant.** Tree renderer (chevron open/close, tick propagation rules v_1/v_2), keep Ctrl-A assign modal mechanics untouched, delete comments/status-filter/View Holes code paths.
**A-6 — Backend.** Endpoints for available/assigned given rig-set + batch assign/unassign + intersection query. (Confirm owner — ticket is filed as frontend.)
**A-7 — Localization ×7 resx** (many new strings: Apply, Available Headings (Unassigned), setup lines, list notes, selector states) **+ bUnit sweep.**

Order: A-1/A-2 first (everything hangs off them), then A-3 → A-4 (prove config) → A-5 (biggest) → A-7. A-6 in parallel backend-side.

## Current endpoint inventory & the generic contract (added 2026-07-08 after code check)

Today: **two separate per-method endpoint families**, disagreeing on verb, ID type, and owning controller:
- DEVOP: `PUT /Development/AssignDrivesToRig/{rigId}` (body: Guid list, `DevelopmentController.cs:635`), `PUT /Development/RemoveDriveFromRig/{rigId}`; reads via `GetDrivesForSite` / `GetAssignedDrivesForRig`.
- PRODOP: `POST /rigs/{id}/AssignRings` (body: long list, `RigsController.cs:401`), `POST /rigs/{id}/UnassignRings`; reads via site drives + per-drive expansion (`RigDetails.razor.cs:233`).

Scenario support matrix (rig is in the ROUTE, so one call = one rig):
| 1 plan→1 rig ✅ | N plans→1 rig ✅ (native shape) | 1 plan→N rigs ❌ | N plans→N rigs ❌ |

**N plans → N rigs: CONFIRMED required** (2026-07-08 check). Evidence chain: (D-IA.2) multi-rig selection is a formal setup step before assigning; (D-IIC) "More than one Heading can be selected"; (D-IID.01) the rig count persists through assignment; and the D-IID summary mockup (image-20260604-100506) shows "2 rigs selected" with headings in the Assigned list immediately after Apply — a list defined as "assigned to (all) rig(s) SELECTED above", which is only possible if Apply wrote to both rigs. The ticket never states the sentence explicitly → confirm at review, esp. **idempotent top-up semantics** (item already on rig A, Apply with A+B → add to B only?).

MC-1825's multi-rig selector therefore **requires an endpoint refactor**, not just UI. Proposed generic contract (kills the per-method-endpoint growth — CF would otherwise be a 3rd family, method #4 a 4th):

```
GET  /plan-assignments/available?siteId=&miningMethod=&rigIds=
GET  /plan-assignments/assigned ?siteId=&miningMethod=&rigIds=     ← intersection rule computed server-side
POST /plan-assignments/assign     { miningMethod, rigIds[], planItemIds[] }   ← one transaction, all 4 matrix cells
POST /plan-assignments/unassign   { miningMethod, rigIds[], planItemIds[] }
```
- Backend: per-method strategy resolves miningMethod → entity (DevelopmentDrive / Ring / Drive). Method #4 = one new strategy, zero new endpoints.
- Read shape is generic `{ id, label, children[] }` — PRODOP returns children (rings under drives), DEV/CF flat → **flat-vs-tree becomes data-driven**, the frontend renders children if present and never knows method names.
- Frontend: ONE AssignPlans component + config provider for text (noun/labels/instructions per MiningMethodKey). It always calls the same endpoints, passing the method key — no per-method branching, not even endpoint selection.
- If backend can't deliver this cycle: wrap the legacy families behind one `IPlanAssignmentClient` frontend adapter (labelled temporary bridge). Caveat: looping per-rig writes = N round trips + partial-failure ambiguity — acceptable only as a bridge, push for the transactional batch endpoint.

### How — strategy pattern pseudo code (added 2026-07-08 after "new strategy = new endpoint?" question)

**Clearing the misconception: methods do NOT share logic — they share the CONTRACT.** "Zero new endpoints" = the URL surface doesn't grow; each method's unique logic (different entities, Guid vs long IDs, different queries, future new models) lives inside its own strategy class. PDF-vs-CSV is the *upload* feature (already config-driven, MC-1822) — assign-plans only links rigs to already-imported plan rows and never touches files.

```csharp
public class PlanItemDto { string Id; string Label; List<PlanItemDto> Children; bool Selectable; }
// Id as string absorbs Guid (DevelopmentDrive) vs long (Ring); Children = PRODOP's tree AS DATA

public interface IPlanAssignmentStrategy
{
    MiningMethod Method { get; }
    Task<IReadOnlyCollection<PlanItemDto>> GetAvailableAsync(long siteId, IReadOnlyCollection<long> rigIds);
    Task<IReadOnlyCollection<PlanItemDto>> GetAssignedAsync(long siteId, IReadOnlyCollection<long> rigIds); // intersection
    Task AssignAsync(long siteId, IReadOnlyCollection<long> rigIds, IReadOnlyCollection<string> planItemIds);   // REQUIRED: idempotent top-up (decision 2026-07-08) — end-state semantics, skip existing links, one transaction, never error/duplicate
    Task UnassignAsync(long siteId, IReadOnlyCollection<long> rigIds, IReadOnlyCollection<string> planItemIds);
}

// ONE controller, no method logic: shared auth/validation, then
//   _strategies[req.MiningMethod].AssignAsync(...)  — strategies injected via DI (IEnumerable → dictionary)

// DevelopmentPlanAssignmentStrategy : queries DevelopmentDrives (Guid), flat DTOs, absorbs AssignDrivesToRig logic
// ProductionPlanAssignmentStrategy  : queries Drives+Rings (long), DTOs with Children, absorbs AssignRings logic
// CutAndFillPlanAssignmentStrategy  : its own entity/queries
// Method #4 with a brand-new plan model = new model+migration (needed under ANY design) + one strategy class + one DI line.
//   Controller untouched, routes untouched, frontend untouched (labels via config provider).
```

Frontend: ONE `PlanAssignmentRepository`, ONE `AssignPlansUiConfig` (text only, per MiningMethodKey), and the modal renders `item.Children.Any()` → chevron+nested — tree-vs-flat is a **data** question, never `if (method == Production)`.

**Honest limit (state it before someone else does):** the contract assumes plan items are identifiable + labelable + optionally hierarchical tick-list items. A future method needing a fundamentally different assignment UI (e.g. map picker) would need a new modal renderer variant — but its endpoints would still fit the strategy.

## Refactor-goal conflicts & design-review points (meeting 2026-07-08 PM)

1. **The ticket's structure invites the anti-pattern.** Three parallel specs → three page edits → more `SiteType`/method conditionals, which §22 of the Front End PRD then pays to remove. Counter-proposal: one page + config provider; CF ships as a config entry. Lead + product said MC-1825 is "not technically in refactor scope" — but it touches the exact pages the refactor rebuilds, so **build it ON the refactor architecture now or pay twice**.
2. **Method nouns/labels are frontend-hardcoded in the ticket.** Refactor principle: frontend never knows method names — labels should key off backend method metadata (`MiningMethodKey` → config), so drilling method #4 = config, not components.
3. **Availability/intersection rules are business logic.** "Hide items fully assigned to selected rigs" computed in Blazor = frontend calculation, violating the "backend provides, frontend displays" goal. Push for the A-6 endpoints; if backend can't deliver this cycle, isolate the logic in ONE frontend service labelled as a temporary bridge (same convention as `FilterMiningMethodTabsByLegacySiteFeatures`).
4. **PRODOP tree is the one legitimate method-specific UI** — model it as a config-selected list renderer (flat | grouped), consistent with §22's "fixed shell, configurable method-specific parts".
5. **Intersection-unassign trap (product question):** an item assigned to 2 of 3 selected rigs appears in NO list (not "available" for none-assigned semantics? it appears in available since SOME assigned — but not in assigned list since not ALL). To unassign it you must reselect exactly the right rigs. Is that acceptable, or should assigned list show "assigned to ANY selected rig" with a count badge?
6. **Bulk assign to partially-assigned rigs — DECIDED (engineering, 2026-07-08): idempotent top-up is the required endpoint behavior.** Inform product rather than ask. The ticket guarantees the mixed case — the modal shows items with SOME rigs assigned (x-IIB.2), so ticking an item rig A already has, with A+B selected, is normal use. Rejected alternatives: erroring "already assigned" contradicts the UI that offered the item; blind insert creates duplicate links/corrupt data. Note: today's single-rig endpoints never faced this (already-assigned items simply weren't listed, RigDetails.razor:99) — multi-rig selection creates the mixed starting state for the first time.

   **Endpoint contract requirement (binding for A-6 implementation):**
   - `AssignAsync(siteId, rigIds[], planItemIds[])` — declares the END STATE: every selected rig has every selected item. Walk each (rig × item) pair; link exists → skip silently; link missing → create. **Never error, never duplicate.** Whole batch in ONE transaction. Calling Apply twice in a row is a no-op the second time.
   - `UnassignAsync` — mirror: link exists → remove; link doesn't → skip silently.
   - Enforce with a **DB-level uniqueness constraint** on the link (rigId + planItemId) per method's link table as the safety net — idempotency must survive concurrent Applies from two browser sessions, not just sequential ones.
   - Unit tests per strategy: mixed pre-assignment state, double-Apply no-op, concurrent-Apply safety.
7. **View Holes is deleted, not relocated** — confirm product accepts losing the feature entirely (it's in the removals list, DC approved).
8. **The 0-selected→reopen-defaults-to-ALL rule** is surprising UX — confirm it survived review intentionally (it's marked DC-reviewed, so likely yes; just flag).

## Related
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]] — the config-provider pattern this must follow
- [[2026-07-07 PRD Front End 15-22 Analysis and Executable Plan]] — §22 General is where 3-way branching would have to be undone
- [[2026-04-28 Assign Rings Ctrl-A Multi-Select Debugging]] — the multi-select mechanics P-IIC preserves
