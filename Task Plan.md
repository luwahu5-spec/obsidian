Date: 2026-06-04  
tags: [planning, task, implementation]  
status: planning  
priority: high

> **Author:** Allen  
> **Date:** 2026-06-04  
> **Estimated Effort:** Medium / Large  
> **Target Completion:** Next sprint / staged delivery

## 🎯 Goal

Refactor the site workflow so the frontend no longer treats a site as “Development” or “Production” directly. Instead, the frontend should render site-level pages from generic drilling method metadata, allowing current and future drilling methods such as Production, Development, and Cut and Fill to be added with minimal frontend changes.

---

## 📐 Scope

## **In Scope:**

- Replace site-list product-type navigation with direct site navigation.
- Introduce a frontend adapter/model layer for drilling methods.
- Prepare site details UI to support drilling method tabs.
- Move upload drill plan and plan summary sections toward method-tab-based rendering.
- Keep frontend components generic where possible.
- Support the current legacy backend shape until backend behaviour metadata is available.
- Keep existing site-level collections such as rigs, drives, drillers, consumable reasons, pause reasons, service types, and service tags.

## **Out of Scope:**

- Backend API implementation.
- Shared repository changes.
- Final backend drilling behaviour metadata design.
- Rig-level drilling method logic until backend confirms the metadata shape.
- Full replacement of upload, plan summary, shift history, or shift report providers in this task.
- Database/entity migration work unless backend later requires it.

---

## 🧩 What Needs to Be Done

- Add a web-only drilling method adapter that translates current site data into generic frontend drilling method models.
- Update the site list so both administrators and regular users click the site name to enter the site directly.
- Remove visible Development | Production site-type links from the site list.
- Introduce a generic drilling method tab layout on the site details page.
- Place method-specific upload drill plan content inside the selected drilling method tab.
- Place method-specific plan summary data inside the selected drilling method tab.
- Keep existing site-level data collections attached to the Site model/context.
- Prepare component interfaces so future backend providers can be plugged in without large frontend rewrites.
- Document temporary frontend assumptions clearly in code comments.

---

## 💡 Why We're Doing This

- **Problem being solved:**  
    The current frontend hardcodes Development and Production checks in many places. Adding a new drilling method, such as Cut and Fill, would increase duplicated conditionals, labels, calculations, and UI branching.
    
- **Impact if skipped:**  
    Every new drilling method would require more frontend-specific logic, making the UI harder to maintain and increasing the risk of inconsistent behaviour across site details, uploads, plan summaries, shift history, and reports.
    
- **Requested by / triggered by:**  
    Product clarification that a site should no longer be categorized as Development or Production at the landing level. Instead, one site page should contain drilling method tabs, and each tab should display method-specific content.
    

---

## 💻 Suggested Code Areas

|Area|File / Module|What Likely Changes|
|---|---|---|
|Drilling method adapter|Minnovare.Core.Web/Shared/Drilling/*|Web-only adapter and frontend models for drilling methods|
|Dependency injection|Minnovare.Core.Web/Startup.cs|Register drilling method adapter|
|Site list|Minnovare.Core.Web/Pages/Blazor/Sites/SitesList.razor|Make site name the main navigation link; remove Development/Production links|
|Site details|Minnovare.Core.Web/Pages/Blazor/Sites/SiteDetails.razor|Add drilling method tab layout|
|Upload section|Existing upload drill plan section/component|Move into method tab and render through generic upload config/component|
|Plan summary|Existing drive/development summary components|Move toward generic provider-driven plan summary section|
|Site edit|Existing edit site page/components|Later align tabs/properties with drilling behaviour metadata|

`public interface IDrillingMethodAdapter { IReadOnlyList<SiteDrillingMethod> GetSiteMethods(Site site); IReadOnlyList<SiteDrillingMethod> GetEnabledSiteMethods(Site site); bool IsMethodEnabled(Site site, string methodKey); } public class SiteDrillingMethod { public string Key { get; set; } public string Label { get; set; } public int? DisplayOrder { get; set; } public bool Enabled { get; set; } // Temporary bridge to current FeatureType-based backend model. public FeatureType? LegacyFeatureType { get; set; } // Reserved for future backend behaviour metadata/providers. public Dictionary<string, object> Metadata { get; set; } }`

---

## 🗺️ Implementation Plan

---

### Phase 1 — Frontend Drilling Method Adapter

**Goal of this phase:**

Create a stable frontend boundary so pages consume generic drilling method data instead of hardcoded Development/Production checks.

**Steps:**

- Add SiteDrillingMethod frontend model.
- Add DrillingMethodDefinition as a temporary compatibility map.
- Add IDrillingMethodAdapter.
- Add DrillingMethodAdapter.
- Register adapter in Startup.cs.
- Keep all adapter code inside Minnovare.Core.Web.
- Add comments explaining that KnownMethods is temporary until backend metadata exists.

**Done when:**

Frontend has a single adapter that converts current site feature data into generic drilling method models, and page components do not need to know the source of the drilling method data.

---

### Phase 2 — Site List Navigation Refactor

**Goal of this phase:**

Remove site-type navigation from the landing page and make the site itself the entry point.

**Steps:**

- Update SitesList.razor.
- Make site name clickable for both administrators and regular users.
- Route site name directly to the site details page.
- Remove Development | Production links from the list.
- Remove or hide product selector navigation from this workflow.
- Keep Edit link for administrators where appropriate.

**Done when:**

All users enter a site by clicking the site name, and the site list no longer presents Development/Production as separate site destinations.

---

### Phase 3 — Site Details Drilling Method Tabs

**Goal of this phase:**

Make the site details page render one tab per enabled drilling method.

**Steps:**

- Inject IDrillingMethodAdapter into the site details page.
- Get enabled drilling methods for the current site.
- Render tab labels from SiteDrillingMethod.Label.
- Track selected method by SiteDrillingMethod.Key.
- Avoid hardcoded checks such as if Production or if Development in the page layout.
- Preserve existing site-level data and layout outside the selected method content.

**Done when:**

The site details page can show tabs based on adapter output, and adding a new method only requires adapter/backend metadata changes, not a new page structure.

---

### Phase 4 — Method-Specific Upload Section

**Goal of this phase:**

Move upload drill plan functionality into each drilling method tab.

**Steps:**

- Create or refactor a generic upload card/component.
- Pass selected drilling method key into the upload component.
- Keep upload display labels/config local for now if backend does not provide upload providers.
- Avoid duplicating upload layout per method.
- Ensure each method tab can render a different upload title, file input, and upload handler.

**Done when:**

Each drilling method tab can show its own upload section, while the layout remains generic and method-specific logic is isolated.

---

### Phase 5 — Method-Specific Plan Summary Section

**Goal of this phase:**

Prepare plan summary data to be rendered per drilling method tab and later driven by backend table/filter providers.

**Steps:**

- Identify current hardcoded plan summary components.
- Introduce a generic plan summary section component.
- Pass selected drilling method key into the section.
- Keep current components working during transition.
- Prepare interfaces for future table provider and filter provider data.
- Support method-specific table title, columns, actions, filters, and archived toggle.

**Done when:**

Plan summary is no longer conceptually tied to a site-level Development/Production switch and can be rendered inside the selected drilling method tab.

---

### Phase 6 — Future Provider Integration

**Goal of this phase:**

Connect backend-provided behaviour metadata when available without changing the frontend page structure again.

**Steps:**

- Replace temporary KnownMethods mapping inside the adapter.
- Map backend drilling behaviour metadata into SiteDrillingMethod.
- Map backend upload/table/filter providers into typed frontend config models.
- Remove legacy FeatureType dependency from the adapter when backend fully supports the new model.
- Keep existing UI components consuming the same generic frontend models.

**Done when:**

Backend controls available drilling methods and method-specific providers, while frontend only renders the returned configuration.

---

## ⚠️ Risks & Unknowns

|Risk / Unknown|Likelihood|Mitigation|
|---|---|---|
|Backend drilling behaviour metadata shape is unknown|High|Use a frontend adapter so only one translation layer changes later|
|Backend may not provide upload provider/config|High|Keep frontend upload config isolated in one generic upload module for now|
|Site details currently has hardcoded Development/Production components|High|Refactor gradually behind method tabs instead of replacing everything at once|
|Product may change tab ordering or labels|Medium|Use Label and optional DisplayOrder from adapter/backend|
|Rig drilling method metadata is unclear|High|Do not depend on rig method properties yet|
|Existing site edit page still uses Development/Production tabs|Medium|Treat site edit as a later behaviour-driven refactor|
|Current build may fail due to locked DLL while app is running|Medium|Stop running app or build to a separate output folder during verification|

---

## ✅ Definition of Done

- Site list no longer displays Development | Production links.
- Both administrator and regular users navigate by clicking the site name.
- Site details page renders drilling method tabs from adapter/backend metadata.
- Upload drill plan section is displayed inside the selected drilling method tab.
- Plan summary data section is displayed inside the selected drilling method tab.
- No page-level hardcoded Development/Production branching remains for the refactored workflow.
- Existing site-level collections remain available on the site context.
- Temporary frontend assumptions are clearly commented.
- Tests passing / manually verified.

---

## 🔗 Related

- Logic Analysis: [[Drilling Method Behaviour Refactor]]
- Debug Log: [[]]
- Commit / PR:
---

 Date: 2026-06-04 
 tags: [planning, task, implementation] 
 status: planning priority: low | medium | high


> **Author:** Allen **Date:** 2026-06-04 **Estimated Effort:** **Target Completion:**

---

## 🎯 Goal

> What is the single, clear outcome this task is meant to achieve? One or two sentences max.

---

## 📐 Scope

> What is **in scope** and what is explicitly **out of scope**? This prevents scope creep.

## **In Scope:**

## **Out of Scope:**

---

## 🧩 What Needs to Be Done

> A high-level breakdown of the work. Not a phase-by-phase plan yet — just the full list of things that need to happen.

- [ ]
- [ ]
- [ ]
- [ ]

---

## 💡 Why We're Doing This

> The reason / motivation behind the task. What problem does this solve? What breaks if we don't do it?

- **Problem being solved:**
- **Impact if skipped:**
- **Requested by / triggered by:**

---

## 💻 Suggested Code Areas

> Which files, modules, classes, or methods will likely need to be touched? This is a working hypothesis — update as you go.

|Area|File / Module|What Likely Changes|
|---|---|---|
||||
||||
||||

```csharp
// Any specific method signatures, interfaces, or patterns worth noting upfront
```

---

## 🗺️ Implementation Plan

> Break the work into sequential phases. Each phase should be completable and independently testable where possible.

---

### Phase 1 — `[Name of Phase]`

**Goal of this phase:**

**Steps:**

- [ ]
- [ ]
- [ ]

**Done when:**

---

### Phase 2 — `[Name of Phase]`

**Goal of this phase:**

**Steps:**

- [ ]
- [ ]
- [ ]

**Done when:**

---

### Phase 3 — `[Name of Phase]` _(if needed)_

**Goal of this phase:**

**Steps:**

- [ ]
- [ ]

**Done when:**

---

## ⚠️ Risks & Unknowns

> What could go wrong? What do you not know yet that could block progress?

|Risk / Unknown|Likelihood|Mitigation|
|---|---|---|
||||
||||

---

## ✅ Definition of Done

> How will you know this task is fully complete? List concrete, verifiable criteria.

- [ ]
- [ ]
- [ ]
- [ ] Tests passing / manually verified

---

## 🔗 Related

- Logic Analysis: [[]]
- Debug Log: [[]]
- Commit / PR: