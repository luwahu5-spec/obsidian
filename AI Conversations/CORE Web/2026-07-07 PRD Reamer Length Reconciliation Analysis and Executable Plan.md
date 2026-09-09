---
date: 2026-07-07
source: Claude Code
project: core-web / core-web-api (Minnovare.Core.Shared submodule)
tags: [planning, prd, reamer-hierarchy, ream-passes, completed-length, outstanding-length, drill-length]
---

# PRD Reamer Length Reconciliation — Code Gap Analysis & Executable Plan

> Analyzed against: PRD "Reamer Length Reconciliation" (DC, Jul 3 2026) and current code in the `Minnovare.Core.Shared` submodule (shared by core-web and core-web-api). Related prior finding: [[2026-06-26 Outstanding Length Excludes Recalculated Holes]] — same calculation chain.
> Background on the feature itself (what Reamer Hierarchy does, config chain, where Ream rows come from): [[2026-07-07 Reamer Hierarchy Feature Explained]].

---

## Page 1 — Root cause in code (why length is overstated)

The entire Completed/Outstanding chain is **computed properties on the shared models** — nothing is persisted, nothing is method-specific per page:

```
Hole.CompletedLength / Hole.TargetTotalLength      (Hole.cs:229 / :265)
  → Ring sums holes                                 (Ring.cs:99 / :112)
    → Drive sums rings                              (Drive.cs:58 / :71)
      → Drive.Outstanding() = Target − Completed    (Drive.cs:100-104)
        → every page renders these                  (DriveDetailsComponent, DriveDetails, RigDetails, RigArchive…)
```

The overstatement comes from exactly **two lines** in `Hole.cs`:

1. **`TargetTotalLength` (Hole.cs:280):** `TargetLength * (Reams.Count() + 1)` — every *generated* ream pass counts toward planned length forever, drilled or not.
2. **`CompletedLength` Done-branch (Hole.cs:245-248):** `if (DrillStatus == DrillStatus.Done) return TargetTotalLength;` — the moment the hole is Done, **all** generated passes are reported as drilled, including skipped ones.

So for a hole with 3 configured passes where the customer skips the middle one: while drilling it reports 3-pass planned length (PRD Req 1 says keep this), and once Done it reports 3 passes completed instead of 2 (PRD Reqs 2-4 say fix this).

Two facts that make this cheap to fix:
- **Skipped passes are already detectable:** a drilled ream has `ActualLength > 0 && DateDrilled.HasValue` (the exact predicate the in-progress branch already uses at Hole.cs:251); a skipped ream has neither.
- **One change propagates everywhere by construction:** Ring, Drive, Plan Summary (active + archived), Drive Page, Rig page, even legacy SitesOld pages all derive from these two properties. The PRD's "consistent across all supported pages" NFR is satisfied automatically — same mechanism as the recalculated-holes exclusion documented on 2026-06-26.

---

## Page 2 — The design (pure computed-property change, no schema, no sync, no PRODOP change)

**Reconciliation trigger:** `DrillStatus == DrillStatus.Done` (the tablet sets Done when the final ream is completed — this is the same trigger the current Done-branch uses, so no new signal is needed).

**New formulas, applied only in the Done state:**

- `CompletedLength (Done)` = `TargetLength × (1 + count of Reams where ActualLength > 0 && DateDrilled.HasValue)` — the hole's own pass plus only the passes actually drilled. *(Reqs 2, 3)*
- `TargetTotalLength (Done)` = same value — once reconciled, skipped passes leave the plan, so `Outstanding = Target − Completed = 0` for that hole and `CompletedPercent` correctly reaches 100%. *(Req 4)*

**Explicitly unchanged (Req 1 + NFR 3, 4):**
- In-progress behaviour: `TargetTotalLength` keeps counting all generated passes and the in-progress `CompletedLength` branch (Hole.cs:251) is untouched — combined planned length is reported "until the final ream has been completed", exactly as the PRD demands.
- `CreateReams` / Reamer Hierarchy generation (Hole.cs:284-331): untouched → hierarchy config stays backwards compatible.
- No DB columns, no migration, no tablet/sync change, no Production Optimiser workflow change — reconciliation is a display-time calculation, not a data mutation.

**Why not the alternatives:**
- *Persisted reconciliation (delete/flag skipped Ream rows when Done arrives):* mutates historical data, needs migration + backfill job + undo story if a "skipped" ream is later drilled. All cost, no benefit — the computed approach yields identical numbers.
- *Per-page fixes:* guarantees the inconsistency the NFRs forbid. The 2026-06-26 note already proved model-level is where this family of rules lives.

**One consequence to surface to product (not a blocker):** because the values are computed, the fix applies **retroactively to all existing Done holes** the moment it deploys — Plan Summary, archived drives, and historical Drive pages will restate. That is literally the PRD's objective (archived drives are in scope), but customer-facing numbers will visibly drop after the release; support/release notes should say so.

---

## Page 2.5 — Per-site opt-in (added 2026-07-08 — reconciliation will NOT apply to all sites)

**Reframing fact:** for sites that drill every pass, the reconciled formula returns identical numbers (all passes have `ActualLength > 0`) — the math never needed gating. The gate exists for two *non-mathematical* risks: (1) **historical data quality** — a site whose reams were drilled but never recorded (old app versions, sync failures) is indistinguishable in data from one that skipped passes; enabling reconciliation there would *incorrectly* shrink their lengths, and only the customer knows which case they are; (2) **controlled restatement** — default-off makes enabling the flag the deliberate, per-customer communication event, killing the retroactive-restatement concern (old Q2).

**Design — default formula vs opt-in (the gate axis is the SITE, not the drilling method):**
- Default = current behaviour (all generated passes count). **All sites default OFF at deployment — deploy changes nothing anywhere.**
- **Storage (revised 2026-07-08 — this ships POST-refactor, so do NOT add a new `FeatureType`):** `SiteFeatures` is the legacy mechanism the refactor is shrinking (it still marks dev/pro site types today); planting a new flag there prolongs it. Best practice: a `bool ReamReconciliationEnabled` **column on `SiteMiningMethodConfiguration`** (the per-site-per-method config table §19/Interpretation B creates) — meaningful on the Production row, default false, normal column-add migration. Semantically exact ("this site's production drilling reports reconciled lengths"), the Production tab renders it for free from the config object, and zero new `site.HasFeature(...)` call sites are created. Scenario served: 30 sites all run Production; only the one/few whose customers deliberately skip passes get the flag.
- Contingency: if reconciliation is ever pulled FORWARD of the refactor's site work, fall back to a `FeatureType.ReamReconciliation` SiteFeatures flag as a labelled temporary bridge, migrated into the config column when S-API-2 lands. Don't build the bridge speculatively.
- Drilling method needs NO configuration role: reams only exist on production holes (`CreateReams` only fires for ream-hierarchy holes; headings have none), so a hole with zero reams computes identically under either formula — "production only" is enforced by the data shape automatically, even on a flagged site.
- UI placement: the checkbox goes on the **Production tab** of Site Create/Edit (next to MWD/ChargePlan, `SiteCreateEdit.razor:204`) — it only means something for production drilling.
- The §22/refactor note below is compatibility only — this feature neither waits for nor contributes to the drilling-method refactor.

**How the flag reaches the calculation (backend contribution #1):**
- `Hole` gets a SETTABLE `bool ApplyReamReconciliation` (settable → serializes across the wire, unlike the computed props).
- The Done branch becomes: `ApplyReamReconciliation ? reconciledFormula : TargetTotalLength` (same for `TargetTotalLength`'s Done reconciliation).
- **Backend stamps the flag at hole-hydration in ONE place** (`ApplyReamReconciliation = productionConfig.ReamReconciliationEnabled` from the site's Production `SiteMiningMethodConfiguration` row); it rides the JSON, the web app deserializes holes already carrying the decision and recomputes identically. Source of truth is server-side; frontend displays — refactor principle honoured within the shared-model architecture.
- Server-side consumers (CSV export, lambdas, summaries) loading holes through the stamped path apply the rule automatically — one hydration point instead of per-consumer checks (backend contribution #2).

**Phase 2 alignment:** when the §22 table-data provider makes lengths backend-computed columns, the flag check moves fully server-side and the stamped property retires; `FeatureType.ReamReconciliation` and the per-site semantics carry over unchanged. Nothing here is throwaway.

## Page 3 — Tickets

**R-0 — Per-site opt-in flag (NEW, revised)** — *Shared + API + Web*
- `ReamReconciliationEnabled` column on `SiteMiningMethodConfiguration` (Production row; default false; column-add migration — depends on §19 S-API-2 having landed).
- Site edit Production tab checkbox rendered from the config object (+ label × 7 resx); hole-hydration stamping helper in the API reads the config row.

**R-1 — `Hole.cs` reconciliation (the core change)** — *Shared submodule*
- Change the Done branch of `CompletedLength` (Hole.cs:245-248) to count only drilled passes.
- Change `TargetTotalLength` (Hole.cs:265-282) to return the reconciled value when `DrillStatus == Done`.
- Extract the "drilled ream" predicate (`ActualLength > 0 && DateDrilled.HasValue`) into one place — it's now used by three branches and must not drift.
- Keep the recalculated-hole early-returns (`RecalculatedHoleId.HasValue → 0`) ahead of the new logic — ordering matters, both rules must compose.

**R-2 — Unit tests** — *Shared submodule (`HoleTests`, `DriveTests`)*
- Test matrix now doubles: every reconciliation case × flag ON/OFF (`ApplyReamReconciliation`). Flag OFF must reproduce today's numbers exactly — existing assertions (e.g. HoleTests line 95: 36 for a 3-pass Done hole) stay valid for the OFF state. Add for ON:
  - Done, all passes drilled → unchanged (36).
  - Done, middle pass skipped → Completed = 24, Target = 24, Outstanding = 0.
  - Done, no reams configured → unchanged.
  - In progress, middle pass skipped → unchanged from today (Req 1).
  - Done + recalculated → still 0.
  - Ring/Drive aggregation with a mix of reconciled and in-progress holes.

**R-3 — Shared version bump + dual submodule update**
- The fix lives in the `Minnovare.Core.Shared` **git submodule consumed by both repos**. Ship: Shared bump (2.23 → next) → update submodule pointer in `core-web-api` and `core-web` in the same release. If only one repo updates, API-serialized values and web-recomputed values disagree — the exact inconsistency NFR 2 forbids.

**R-4 — Consumer verification pass (no code expected)**
- Verify rendering on: Plan Summary Active Drives (`DriveDetailsComponent`), Plan Summary Archived (`RigArchive`/`ArchivedDrives`), Drive Page (`DriveDetails`), Rig page Outstanding (`RigDetails` — *not* named in the PRD's scope list but uses the same properties and will change too; tell product, it's the consistency they asked for), legacy `SitesOld` pages.
- Verify the summary Lambdas (`CalculateSiteSummaryData`, `ProductivityReportProcessor`) and `ShiftSummary` don't independently re-derive planned length from ream counts — if any do, they get the same predicate or are confirmed out of scope (shift reports measure *actual* drilled metres, which never included skipped passes).

**R-5 — Test-case SQL + manual verification recipe**
- Seed a Done hole with a skipped intermediate ream in the dev DB (ream row with `ActualLength = 0`, `DateDrilled = NULL`, siblings drilled) and confirm all three pages show the reconciled numbers. Existing sql-testdata skill covers the seeding pattern.

Sizing: R-1/R-2 are a day-scale change; R-3 is process; R-4/R-5 are the real time cost (verification breadth). No migration, no localization changes (numbers only, no labels).

---

## Page 4 — Edge cases & questions for product

**Q1 — Is `DrillStatus = Done` reliably set when the final ream is completed with skipped intermediates?** The design keys reconciliation on Done (same trigger as today's overstating branch, so almost certainly yes — the PRD's complaint only exists because Done holes report full length). Confirm with the tablet team that a skipped intermediate pass cannot block the Done transition. If there's any path where the final ream is drilled but the hole never reaches Done, reconciliation would need a fallback trigger (largest-diameter ream drilled) — cheap to add, but only if the case is real.

**Q2 — Retroactive restatement — RESOLVED by the per-site flag (2026-07-08).** With reconciliation default-OFF, deployment changes no numbers anywhere; enabling `FeatureType.ReamReconciliation` per site is the deliberate restatement event, done with the customer informed. Remaining product task: define the enablement checklist for Support (confirm with the customer that unrecorded-but-drilled reams aren't a factor at that site before flipping the flag — that's the one case the data cannot distinguish from genuine skipping).

**Q3 — A ream drilled with `ActualLength = 0`.** The drilled-pass predicate requires `ActualLength > 0`; a pass the tablet recorded with a date but zero length counts as skipped. This matches the in-progress branch's existing interpretation — flagging it so the behaviour is a decision, not an accident.

**Q4 — Skipped FINAL ream.** If the customer never completes the final ream, the hole stays in-progress and keeps reporting full planned length forever (Req 1 says exactly this). Confirm that's accepted — i.e., reconciliation intentionally never fires for abandoned holes; archiving the drive is the existing mechanism for taking them out of reporting.

**Q5 — Scope ride-alongs.** The Rig page and legacy SitesOld pages use the same properties and will reconcile too, though the PRD scope list names only Plan Summary (active/archived) + Drive Page. This is the consistency NFR working as intended — listed here so nobody is surprised in UAT.

---

*Each "Page N" section is sized to paste as one Loop page.*
