---
date: 2026-07-07
updated: 2026-07-07
source: Claude Code
project: core-web / core-web-api
tags: [domain-knowledge, reamer-hierarchy, ream-passes, drill-length, production, site-settings]
---

# Reamer Hierarchy — what the feature actually does

> **Takeaway:** one production hole = several physical drilling passes (pilot + progressively larger reams). The Reamer Hierarchy is the per-site *expectation* of which passes each design diameter needs; at CSV import it materializes as `Ream` child rows on the hole, and every row counts as `TargetLength` toward planned/completed length. The hierarchy config can express "this site never drills size X" — it **cannot** express per-hole ad-hoc skipping, which is the gap the Reamer Length Reconciliation PRD fills: [[2026-07-07 PRD Reamer Length Reconciliation Analysis and Executable Plan]].

## The physical reality being modelled

A large-diameter production hole is not drilled in one go: the rig drills a small **pilot pass**, then **reams** the same hole with progressively larger bits up to the design diameter. Each pass runs the hole's full length.

Example: 12 m hole, design 127 mm, passes 64 → 89 → 102 → 127 = 4 passes × 12 m = **48 m of real drilling** for one 12 m hole. This is why ream passes legitimately count toward drilled length.

## The configuration chain

1. **Site Ream Sizes** — Site Create/Edit, Production tab (`SiteCreateEdit.razor:270`): comma list of every bit size on site (e.g. `"64,89,102,127"`), stored as string on `Site.ReamSizes`.
2. **Reamer Hierarchy** — `ReamHierarchy.razor(.cs)` component rendered under Ream Sizes. Shape: `Dictionary<float, List<float>>` — key = design diameter, value = expected intermediate sizes. Default = every smaller size (`_reamSizes.Take(i)`, ReamHierarchy.razor.cs:105); Support unticks sizes the site skips as standard practice. Stored per site (migration `20250912073914_SiteReamHierarchies`).
3. **Rig ream selection** (related, separate) — `RigEdit.razor:93`: which sizes a rig uses; drives tablet display and per-ream MWD pressure settings. Not part of the length calculation.

## Where it takes effect

On drill-plan CSV upload, `CsvImporter.cs:413` calls `hole.CreateReams(reamHierarchy, reamSizes)` (`Hole.cs:284-331`):
- Design diameter found in hierarchy → one `Ream` row per configured size smaller than the design diameter.
- Diameter **not** in hierarchy → fallback: rows for **all** site ream sizes smaller than the diameter.
- Diameter mapped to empty list → no reams.

### The three-state design (key absent ≠ key empty — not a contradiction)

The dictionary deliberately distinguishes two states that look the same but mean opposite things:

| State | Human meaning | Result |
|---|---|---|
| Key **absent** (`!ContainsKey`) | "Nobody ever configured this diameter" — unknown intent | Safe fallback: ALL site ream sizes (= the original pre-hierarchy behaviour) |
| Key **present, value empty/null** | "Support explicitly unticked everything" — active decision | No reams: hole drilled in a single pass |
| Key present with sizes | Configured expectation | Exactly those passes |

Think of it as a form field: *a question never asked is not the same as a question answered "none".*

When does "key absent" actually happen?
1. **Legacy sites** — the hierarchy arrived with migration `20250912073914_SiteReamHierarchies` (Sept 2025); sites configured before it have no keys, and the fallback branch IS the old behaviour → this code path is how the "hierarchy config remains backwards compatible" NFR is implemented.
2. **Unknown diameters in the CSV** — the UI only generates keys for the site's configured ream sizes (`ReamHierarchy.razor.cs:102-107`); a CSV hole diameter that isn't a site ream size (typo, new bit not yet added to site settings) has no key.

Wrinkle: the code checks `hierarchySizes == null`, but the UI stores an **empty list** when all are unticked — both produce zero reams (empty list iterates zero times in `CreateReams`), the null-check just guards deserialization.

The `Ream` rows are the *expectation*. As drilling progresses the tablet stamps each pass with `ActualLength` + `DateDrilled`; the final design-diameter pass is the hole record itself, and completing it sets `DrillStatus = Done`.

## How it feeds the length numbers

- **Planned:** `TargetTotalLength = TargetLength × (Reams.Count + 1)` — hole pass + each ream (`Hole.cs:280`).
- **In progress:** largest drilled ream found by diameter-descending `FindIndex`; it plus all smaller passes count as done (`Hole.cs:251`).
- **Done:** returns full `TargetTotalLength` unconditionally (`Hole.cs:245-248`) ← the line the reconciliation PRD changes.
- Aggregation: Ring sums holes, Drive sums rings, `Drive.Outstanding() = Target − Completed` (`Drive.cs:100-104`) — all computed properties, nothing persisted. Same chain as [[2026-06-26 Outstanding Length Excludes Recalculated Holes]].

## Questions asked
- 2026-07-07 — "Walk me through the Reamer Hierarchy feature — I know it exists but not what it does." → full walkthrough; this note.
- 2026-07-07 — "Isn't 'diameter not in hierarchy → all sizes' conflicted with 'empty list → no reams'? Aren't they the same?" → No: dictionary key **absent** = never configured (safe fallback = legacy behaviour), key **present but empty** = explicit "no reams" decision. See *The three-state design* section.

## The gap the PRD fills

Driller skips a pass on the day (e.g. pilot 64 → straight to 102 → final 127): real drilling 36 m, reported 48 m, because the skipped 89 mm `Ream` row still exists and Done counts every row. Config can't capture ad-hoc decisions; the evidence is already in data (skipped pass = `Ream` row with no `ActualLength`/`DateDrilled`), which is why the fix is a computed-property change, not a feature: [[2026-07-07 PRD Reamer Length Reconciliation Analysis and Executable Plan]].
