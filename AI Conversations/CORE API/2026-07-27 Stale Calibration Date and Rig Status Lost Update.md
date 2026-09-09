---
date: 2026-07-27
source: Codex (VS Code)
project: core-web-api, core-web
tags: [debugging, calibration, rig-status-summary, acknowledgement, concurrency, lost-update, core-sync]
---

# Stale calibration date caused by a probable `RigStatusSummary` lost update

## Executive summary

Big Bell rig `LH017` received a new calibration on 9 July 2026. The new values appeared in the Calibration Deltas chart and Calibration Details Log, but the dashboard continued to show:

```text
Last Calibration Date = 17/03/2026
Calibration Due       = 14/04/2026
```

Production database evidence established that:

- Raw calibration history contains 9 July.
- Daily calibration-summary history contains 9 July.
- The 9 July summary has a higher identity ID than the March summary.
- There is exactly one `RigStatusSummary` row for the rig.
- That status row still contains 17 March.
- Both the March and July invalid calibrations have been acknowledged.

The dashboard is not calculating the latest date from calibration history. It trusts `RigStatusSummaries.LastCalibrationUpdate`, so its displayed date and derived due date are stale whenever that snapshot is stale.

The most probable root cause is a lost-update race in `RigStatusSummaryRepository.UpdateDetails`. A sync-only request loads the complete status row and calls `_dbSet.Update(rigSummary)`. `Update` marks the whole entity as modified, including calibration fields the request did not intend to change. If that request loaded the March state, overlaps the July calibration request, and saves last, it can restore the March date and `Okay` state while retaining its newer sync time.

The acknowledgement endpoint then preserves the stale date because it writes `rigSummary.LastCalibrationUpdate` back instead of using the date of the calibration being acknowledged.

This lost-update sequence is the strongest explanation supported by the current code and production data, but it cannot be historically proven without the production API logs or a database audit/concurrency column. A missing outer `OrderBy` in calibration rebuilding remains a separate correctness defect; production ID evidence does not support it as the cause of this specific LH017 incident.

## Reported incident

Site and rig:

```text
Site: Big Bell
Rig:  LH017
RigId: 347
```

Timeline reported by Testing:

```text
17/03/2026 - previous invalid calibration
09/07/2026 - new invalid calibration received
```

Observed UI behaviour:

- The 9 July values appeared in the Calibration Deltas chart.
- The 9 July values appeared in the Calibration Details Log.
- Last Calibration Date remained 17 March.
- Calibration Due remained 14 April.
- The rig remained calibration-out-of-date even though a new calibration existed.

Testing proposed two possible explanations:

1. An invalid calibration was received on top of an earlier invalid calibration.
2. When Calibration Error and Calibration Out of Date both exist, receiving a calibration resets only one error.

Those scenarios describe visible alarm combinations, but neither explains why the status date itself stayed in March.

## The three calibration data layers

The system has three related but distinct representations of calibration data.

### 1. `SensorCalibrationData`: raw calibration history

Database table:

```text
SensorCalibrations
```

Important fields:

```text
Id
RigId
DateRecorded
AzimuthOffset
InclinationOffset
RollOffset
SensorId
SensorPosition
```

This is the original data received from Production Optimizer. There can be multiple raw readings for the same rig and calendar day.

The Calibration Details Log reads this raw history. Therefore, the log can correctly show a new calibration even when the status snapshot is stale.

The latest raw calibration is determined by:

```sql
MAX(SensorCalibrations.DateRecorded)
```

For LH017, the relevant raw records were:

```text
Id 19026 - 2026-03-17 08:32:19.812 +00:00
Id 20353 - 2026-07-09 02:59:27.478 +00:00
```

Therefore, the latest raw calibration is 9 July.

### 2. `RigCalibrationSummary`: one calculated summary per calibration day

Database table:

```text
RigCalibrationSummaries
```

Important fields:

```text
Id
RigId
Date
AzimuthCalibrationValue
InclinationCalibrationValue
RollCalibrationValue
AzimuthDelta
InclinationDelta
RollDelta
AzimuthDeltaOutOfRange
InclinationDeltaOutOfRange
RollDeltaOutOfRange
IsAcked
```

The API groups raw calibrations by rig and calendar day, selects the final reading within each day, and compares that reading with the previous daily calibration to calculate deltas.

```text
Raw SensorCalibrationData
          |
          v
Group by rig and calendar date
          |
          v
Select the latest reading within each date
          |
          v
Calculate deltas from the previous daily calibration
          |
          v
RigCalibrationSummary
```

The Calibration Deltas chart reads these summary records and orders them by `Date`.

Production evidence for LH017:

```text
Id 8394 - Date 2026-03-17
Id 8962 - Date 2026-07-09
```

Both rows had all three out-of-range flags set:

```text
AzimuthDeltaOutOfRange     = 1
InclinationDeltaOutOfRange = 1
RollDeltaOutOfRange        = 1
```

Both rows also had:

```text
IsAcked = 1
```

The correct latest daily calibration is:

```sql
MAX(RigCalibrationSummaries.Date)
```

For LH017, that is 9 July.

### 3. `RigStatusSummary`: current dashboard snapshot

Database table:

```text
RigStatusSummaries
```

Important fields:

```text
Id
RigId
LastCalibrationUpdate
CalibrationState
LastSyncDate
SensorErrorCount
```

This is not calibration history. It is intended to be one current-status row per rig.

When a new calibration arrives, the API should update the existing row rather than insert another status row:

```text
Before July calibration:
LastCalibrationUpdate = 17/03/2026

Expected after July calibration:
LastCalibrationUpdate = 09/07/2026
```

Production evidence showed exactly one status row:

```text
Id                    = 274
RigId                 = 347
LastCalibrationUpdate = 2026-03-17
CalibrationState      = Okay
LastSyncDate          = 2026-07-23 22:24:23.5220344 +00:00
```

The row's continuing `LastSyncDate` updates show that later sync activity was reaching and modifying the status row while its calibration date remained stale.

## Which value decides the most recent calibration?

There is no single source used consistently by every screen or endpoint.

| Consumer or operation | Source used as "latest" |
|---|---|
| Calibration Details Log | Maximum raw `SensorCalibrationData.DateRecorded` |
| Calibration Deltas chart | Latest `RigCalibrationSummary` after ordering by `Date` |
| Dashboard Last Calibration Date | `RigStatusSummary.LastCalibrationUpdate` |
| Calibration Due | `LastCalibrationUpdate + 28 days` |
| Calibration Out of Date | Whether `LastCalibrationUpdate` is more than 28 days old |
| Current acknowledgement API | Highest `RigCalibrationSummary.Id` |
| Business truth | Latest calibration by `RigCalibrationSummary.Date`, supported by raw `DateRecorded` |

For LH017:

```text
Latest raw date:                 09/07/2026
Latest summary date:             09/07/2026
Status snapshot date:            17/03/2026
Latest acknowledged calibration: 09/07/2026
```

The business truth is 9 July. The dashboard is wrong because it trusts the stale status snapshot.

## How Calibration Due and Calibration Out of Date work

Calibration Due is not an independent database field. CORE Web derives it from the status date:

```csharp
calibrationDueDate = rigSummary.LastCalibrationUpdate
    .Value
    .ToUniversalTime()
    .AddDays(Errors.CalibrationTimeoutThresholdDays);
```

The configured calibration threshold is 28 days.

For the stale LH017 status:

```text
17/03/2026 + 28 days = 14/04/2026
```

This exactly explains the reported due date.

The dashboard also calculates the out-of-date condition from that same field:

```csharp
(DateTime.UtcNow - LastCalibratedDate.ToUniversalTime()).Days > 28
```

Therefore, one stale field produces all three symptoms:

```text
Wrong Last Calibration Date
Wrong Calibration Due date
Incorrect Calibration Out of Date alarm
```

## Calibration validity versus calibration age

Calibration validity and calibration age are separate concepts.

Validity is based on calculated delta flags:

```text
AzimuthDeltaOutOfRange
InclinationDeltaOutOfRange
RollDeltaOutOfRange
```

Age is based on:

```text
RigStatusSummary.LastCalibrationUpdate
```

This produces four meaningful states:

| Age    | Delta result               | Expected visible state                        |
| ------ | -------------------------- | --------------------------------------------- |
| Recent | Valid                      | No calibration alarms                         |
| Recent | Invalid and unacknowledged | Calibration Error only                        |
| Old    | Valid or acknowledged      | Calibration Out of Date only                  |
| Old    | Invalid and unacknowledged | Calibration Error and Calibration Out of Date |

An invalid calibration is still a real new calibration. Its date must advance even when its values remain outside tolerance.

Therefore, a new invalid calibration received on 9 July should initially produce:

```text
LastCalibrationUpdate = 09/07/2026
CalibrationState      = Error
CalibrationOutOfDate  = false
```

This is why invalid-on-invalid does not explain the stale date.

## What acknowledgement means

Acknowledgement does not make an invalid calibration mathematically valid. It records that the operator knows about the deviation and suppresses the current alarm.

The summary retains the actual results:

```text
AzimuthDeltaOutOfRange     = true
InclinationDeltaOutOfRange = true
RollDeltaOutOfRange        = true
```

Acknowledgement changes:

```text
IsAcked = true
CalibrationState = Okay
```

For LH017, both the March and July invalid summaries were acknowledged. This correctly explains why the status state is now `Okay`.

It does not explain why the status date is March.

### Current acknowledgement date behaviour

`ChangeCalibrationAckState` selects what it considers the newest summary, changes `IsAcked`, and updates `RigStatusSummary`.

The current code passes the status row's existing date back into the update:

```csharp
UpdateRigSummary(
    rig,
    hasCalibrationErrors: calibrationError,
    lastCalibrationDate: rigSummary.LastCalibrationUpdate);
```

If the status row is already stale, acknowledgement preserves the stale value:

```text
Calibration being acknowledged = 09/07/2026
Existing status date            = 17/03/2026
Date written by acknowledgement = 17/03/2026
```

Acknowledgement therefore contributed to the persistence of the LH017 problem, but it does not by itself reveal how the status first became stale.

## Investigation process

### Step 1: Trace the dashboard value to its source

The dashboard obtains status data from:

```text
GET /Rigs/RigStatusSummaries
```

CORE Web maps:

```text
RigStatusSummary.LastCalibrationUpdate
    -> RigStatusInfo.LastCalibratedDate
    -> Last Calibration Date display
```

Calibration Due is then computed by adding 28 days.

This established that the dashboard and calibration charts do not share the same source of truth.

### Step 2: Trace calibration ingestion

`POST /Rigs/{id}/SensorCalibrations`:

1. Saves raw `SensorCalibrationData`.
2. Finds the earliest incoming date.
3. Deletes overlapping daily summaries when a rebuild is required.
4. Groups raw records per day.
5. Calculates daily summary deltas.
6. Adds `RigCalibrationSummary` rows.
7. Updates `RigStatusSummary` using the final calculated summary.
8. Saves summary and status changes.

### Step 3: Investigate missing SQL ordering

The daily rebuild query was found to be:

```csharp
.GroupBy(c => new { Day = c.DateRecorded.Date })
.Select(g => g.OrderByDescending(s => s.DateRecorded).First());
```

The inner ordering selects the latest reading within each day. It does not contractually order the selected days relative to one another. The later use of `summaryData.LastOrDefault()` therefore relies on an order that the LINQ query does not explicitly request.

This was initially considered the likely explanation for random rigs.

### Step 4: Test the ordering theory locally

The local database was queried read-only:

```text
453 rigs
18,217 raw calibration rows
7,476 calibration summaries
404 status summaries
309 rigs with multiple calibration days
```

One local rig had the same architectural mismatch:

```text
Site: New Holland
Rig:  JB236 - New Holland
Status date:         08/08/2024
Latest summary date: 19/11/2024
Latest raw date:     19/11/2024
Gap:                 103 days
```

However, the endpoint-equivalent unordered query happened to return chronological results for all 309 multi-day local rigs:

```text
Non-chronological results: 0
```

This did not make the query safe, but it meant the ordering hypothesis had not been reproduced locally. The earlier conclusion was reduced from "root cause" to "separate latent defect."

### Step 5: Inspect production LH017 data

Production queries established:

```text
Raw calibration:
  17/03/2026, Id 19026
  09/07/2026, Id 20353

Calibration summary:
  17/03/2026, Id 8394
  09/07/2026, Id 8962

Status summary:
  one row, Id 274
  LastCalibrationUpdate = 17/03/2026
  CalibrationState = Okay

Acknowledgement:
  March IsAcked = 1
  July  IsAcked = 1
```

The July summary had the higher ID. Nearby summary IDs were associated with recent July calibrations from other rigs, consistent with ordinary ongoing API activity rather than March being inserted after July during one reversed rebuild.

This ruled against the missing outer ordering as the explanation for LH017.

There were no duplicate status rows, ruling out the dashboard selecting the wrong duplicate.

### Step 6: Identify the full-row update risk

`RigStatusSummaryRepository.UpdateDetails` loads a tracked status entity:

```csharp
var rigSummary = _dbSet.FirstOrDefault(rs => rs.Rig.Id == rig.Id);
```

It conditionally changes only the requested fields. For example, a normal sync request may change only:

```csharp
rigSummary.LastSyncDate = lastSyncDate.Value;
```

But the method then calls:

```csharp
_dbSet.Update(rigSummary);
```

For an existing tracked entity, `Update` is unnecessary and marks the complete entity as modified. Consequently, a sync-only request can issue an update that includes stale calibration values it never intended to change.

`RigStatusSummary` has no row-version or other optimistic concurrency token, so EF/SQL Server cannot detect the lost update.

## Most probable root cause: sync-only request overwrites calibration fields

The race is not between the March and July calibration events. It is between two requests executing around the July sync while one request is carrying March as an old in-memory value.

Probable sequence:

```text
Initial status row:
  LastCalibrationUpdate = 17 March
  CalibrationState      = Okay

Request A: ordinary sync request
  Loads the complete status row containing March.
  Intends to change only LastSyncDate.

Request B: July calibration request
  Saves the July raw calibration.
  Saves the July daily summary.
  Updates status to:
    LastCalibrationUpdate = 9 July
    CalibrationState      = Error

Request A saves after Request B
  Because Update() marked every property modified, it writes:
    LastSyncDate           = new sync time
    LastCalibrationUpdate = 17 March (stale)
    CalibrationState      = Okay (stale)

Final database state:
  Raw calibration         = July
  Calibration summary     = July
  Status date             = March
  Status state            = Okay
```

This explains all important characteristics:

- The failure is random because it depends on request timing.
- March and July do not need to be concurrent calibration events.
- Multiple sync endpoints update `LastSyncDate` on the shared status row.
- The new raw and summary records survive.
- The single status row can revert to old values.
- Later syncs continue moving `LastSyncDate` forward while retaining the stale March calibration date.
- Later acknowledgement of July changes or preserves the state as `Okay` but reuses the stale March date.

The production `DeviceSyncLogs` showed multiple distinct sync IDs being processed within seconds, which is consistent with the request-overlap environment needed for this race. Exact historical proof still requires production request/API logs because the status table has no audit timestamp or concurrency version.

## Relationship to Testing's proposed causes

### Theory 1: invalid calibration received after another invalid calibration

Not the cause of the stale date.

The status update sets `LastCalibrationUpdate` independently of the out-of-range result. A new invalid calibration should advance the date while keeping the Calibration Error active until acknowledgement.

### Theory 2: both Calibration Error and Calibration Out of Date exist, and only one resets

The two alarms are independent, so a recent invalid calibration is expected to clear only the out-of-date alarm while retaining Calibration Error.

What happened here is different: the status snapshot reverted to or retained March, so the dashboard continued deriving the out-of-date alarm from an old date. Acknowledgement then set/preserved `CalibrationState = Okay` while preserving that stale date.

Testing's intuition that the error states interact with the visible symptom was useful, but the code-level failure is the stale shared status-row update rather than invalid calibration semantics.

## Recommended fixes

### Fix 1: remove the full-row `Update` call

This is the primary fix.

Current code:

```csharp
if (!isNew)
{
    _dbSet.Update(rigSummary);
}
```

Remove that block.

The entity was loaded through the current tracked `DbSet`, so EF change detection already knows which properties were assigned. Without `Update`, a sync-only request will update only `LastSyncDate` rather than writing stale calibration fields.

Desired structure:

```csharp
var rigSummary = _dbSet.FirstOrDefault(rs => rs.Rig.Id == rig.Id);

if (rigSummary == null)
{
    rigSummary = new RigStatusSummary
    {
        Rig = rig,
        Site = rig.Site
    };

    _dbSet.Add(rigSummary);
}

if (lastSyncDate.HasValue)
{
    rigSummary.LastSyncDate = lastSyncDate.Value;
}

if (hasCalibrationErrors.HasValue)
{
    rigSummary.CalibrationState = hasCalibrationErrors.Value
        ? CalibrationState.Error
        : CalibrationState.Okay;

    rigSummary.LastCalibrationUpdate = lastCalibrationDate.Value;
}

// Do not call Update for the existing tracked entity.
```

Why this fix matters:

- It prevents unrelated sync writes from including calibration fields.
- It is narrowly scoped and does not change the public API.
- It lets EF produce property-level updates based on actual changes.
- It reduces the lost-update surface without requiring an immediate migration.

### Fix 2: make acknowledgement use the acknowledged calibration's date

Select the latest summary by business date, with ID only as a tie-breaker:

```csharp
var calibrationSummary = _context.RigCalibrationSummaries
    .Where(rcs => rcs.Rig.Id == rigId)
    .OrderByDescending(rcs => rcs.Date)
    .ThenByDescending(rcs => rcs.Id)
    .FirstOrDefault();
```

Then update status with:

```csharp
lastCalibrationDate: calibrationSummary.Date
```

instead of:

```csharp
lastCalibrationDate: rigSummary.LastCalibrationUpdate
```

Why:

- Acknowledging July should make the status snapshot refer to July.
- It provides self-repair if the status date is already stale.
- It uses calibration history as the authority instead of feeding the stale snapshot back into itself.

### Fix 3: explicitly order rebuilt calibration days

Change the daily rebuild query to include an outer chronological order:

```csharp
var newCalibrations = _context.SensorCalibrations
    .Where(x => x.RigId == id && x.DateRecorded.Date >= minDate)
    .GroupBy(c => new { Day = c.DateRecorded.Date })
    .Select(g => g.OrderByDescending(s => s.DateRecorded).First())
    .OrderBy(c => c.DateRecorded)
    .ToList();
```

Why:

- The inner order chooses the final reading within a day.
- The outer order guarantees daily summaries are processed chronologically.
- Delta calculation is then deterministic.
- `summaryData.LastOrDefault()` reliably represents the newest date.

This is a valid hardening fix even though production evidence does not identify it as the LH017 cause.

### Fix 4: consider a `rowversion` concurrency token

As a later hardening change, add a SQL `rowversion` to `RigStatusSummary` and configure it as an EF concurrency token.

Why:

- Overlapping writes to the same snapshot row would produce a detectable `DbUpdateConcurrencyException` rather than silently losing an update.
- It makes future full-row mistakes safer.

This is broader than the primary fix because it requires a migration and a defined retry/conflict policy.

### Fix 5: repair existing production snapshot rows

After deploying the code changes, identify status rows whose date differs from the latest summary date and repair them from the latest `RigCalibrationSummary` ordered by `Date`.

For LH017, the corrected status should be:

```text
LastCalibrationUpdate = 09/07/2026
CalibrationState      = Okay
```

`Okay` is correct because the latest invalid calibration has been acknowledged.

## Required regression tests

### Test 1: sync-only update must not modify calibration fields

```text
Given a status row with a calibration date and state
When UpdateDetails is called with only lastSyncDate
Then only LastSyncDate changes
And calibration date/state remain unchanged
```

### Test 2: overlapping-context lost-update reproduction

Use two relational EF contexts:

```text
Context A loads the status row.
Context B updates LastCalibrationUpdate and CalibrationState, then saves.
Context A changes only LastSyncDate, then saves.

Expected after the fix:
  LastSyncDate comes from A.
  Calibration date/state remain from B.
```

The current code should fail this test because `Update` writes the stale full entity.

### Test 3: invalid-on-invalid advances the date

```text
Existing March calibration is invalid.
Post a July calibration that is also invalid.

Expected:
  LastCalibrationUpdate = July
  CalibrationState = Error
  CalibrationOutOfDate = false
```

### Test 4: acknowledgement repairs the status date

```text
Latest summary date = July
Status snapshot date = March
Ack July

Expected:
  July summary IsAcked = true
  CalibrationState = Okay
  LastCalibrationUpdate = July
```

### Test 5: multi-day rebuild is chronological

Post or seed multiple calibration dates and verify:

- Deltas are calculated from earlier date to later date.
- Latest status date equals maximum summary date.
- Input ordering and SQL execution-plan ordering do not affect the result.

Use a relational provider for these tests. The existing calibration controller tests post one calibration at a time and are skipped under the EF Core 8 in-memory setup, so they do not protect this workflow.

## Diagnostic SQL

Find status rows that disagree with calibration history:

```sql
WITH LatestSummary AS
(
    SELECT RigId, MAX([Date]) AS LatestSummaryDate
    FROM RigCalibrationSummaries
    GROUP BY RigId
),
LatestRaw AS
(
    SELECT RigId, MAX(CONVERT(date, DateRecorded)) AS LatestRawDate
    FROM SensorCalibrations
    GROUP BY RigId
)
SELECT
    s.Name AS SiteName,
    r.Id AS RigId,
    r.Name AS RigName,
    rss.LastCalibrationUpdate,
    rss.CalibrationState,
    ls.LatestSummaryDate,
    lr.LatestRawDate
FROM RigStatusSummaries rss
JOIN Rigs r ON r.Id = rss.RigId
JOIN Sites s ON s.Id = r.SiteId
LEFT JOIN LatestSummary ls ON ls.RigId = r.Id
LEFT JOIN LatestRaw lr ON lr.RigId = r.Id
WHERE rss.LastCalibrationUpdate IS NULL
   OR CONVERT(date, rss.LastCalibrationUpdate) <> ls.LatestSummaryDate
   OR CONVERT(date, rss.LastCalibrationUpdate) <> lr.LatestRawDate
ORDER BY s.Name, r.Name;
```

Inspect one rig's three data layers:

```sql
DECLARE @RigId bigint = 347;

SELECT *
FROM RigStatusSummaries
WHERE RigId = @RigId;

SELECT
    Id,
    [Date],
    IsAcked,
    AzimuthDeltaOutOfRange,
    InclinationDeltaOutOfRange,
    RollDeltaOutOfRange
FROM RigCalibrationSummaries
WHERE RigId = @RigId
ORDER BY [Date] DESC;

SELECT
    Id,
    DateRecorded,
    AzimuthOffset,
    InclinationOffset,
    RollOffset
FROM SensorCalibrations
WHERE RigId = @RigId
ORDER BY DateRecorded DESC;
```

## Final conclusion

The dashboard is displaying exactly what `RigStatusSummary` tells it to display. The dashboard is not the source of the bad date.

The July calibration exists correctly in both raw and calculated history. The defect is that the one-row current-status snapshot remained or reverted to March. The most probable mechanism is a sync-only request performing a full-row EF update from a stale snapshot and silently overwriting the calibration fields written by the July request. Acknowledgement then preserved the stale date instead of repairing it.

The primary fix is to stop calling `Update` on the already tracked existing `RigStatusSummary`. Acknowledgement should use the selected calibration summary's date, and calibration rebuilds should be explicitly chronological. Existing inconsistent production rows must then be repaired from the latest calibration history.

## Related notes

- [[2026-06-02 EF Core 8 Upgrade NullReference in ChangeCalibrationAckState]]
- [[2026-06-25 PostShift SQL Timeout Analysis]]

