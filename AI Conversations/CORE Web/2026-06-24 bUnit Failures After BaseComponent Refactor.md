---
date: 2026-06-24
source: Codex (VS Code)
project: core-web
tags: [debugging, bunit, testing, dependency-injection, base-component]
---

# bUnit test failures after components inherited BaseComponent

## Symptom
After refactoring components to inherit a shared `BaseComponent`, previously green bUnit tests (SensorLogTests, ReamHierarchyTests, CutFaceComponentTests) failed with DI errors.

## Root cause
**bUnit must satisfy every `[Inject]` property of the whole inheritance chain — even if the test never exercises that code path.** Inheriting `BaseComponent` added shared injected services the test contexts never registered.

## Fix pattern
Register test-only mocks for the base dependencies in each affected `TestContext` rather than weakening production DI. For localization, a **pass-through `IStringLocalizer<ApiPageFilter>`** (key resolves to itself) mirrors production enough that markup assertions stay unchanged.

## Concrete earlier instance (2026-06-17)
`ToastService` added as `[Inject]` to `CutFaceComponent`/`ReamHierarchy` broke their tests the same way. Since `ToastService` has no constructor dependencies, the fix was simply `Services.AddScoped<ToastService>();` in each test constructor — no mock needed.

## Lesson
When adding injections to a shared base component, sweep all bUnit test setups — run the affected test project immediately to expose every context missing the new registrations.
