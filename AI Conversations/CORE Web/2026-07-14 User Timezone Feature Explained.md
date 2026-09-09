---
date: 2026-07-14
updated: 2026-07-14
source: Claude Code
project: core-web
tags: [domain-knowledge, timezone, user-profile, UserTimeService, date-display]
---

# User Timezone — what the Edit User timezone actually does

> **Takeaway:** The timezone picked in Edit User is purely a **display preference for timestamps**. It feeds one service — `UserTimeService` — which ~40 pages use to convert UTC values into "the viewer's clock" before rendering. It never changes what data is loaded. It coexists with the **site timezone**, which governs operational boundaries (shift start/end, date-range filters, breadcrumb site clock). Mental model: *site timezone = when things happened at the mine; user timezone = what my watch says.* If the user has no timezone set, the app falls back to their first associated site's timezone, then to W. Australia (+8).

## The business/physical reality being modelled

A mine site in Western Australia runs shifts on Perth time, but the engineer reviewing the data may sit in Canada. Two different questions need two different clocks:
- "Which shift does this record belong to?" → **site** timezone (operational truth).
- "When did this happen, in terms I can read?" → **user** timezone (viewer convenience).

The Edit User timezone answers only the second question.

## The configuration chain

1. **UI**: Edit User page renders a `<select>` over `TimeZoneInfo.GetSystemTimeZones()` bound to `InputUser.TimeZone` — `Pages/Blazor/Users/EditUser.razor:45-47`.
2. **Save**: the value is put into the update payload (`EditUser.razor.cs:187`) and sent to the API as a plain string attribute `"TimeZone"` (`Repositories/UserRepository.cs:107`). Storage lives backend-side (Keycloak user attribute via CORE API).
3. **Model**: `User.TimeZone` (string) lazily materialises `User.TimeZoneInstance` via `TZConvert.GetTimeZoneInfo` — `Minnovare.Core.Shared/Models/User.cs:76-114`. On parse failure it silently falls back to `TimeZoneInfo.Local` (the **server's** zone) — `User.cs:103`.
4. **Auto-default on invite**: when a user is added to a site, their timezone is initialised to that site's timezone — `Pages/Blazor/Sites/AddUser.razor.cs:173` (`user.TimeZone = _siteTz`).

## Where it takes effect — `UserTimeService`

`Shared/UserTimeService.cs` (registered **scoped** at `Startup.cs:107`) is the single consumer of the user's timezone. Resolution order in `GetUserTimezoneInfoAsync()` (`UserTimeService.cs:104-145`):

1. Default seed: `"W. Australia Standard Time"` (+8) — line 112.
2. Fetch the logged-in user; if `user.TimeZoneInstance != null` use it — lines 119-122.
3. Else fall back to the user's **first associated site's** timezone (SiteId claim) — lines 127-133.

The zone is cached per circuit (`_userTimeZone` field), so **changing it in Edit User only shows after a new circuit/page session**. It exposes:
- `ConvertToUserTimezoneDisplay(...)` / `ConvertToUserTimezoneAsync(...)` — UTC → user clock for rendering.
- `ConvertFromUserTimezone(...)` / `GetTimeWithUserOffset(...)` — user clock → UTC for the few inputs typed "in my time".
- `UserTimeZoneDisplay` — the "(UTC+08:00)" chip shown next to times (`UserTimeService.cs:44-60`).

## How it feeds the UI (~40 consumers)

Any timestamp rendered through `UserTimeService` moves when the Edit User timezone changes. Representative examples:

| View | What shifts | Evidence |
|---|---|---|
| **MetaDataTable** (new generic table) | every `DateTime`/`DateTimeOffset` cell | `Pages/Blazor/Shared/TableData/MetaDataTable.razor.cs:121-122` |
| Drive details | Date Created / Date Drilled | `Shared/DriveDetailsComponent.razor:186,189` |
| Shift details | target action times + "(UTC+08:00)" suffix | `Pages/Blazor/Rigs/ShiftDetails.razor:579` |
| System Status Dashboard | passes the user TZ id to the chart JS | `RigsDashboard/SystemStatusDashboard.razor.cs:171` |
| Productivity report | report expiry date | `Reports/ProductivityReport/ProductivityReportRequestPage.razor.cs:108` |

Plus compliance views, MWD reports, driller archive/merge pages, calibration logs, etc. (42 files reference `UserTimeService`).

**What it does NOT affect** — these use the **site** timezone instead:
- Shift-list date-range filters sent to the API: `Pages/Blazor/Rigs/ShiftList.razor.cs:198-199,290-291` convert with `_siteTimeZone` (from `site.TimeZoneInstance`, line 120).
- The breadcrumb header's site clock (`SiteTimeZone` attribute passed into `BreadcrumbHeader` from ~15 pages, e.g. `Rigs/RigDetails.razor.cs:93`).
- The new Shift History table request carries `SiteTimeZone`, not user TZ — `TableData/ShiftHistoryTableDataRequest.cs:16`.

So the same Shift History page mixes both on purpose: which shifts fall in the period = site clock; how each timestamp is printed in a cell = user clock (via MetaDataTable).

## Design subtleties & edge cases

- **Worked example** (what a change looks like): a drive drilled at 2026-07-14 02:00 UTC renders as 10:00 AM under Perth (+8), 12:00 PM under Brisbane (+10), 2:00 AM under UTC — same record, different wall clock (`DriveDetailsComponent.razor:189`). Switching between two same-offset zones (Perth → Singapore) changes nothing visible except the display-name chip.
- **Stale cache**: `_userTimeZone` is cached per scoped service (`UserTimeService.cs:106` only fetches when null); a timezone change in Edit User does not repaint already-open circuits — leave and re-enter the page (or refresh) to see it.
- **Silent fallback to server zone**: an unparseable stored timezone string yields `TimeZoneInfo.Local` (`User.cs:103`) — on AWS that's the container's zone, not the user's; hard to spot.
- **DST ignored in offsets**: `GetTimeWithUserOffset` uses `BaseUtcOffset` (`UserTimeService.cs:94,101`), and `TimeService.TimeZoneOffset` does too (`Shared/Minnovare.Core.Shared/Services/TimeService.cs:34`) — the "(UTC+xx)" label can be off by 1h during DST in DST-observing zones.
- **Windows vs IANA ids**: the Edit User dropdown offers `TimeZoneInfo.GetSystemTimeZones()` of the *web server* (Windows ids locally, IANA on Linux containers); `TZConvert` papers over the difference at read time.
- **Hard default is Perth**: three separate places default to +8 (UserTimeService line 112, `GetTimeWithUserOffset` lines 94/101, `Site.TimeZone` default at `Site.cs:202`).

## Questions asked
- 2026-07-14 — "In Edit User we can change the timezone but I don't know what it really does — does it affect any view?" → Yes: it drives `UserTimeService`, which formats timestamps on ~40 pages (including the new MetaDataTable); it does NOT affect site-timezone logic like shift filters or the breadcrumb site clock. See "How it feeds the UI".
- 2026-07-14 — "If I change my timezone, will I actually see any page look different?" → Yes, every rendered timestamp shifts (02:00 UTC → 10 AM Perth / 12 PM Brisbane), but only after re-entering the page (circuit cache), and only if the new zone has a different offset. See the worked example in "Design subtleties".

## Related
- [[2026-07-09 Generic Metadata Tables Feature Explained]] — MetaDataTable renders date cells through user timezone
- [[2026-07-10 MC-1833 Shift History Group Columns Executable Plan]] — Shift History request uses SiteTimeZone
- [[Knowledge Priorities and Correlations]]
