---
date: 2026-04-13
source: Codex (VS Code)
project: core-web
tags: [sql, park-snapshots, rings, investigation, test-data]
---

# Finding rings with park-log history (SQL detective work)

## Data model
A ring "has a park log" only if `ParkSnapshots` has rows for that `RingId`. Chain: `Rings.DriveId → Drives.Id`, `Drives.SiteId → Sites.Id`, `ParkSnapshots.RigId → Rigs.Id`. A park log = **Ring + Snapshot + Rig**, not just ring↔rig assignment. Ring details URL shape: `/{SiteId}/Rings/{RingId}`.

## Finding test cases: rings with multi-rig history
To find rings whose park log shows *another* rig (clickable rig link in UI), search from the snapshots side:
```sql
SELECT r.Id AS RingId, r.Name, d.SiteId, d.Name AS DriveName,
       COUNT(DISTINCT ps.RigId) AS DistinctRigCount,
       CONCAT('/', d.SiteId, '/Rings/', r.Id) AS RingDetailsUrl
FROM dbo.Rings r
JOIN dbo.Drives d ON d.Id = r.DriveId
JOIN dbo.ParkSnapshots ps ON ps.RingId = r.Id
WHERE ps.RigId IS NOT NULL
GROUP BY r.Id, r.Name, d.SiteId, d.Name
HAVING COUNT(DISTINCT ps.RigId) > 1;
```

## Technique
When hunting UI test cases, **query from the data that produces the UI feature** (snapshots) back to the entities, rather than from the entity you happen to know (rig). "Current assignment" lives in a separate join table (RigAssignedRings-style) — plain Rings/ParkSnapshots can't tell you current binding.
