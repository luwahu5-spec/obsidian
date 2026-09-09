---
date: 2026-06-12
source: Claude Code (VS Code)
project: load-testing
tags: [load-testing, core-sync, datasets, payloads]
---

# Core Sync load-test dataset construction rules

Each dataset folder (`dataset-001` … `dataset-050`, under `payloads/<set>/dataset_<env>/`) contains: `drill-plan.json`, `metadata.json`, `pre-start-data.json`, `sensor-calibrations.json`, `sensor-states.json`, `shift.json`.

## Consistency rules (a dataset is invalid unless all hold)
1. **tabletId** identical across `metadata.json`, `shift.json`, and the shift item inside `pre-start-data.json` — `metadata.json` is the source of truth.
2. **Engine hours** (`engineHours`, `drifterHours`, `percussionHours`) must be **whole numbers** in both `shift.json` and the shift item in `pre-start-data.json`.
3. **rigId/siteId must be unique across datasets** (check `sensor-calibrations.json` / `sensor-states.json`) — duplicated rig IDs between e.g. dataset-001 and dataset-011 corrupt concurrent test runs.

Environment-specific sets exist: `dataset_local`, `dataset_staging`, `dataset_dev` — same rules for all.

## Related
- [[2026-06-05 k6 Load Testing Setup for Core Sync]]
