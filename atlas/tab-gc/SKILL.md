---
name: tab-gc
description: Close or reduce browser tabs by identifying stale, duplicate, or low-value tabs. Use when the user asks for tab cleanup, tab GC, closing unused/old tabs, or reducing tab count, with a focus on closing infrequently used tabs while preserving active work.
---

# Tab GC

## Workflow

1. Gather a tab inventory with title, URL, window, pinned status, and last accessed time if available.
2. Confirm the "infrequently used" threshold.
   - If the user provides a cutoff (time, count, or category), use it.
   - Otherwise propose a default cutoff (for example, not accessed in the last 7 days) and confirm.
3. Protect tabs that are likely active work: pinned tabs, tabs with unsaved input, or tabs actively playing media.
4. Identify closure candidates in priority order:
   - Tabs not accessed since the cutoff.
   - Duplicates and near-duplicates.
   - Low-value results (search pages, error pages, login landings).
5. Close candidates in small batches, confirming any ambiguous cases.
6. Report what was closed with titles and URLs so the user can restore if needed.
