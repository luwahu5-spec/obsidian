---
date: 2026-05-19
source: Codex (VS Code)
project: core-web
tags: [planning, roadmap, users, mining-methods, site-refactor]
---

# Plan — order of site-type feature implementation

## Question
Given items: Users (15), Site Creation Page (19), Rig Creation Page (20), Settings Pages (21), General (22) — what order to implement?

## Agreed order & reasoning
1. **Users** — must be first. The tab logic depends on `userId + siteId → backend returns available mining methods`. Without per-user/per-site mining-method assignment there is no source of truth behind the endpoint. Includes: user create/edit mining-method assignment, per-site assignment, admin users likely get all methods.
2. **Site Creation Page**
3. **Rig Creation Page**
4. **Settings Pages**
5. **General**

## Planning principle used
Order features by **data dependency**: build the source-of-truth (user/site assignments) before the UIs that consume it, even if the consuming UI is more visible.

Also from this session: for the SiteDetails "Plan summary data" (backend table data provider not ready), the recommendation was to separate the **ideal backend-owned design** from a **practical transition path** when the backend provider doesn't exist yet.
