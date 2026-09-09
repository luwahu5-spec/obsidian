---
date: 2026-07-02
source: Claude Code (VS Code)
project: core-web-api
tags: [debugging, csv-import, drill-plan, sites-controller]
---

# "Invalid CSV Layout" — the real rules of ImportDrillPlanAsync

## Accepted upload formats (`SitesController.cs` ~553)
1. **Direct CSV** — MIME type must be in `CsvMimeTypes.All` (`Globals.cs` ~229: text/csv, text/plain, application/vnd.ms-excel, …) **and** filename must end `.csv`.
2. **ZIP archive** containing the CSV.

## Root cause of the error (not site-related!)
`CsvImporter.cs` (~47–74) reads the **first row** and does an **exact case-insensitive string match** against **6 hardcoded header layouts** (Deswik format etc.). No match → `indexConfig == null` → throws `"Invalid CSV Layout. Please call Minnovare Support."` at ~467.

So: the failure depends entirely on the **header row text**, not the site, the data rows, or permissions. A file exported from the Drive section can still fail if the export header doesn't exactly equal one of the 6 import layouts — which is the export/import mismatch seen from the frontend in [[2026-06-25 Upload Error Popup and CSV Import Mismatch]].

## Debugging rule
For "invalid layout/format" errors, diff the file's header row byte-for-byte against the parser's expected header list before investigating anything else.
