---
date: 2026-06-19
source: Codex (VS Code)
project: core-web-api
tags: [debugging, prestart-data, dynamic-forms, postman, fk-violation, payload]
---

# Crafting a valid PostPreStartData payload (NRE + FK errors)

## Symptom chain
1. `NullReferenceException` at ShiftsController ~1549 — payload used key `Groups`; the model expects **`groupData`**. (Property names in the app's JSON are display-only; the save path binds `DynamicItemGroup`/`DynamicItem` objects.)
2. After fixing that: FK violation — the example `dynamicItemGroup.id: 123` doesn't exist in `dbo.DynamicItemGroup`.

## The rule
**You cannot invent dynamic-form IDs.** Source of truth: `GET /DynamicForms/GetByRig/{rigId}` — reuse the real IDs from `itemGroups[].id`, `itemGroups[].items[].id`, `itemGroups[].items[].itemTemplate.id` in the POST body.

Minimal shape that passes parsing (empty pre-start form):
```json
{ "id": "<guid>", "shift": { "startTime": "...", "endTime": "...", "rigId": 121, "name": "Night", "number": 1, ... }, "groupData": [] }
```

## Debugging insight
An FK error after a parse-level NRE is *progress* — it proves the payload now reaches EF save; the remaining problem is data validity, not shape.
