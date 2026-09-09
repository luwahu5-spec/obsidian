---
date: 2026-06-26
source: Claude Code (VS Code)
project: core-web
tags: [domain-knowledge, drill-length, recalculated-holes, hole-model]
---

# Does "Outstanding" drill length exclude Recalculated holes? Yes.

## Question
For PRODOP, does Total/Individual remaining ("Outstanding") drill length already exclude recalculated data? (A Recalculated Hole = hole no longer drilled per plan because it was Smart Collared with offsets, e.g. a bolt in the ceiling forced a position shift.)

## Answer — exclusion happens at the `Hole` model level
`Minnovare.Core.Shared/Models/Hole.cs`:
- `Hole.TargetTotalLength` (~275): `if (RecalculatedHoleId.HasValue) return 0;` — target length comes from the *new* hole instead.
- `Hole.CompletedLength` (~240): also returns 0 when `RecalculatedHoleId.HasValue`.

UI formula: Outstanding = `drive.TargetTotalLength - drive.CompletedLength` (and per-ring equivalent) — since both sides already zero out recalculated holes, **Outstanding excludes them everywhere automatically**.

## Search tip
"Outstanding" was a description, not a code identifier — locate the value by going to the page component (RigDetails) and following the computed properties, not by keyword search.
