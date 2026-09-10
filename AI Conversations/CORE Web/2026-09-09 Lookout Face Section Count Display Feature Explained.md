---
date: 2026-09-09
updated: 2026-09-09
source: Claude Code
project: core-web
tags: [domain-knowledge, lookout, cut-face, svg, blazor]
---

# Lookout Face Section Count Display — what the feature actually does

> **Takeaway:** The numbers overlaid on each cut-face section in the Lookout page are not part of the polygon shape — they're separate SVG `<text>` elements positioned at each polygon's mathematical centroid. The value shown is simply `List<CutLookoutDisplay>.Count` for that section, computed live from a `Dictionary<int, List<CutLookoutDisplay>>` the parent page builds by grouping all lookouts for the cut by `SectionIndex`.

## The business/physical reality being modelled
A drive's face is divided into a fixed 5×3 grid of physical drilling sections — 5 rows (Lifters, Knees, Vertical Bur, Burden, Backs) × 3 columns (Left Perimeter, Horizontal Burn, Right Perimeter) = 15 sections. Each recorded "lookout" (a horizontal/vertical deviation measurement) belongs to exactly one of these 15 sections. The diagram exists so a user can see, at a glance, how many lookouts have been logged per section, then click a section to drill into its details.

## The configuration chain
- Grid geometry (row/column names, spacing percentages) is hard-coded in `DrawCutSections()`, not configurable per site: [CutFaceComponent.razor.cs:32-80](Minnovare.Core.Web/Pages/Blazor/Shared/CutFaceComponent.razor.cs#L32-L80).
- Each section gets an `Index = rowIndex * cols + colIndex` (0-14), used as the join key against the lookout data: [CutFaceComponent.razor.cs:74](Minnovare.Core.Web/Pages/Blazor/Shared/CutFaceComponent.razor.cs#L74).

## Where it takes effect
- Parent page `LookoutDetails.razor.cs` loads all lookouts for the cut (`GetAllLookoutDataByCut`) and groups them into `_lookoutMap` (`Dictionary<int, List<CutLookoutDisplay>>`) keyed by `lookout.SectionIndex`: [LookoutDetails.razor.cs:124-139](Minnovare.Core.Web/Pages/Blazor/Development/LookoutDetails.razor.cs#L124-L139).
- That map is passed straight into the child component as `DataSource="@_lookoutMap"`: [LookoutDetails.razor:27-31](Minnovare.Core.Web/Pages/Blazor/Development/LookoutDetails.razor#L27-L31).
- Selecting a section (`OnSelectedSectionChanged`) swaps the table's `_displayLookout` to that section's list (or all lookouts if deselected, index `< 0`), sorted by `LastUpdated` descending: [LookoutDetails.razor.cs:145-164](Minnovare.Core.Web/Pages/Blazor/Development/LookoutDetails.razor.cs#L145-L164).

## How it feeds the numbers/UI
- `CaculateLookoutCount(index)` looks up `index` in `DataSource`; returns `value.Count` if present, else `0`: [CutFaceComponent.razor.cs:83-93](Minnovare.Core.Web/Pages/Blazor/Shared/CutFaceComponent.razor.cs#L83-L93). This becomes `FaceSection.Count`.
- Each `FaceSection`'s polygon points are computed from the grid geometry; `GetCentroidX`/`GetCentroidY` run the standard polygon-centroid (shoelace-formula-based) calculation over those points: [CutFaceComponent.razor.cs:113-143](Minnovare.Core.Web/Pages/Blazor/Shared/CutFaceComponent.razor.cs#L113-L143).
- The markup draws one `<g>` per section containing a `<polygon>` (the shape, click target) and a `<text>` positioned at `(centroidX, centroidY)` with `text-anchor="middle"` / `alignment-baseline="central"`, so the count digit sits visually centered regardless of the section's irregular trapezoid shape: [CutFaceComponent.razor:6-17](Minnovare.Core.Web/Pages/Blazor/Shared/CutFaceComponent.razor#L6-L17).
- The selected section (blue highlight in the screenshot) is driven by CSS class `"selected"` vs `"stroked"` from `GetPolygonClass`, compared against the `SelectedIdx` parameter — independent of the count logic: [CutFaceComponent.razor.cs:108-111](Minnovare.Core.Web/Pages/Blazor/Shared/CutFaceComponent.razor.cs#L108-L111).

## Design subtleties & edge cases
- A section with zero lookouts still renders — `CaculateLookoutCount` falls back to `0` rather than hiding the section or its text.
- The centroid formula works for any convex/simple polygon, which matters because sections aren't uniform rectangles — the Backs row and perimeter columns are trapezoids due to the `xPointPercentages`/`rowWidthPercentages` tapering ([CutFaceComponent.razor.cs:35-37](Minnovare.Core.Web/Pages/Blazor/Shared/CutFaceComponent.razor.cs#L35-L37)), so a naive bounding-box center would be visually off-center for those.
- `SectionIndex` is the only link between the visual grid and lookout records — if the grid geometry (rows/cols) ever changes, the index math (`rowIndex * cols + colIndex`) must stay in sync with however `SectionIndex` is assigned when a lookout is created, or counts will land on the wrong section.

## Questions asked
- 2026-09-09 — "How are the integers displayed over each face section?" → SVG `<text>` at each polygon's centroid, value = `List<CutLookoutDisplay>.Count` per section from `_lookoutMap` grouped by `SectionIndex`. See "How it feeds the numbers/UI".

## Related
- (none yet)
