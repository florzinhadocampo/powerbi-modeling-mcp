# Cleanup report template

Use this format when presenting findings to the user. Keep it scannable. Always wait for explicit approval before executing any deletion.

---

## Power BI Model Cleanup — Findings

**Model:** `<model name>` on `<server / workspace>`
**Compatibility level:** `<level>`
**Scanned at:** `<ISO timestamp>`

### Summary

| Bucket            | Count | Notes                                     |
| ----------------- | ----- | ----------------------------------------- |
| Safe to remove    |       | Unreferenced & hidden, or never queried.  |
| Likely removable  |       | Unreferenced but visible to consumers.    |
| Smells            |       | Not for auto-removal. Suggestions only.   |
| Protected / keep  |       | Matched a safety rule.                    |

If trace-based usage was captured: `Trace window: <start> → <end>, <N> distinct queries observed.` Otherwise: `Trace not run — usage is model-internal only.`

### Safe to remove

Grouped by table. One row per object.

| Object                         | Kind    | Evidence                                                        | Size impact       |
| ------------------------------ | ------- | --------------------------------------------------------------- | ----------------- |
| `Sales[AmountRaw]`             | Column  | No CALCDEPENDENCY hits; not a relationship/sort/hierarchy target; hidden. | 4.2 MB dict + 1.1 MB data |
| `Sales[Margin pct v2]`         | Measure | No CALCDEPENDENCY hits; not in any calc item / detail rows / RLS.         | n/a               |

### Likely removable (needs confirmation)

| Object                         | Kind    | Why uncertain                                                   |
| ------------------------------ | ------- | --------------------------------------------------------------- |
| `Product[DescriptionLong]`     | Column  | Unreferenced in model but visible; may be used in report visuals. |
| `Finance[LegacyRatio]`         | Measure | Unreferenced in model; display folder suggests external consumers. |

### Smells (suggestions only)

| Object / Pattern                    | Smell                                                   | Suggestion                                  |
| ----------------------------------- | ------------------------------------------------------- | ------------------------------------------- |
| `Sales[LineTotal]` (calc column)    | Row-level `[Qty]*[Price]` only used inside a `SUMX`.    | Rewrite the one measure as `SUMX(Sales, Sales[Qty]*Sales[Price])` and drop the calc column. |
| Measures `Revenue`, `Revenue 2`     | Identical expressions after normalization.              | Keep one, redirect dependents, delete other. |
| `Customer[Notes]`                   | String column, cardinality 98% of rows.                 | Confirm reporting need; if none, drop.      |
| `DimDate` auto date/time hidden     | Auto date/time enabled — creates one hidden table per date column. | Disable in model settings.                  |

### Protected / kept

Short list — explain why each was protected (key, relationship endpoint, sort-by target, RLS reference, marked-as-date, calculation group column, etc.).

### Proposed execution plan

If the user approves, the changes will run inside a single transaction in this order:

1. Delete N measures
2. Delete N calculated columns
3. Delete N hierarchy levels / hierarchies
4. Delete N relationships touching dropped columns
5. Delete N regular columns
6. Delete N calculated tables
7. Commit. On any failure: roll back, report the failing step.

### Next step

Reply with one of:
- `approve all safe` — execute the "Safe to remove" bucket only.
- `approve safe + <table>` — safe bucket plus "likely" rows for a specific table.
- `approve <comma-separated qualified names>` — granular.
- `skip` — no changes.

> Reminder: back up (or confirm clean git state for PBIP) before approving.
