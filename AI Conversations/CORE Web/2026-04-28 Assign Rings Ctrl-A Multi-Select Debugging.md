---
date: 2026-04-28
source: Codex (VS Code)
project: core-web
tags: [debugging, blazor, select, appearance, scoped-css, rig-details]
---

# Assign Rings Ctrl+A stopped working — `appearance: base-select` on multi-select

## Symptom
In RigDetails, the Assign Rings picker (native `<select multiple>`) no longer responded to Ctrl+A locally, though production behaved fine.

## Expected flow
focus ring box → `Ctrl+A` → browser selects all options → Blazor `@onchange` updates `_ringsSelectedForAssignment` → Apply button enables.

## Root cause
A global SCSS rule applied `appearance: base-select` to all `select.form-select` — **including the multi-select**. Chrome then stops treating it as a native multi-select, killing native Ctrl+A. The regression was reintroduced by an incomplete revert (the CSS guard was lost while Razor/JS changes were reverted).

## Fix
Exclude multi-selects from custom single-select styling — `select:not([multiple])` — and update **all three artifacts**: `base.scss` (source), compiled `site.css`, and the minified CSS (the browser may load any of them depending on environment). JS keydown workarounds were rejected; the CSS guard is the correct fix.

## Lessons
- After a partial/interrupted `git restore` (e.g. rejected permission prompt), verify the diff actually reverted — here a stale compiled CSS kept the bug alive.
- `base.scss` "modified" with no content diff = line-ending noise, but *check* rather than assume.
- Scoped-CSS colors not applying usually means a more specific rule wins or the scoped attribute selector doesn't match the rendered element — inspect the winning rule in DevTools, rebuilding won't help.
