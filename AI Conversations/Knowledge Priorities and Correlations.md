---
tags: [meta, priorities, correlations]
updated: 2026-07-06
---

# Knowledge Priorities & Correlations

Which knowledge types pay back the most, ranked. Each point links to the notes that prove it.

**Curation rules (2026-07-06 cleanup):** the vault keeps only notes that pass the *"act differently" test* — they must change how a future, different problem gets approached. Action-execution knowledge (git merging/stash/backport mechanics, PR splitting, running migrators, setup steps) is excluded on principle: it is re-derivable on demand. Seven notes were deleted under these rules; new notes are written takeaway-first by `/save-knowledge`.

## Priority ranking (most → least valuable)

### P1 — Production incidents where local reproduction fails
The rarest and most expensive knowledge: bugs that depend on **accumulated state**, not payload shape. You cannot re-derive these later; the insight only exists because you lived through it.
- [[2026-06-29 EF Core Hole.Id Key Error 500 in Prod]] — prod 500 caused by DB corruption from *prior* failed syncs; a clean local DB can never show it ⭐ the single most valuable note in the vault. Also carries the shipping rule: hotfix from the release branch PROD actually runs.
- [[2026-06-25 PostShift SQL Timeout Analysis]] — why timeouts look "random" and why disappearing ≠ fixed
- [[2026-06-18 Core Sync DB Connection Pool Findings]] — systemic multipliers (pool size, async void, manual contexts)

### P2 — Auth/infrastructure configuration debugging
High value because the failure is **outside the code you can read** — Keycloak realms, tokens, AWS. Symptoms are generic (401/403) and the diagnosis path is expensive to rediscover.
- [[2026-05-04 Keycloak SSO and Audience Mapper]] — the `aud: api` fix; audience-mapper misconfiguration
- [[2026-04-29 Keycloak GetUsersForSite 403 Debugging]] — realm-specific admin permissions; the "fails for every site, cloud-only, immediately" ⇒ config-not-data diagnostic pivot
- [[2026-05-05 How Authorization Flows from Core Web to Core API]] — the 401 debugging playbook + `/Identity` endpoint

### P3 — Framework/version behavior changes (upgrade traps)
Bugs where *your code didn't change but behavior did*. These repeat every upgrade and the patterns transfer across the codebase.
- [[2026-06-02 EF Core 8 Upgrade NullReference in ChangeCalibrationAckState]] — materialize-then-navigate pattern
- [[2026-06-25 Upload Error Popup and CSV Import Mismatch]] — Bootstrap 4 attribute silently dead in Bootstrap 5
- [[2026-04-28 Assign Rings Ctrl-A Multi-Select Debugging]] — CSS `appearance` changing native element behavior
- [[2026-06-24 bUnit Failures After BaseComponent Refactor]] — bUnit must satisfy every `[Inject]` in the inheritance chain
- [[2026-06-24 MiningMethod Merge Missing Migration]] — EF model change and its migration must ship together; any `ToList()` on the entity fails otherwise

### P4 — Hidden contracts & defaults (the "why is data missing" family)
API defaults and gatekeeper logic that silently filter data. Cheap to write down, painful to rediscover.
- [[2026-05-25 Sites API siteType Default and Drillers 403 Analysis]] — `siteType=Production` default
- [[2026-05-19 System Status Dashboard Missing Sites Debugging]] — two-gatekeeper rendering
- [[2026-07-02 Invalid CSV Layout Root Cause]] — exact header match against 6 hardcoded layouts
- [[2026-06-19 PreStartData Payload and Dynamic Form IDs]] — IDs must come from the source-of-truth endpoint
- [[2026-06-26 Outstanding Length Excludes Recalculated Holes]] — domain rule enforced at model level
- [[2026-05-21 Drill Plan Upload NRE and Duplicate Ring Debugging]] — POST DrillPlan is *full-state replacement*, and the "Error processing Hole" log line belongs to PUT DrillerDrillPlan, not POST

### P5 — Architecture & planning decisions (with the *why*)
Valuable mainly when the reasoning is recorded — the decision itself is visible in code, the rejected alternatives are not.
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]] — the biggest structural decision in the vault: adapter layer, config-driven pages, "frontend never knows method names" ⭐
- [[2026-03-27 Deswik K92 Shift Integration API Design]] / [[2026-04-23 Deswik Shift Endpoint Implementation]] — DTO consolidation reasoning, "Deswik computes nothing" principle
- [[2026-05-19 Site Feature Implementation Order Plan]] — order features by data dependency, not visibility
- [[2026-06-05 k6 Load Testing Setup for Core Sync]] / [[2026-06-16 Playwright Web Client Load Testing]] — replicate-before-fix discipline, incident-replay pattern, real-OIDC-login requirement

### P6 — Reusable techniques & reference data
Methods and operational reference still in active use. Kept because they answer "how do I do this next time" directly.
- [[2026-05-05 How Authorization Flows from Core Web to Core API]] — doubles as the 401 playbook
- [[2026-07-02 CORE API Unavailable Error Causes]] — SocketException/port checklist
- [[2026-06-12 When the Blazor Error Banner Appears]] — banner taxonomy + the 5 crash causes and their fix family
- [[2026-04-13 Park Log SQL Detective Queries]] — find UI test cases by querying from the data that produces the feature
- [[2026-04-22 Unit Tests Secretly Using Real SQL Server]] — TestDatabaseFixture reality + "in-memory EF is still DB coupling"
- [[2026-06-12 Core Sync Dataset Rules]] — load-test dataset validity rules (operational reference)

### Deliberately NOT kept
- Action-execution records: git merging/stash/backport/cherry-pick mechanics, PR splitting, migration-runner steps, environment setup. Re-derivable; visible in git history.
- Single-fact regressions with obvious causes once found (commented-out markup, stale redirect routes).
- UI/style task execution and routine implementation chats.

## Correlation map (which questions belong together)

**The core-sync saga** — one prod issue spawning many sessions over two months:
[[2026-06-05 k6 Load Testing Setup for Core Sync]] → [[2026-06-12 Core Sync Dataset Rules]] → [[2026-06-25 PostShift SQL Timeout Analysis]] + [[2026-06-18 Core Sync DB Connection Pool Findings]] → [[2026-06-29 EF Core Hole.Id Key Error 500 in Prod]] (root cause + shipping rule).
*Pattern: replicate → measure → root-cause → fix → ship. The load-testing questions and the EF bug questions were the same investigation.*

**The auth cluster** — every 401/403 traces to one mental model (token claims vs API expectations):
[[2026-05-05 How Authorization Flows from Core Web to Core API]] is the hub; [[2026-05-04 Keycloak SSO and Audience Mapper]], [[2026-04-29 Keycloak GetUsersForSite 403 Debugging]], and the 403 half of [[2026-05-25 Sites API siteType Default and Drillers 403 Analysis]] are instances.

**The drill-plan/EF cluster** — all rooted in POST DrillPlan being *full-state replacement* with self-referencing hole FKs:
[[2026-05-21 Drill Plan Upload NRE and Duplicate Ring Debugging]] ↔ [[2026-06-29 EF Core Hole.Id Key Error 500 in Prod]] ↔ [[2026-06-26 Outstanding Length Excludes Recalculated Holes]] (same RecalculatedHole concept, benign side).

**The "missing data" family** — one lesson, three costumes: *diff the exact API call/params/format the failing consumer uses*:
[[2026-05-19 System Status Dashboard Missing Sites Debugging]] ↔ [[2026-05-25 Sites API siteType Default and Drillers 403 Analysis]] ↔ [[2026-07-02 Invalid CSV Layout Root Cause]] ↔ export/import mismatch in [[2026-06-25 Upload Error Popup and CSV Import Mismatch]].

**The upgrade/refactor-trap family** — code unchanged, behavior changed:
[[2026-06-02 EF Core 8 Upgrade NullReference in ChangeCalibrationAckState]] ↔ [[2026-06-25 Upload Error Popup and CSV Import Mismatch]] ↔ [[2026-04-28 Assign Rings Ctrl-A Multi-Select Debugging]] ↔ [[2026-06-24 bUnit Failures After BaseComponent Refactor]].

**The core-refactor 2.23 thread** — one architecture decision rippling across both repos and the tooling:
[[2026-05-19 Core Refactor 2.23 Architecture Plan]] (the design) → [[2026-05-19 Site Feature Implementation Order Plan]] (sequencing) → [[2026-06-24 MiningMethod Merge Missing Migration]] (backend enum + migration) → [[2026-06-16 Playwright Web Client Load Testing]] (v2.23 test scripts).
