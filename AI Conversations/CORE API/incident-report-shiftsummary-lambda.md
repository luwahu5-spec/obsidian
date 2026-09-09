# Incident Report: Production-Drilling Shift Summaries Not Generated (2.23 Environments)

**Status:** Root cause confirmed — fix pending
**Severity:** ⟨Sev 2 — data pipeline outage, no data loss (recoverable by backfill)⟩
**Affected component:** `CalculateSiteSummaryData` Lambda / `DampedDelay-CalculateSiteSummaryData-V2` Step Function
**Affected environments:** Testing, CloudDevelopment (any environment whose Lambda alias resolves to the affected version; Production unaffected)
**Duration:** ~2 weeks (⟨~2026-08-11⟩ → ongoing at time of writing)
**Author:** Allen
**Date:** 2026-08-25

---

## Summary

Shift summaries for **production drilling** shifts stopped being generated on the 2.23 (Testing/CloudDevelopment) environments. Development drilling summaries continued to appear, which initially made this look like a release-2.23 code regression. Investigation showed release 2.23's code is unaffected: the root cause is that **the wrong deployment package — the DailyEmailGenerator Lambda's build — was uploaded to the `CalculateSiteSummaryData` Lambda function** approximately two weeks ago. Every invocation since then has failed instantly with `Runtime.InvalidEntrypoint` before executing any code. Production (2.22) was unaffected only because its Lambda alias is pinned to an older, healthy published version.

## Impact

- No `ShiftSummary` records generated for production-drilling shifts on affected environments for ~2 weeks. Shift History pages for production rigs show missing shifts on those environments.
- Development-drilling summaries were **not** affected: they are calculated inline by the Web API (`DevelopmentController` → `ShiftSummaryService`) and do not depend on the Lambda pipeline.
- The parallel step-function branches (`Calculate Hole Settle`, `Calculate Site Compliance`) ⟨verify: likely also affected for the same executions — confirm and update⟩.
- No source data was lost. Shifts, holes, and reams synced normally; only the derived summary records are missing and can be regenerated.
- DailyEmailGenerator's own function still carries its correct package — the mis-upload was one-directional. ⟨Sanity-check that daily emails have been arriving.⟩

## Root cause

`CalculateSiteSummaryData` is a **custom-runtime** Lambda (`function-runtime: provided`). Custom-runtime functions require an executable named `bootstrap` at the package root; the project produces it via `<AssemblyName>bootstrap</AssemblyName>` and a self-contained publish.

On ⟨~2026-08-11⟩, a deployment package built from **`Minnovare.Core.DailyEmailGenerator.Lambda`** was uploaded to the `CalculateSiteSummaryData` function's `$LATEST`. That package contains no `bootstrap` file, so the function cannot start:

```
{
  "errorType": "Runtime.InvalidEntrypoint",
  "errorMessage": "Error: Couldn't find valid bootstrap(s): [/var/task/bootstrap /opt/bootstrap]"
}
```

Because Lambda aliases map environments to versions (per [Step Function / Lambda build and update](https://minnovare.atlassian.net/wiki/spaces/SOFTWARE/pages/68124673)):

- **Production alias** → pinned to an older published version containing a valid `bootstrap` → 2.22/Production kept working.
- **Testing / CloudDevelopment aliases** → resolve to the bad upload → every summary calculation failed.

Release 2.23's application code was ruled out: the trigger code (`ShiftsController` → `StartExecutionAsync`) and the entire summary-calculation path (`ShiftSummaryService`, repositories) are identical between `release/2.22` and `release/2.23`.

### Evidence

- Package downloaded from `CalculateSiteSummaryData` `$LATEST`: **no `bootstrap`**; entry assembly is `Minnovare.Core.DailyEmailGenerator.Lambda` (per `.deps.json` / `.runtimeconfig.json`).
- File list is **identical** to the DailyEmailGenerator function's package, and MD5 checksums match, e.g. `Minnovare.Core.DailyEmailGenerator.Lambda.dll` = `5d287541dedd1449e0b733c597497dca` in both.
- Package downloaded from the **Production-alias version**: contains `bootstrap` (valid custom-runtime package).
- Step Function execution history: failed executions show green up to the Parallel state, with `GenerateSummary` failing on `Runtime.InvalidEntrypoint`.

## Why it was hard to detect

1. **Silent partial failure.** Development summaries kept working (inline API calculation), masking the pipeline outage and misdirecting suspicion at release 2.23.
2. **"Succeeded" ≠ "did work."** Most Step Function executions end green via the dedup path (`ExecutionIdCompare` → `Done`) without invoking any Lambda; only the winning execution per site per 10-minute window actually calls it. Failures were sparse among thousands of green executions.
3. **Console limitations.** The executions list is browsable only ~2,000 executions (~2 days at current volume) deep, and cannot filter by input fields (e.g. Environment) — `ListExecutions` has no time-range or input filter.
4. **No alerting** on Lambda errors or on step-function execution failures for this state machine.
5. The Lambda update is a **manual step outside TeamCity/Octopus**, so no pipeline log ties the upload to a release.

## Timeline (UTC+8)

- **⟨~2026-08-11⟩** — DailyEmailGenerator package uploaded to `CalculateSiteSummaryData` (`$LATEST`). Production-drilling summaries stop generating on Testing/CloudDevelopment.
- **⟨date⟩** — Missing summaries noticed on 2.23 environment; initially suspected 2.23 CI/CD or code regression.
- **2026-08-25** — Investigation: code diff 2.22↔2.23 ruled out application code; step-function graph analysis identified the dedup/silent-exit paths; two failed executions located with `Runtime.InvalidEntrypoint`; package downloads + checksums confirmed the wrong-artifact upload. Root cause confirmed.
- **⟨date⟩** — Fix deployed (see below).

## Resolution (pending)

1. Rebuild `CalculateSiteSummaryData` from `release/2.23` — first correcting the stale `"framework": "net6.0"` in `aws-lambda-tools-defaults.json` to `net8.0`.
2. Verify the produced zip has `bootstrap` at its root before uploading.
3. Upload to `$LATEST`, **publish a new version** (description: release number), and point the `Testing` alias at it; verify the `CloudDevelopment` / `Staging` aliases while there.
4. Smoke test: sync a production-drilling shift on Testing; confirm the winning execution's `GenerateSummary` goes green and a `ShiftSummary` row appears in `MinnovareTest`.
5. **Backfill:** normal runs only recalculate 7 days back, so start one execution per affected site with `ForceReset: true` (off-peak — it rebuilds that site's summaries from scratch).

## Action items

| # | Action | Owner | Ticket |
|---|--------|-------|--------|
| 1 | Redeploy correct `CalculateSiteSummaryData` package; repoint Testing alias | ⟨⟩ | ⟨⟩ |
| 2 | Backfill affected sites with `ForceReset: true` | ⟨⟩ | ⟨⟩ |
| 3 | Automate Lambda build/publish/alias update in TeamCity/Octopus (remove the manual upload step) | ⟨⟩ | ⟨⟩ |
| 4 | Add CloudWatch alarms: Lambda `Errors` metric on `CalculateSiteSummaryData` (all aliases) + Step Function `ExecutionsFailed` | ⟨⟩ | ⟨⟩ |
| 5 | Set execution `Name` in `StartExecutionAsync` (e.g. `{Environment}-{SiteId}-{timestamp}`) so executions are searchable in the console | ⟨⟩ | ⟨⟩ |
| 6 | Fix `aws-lambda-tools-defaults.json` (`framework` → `net8.0`) in all Lambda projects | ⟨⟩ | ⟨⟩ |
| 7 | Update the [Step Function / Lambda build and update] wiki page with a package-verification step ("confirm `bootstrap` exists in the zip before upload") | ⟨⟩ | ⟨⟩ |
| 8 | Confirm DailyEmailGenerator and the Hole Settle / Compliance lambdas were not also mis-deployed | ⟨⟩ | ⟨⟩ |

## Lessons learned

- **The only deploy step outside the pipeline was the one that failed.** TeamCity/Octopus cover the Web API; the Lambda is uploaded by hand, and a hand can pick the wrong zip. Automation (action 3) removes the whole failure class.
- A subsystem that "half works" is more misleading than one that fails outright — the dev/production split here sent the investigation toward release code first.
- Green step-function executions need interpretation: in this design, most successes are deliberate no-ops.
