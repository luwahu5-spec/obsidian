---
date: 2026-06-25
source: Claude Code (VS Code)
project: core-web
tags: [debugging, bootstrap5, blazor, csv-import, site-upload]
---

# Upload card debugging — dead X button + "Invalid CSV Layout"

*(UI-styling parts of this session omitted; two diagnostic findings kept.)*

## Dead close button — Bootstrap 4 syntax in a Bootstrap 5 app
The alert's X used `data-dismiss="alert"` — **Bootstrap 4 attribute, silently a no-op in Bootstrap 5** (BS5 needs `data-bs-dismiss`). Fixed properly with a Blazor `@onclick` handler that clears the error, resets `_uploadFiles`, and **regenerates the `InputFile` element — the only way to truly reset a file picker in Blazor**.

## "Import Failed … Invalid CSV Layout" on a file we exported ourselves
Export comes from `DriveRepository.ExportDriveAsCSV` (one backend endpoint); import goes to `/Sites/{siteId}/ImportDrillPlanAsync` (a different endpoint) — **the export format and import parser don't agree on column layout**. Frontend can't fix it; it's a backend format mismatch to raise with the API side.

## Localization mechanics worth remembering
Renaming a button label = rename the `@Localizer` key in the component **and** the key in all 7 `.resx` files (en, en-AU, en-US, es, fr, fr-CA, pt); non-English locales keep their translations under the new key.
