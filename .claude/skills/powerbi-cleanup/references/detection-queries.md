# Detection queries

Reusable DAX / DMV queries for the powerbi-cleanup skill. Run each via `dax_query_operations`. Queries use `INFO.VIEW.*` where available (human-readable) and fall back to `INFO.*` / `$SYSTEM.DISCOVER_*` for fields the view does not expose.

## Inventory

### All tables with row count and type
```dax
EVALUATE
SELECTCOLUMNS(
    INFO.VIEW.TABLES(),
    "Table", [Name],
    "IsHidden", [IsHidden],
    "IsCalculated", [Expression] <> BLANK(),
    "IsCalculationGroup", [IsCalculationGroup],
    "IsPrivate", [IsPrivate],
    "RowCount", [RowCount]
)
```

### All columns with cardinality and size
```dax
EVALUATE
SELECTCOLUMNS(
    INFO.VIEW.COLUMNS(),
    "Table", [Table],
    "Column", [Name],
    "DataType", [DataType],
    "IsHidden", [IsHidden],
    "IsKey", [IsKey],
    "IsCalculated", [IsCalculated],
    "Cardinality", [Cardinality],
    "DictionarySize", [DictionarySize],
    "DataSize", [DataSize],
    "SortByColumn", [SortByColumn]
)
```
If `INFO.VIEW.COLUMNS()` is unavailable, fall back to `INFO.COLUMNS()` joined to `INFO.TABLES()` and `$SYSTEM.DISCOVER_STORAGE_TABLE_COLUMNS` for size.

### All measures
```dax
EVALUATE
SELECTCOLUMNS(
    INFO.VIEW.MEASURES(),
    "Table", [Table],
    "Measure", [Name],
    "Expression", [Expression],
    "DisplayFolder", [DisplayFolder],
    "IsHidden", [IsHidden],
    "FormatString", [FormatString],
    "Description", [Description],
    "DetailRowsExpression", [DetailRowsDefinitionExpression]
)
```

## Dependency graph

### Full calc dependency (model-internal references)
```dax
EVALUATE INFO.CALCDEPENDENCY()
```
Returns rows like `Object / ObjectType / ReferencedObject / ReferencedObjectType / ReferencedTable / Expression`. An object is **model-referenced** if it appears as any of the `Referenced*` values.

### Relationship endpoints (both active and inactive)
```dax
EVALUATE
SELECTCOLUMNS(
    INFO.RELATIONSHIPS(),
    "FromTable", LOOKUPVALUE( INFO.TABLES()[Name], INFO.TABLES()[ID], [FromTableID] ),
    "FromColumn", LOOKUPVALUE( INFO.COLUMNS()[ExplicitName], INFO.COLUMNS()[ID], [FromColumnID] ),
    "ToTable", LOOKUPVALUE( INFO.TABLES()[Name], INFO.TABLES()[ID], [ToTableID] ),
    "ToColumn", LOOKUPVALUE( INFO.COLUMNS()[ExplicitName], INFO.COLUMNS()[ID], [ToColumnID] ),
    "IsActive", [IsActive]
)
```

### Hierarchy levels (columns that back hierarchies)
```dax
EVALUATE INFO.LEVELS()
```

### Sort-by targets
```dax
EVALUATE
FILTER(
    INFO.COLUMNS(),
    [SortByColumnID] <> BLANK()
)
```

### RLS filter expressions (may reference columns by name)
```dax
EVALUATE INFO.TABLEPERMISSIONS()
```

### Calculation items (may reference measures)
```dax
EVALUATE INFO.CALCULATIONITEMS()
```

### Perspective members (visibility only, but useful context)
```dax
EVALUATE INFO.PERSPECTIVECOLUMNS()
EVALUATE INFO.PERSPECTIVEMEASURES()
EVALUATE INFO.PERSPECTIVEHIERARCHIES()
```

### DMV fallback
```sql
SELECT * FROM $SYSTEM.DISCOVER_CALC_DEPENDENCY
```

## Classification logic

Build three sets in memory from the results above:

- `REFERENCED_COLUMNS` = union of:
  - all `ReferencedObject` where `ReferencedObjectType = 'COLUMN'` in `INFO.CALCDEPENDENCY()`
  - relationship `FromColumn` + `ToColumn`
  - sort-by target columns
  - columns that appear as `INFO.LEVELS()` level columns
  - any column name found as a substring match inside an RLS filter expression (flag only — substring is approximate)

- `REFERENCED_MEASURES` = union of:
  - `ReferencedObject` where `ReferencedObjectType = 'MEASURE'`
  - measures referenced inside calculation item expressions
  - measures referenced inside detail rows expressions
  - measures referenced inside other measures' expressions

- `PROTECTED` = date-table date columns, primary key columns, calculation group columns, columns inside a marked-as-date table, parameter-backing columns.

Then:

- **Unused column** = `not in REFERENCED_COLUMNS and not in PROTECTED`
- **Unused measure** = `not in REFERENCED_MEASURES`

## Smell detection

### Duplicate measures by expression
```dax
EVALUATE
VAR Normalized =
    SELECTCOLUMNS(
        INFO.VIEW.MEASURES(),
        "Table", [Table],
        "Measure", [Name],
        "Normalized",
            SUBSTITUTE( SUBSTITUTE( UPPER( TRIM( [Expression] ) ), UNICHAR(10), "" ), UNICHAR(13), "" )
    )
RETURN
    FILTER(
        Normalized,
        COUNTROWS( FILTER( Normalized, [Normalized] = EARLIER( [Normalized] ) ) ) > 1
    )
```

### High-cardinality text columns
Threshold: cardinality > 1000 AND cardinality / row_count > 0.9 AND DataType = "String" AND not key and not hidden.
```dax
EVALUATE
VAR Tbl = INFO.VIEW.TABLES()
RETURN
FILTER(
    ADDCOLUMNS(
        INFO.VIEW.COLUMNS(),
        "TableRows", LOOKUPVALUE( Tbl[RowCount], Tbl[Name], [Table] )
    ),
    [DataType] = "String"
        && NOT [IsKey]
        && NOT [IsHidden]
        && [Cardinality] > 1000
        && DIVIDE( [Cardinality], [TableRows] ) > 0.9
)
```

### Zero-row tables (candidate junk)
```dax
EVALUATE
FILTER(
    INFO.VIEW.TABLES(),
    [RowCount] = 0
        && NOT [IsCalculationGroup]
        && NOT [IsPrivate]
)
```

### Calc columns that look like measure candidates
Heuristic: expression contains a scalar aggregation (`SUM`, `AVERAGE`, `COUNT*`) at the outermost level, or is a simple row-level arithmetic between two numeric columns used only inside a single SUMX. Flag and suggest rewrite; do not auto-convert.

### Inactive relationships never activated
Cross-check: relationship `IsActive = FALSE` AND no `INFO.CALCDEPENDENCY()` row references the relationship (look for `USERELATIONSHIP` in measure expressions text).

### Stale translations
```dax
EVALUATE
INFO.OBJECTTRANSLATIONS()
```
Group by culture; if translated-object-count / total-objects < 0.2, flag the culture as stale.
