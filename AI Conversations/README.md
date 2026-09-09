---
tags: [index, moc]
updated: 2026-07-20
---

# AI Conversations — Knowledge Base Index

Distilled from Claude Code + Codex (VS Code) conversation history (Mar–Jul 2026). Kept only if it passes the **"act differently" test**: debugging investigations and planning/architecture sessions with transferable takeaways. Routine implementation chats and action-execution records (git mechanics, setup steps) are excluded on purpose.

➡️ **Start here:** [[Knowledge Priorities and Correlations]] — which knowledge types matter most, and how the notes connect.
🧰 **Working with Claude:** [[Claude Workflow Playbook]] — skills, prompt templates, model strategy, methodology rules.
🛠️ Maintain: `/save-knowledge` distills new sessions into this vault (takeaway-first format); `/cross-review` makes Claude and Codex audit each other.

## 🔥 The PROD core-sync saga (one story across many sessions)
1. [[2026-06-05 k6 Load Testing Setup for Core Sync]] — replicate-before-fix discipline, k6 setup, SyncId root-cause lead
2. [[2026-06-12 Core Sync Dataset Rules]] — payload dataset construction/validation rules
3. [[2026-06-25 PostShift SQL Timeout Analysis]] — reading EF logs, insert vs update, why timeouts look random
4. [[2026-06-18 Core Sync DB Connection Pool Findings]] — pool size, async void, manual DbContexts
5. [[2026-06-29 EF Core Hole.Id Key Error 500 in Prod]] — the actual bug, why it wouldn't reproduce locally, and the hotfix shipping rule ⭐

## CORE API — debugging
- [[2026-08-04 Duplicate Rig Status Summaries Break Site Detail Page]] — site rig cards call the single-status endpoint; duplicate rows throw `SingleOrDefaultAsync`, while the schema does not enforce one row per rig
- [[2026-07-27 Stale Calibration Date and Rig Status Lost Update]] — raw/history data reached July while a full-row status update probably restored March; acknowledgement preserved the stale snapshot
- [[2026-05-21 Drill Plan Upload NRE and Duplicate Ring Debugging]] — POST DrillPlan is full-state replacement; duplicate ring names
- [[2026-06-02 EF Core 8 Upgrade NullReference in ChangeCalibrationAckState]] — materialize-then-navigate pattern breaks on EF 8
- [[2026-04-29 Keycloak GetUsersForSite 403 Debugging]] — realm-specific admin permissions; AdminId is a role id
- [[2026-05-05 How Authorization Flows from Core Web to Core API]] — bearer-only forwarding; /Identity endpoint; 401 playbook
- [[2026-05-25 Sites API siteType Default and Drillers 403 Analysis]] — `siteType=Production` default; admin bypass in SiteAuthorizationHandler
- [[2026-06-19 PreStartData Payload and Dynamic Form IDs]] — dynamic-form IDs must come from GET /DynamicForms/GetByRig
- [[2026-07-02 Invalid CSV Layout Root Cause]] — exact header match against 6 hardcoded layouts
- [[2026-04-22 Unit Tests Secretly Using Real SQL Server]] — TestDatabaseFixture; DB-coupling concept; IgnoreOnLinuxFact
- [[2026-06-24 MiningMethod Merge Missing Migration]] — model/migration must ship together
- [[2026-04-13 Park Log SQL Detective Queries]] — finding UI test cases from the data side

## CORE API — feature knowledge (living docs)
- [[2026-09-03 Sync After Login Endpoints Feature Explained]] — the tablet login burst: what each of the ~10 endpoints returns; siteType is ignored on /sites/{id}, showX=2 means ShowAll, and two "reads" actually write
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]] — Keycloak OIDC chain end-to-end; dual-store user management is THE constraint for SSO; verified Google + Entra ID brokering recipe (link-or-reject flow) ⭐

## CORE API — planning / design
- [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] — six dark-shippable backend tickets (schema → KcClient → creation/G3 → G4 → audit → conversions) + parallel infra track ⭐
- [[2026-03-27 Deswik K92 Shift Integration API Design]] — requirements, endpoint reuse, "Deswik computes nothing"
- [[2026-04-23 Deswik Shift Endpoint Implementation]] — DTO consolidation, placement rules, validation tests

## CORE Web — debugging
- [[2026-05-04 Keycloak SSO and Audience Mapper]] — enterprise SSO brokering; the `aud: api` fix ⭐
- [[2026-06-12 When the Blazor Error Banner Appears]] — blazor-error-ui vs reconnect modal; 5 crash causes
- [[2026-05-19 System Status Dashboard Missing Sites Debugging]] — two gatekeepers; inconsistent API params
- [[2026-04-28 Assign Rings Ctrl-A Multi-Select Debugging]] — `appearance: base-select` kills native multi-select
- [[2026-06-24 bUnit Failures After BaseComponent Refactor]] — bUnit must satisfy every [Inject]
- [[2026-06-25 Upload Error Popup and CSV Import Mismatch]] — Bootstrap 4 `data-dismiss` no-op in BS5; export/import format mismatch
- [[2026-07-02 CORE API Unavailable Error Causes]] — SocketException/port checklist
- [[2026-06-26 Outstanding Length Excludes Recalculated Holes]] — domain fact at Hole model level

## CORE Web — planning
- [[2026-08-12 Plan Data Metadata Table Refactor Executable Plan]] — unified Plan Data route, mixed identifier contract, reusable table selection, status/comment metadata, and legacy detail bridge
- [[2026-07-22 Mining Method Architecture Decision Points]] — explicit Site-to-MiningMethod binding, administrator bypass, Cut and Fill, API ownership, and migration decisions
- [[2026-05-19 Core Refactor 2.23 Architecture Plan]] — one site page, method tabs, adapter layer, config-driven uploads ⭐
- [[2026-05-19 Site Feature Implementation Order Plan]] — order by data dependency (Users first)
- [[2026-07-07 PRD Front End 15-22 Analysis and Executable Plan]] — Users/Site/Rig/Settings/General epics; decisions: per-method config independent (B), config stays site-level, user methods global ⭐
- [[2026-07-07 PRD Reamer Length Reconciliation Analysis and Executable Plan]] — two-line fix in shared Hole model; ships via submodule bump to BOTH repos
- [[2026-07-08 MC-1825 Assign Plans Requirements Organized and Execution Plan]] — one spec ×3 methods; hidden multi-rig bulk semantics; build on refactor architecture or pay twice
- [[2026-07-08 PRD AI Assistant in CORE Web Analysis and Plan]] — "frontend-only" contradicted by the PRD's own NFRs; backend proxy pattern; session/memory/history ownership table
- [[2026-07-10 MC-1833 Shift History Group Columns Executable Plan]] — one true group (dynamic diameters), rest are two-line labels; 4 contract additions; reuses MetaDataTable as-is
- [[2026-07-09 Generic MetaDataTable Component Executable Plan]] — frontend-first metadata tables; FrontendTableDataProvider fakes the backend; tab-switch column demo with zero API work
- [[2026-07-14 PRD Enterprise Authentication User Management Analysis]] — Freeport/Entra SSO PRD vs investigated architecture; User Type is the hard new concept; PoC first-login flow = FR-A 10 ⭐

## CORE Web — feature knowledge (living docs)
- [[2026-08-06 Archived Plan Summary Metadata Table Implementation]] — shared archived metadata host/card, dedicated method routes, API rendering chain, and legacy action bridge
- [[2026-07-07 Reamer Hierarchy Feature Explained]] — one hole = several drilling passes; three-state hierarchy design (absent ≠ empty)
- [[2026-07-07 Development Drive Last Updated Feature Explained]] — computed MAX(DateCompleted) per drive, not an audit timestamp; client-supplied, unvalidated
- [[2026-07-09 Generic Metadata Tables Feature Explained]] — current Plan Summary, Shift History, and Shift Report logic; TableData contract mapping + 7 contract gaps (totals row, diameter groups, paging, localization) ⭐
- [[2026-07-14 User Timezone Feature Explained]] — Edit User timezone = display-only preference via UserTimeService (~40 views); site timezone still owns shift boundaries and filters
- [[2026-08-27 BTokenRefresh Feature Explained]] — the escape hatch out of the Blazor circuit for token refresh; why the cookie is unreachable over SignalR ⭐
- [[2026-07-20 Site Drilling Method Enablement Feature Explained]] — SiteFeatures Enabled bit; Edit Site can already disable with no confirmation; disable = portal hidden + sync stops, data never deleted; PRD "history stays visible" ≠ current behaviour ⭐

## Load Testing
- [[2026-06-05 k6 Load Testing Setup for Core Sync]] — backend (k6)
- [[2026-06-16 Playwright Web Client Load Testing]] — web client (real OIDC login, per-realm credentials)
- [[2026-06-12 Core Sync Dataset Rules]] — payload datasets

---
*Cleanup 2026-07-06: 7 notes deleted (git/PR action-execution records and single-fact regressions with no transferable takeaway). Source logs: `~/.claude/projects/` (Claude Code) and `~/.codex/sessions/` (Codex).*
