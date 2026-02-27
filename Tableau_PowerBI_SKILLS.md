# Tableau to Power BI Migration Guide

This comprehensive guide documents the complete process of converting Tableau workbooks (`.twb` / `.twbx` format) and data sources to Power BI reports (`.pbip` format) using Power BI Desktop and semantic model definitions.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Project Structure](#project-structure)
4. [Migration Process](#migration-process)
   - [Step 1: Analyze Tableau Data Sources](#step-1-analyze-tableau-data-sources)
   - [Step 2: Identify Calculated Fields and Transformations](#step-2-identify-calculated-fields-and-transformations)
   - [Step 3: Create Power Query / DAX Equivalents](#step-3-create-power-query--dax-equivalents)
   - [Step 4: Analyze Tableau Worksheets and Dashboards](#step-4-analyze-tableau-worksheets-and-dashboards)
   - [Step 5: Extract Color Palettes](#step-5-extract-color-palettes)
   - [Step 6: Build Power BI Semantic Model (.tmdl)](#step-6-build-power-bi-semantic-model-tmdl)
   - [Step 7: Create Report Pages (.json)](#step-7-create-report-pages-json)
   - [Step 8: Configure Model Relationships](#step-8-configure-model-relationships)
   - [Step 9: Apply Themes](#step-9-apply-themes)
   - [Step 10: Publish](#step-10-publish)
5. [Semantic Model Specification](#semantic-model-specification)
6. [Visual Templates](#visual-templates)
7. [Reference Tables](#reference-tables)
8. [Troubleshooting](#troubleshooting)
9. [Complete Example: Superstore Sales](#complete-example-superstore-sales-migration)

---

## Overview

The migration process converts:
- **Tableau Data Sources (.tds / embedded)** → Power BI Semantic Model (`.tmdl` tables)
- **Tableau Calculated Fields** → Power BI DAX Measures and Calculated Columns
- **Tableau Worksheets** → Power BI Report Visuals
- **Tableau Dashboards** → Power BI Report Pages
- **Tableau Parameters** → Power BI What-If Parameters or Slicers
- **Tableau Filters** → Power BI Slicers / Report-level Filters

## Prerequisites

- Power BI Desktop installed (latest version recommended)
- Access to underlying data sources connected in the Tableau workbook
- Tableau workbook in `.twb` (XML) or `.twbx` (packaged) format
- Power BI Pro or Premium license for publishing (optional for local migration)

## Placeholders Reference

Throughout this guide, replace these placeholders with your actual values:

| Placeholder | Description | Example |
|-------------|-------------|---------|
| `<WorkbookName>` | Name of your Tableau workbook | `SuperstoreSales` |
| `<reportname>` | Lowercase report name (no spaces) | `superstoresales` |
| `<datasource_name>` | Tableau data source logical name | `Sample - Superstore` |
| `<server>` | Database server hostname | `sqlserver.company.com` |
| `<database>` | Database name | `superstore_db` |
| `<schema>` | Schema name | `dbo` |
| `<table>` | Table or view name | `Orders` |
| `<workspace_url>` | Power BI Service workspace URL | `https://app.powerbi.com/groups/<guid>` |

## Project Structure

```
├── <WorkbookName>.pbip                  # Power BI project file (entry point)
├── <WorkbookName>.SemanticModel/
│   └── definition/
│       ├── model.tmdl                   # Model-level settings (name, culture, etc.)
│       ├── relationships.tmdl           # Table relationships
│       └── tables/
│           ├── <Table1>.tmdl            # Table definition (columns, measures, M query)
│           ├── <Table2>.tmdl            # Additional tables
│           └── _Measures.tmdl           # Standalone measures table (if applicable)
├── <WorkbookName>.Report/
│   └── definition/
│       ├── report.json                  # Report-level settings and theme reference
│       └── pages/
│           ├── <pageId1>/
│           │   └── page.json            # Page layout and visual definitions
│           └── <pageId2>/
│               └── page.json
├── StaticResources/
│   └── RegisteredResources/
│       └── <ThemeName>.json             # Custom theme JSON
└── tableau_source/                      # Original Tableau files (reference)
    └── <WorkbookName>.twb
```

---

## Migration Process

### Step 1: Analyze Tableau Data Sources

Open the `.twb` file (it is XML) and locate the `<datasources>` section. For each `<datasource>` element:

**For each data source, determine the action:**

| Scenario | Action |
|----------|--------|
| Live connection to a database table | Import or DirectQuery in Power BI |
| Tableau extract (.hyper) | Re-import from the original source into Power BI |
| Blended data source (primary + secondary) | Reproduce as a relationship or merged query in Power BI |
| Published data source on Tableau Server | Connect Power BI to the same underlying database |

**Create a data source analysis like this:**

| Tableau Data Source | Connection Type | Tables Used | Has Calculated Fields | Power BI Action |
|---------------------|----------------|-------------|----------------------|----------------|
| `<datasource_name>` | SQL Server / Extract / etc. | `<table1>`, `<table2>` | Yes/No | **Import** / **DirectQuery** |

**Key XML elements to inspect:**
```xml
<datasource name='...' caption='...' connection='...'>
  <connection class='sqlserver' server='...' dbname='...' />
  <relation name='<table>' table='[<schema>].[<table>]' type='table' />
</datasource>
```

### Step 2: Identify Calculated Fields and Transformations

In the `.twb` XML, look inside each `<datasource>` for `<column>` elements with a `<calculation>` child, or for `role='measure'` / `role='dimension'` with a formula attribute.

**Calculated field (Tableau formula):**
```xml
<column caption='Profit Ratio' datatype='real' name='[Profit Ratio]' role='measure'>
  <calculation class='tableau' formula='SUM([Profit])/SUM([Sales])' />
</column>
```

**Groups / Bins / Sets:**
```xml
<column caption='Segment Group' datatype='string' name='[Segment Group]'>
  <calculation class='tableau' formula='IF [Segment]="Consumer" THEN "B2C" ELSE "B2B" END' />
</column>
```

**Table calculations** (e.g., `RUNNING_SUM`, `RANK`, `WINDOW_AVG`): These have no direct one-to-one equivalent in a measure definition — they must be rewritten as DAX time-intelligence or window functions.

**Parameters:**
```xml
<column caption='Top N' datatype='integer' name='[Parameters].[Top N]' param-domain-type='range'>
  <calculation formula='10' />
</column>
```

**Common transformation patterns to look for:**
- `DATETRUNC('month', [Order Date])` → `DATE(YEAR([Date]), MONTH([Date]), 1)` in DAX or Power Query
- `DATEDIFF('day', [Start], [End])` → `DATEDIFF([Start], [End], DAY)` in DAX
- `STR([Number])` → `FORMAT([Number], "General")` in DAX
- `TRIM([String])` → `TRIM([String])` in DAX or `Text.Trim` in M
- `UPPER([String])` → `UPPER([String])` in DAX or `Text.Upper` in M

### Step 3: Create Power Query / DAX Equivalents

For each Tableau table or data source, create the corresponding Power BI table definition in `<WorkbookName>.SemanticModel/definition/tables/`.

**Naming Conventions:**
- **File:** `<TableName>.tmdl`
- **Table display name:** Match Tableau logical name where possible
- **Measure names:** Match Tableau calculated field captions exactly

**Template — Table with Power Query (M) source:**
```
table <TableName>
    lineageTag: <guid>

    partition <TableName> = m
        mode: import
        source =
            let
                Source = Sql.Database("<server>", "<database>"),
                Schema = Source{[Schema="<schema>",Item="<table>"]}[Data],
                #"Trimmed Columns" = Table.TransformColumns(Schema, {
                    {"<text_column>", Text.Trim, type text}
                }),
                #"Added Custom" = Table.AddColumn(#"Trimmed Columns", "<CalcColumn>",
                    each [<col1>] * [<col2>], type number)
            in
                #"Added Custom"

    column <OriginalColumn>
        dataType: string
        lineageTag: <guid>
        summarizeBy: none
        sourceColumn: <OriginalColumn>

    column <CalcColumn>
        dataType: decimal
        lineageTag: <guid>
        summarizeBy: sum
        sourceColumn: <CalcColumn>

    measure '<MeasureName>' = SUM('<TableName>'[<column>])
        lineageTag: <guid>
        formatString: \#,0.00
```

**Template — Calculated date dimension (equivalent to Tableau date scaffold):**
```
table DimDate
    lineageTag: <guid>

    partition DimDate = m
        mode: import
        source =
            let
                StartDate = List.Min(<TableName>[<DateColumn>]),
                EndDate   = List.Max(<TableName>[<DateColumn>]),
                DateList  = List.Dates(StartDate, Duration.Days(EndDate - StartDate) + 1,
                                #duration(1, 0, 0, 0)),
                DateTable = Table.FromList(DateList, Splitter.SplitByNothing(),
                                {"Date"}, null, ExtraValues.Error),
                #"Changed Type" = Table.TransformColumnTypes(DateTable, {{"Date", type date}}),
                #"Added Year"   = Table.AddColumn(#"Changed Type", "Year",
                                    each Date.Year([Date]), Int64.Type),
                #"Added Month"  = Table.AddColumn(#"Added Year", "MonthNumber",
                                    each Date.Month([Date]), Int64.Type),
                #"Added MonthName" = Table.AddColumn(#"Added Month", "Month",
                                    each Date.ToText([Date], "MMM"), type text),
                #"Added Quarter" = Table.AddColumn(#"Added MonthName", "Quarter",
                                    each "Q" & Text.From(Date.QuarterOfYear([Date])), type text)
            in
                #"Added Quarter"
```

### Step 4: Analyze Tableau Worksheets and Dashboards

In the `.twb` XML, locate the `<worksheets>` and `<dashboards>` sections.

**For each `<worksheet>`, extract:**

1. **Chart type** (`<mark class='...'>`):
   - `bar` → Power BI `barChart` / `clusteredBarChart`
   - `line` → Power BI `lineChart`
   - `area` → Power BI `areaChart`
   - `circle` / `shape` → Power BI `scatterChart`
   - `text` → Power BI `tableEx` or `matrix`
   - `pie` → Power BI `pieChart`
   - `map` → Power BI `map` or `filledMap`

2. **Rows / Columns shelf fields** (`<rows>` and `<cols>` elements)

3. **Marks card** — Color, Size, Label, Detail, Tooltip fields

4. **Filters** (`<filter>` elements inside `<datasource-dependencies>`)

5. **Sort order** and **top N** settings

**For each `<dashboard>`, extract:**

- Dashboard title and size
- List of worksheet objects and their layout zones (`<zone>` elements with `x`, `y`, `w`, `h` attributes)
- Text boxes, images, and blank padding objects

**Create a visual analysis table like this:**

| Worksheet Name | Tableau Mark Type | Power BI Visual | Dimensions (Rows/Cols) | Measures | Filters |
|---------------|------------------|-----------------|------------------------|----------|---------|
| `<sheet_name>` | bar | clusteredBarChart | Category | SUM(Sales) | Region = 'West' |

### Step 5: Extract Color Palettes

In the `.twb` XML, locate `<preferences>` and `<encoding>` elements for color settings. Custom color palettes are defined in the Tableau Repository `Preferences.tps` file or embedded as:

```xml
<color-palette name="Custom Palette" type="regular">
  <color>#1F4B99</color>
  <color>#2EA8E0</color>
  <color>#FF9F1C</color>
  <color>#2ECC71</color>
  <color>#E74C3C</color>
</color-palette>
```

**Tableau Palette Type Mapping:**

| Tableau Palette Type | Power BI Equivalent |
|----------------------|---------------------|
| `regular` (categorical) | `dataColors` array in theme JSON |
| `ordered-sequential` | Custom sequential palette in conditional formatting |
| `ordered-diverging` | Custom diverging palette in conditional formatting |

**Tableau Mark Color → Power BI Data Color:**
- Note the hex values from the Tableau palette in order
- Map position 0 → first color in `dataColors` array in Power BI theme JSON

### Step 6: Build Power BI Semantic Model (.tmdl)

Create the `.tmdl` files in `<WorkbookName>.SemanticModel/definition/`.

**model.tmdl:**
```
model Model
    culture: en-US

    annotation PBIDesktopVersion = <version>
```

**relationships.tmdl — use Tableau join relationships as a guide:**
```
relationship '<ForeignTable>_<PKTable>'
    fromColumn: '<ForeignTable>'[<foreign_key>]
    toColumn: '<PKTable>'[<primary_key>]
```

> **Important:** Tableau joins are query-time (each worksheet can have a different join). Power BI relationships are model-level. When Tableau worksheets use **different join paths** to the same tables, you may need to create **role-playing dimensions** or **inactive relationships** and use `USERELATIONSHIP()` in DAX.

### Step 7: Create Report Pages (.json)

Each Tableau dashboard becomes one Power BI report page. Each Tableau worksheet placed on the dashboard becomes one visual on that page.

**Report layout mapping:**

Tableau dashboard zones use pixel coordinates. Power BI uses a relative canvas (default 1280 × 720 px). Convert using:

```
pbi_x = (tableau_x / dashboard_width)  * 1280
pbi_y = (tableau_y / dashboard_height) * 720
pbi_w = (tableau_w / dashboard_width)  * 1280
pbi_h = (tableau_h / dashboard_height) * 720
```

See [Visual Templates](#visual-templates) for complete JSON structures for each visual type.

**page.json base structure:**
```json
{
  "name": "<pageId>",
  "displayName": "<Dashboard Title>",
  "width": 1280,
  "height": 720,
  "defaultFilterType": "Dropdown",
  "visualContainers": []
}
```

### Step 8: Configure Model Relationships

Review every Tableau join in the workbook and create the equivalent in `relationships.tmdl`.

**Tableau join types → Power BI relationship cardinality:**

| Tableau Join | Power BI Cardinality | Cross-Filter Direction |
|-------------|----------------------|------------------------|
| Inner Join (1:M) | Many-to-One `*:1` | Single (from fact to dim) |
| Left Join (1:M) | Many-to-One `*:1` | Single |
| Full Outer Join | Many-to-Many `*:*` | Both |
| Self-join | Calculated column or DAX | N/A |

**Example relationships.tmdl:**
```
relationship 'Orders_Products'
    fromColumn: 'Orders'[ProductID]
    toColumn: 'Products'[ProductID]

relationship 'Orders_Customers'
    fromColumn: 'Orders'[CustomerID]
    toColumn: 'Customers'[CustomerID]
```

### Step 9: Apply Themes

Create `StaticResources/RegisteredResources/<ThemeName>.json` to replicate the Tableau color palette:

```json
{
  "name": "<ThemeName>",
  "dataColors": [
    "#1F4B99",
    "#2EA8E0",
    "#FF9F1C",
    "#2ECC71",
    "#E74C3C",
    "#7F8C8D",
    "#9B59B6",
    "#F39C12",
    "#3599B8",
    "#D35400"
  ],
  "background": "#FFFFFF",
  "foreground": "#252423",
  "tableAccent": "#1F4B99"
}
```

Reference the theme in `report.json`:
```json
{
  "themeCollection": {
    "baseTheme": {
      "name": "<ThemeName>",
      "reportVersionAtImport": "5.43",
      "type": "SharedResources"
    }
  }
}
```

### Step 10: Publish

```powershell
# Option A: Publish via Power BI Desktop
# File > Publish > Publish to Power BI → select workspace

# Option B: Publish via Power BI REST API (CI/CD)
$headers = @{ Authorization = "Bearer $token" }
Invoke-RestMethod -Uri "https://api.powerbi.com/v1.0/myorg/groups/<workspace_id>/imports" `
    -Method POST -Headers $headers `
    -InFile "<WorkbookName>.pbix" -ContentType "multipart/form-data"

# Option C: pbip format deployment using Power BI Project files
# Open .pbip in Power BI Desktop and publish from there
```

**What gets created:**
1. Semantic Model (dataset) in the Power BI workspace
2. Report linked to the dataset
3. Scheduled refresh (configure separately in the Service)

---

## Semantic Model Specification

### Table Definition Rules

1. **lineageTag**: Generate a unique GUID for every table, column, and measure
2. **Calculated columns**: Define inside the table block with `isDataTypeInferred: true` if type is auto-detected
3. **Measures**: Can live in their source table or in a dedicated `_Measures` table
4. **Summarization default**: Set `summarizeBy: none` on key/ID columns to prevent accidental aggregation

### DAX Measure Templates

```
// Simple aggregation
measure 'Sales Amount' = SUM('Orders'[Sales])
    formatString: \$#,0.00

// Safe division (equivalent to Tableau's ATTR or ZN()/ZN())
measure 'Profit Ratio' = DIVIDE(SUM('Orders'[Profit]), SUM('Orders'[Sales]))
    formatString: 0.00%

// Count distinct
measure 'Customer Count' = DISTINCTCOUNT('Orders'[CustomerID])
    formatString: #,0

// Year-over-year growth
measure 'YoY Growth' =
    VAR CurrentYear = [Sales Amount]
    VAR PriorYear   = CALCULATE([Sales Amount], SAMEPERIODLASTYEAR('DimDate'[Date]))
    RETURN DIVIDE(CurrentYear - PriorYear, PriorYear)
    formatString: 0.0%

// Running total (equivalent to Tableau RUNNING_SUM)
measure 'Running Sales' =
    CALCULATE(
        [Sales Amount],
        FILTER(ALLSELECTED('DimDate'[Date]),
               'DimDate'[Date] <= MAX('DimDate'[Date]))
    )
```

### Filter Context Rules

- **Tableau context filters** → Power BI `ALLEXCEPT()` or `KEEPFILTERS()` in DAX
- **Tableau fixed LOD** (`FIXED [Dim] : AGG`) → Power BI `CALCULATE(..., ALL(<other_dims>))`
- **Tableau include LOD** (`INCLUDE [Dim] : AGG`) → Granularity added to the visual, or DAX `SUMMARIZE`
- **Tableau exclude LOD** (`EXCLUDE [Dim] : AGG`) → Power BI `CALCULATE(..., ALL([Dim]))`

---

## Visual Templates

### KPI Card

Tableau `BAN` (Big Ass Number) / Single-value text mark → Power BI `card`

```json
{
  "name": "<visualId>",
  "position": { "x": 0, "y": 0, "width": 200, "height": 120 },
  "visual": {
    "visualType": "card",
    "projections": {
      "Values": [{ "queryRef": "<MeasureName>" }]
    },
    "prototypeQuery": {
      "Select": [
        { "Measure": { "Expression": { "SourceRef": { "Source": "t" } }, "Property": "<MeasureName>" },
          "Name": "t.<MeasureName>" }
      ],
      "From": [{ "Name": "t", "Entity": "<TableName>", "Type": 0 }]
    },
    "objects": {
      "labels": [{ "properties": { "show": { "expr": { "Literal": { "Value": "true" } } } } }]
    }
  }
}
```

### Bar Chart

Tableau `bar` mark → Power BI `clusteredBarChart` (horizontal) or `clusteredColumnChart` (vertical)

```json
{
  "name": "<visualId>",
  "position": { "x": 0, "y": 120, "width": 640, "height": 300 },
  "visual": {
    "visualType": "clusteredBarChart",
    "projections": {
      "Category": [{ "queryRef": "<DimensionField>" }],
      "Y":        [{ "queryRef": "<MeasureName>" }],
      "Series":   [{ "queryRef": "<ColorField>" }]
    },
    "prototypeQuery": {
      "Select": [
        { "Column": { "Expression": { "SourceRef": { "Source": "t" } }, "Property": "<DimensionField>" },
          "Name": "t.<DimensionField>" },
        { "Measure": { "Expression": { "SourceRef": { "Source": "t" } }, "Property": "<MeasureName>" },
          "Name": "t.<MeasureName>" }
      ],
      "From": [{ "Name": "t", "Entity": "<TableName>", "Type": 0 }],
      "OrderBy": [
        { "Direction": 2,
          "Expression": { "Measure": { "Expression": { "SourceRef": { "Source": "t" } }, "Property": "<MeasureName>" } } }
      ]
    }
  }
}
```

> **Tip:** Tableau sorts bars by measure value descending by default. Replicate with `OrderBy Direction: 2` (descending) in the prototypeQuery.

### Line Chart

Tableau `line` mark → Power BI `lineChart`

```json
{
  "name": "<visualId>",
  "position": { "x": 640, "y": 120, "width": 640, "height": 300 },
  "visual": {
    "visualType": "lineChart",
    "projections": {
      "Category": [{ "queryRef": "<DateField>" }],
      "Y":         [{ "queryRef": "<MeasureName>" }],
      "Series":    [{ "queryRef": "<SeriesField>" }]
    },
    "prototypeQuery": {
      "Select": [
        { "Column": { "Expression": { "SourceRef": { "Source": "t" } }, "Property": "<DateField>" },
          "Name": "t.<DateField>" },
        { "Measure": { "Expression": { "SourceRef": { "Source": "t" } }, "Property": "<MeasureName>" },
          "Name": "t.<MeasureName>" }
      ],
      "From": [{ "Name": "t", "Entity": "<TableName>", "Type": 0 }],
      "OrderBy": [
        { "Direction": 1,
          "Expression": { "Column": { "Expression": { "SourceRef": { "Source": "t" } }, "Property": "<DateField>" } } }
      ]
    },
    "objects": {
      "categoryAxis": [{ "properties": {
        "axisType": { "expr": { "Literal": { "Value": "'Continuous'" } } }
      }}]
    }
  }
}
```

### Area Chart

Tableau `area` mark → Power BI `areaChart`

Same structure as line chart with `"visualType": "areaChart"`. For **stacked** area (Tableau mark type `area` with multiple series), use `"visualType": "stackedAreaChart"`.

### Scatter Plot

Tableau `circle` / `shape` mark with two measures on rows and columns → Power BI `scatterChart`

```json
{
  "name": "<visualId>",
  "position": { "x": 0, "y": 420, "width": 640, "height": 300 },
  "visual": {
    "visualType": "scatterChart",
    "projections": {
      "X":      [{ "queryRef": "<XMeasure>" }],
      "Y":      [{ "queryRef": "<YMeasure>" }],
      "Size":   [{ "queryRef": "<SizeMeasure>" }],
      "Details":[{ "queryRef": "<DetailDimension>" }],
      "Series": [{ "queryRef": "<ColorDimension>" }]
    }
  }
}
```

> **Note:** Tableau scatter plots using `ATTR([Dimension])` for detail → use `Details` projection in Power BI. Tableau's `SIZE()` mark encoding → use `Size` projection.

### Pie / Donut Chart

Tableau `pie` mark → Power BI `pieChart` or `donutChart`

```json
{
  "name": "<visualId>",
  "position": { "x": 640, "y": 420, "width": 320, "height": 300 },
  "visual": {
    "visualType": "donutChart",
    "projections": {
      "Category": [{ "queryRef": "<CategoryField>" }],
      "Y":         [{ "queryRef": "<MeasureName>" }]
    }
  }
}
```

### Table / Crosstab

Tableau `text` mark (crosstab) → Power BI `tableEx` (flat table) or `matrix` (pivot)

**Flat table** (no row/column nesting):
```json
{
  "name": "<visualId>",
  "position": { "x": 0, "y": 420, "width": 1280, "height": 300 },
  "visual": {
    "visualType": "tableEx",
    "projections": {
      "Values": [
        { "queryRef": "<Field1>" },
        { "queryRef": "<Field2>" },
        { "queryRef": "<MeasureName>" }
      ]
    }
  }
}
```

**Pivot / crosstab** (row and column shelves both populated in Tableau):
```json
{
  "name": "<visualId>",
  "position": { "x": 0, "y": 420, "width": 1280, "height": 300 },
  "visual": {
    "visualType": "matrix",
    "projections": {
      "Rows":    [{ "queryRef": "<RowDimension>" }],
      "Columns": [{ "queryRef": "<ColDimension>" }],
      "Values":  [{ "queryRef": "<MeasureName>" }]
    }
  }
}
```

### Map Visual

Tableau `filled map` → Power BI `filledMap`
Tableau `symbol map` → Power BI `map`

```json
{
  "name": "<visualId>",
  "position": { "x": 0, "y": 0, "width": 640, "height": 400 },
  "visual": {
    "visualType": "filledMap",
    "projections": {
      "Location":  [{ "queryRef": "<GeoField>" }],
      "ColorSaturation": [{ "queryRef": "<MeasureName>" }],
      "Tooltips":  [{ "queryRef": "<TooltipMeasure>" }]
    }
  }
}
```

> **Note:** Tableau geographic roles (`Country`, `State`, `City`) map to Power BI data category. Set `dataCategory` on the column in the `.tmdl` file:
> ```
> column Country
>     dataCategory: Country
> ```

### Slicer (from Tableau Filter / Parameter)

Tableau **Quick Filter** (dropdown) → Power BI `slicer` (dropdown or list)

```json
{
  "name": "<visualId>",
  "position": { "x": 0, "y": 0, "width": 200, "height": 300 },
  "visual": {
    "visualType": "slicer",
    "projections": {
      "Field": [{ "queryRef": "<FilterField>" }]
    },
    "objects": {
      "data": [{ "properties": {
        "mode": { "expr": { "Literal": { "Value": "'Dropdown'" } } }
      }}]
    }
  }
}
```

**Tableau Parameter** (single-value numeric or string) → Power BI **What-If Parameter** slicer:
- Create a calculated table in Power BI with the parameter range
- Create a measure that uses `SELECTEDVALUE()` to read the slicer value
- This replicates Tableau `[Parameters].[<name>]` references in calculated fields

---

## Reference Tables

### Visual Type Mapping

This table covers all 24 chart types available in Tableau's **Show Me** menu, plus additional map variants documented in Tableau's visualization reference library. The Show Me types are listed first in their canonical menu order, followed by supplementary map types.

#### Show Me Chart Types (all 24)

| # | Tableau Show Me Name | Tableau Mark Type / Context | Power BI Visual Type | Migration Notes |
|---|----------------------|-----------------------------|----------------------|-----------------|
| 1 | **Text Table** (Crosstab) | `text` mark; dimensions on both Rows and Columns shelves | `tableEx` (flat) or `matrix` (pivoted) | Use `matrix` when Tableau has dimensions on both Rows and Columns (true crosstab). Use `tableEx` for flat row-level tables. Add conditional formatting in Power BI to replicate any color-coded text values. |
| 2 | **Heat Map** | `square` or `circle` mark; 1–2 measures (one on Color, one on Size); 1+ dimensions | `matrix` with conditional formatting on both background color and font size / cell size | Tableau heat maps encode two measures simultaneously (color = one measure, size = another). In Power BI, apply conditional formatting to a `matrix` for color; cell size is not independently controllable — use color intensity alone or a custom visual if size encoding is critical. |
| 3 | **Highlight Table** | `square` mark; 1 measure on Color; dimensions on Rows and Columns | `matrix` with background-color conditional formatting | Simpler than a heat map — only one measure encoded via color. Replicate exactly using Power BI `matrix` + conditional formatting (background color scale) on the value field. |
| 4 | **Symbol Map** | `circle` or `shape` mark on a map; geographic dimension + optional size/color measure | `map` (bubble / proportional symbol map) | Tableau symbol maps show proportional circles sized by a measure over a geographic background. Use Power BI's `map` visual with `Bubble size` bound to the same measure. For lat/long data, bind directly; for named geo fields set `dataCategory` on the column in `.tmdl`. |
| 5 | **Filled Map** (Choropleth) | `map` mark type with a geographic dimension (country, state, etc.) on Detail and a measure on Color | `filledMap` | Tableau filled maps shade geographic regions by a measure value using a color gradient. Power BI `filledMap` is a direct equivalent. Set `dataCategory: Country`, `StateOrProvince`, `County`, etc. on the geographic column in `.tmdl`. For sub-national granularity, fully qualify the field (e.g., `"State, Country"`) to avoid mismatches. |
| 6 | **Pie Chart** | `pie` mark; dimension on Color; measure on Angle (Size) | `pieChart` | Direct equivalent. Limit to 6–8 slices for readability, matching Tableau best practice. |
| 7 | **Horizontal Bar Chart** | `bar` mark; measure on Columns, dimension on Rows | `clusteredBarChart` | Direct equivalent (horizontal orientation). Tableau default-sorts bars by measure descending — replicate with `OrderBy Direction: 2` in the Power BI query. |
| 8 | **Stacked Bar Chart** | `bar` mark; dimension on Color shelf; one axis is a measure | `stackedBarChart` (horizontal) or `stackedColumnChart` (vertical) | Choose orientation to match the Tableau worksheet. For 100% stacked, use `hundredPercentStackedBarChart` / `hundredPercentStackedColumnChart`. |
| 9 | **Side-by-Side Bar Chart** | `bar` mark; dimension on Color AND a second dimension on the axis | `clusteredBarChart` or `clusteredColumnChart` | Side-by-side bars = clustered bars in Power BI. The Color shelf dimension becomes the Legend / Series field. |
| 10 | **Treemap** | `square` mark arranged in nested rectangles; dimension on Color and/or Detail; measure on Size | `treemap` | Direct equivalent. Tableau allows nested treemaps (two dimensions creating parent/child rectangles). Power BI `treemap` supports one level of grouping natively; use a custom visual (e.g., Sunburst) for nested hierarchies. |
| 11 | **Circle View** | `circle` mark; dimension on one axis; measure on the other; marks sized/colored | `scatterChart` (set all marks to same size) or `dotPlot` custom visual | Circle views are essentially dot plots. The closest native Power BI visual is a scatter chart with a fixed bubble size. Alternatively use the Dot Plot custom visual from AppSource. |
| 12 | **Side-by-Side Circle View** | `circle` mark; two dimensions (one on Columns, one on Rows); measure on Size | `scatterChart` with small multiples, or `matrix` with embedded scatter | Side-by-side circle views are less common; approximate with a scatter chart using the second dimension as small multiples or with a custom visual. |
| 13 | **Line Chart (Continuous)** | `line` mark; continuous date field on Columns; measure on Rows | `lineChart` | Set the axis type to `Continuous` in Power BI formatting pane. Continuous = axis is a true date scale with proportional spacing. |
| 14 | **Line Chart (Discrete)** | `line` mark; discrete (blue pill) date field on Columns; measure on Rows | `lineChart` with categorical X axis | Discrete date pills in Tableau (e.g., MONTH([Date]) as a discrete header) become a categorical/text axis. Replicate by formatting the axis type as Categorical in Power BI. |
| 15 | **Dual Line Chart** | `line` mark; two separate measures each on its own Y axis (dual axis) | `lineClusteredColumnComboChart` or `lineStackedColumnComboChart` | Power BI combo charts support a secondary Y axis. Assign one measure to the line and one to the column, then switch the column to a line in format settings. True dual-line (both as lines) is not natively supported — use a custom visual or overlay two charts if needed. |
| 16 | **Area Chart (Continuous)** | `area` mark; continuous date on Columns; single or multiple measures stacked | `areaChart` (single series) or `stackedAreaChart` (multiple series) | Continuous date axis = set axis to Continuous in Power BI. |
| 17 | **Area Chart (Discrete)** | `area` mark; discrete date on Columns | `areaChart` with categorical X axis | Same as discrete line chart — format axis as Categorical. |
| 18 | **Dual Combination Chart** | Combo mark; two measures, one as bar and one as line on dual axes | `lineClusteredColumnComboChart` | This is Tableau's primary "combo chart" Show Me type. Power BI combo charts support one line and one bar/column series per secondary axis. |
| 19 | **Scatter Plot** | `circle` mark; one measure on Columns, another on Rows; optional dimension on Color/Detail | `scatterChart` | Direct equivalent. Tableau's scatter plots often use `ATTR([Dimension])` for detail → use the `Details` projection. Size encoding → `Bubble Size` projection. |
| 20 | **Histogram** | `bar` mark; single continuous measure with auto-created bin field on Columns; COUNT on Rows | `columnChart` using a binned calculated column, or Power BI's native histogram bin via field formatting | Power BI does not have a one-click histogram Show Me equivalent. Create a bin calculated column in `.tmdl` using integer division (`INT([value] / <bin_size>) * <bin_size>`) and then use a `clusteredColumnChart` with count. |
| 21 | **Box-and-Whisker Plot** | `circle` mark (disaggregated) with box plot applied; dimension on Columns; measure on Rows | `boxWhiskerChart` (custom visual from AppSource) or approximation with error bars on a `columnChart` | There is no native box-and-whisker in standard Power BI visuals. The "Box and Whisker Chart" from Microsoft AppSource is the closest equivalent. Alternatively, pre-compute quartiles in DAX (Q1, Q2, Q3, whisker min/max) and render as a layered bar chart with error bars. |
| 22 | **Gantt Chart** | `gantt bar` mark; date on Columns; dimension on Rows; measure on Size (duration) | `clusteredBarChart` (horizontal, two measures: start and duration) | Power BI has no native Gantt. Approximation: create two calculated columns (start position offset and bar width as duration), then use a clustered horizontal bar chart. For a true Gantt, use the "Gantt Chart" custom visual from AppSource. |
| 23 | **Bullet Graph** | `bar` mark with reference lines and reference distributions; measure (actual) + measure (target) | `bulletChart` custom visual (AppSource), or `clusteredBarChart` with a reference line constant | Power BI has no native bullet graph. The Bullet Chart custom visual from AppSource is the closest. Alternatively, use a clustered bar chart with a target measure displayed as a data label or constant line overlay. |
| 24 | **Packed Bubbles** | `circle` mark without axes; dimension on Color/Detail; measure on Size; circles packed into the available space | `treemap` (size-based, no axes) or `scatterChart` with fixed positions, or "Bubble Chart" custom visual | Packed bubble charts have no native Power BI equivalent. A `treemap` conveys a similar size-as-measure encoding in a structured layout. For a true packed bubble appearance, use the "Bubble Chart" or "Aquarium" custom visual from AppSource. |

#### Additional Map Types (beyond Show Me)

These map types are available in Tableau beyond the standard Show Me menu and require specific handling in Power BI:

| Tableau Map Type | Description | Power BI Equivalent | Migration Notes |
|-----------------|-------------|---------------------|----------------|
| **Choropleth / Filled Map** | Shaded geographic regions (country, state, county) colored by a measure | `filledMap` | See row 5 above. The Tableau docs distinguish choropleth (boundary-based shading) from isopleth (contour-based). Power BI `filledMap` is choropleth only. |
| **Isopleth / Density Area Map** | Contour map showing continuous density gradients across a region, not bound by political boundaries (e.g., weather radar) | No native equivalent | Power BI does not support isopleth/contour maps natively. Use Azure Maps custom visual or export data to a dedicated mapping tool (e.g., ArcGIS, Mapbox). The ArcGIS Maps for Power BI visual (built-in) supports heat density overlays as a partial substitute. |
| **Symbol Map (Proportional)** | Circles sized by a measure placed at geographic points | `map` with Bubble Size | See row 4 above. |
| **Point Distribution Map** | Individual data points plotted as dots at geographic coordinates (no sizing) | `map` with fixed small bubble size | Bind lat/long or geo field to the map visual; set bubble size to a constant or a very small measure. |
| **Density Map (Geographic Heatmap)** | Heat density overlay on a geographic map showing concentration of overlapping data points (e.g., taxi pickups per area) | ArcGIS Maps for Power BI or Azure Maps custom visual | Tableau's density mark type on a map creates a kernel density heat overlay. Power BI's built-in `map` visual does not support this natively; ArcGIS Maps for Power BI provides a heat map layer option. |
| **Flow Map / Path Map** | Lines connecting geographic points to show movement or paths over time (e.g., storm tracks, trade routes) | No native equivalent | Use the "Route Map" or "Icon Map" custom visuals from AppSource. Alternatively, pre-compute path geometry and render as a custom shape layer in ArcGIS Maps for Power BI. |
| **Tile Grid Map (Shape Map)** | Equal-sized tile representation of geographic regions to avoid visual bias from area size differences (e.g., US state grid) | `shapeMap` (Power BI Shape Map visual, requires custom TopoJSON) | Power BI's Shape Map visual supports custom region definitions via TopoJSON files. Download or create a tile-grid TopoJSON for your regions and bind the geographic dimension. |

### Tableau Function Reference & DAX / Power Query M Equivalents

The sub-sections below cover every function category from the Tableau help documentation (alphabetical and categorical references). Each entry shows the Tableau syntax, a brief description, and the Power BI DAX equivalent (or Power Query M equivalent where the conversion happens at data-prep time). Functions with no reasonable Power BI equivalent are flagged.

---

#### Aggregate Functions

These functions aggregate values across rows. In Tableau they are drag-and-drop; in Power BI they become DAX measures.

| Tableau Syntax | Output | Description | Power BI DAX Equivalent |
|----------------|--------|-------------|------------------------|
| `SUM(expression)` | Number | Sum of all values; nulls ignored | `SUM('Table'[col])` |
| `AVG(expression)` | Number | Average of all values; nulls ignored | `AVERAGE('Table'[col])` |
| `COUNT(expression)` | Integer | Count of non-null items | `COUNT('Table'[col])` |
| `COUNTD(expression)` | Integer | Count of distinct non-null items | `DISTINCTCOUNT('Table'[col])` |
| `MIN(expression)` | Varies | Minimum value; works on numbers and dates | `MIN('Table'[col])` |
| `MAX(expression)` | Varies | Maximum value; works on numbers and dates | `MAX('Table'[col])` |
| `MEDIAN(expression)` | Number | Median value; nulls ignored | `MEDIAN('Table'[col])` |
| `PERCENTILE(expression, number)` | Number | Percentile value (number must be 0–1 constant, e.g. 0.9) | `PERCENTILEX.INC(ALL('Table'), 'Table'[col], 0.9)` |
| `STDEV(expression)` | Number | Sample standard deviation | `STDEV.S('Table'[col])` or `STDEVX.S(ALL('Table'), 'Table'[col])` |
| `STDEVP(expression)` | Number | Population (biased) standard deviation | `STDEV.P('Table'[col])` |
| `VAR(expression)` | Number | Sample variance | `VAR.S('Table'[col])` or `VARX.S(ALL('Table'), 'Table'[col])` |
| `VARP(expression)` | Number | Population (biased) variance | `VAR.P('Table'[col])` |
| `CORR(expr1, expr2)` | Number (−1 to 1) | Pearson correlation coefficient between two expressions | No direct DAX aggregate; use `WINDOW_CORR` pattern or calculate in SQL |
| `COVAR(expr1, expr2)` | Number | Sample covariance of two expressions | No direct DAX aggregate; pre-compute in Power Query or SQL |
| `COVARP(expr1, expr2)` | Number | Population covariance of two expressions | No direct DAX aggregate; pre-compute in Power Query or SQL |
| `ATTR(expression)` | Varies or `*` | Returns value if single for all rows, else `*` (asterisk) | `IF(DISTINCTCOUNT('Table'[col]) = 1, MIN('Table'[col]), BLANK())` |
| `COLLECT(spatial)` | Geometry | Aggregate that combines spatial field values | No equivalent; Power BI does not aggregate spatial objects natively |

---

#### Number (Math) Functions

| Tableau Syntax | Output | Description | Power BI DAX Equivalent |
|----------------|--------|-------------|------------------------|
| `ABS(number)` | Number | Absolute value | `ABS([measure])` |
| `CEILING(number)` | Integer | Round up to nearest integer | `CEILING([measure], 1)` |
| `FLOOR(number)` | Integer | Round down to nearest integer | `FLOOR([measure], 1)` |
| `ROUND(number, decimals)` | Number | Round to specified decimal places | `ROUND([measure], decimals)` |
| `DIV(integer1, integer2)` | Integer | Integer quotient (floor division) | `INT([int1] / [int2])` |
| `SQRT(number)` | Number | Square root | `SQRT([measure])` |
| `SQUARE(number)` | Number | Square of a number | `[measure] ^ 2` or `POWER([measure], 2)` |
| `POWER(number, power)` | Number | Number raised to power | `POWER([measure], power)` |
| `EXP(number)` | Number | e raised to the power of number | `EXP([measure])` |
| `LN(number)` | Number | Natural logarithm; null if ≤ 0 | `LN([measure])` |
| `LOG(number, [base])` | Number | Logarithm; default base 10 | `LOG([measure], base)` |
| `PI()` | Number | Returns 3.14159… | `PI()` |
| `SIGN(number)` | −1 / 0 / 1 | Sign of a number | `SIGN([measure])` |
| `MIN(a, b)` | Varies | Minimum of two values (non-aggregate form) | `MIN([a], [b])` |
| `MAX(a, b)` | Varies | Maximum of two values (non-aggregate form) | `MAX([a], [b])` |
| `DEGREES(number)` | Number | Converts radians to degrees | `DEGREES([measure])` |
| `RADIANS(number)` | Number | Converts degrees to radians | `RADIANS([measure])` |
| `SIN(number)` | Number | Sine of angle in radians | `SIN([measure])` |
| `COS(number)` | Number | Cosine of angle in radians | `COS([measure])` |
| `TAN(number)` | Number | Tangent of angle in radians | `TAN([measure])` |
| `ASIN(number)` | Number | Arcsine; result in radians | `ASIN([measure])` |
| `ACOS(number)` | Number | Arccosine; result in radians | `ACOS([measure])` |
| `ATAN(number)` | Number | Arctangent; result in radians | `ATAN([measure])` |
| `ATAN2(y, x)` | Number | Arctangent of y/x; result in radians | `ATAN2([y], [x])` (not native; use `ATAN([y]/[x])` with quadrant logic) |
| `COT(number)` | Number | Cotangent of angle in radians | `1 / TAN([measure])` |
| `HEXBINX(x, y)` | Number | X coordinate of nearest hexagonal bin | No direct equivalent; custom visual or pre-compute |
| `HEXBINY(x, y)` | Number | Y coordinate of nearest hexagonal bin | No direct equivalent; custom visual or pre-compute |

---

#### String Functions

| Tableau Syntax | Output | Description | Power BI DAX Equivalent | Power Query M (data-prep) |
|----------------|--------|-------------|------------------------|--------------------------|
| `LEN(string)` | Number | Length of string | `LEN([col])` | `Text.Length([col])` |
| `LEFT(string, n)` | String | Leftmost n characters | `LEFT([col], n)` | `Text.Start([col], n)` |
| `RIGHT(string, n)` | String | Rightmost n characters | `RIGHT([col], n)` | `Text.End([col], n)` |
| `MID(string, start, [length])` | String | Substring from position start | `MID([col], start, length)` | `Text.Middle([col], start-1, length)` |
| `UPPER(string)` | String | Convert to uppercase | `UPPER([col])` | `Text.Upper([col])` |
| `LOWER(string)` | String | Convert to lowercase | `LOWER([col])` | `Text.Lower([col])` |
| `PROPER(string)` | String | Title-case each word | `PROPER([col])` | `Text.Proper([col])` |
| `TRIM(string)` | String | Remove leading and trailing spaces | `TRIM([col])` | `Text.Trim([col])` |
| `LTRIM(string)` | String | Remove leading spaces only | `-- no direct; use` `TRIM` or regex | `Text.TrimStart([col])` |
| `RTRIM(string)` | String | Remove trailing spaces only | `-- no direct; use` `TRIM` or regex | `Text.TrimEnd([col])` |
| `SPACE(number)` | String | Returns a string of n spaces | `REPT(" ", n)` | `Text.Repeat(" ", n)` |
| `ASCII(string)` | Number | ASCII code of first character | `CODE([col])` | `Character.ToNumber(Text.At([col], 0))` |
| `CHAR(number)` | String | Character for ASCII code | `UNICHAR(number)` | `Character.FromNumber(number)` |
| `CONTAINS(string, substring)` | Boolean | True if string contains substring | `CONTAINSSTRING([col], "sub")` | `Text.Contains([col], "sub")` |
| `STARTSWITH(string, substring)` | Boolean | True if string starts with substring | `LEFT([col], LEN("sub")) = "sub"` | `Text.StartsWith([col], "sub")` |
| `ENDSWITH(string, substring)` | Boolean | True if string ends with substring (trailing whitespace ignored) | `RIGHT([col], LEN("sub")) = "sub"` | `Text.EndsWith([col], "sub")` |
| `FIND(string, substring, [start])` | Number | Position of first occurrence; 0 if not found | `FIND("sub", [col], start, 0)` | `Text.PositionOf([col], "sub")` |
| `FINDNTH(string, substring, n)` | Number | Position of nth occurrence | No direct DAX; use custom M column | Custom M: loop with `Text.PositionOf` and offset |
| `REPLACE(string, old, new)` | String | Replace all occurrences of old with new | `SUBSTITUTE([col], "old", "new")` | `Text.Replace([col], "old", "new")` |
| `SPLIT(string, delimiter, token)` | String | Returns the nth token after splitting by delimiter | No direct DAX; pre-compute in Power Query | `Text.Split([col], delim){token-1}` |
| `STR(expression)` | String | Converts expression to string | `FORMAT([col], "General")` | `Text.From([col])` |
| `INT(expression)` | Integer | Casts to integer (truncates toward zero) | `INT([col])` or `TRUNC([col], 0)` | `Int64.From([col])` |
| `FLOAT(expression)` | Decimal | Casts to floating-point number | `CONVERT([col], DOUBLE)` | `Number.From([col])` |

---

#### Date Functions

Valid `date_part` values in Tableau: `'year'`, `'quarter'`, `'month'`, `'week'`, `'day'`, `'hour'`, `'minute'`, `'second'`, `'dayofyear'`, `'weekday'`, ISO variants (`'iso-year'`, `'iso-quarter'`, `'iso-week'`, `'iso-weekday'`).

| Tableau Syntax | Output | Description | Power BI DAX Equivalent |
|----------------|--------|-------------|------------------------|
| `DATE(expression)` | Date | Converts string/number/datetime to date | `DATE(YEAR([col]), MONTH([col]), DAY([col]))` or `DATEVALUE([col])` |
| `DATETIME(expression)` | Datetime | Converts to datetime | `CONVERT([col], DATETIME)` |
| `TODAY()` | Date | Current date | `TODAY()` |
| `NOW()` | Datetime | Current local date and time | `NOW()` |
| `YEAR(date)` | Integer | Year component (1–9999) | `YEAR([date])` |
| `QUARTER(date)` | Integer | Quarter (1–4) | `QUARTER([date])` |
| `MONTH(date)` | Integer | Month number (1–12) | `MONTH([date])` |
| `WEEK(date)` | Integer | Week of year (1–53) | `WEEKNUM([date])` |
| `DAY(date)` | Integer | Day of month (1–31) | `DAY([date])` |
| `ISOYEAR(date)` | Integer | ISO 8601 week-based year | `-- No direct; use` `WEEKNUM([date], 21)` pattern |
| `ISOQUARTER(date)` | Integer | ISO 8601 week-based quarter | No direct DAX equivalent; compute from `ISOWEEK` |
| `ISOWEEK(date)` | Integer | ISO 8601 week number | `WEEKNUM([date], 21)` |
| `ISOWEEKDAY(date)` | Integer | ISO 8601 weekday (Monday=1) | `WEEKDAY([date], 2)` |
| `DATEPART(date_part, date, [start_of_week])` | Integer | Returns date part as integer | `YEAR/MONTH/DAY/QUARTER/WEEKNUM/WEEKDAY([date])` |
| `DATENAME(date_part, date, [start_of_week])` | String | Returns date part as string | `FORMAT([date], "MMMM")` / `"yyyy"` etc. |
| `DATETRUNC(date_part, date, [start_of_week])` | Date | Truncates date to specified precision (like rounding down) | `DATE(YEAR([d]), MONTH([d]), 1)` for month; `STARTOFQUARTER([d])` for quarter; etc. |
| `DATEADD(date_part, interval, date)` | Date | Adds interval date parts to a date | `DATEADD([date], interval, date_part)` — DAX `DATEADD` takes different args; or `EDATE([d], n)` for months |
| `DATEDIFF(date_part, date1, date2, [start_of_week])` | Integer | Difference between dates in specified units | `DATEDIFF([date1], [date2], date_part)` |
| `DATEPARSE(format, string)` | Date | Parses a string to date using a specific format | `DATEVALUE([col])` or `DATE(LEFT([col],4), MID([col],6,2), RIGHT([col],2))` |
| `MAKEDATE(year, month, day)` | Date | Constructs a date from year, month, day integers | `DATE(year, month, day)` |
| `MAKEDATETIME(date, time)` | Datetime | Combines date and datetime (MySQL-compatible only) | `[date] + TIME(HOUR([time]), MINUTE([time]), SECOND([time]))` |
| `MAKETIME(hour, minute, second)` | Datetime | Constructs a time value | `TIME(hour, minute, second)` |
| `ISDATE(string)` | Boolean | True if string is a valid date | `IFERROR(DATEVALUE([col]) > 0, FALSE)` |

---

#### Logical Functions

| Tableau Syntax | Output | Description | Power BI DAX Equivalent |
|----------------|--------|-------------|------------------------|
| `IF test THEN a [ELSEIF test2 THEN b ...] [ELSE c] END` | Varies | Multi-branch conditional | `IF(test, a, IF(test2, b, c))` |
| `IIF(test, then, else, [unknown])` | Varies | Inline if; optional 4th arg handles NULL test | `IF(test, then, else)` — use `ISBLANK` to handle nulls explicitly |
| `CASE expr WHEN v1 THEN r1 [ELSE d] END` | Varies | Switch/case on expression | `SWITCH([col], v1, r1, d)` |
| `ELSEIF test THEN result` | — | Clause within IF block | Nested `IF(...)` in DAX |
| `ELSE default` | — | Default clause for IF/CASE | Last argument in DAX `IF`/`SWITCH` |
| `END` | — | Closes IF/CASE block | Not needed in DAX (functions use parentheses) |
| `WHEN value THEN result` | — | Clause within CASE block | Middle arguments in `SWITCH` |
| `THEN result` | — | Keyword in IF/CASE | Part of `IF(cond, result, ...)` |
| `AND` | Boolean | Logical AND; short-circuit evaluation | `&&` operator or `AND(cond1, cond2)` |
| `OR` | Boolean | Logical OR | `\|\|` operator or `OR(cond1, cond2)` |
| `NOT` | Boolean | Logical NOT | `NOT(cond)` |
| `IN` | Boolean | True if expr matches any value in a list or set | `[col] IN {v1, v2}` — use `CONTAINSROW` or `OR(... = v1, ... = v2)` |
| `ISNULL(expression)` | Boolean | True if expression is NULL | `ISBLANK([col])` |
| `IFNULL(expr1, expr2)` | Varies | Returns expr1 if non-null, else expr2 | `IF(ISBLANK([expr1]), [expr2], [expr1])` |
| `ZN(expression)` | Number | Returns expression if not null, else 0 | `IF(ISBLANK([measure]), 0, [measure])` |
| `ISDATE(string)` | Boolean | True if string is a valid date | See Date Functions above |

---

#### LOD (Level of Detail) Expressions

LOD expressions are Tableau's most powerful—and most complex—feature to migrate. They override the default level of aggregation.

| Tableau LOD Syntax | Description | Power BI DAX Equivalent |
|--------------------|-------------|------------------------|
| `{ FIXED [Dim1], [Dim2] : AGG([Measure]) }` | Computes aggregate at the specified dimension level, ignoring viz-level filters (except context filters) | `CALCULATE([Measure], ALL('Table'), ALLEXCEPT('Table', 'Table'[Dim1], 'Table'[Dim2]))` |
| `{ INCLUDE [Dim] : AGG([Measure]) }` | Adds a dimension to the viz's granularity before aggregating | Add `[Dim]` to the visual's row/group context, or use `SUMMARIZE` in a calculated table |
| `{ EXCLUDE [Dim] : AGG([Measure]) }` | Removes a dimension from the viz's granularity before aggregating | `CALCULATE([Measure], ALL('Table'[Dim]))` |
| `{ FIXED : MAX([Date]) }` | Grand total LOD — ignores all dimensions | `CALCULATE(MAX('Table'[Date]), ALL('Table'))` |
| `{ FIXED [Customer] : MIN([Order Date]) }` | First order date per customer (common cohort pattern) | `CALCULATE(MIN('Table'[Order Date]), ALLEXCEPT('Table', 'Table'[Customer]))` |

> **Migration warning:** Tableau LOD expressions are evaluated *before* dimension filters but *after* context filters (blue pills). Power BI `CALCULATE` with `ALL`/`ALLEXCEPT` mimics fixed LODs but responds differently to slicers. Always validate row-by-row against Tableau output.

---

#### Table Calculation Functions

Table calculations are post-aggregation window functions computed across the *partition* (a subset of the table defined by "Compute Using"). They have no simple single-function DAX equivalent — each requires a `CALCULATE` + `FILTER(ALLSELECTED(...))` pattern.

| Tableau Syntax | Description | Power BI DAX Pattern |
|----------------|-------------|---------------------|
| `FIRST()` | Offset from current row to first row in partition (negative) | `ROW_NUMBER()` style via `RANKX` with reversed sort |
| `LAST()` | Offset from current row to last row in partition (positive) | `COUNTROWS(ALLSELECTED(...)) - RANKX(...)` |
| `INDEX()` | Row index within partition (1-based) | `RANKX(ALLSELECTED('Table'), [sort_field], , ASC, Dense)` |
| `SIZE()` | Number of rows in partition | `COUNTROWS(ALLSELECTED('Table'))` |
| `RANK(expr, ['asc'\|'desc'])` | Standard competition rank (ties share rank, gaps follow) | `RANKX(ALL('Table'), [Measure])` |
| `RANK_DENSE(expr, ['asc'\|'desc'])` | Dense rank (no gaps after ties) | `RANKX(ALL('Table'), [Measure], , , Dense)` |
| `RANK_MODIFIED(expr, ['asc'\|'desc'])` | Modified competition rank (ties get last position) | No direct; pre-compute in a calculated table |
| `RANK_PERCENTILE(expr, ['asc'\|'desc'])` | Percentile rank (0.0–1.0) | `PERCENTILEX.INC(ALL('Table'), 'Table'[col], ...)` |
| `RANK_UNIQUE(expr, ['asc'\|'desc'])` | Unique rank (no ties; arbitrary tiebreak) | `RANKX(ALL('Table'), [Measure], , , Skip)` with row-level tie-break col |
| `LOOKUP(expr, offset)` | Value from a row at relative offset | No direct; use `OFFSET` in DAX (2022+) or pre-compute in Power Query |
| `PREVIOUS_VALUE(expr)` | Value from previous row (running product pattern) | `CALCULATE([Measure], FILTER(ALLSELECTED('DimDate'), 'DimDate'[Date] = EARLIER('DimDate'[Date]) - 1))` |
| `RUNNING_SUM(expr)` | Cumulative sum from first to current row | `CALCULATE(SUM('T'[col]), FILTER(ALLSELECTED('D'[Date]), 'D'[Date] <= MAX('D'[Date])))` |
| `RUNNING_AVG(expr)` | Cumulative average from first to current row | Same `CALCULATE` pattern with `AVERAGEX` |
| `RUNNING_COUNT(expr)` | Cumulative count | Same pattern with `COUNTROWS` |
| `RUNNING_MAX(expr)` | Running maximum | `CALCULATE(MAX('T'[col]), FILTER(ALLSELECTED(...), date <= MAX(date)))` |
| `RUNNING_MIN(expr)` | Running minimum | Same pattern with `MIN` |
| `TOTAL(expr)` | Grand total of expression in the partition | `CALCULATE([Measure], ALL('Table'[partition_dim]))` |
| `WINDOW_SUM(expr, [start, end])` | Sum within a window of rows | Rolling sum: `CALCULATE(SUM(...), FILTER(ALLSELECTED(...), date >= date-n && date <= date))` |
| `WINDOW_AVG(expr, [start, end])` | Average within a window | Same pattern with `AVERAGEX` |
| `WINDOW_MIN(expr, [start, end])` | Minimum within a window | Same pattern with `MIN` |
| `WINDOW_MAX(expr, [start, end])` | Maximum within a window | Same pattern with `MAX` |
| `WINDOW_COUNT(expr, [start, end])` | Count within a window | Same pattern with `COUNTROWS` |
| `WINDOW_MEDIAN(expr, [start, end])` | Median within a window | Same pattern with `MEDIANX` |
| `WINDOW_PERCENTILE(expr, pct, [start, end])` | Percentile within a window | Same pattern with `PERCENTILEX.INC` |
| `WINDOW_STDEV(expr, [start, end])` | Sample std dev within a window | Same pattern with `STDEVX.S` |
| `WINDOW_STDEVP(expr, [start, end])` | Population std dev within a window | Same pattern with `STDEVX.P` |
| `WINDOW_VAR(expr, [start, end])` | Sample variance within a window | Same pattern with `VARX.S` |
| `WINDOW_VARP(expr, [start, end])` | Population variance within a window | Same pattern with `VARX.P` |
| `WINDOW_CORR(expr1, expr2, [start, end])` | Pearson correlation within a window | Pre-compute in SQL or Power Query; no native DAX window corr |
| `WINDOW_COVAR(expr1, expr2, [start, end])` | Sample covariance within a window | Pre-compute in SQL or Power Query |
| `WINDOW_COVARP(expr1, expr2, [start, end])` | Population covariance within a window | Pre-compute in SQL or Power Query |
| `MODEL_PERCENTILE(target, predictor(s))` | Probability (0–1) from Bayesian model | No equivalent; use Python/R visual or Azure ML integration |
| `MODEL_QUANTILE(quantile, target, predictor(s))` | Predicted value at given quantile | No equivalent; use Python/R visual or Azure ML integration |
| `SCRIPT_BOOL / INT / REAL / STR(script, args)` | Pass expression to external R/Python/Matlab | Use Power BI Python/R visual or call Azure ML scoring endpoint |
| `MODEL_EXTENSION_BOOL/INT/REAL/STRING` | Pass data to a deployed TabPy model | Use Power BI Python/R visual script or Azure ML endpoint |

---

#### User / Security Functions

These replicate Tableau's row-level security (RLS) and user-filter patterns.

| Tableau Syntax | Output | Description | Power BI DAX Equivalent |
|----------------|--------|-------------|------------------------|
| `USERNAME()` | String | Current user's username | `USERPRINCIPALNAME()` |
| `FULLNAME()` | String | Current user's full name | `-- Not directly available; USERPRINCIPALNAME() returns email` |
| `USERDOMAIN()` | String | Current user's domain | `-- Not available; use USERPRINCIPALNAME()` |
| `ISMEMBEROF("Group Name")` | Boolean | True if current user is in the specified group | `-- Use RLS roles in Power BI Service; no runtime function` |
| `ISUSERNAME("username")` | Boolean | True if current user matches specified username | `USERPRINCIPALNAME() = "user@domain.com"` |
| `ISFULLNAME("Full Name")` | Boolean | True if current user's full name matches | No direct equivalent; use `USERPRINCIPALNAME()` comparison |

> **Migration pattern for Tableau user filters:** In Tableau, `[User Field] = USERNAME()` creates a row-level filter. In Power BI, implement this as a **Row-Level Security (RLS) role** in the semantic model: define a DAX filter `'Table'[UserField] = USERPRINCIPALNAME()` on the table. Apply the role in Power BI Service under Dataset Settings → Security.

---

#### Spatial Functions

These Tableau functions have limited Power BI equivalents. Power BI's spatial capabilities are primarily handled through the map visuals or custom visuals (ArcGIS Maps).

| Tableau Syntax | Output | Description | Power BI Equivalent |
|----------------|--------|-------------|---------------------|
| `MAKEPOINT(lat, long, [SRID])` | Geometry | Creates a spatial point from lat/long | Set `dataCategory: Latitude` and `Longitude` on columns; Power BI map uses them automatically |
| `MAKELINE(point1, point2)` | Geometry (line) | Line between two spatial points | No native equivalent; use ArcGIS Maps or Mapbox custom visual |
| `DISTANCE(point1, point2, 'units')` | Number | Distance between two points | Pre-compute in Power Query using Haversine formula |
| `AREA(polygon, 'units')` | Number | Surface area of a spatial polygon | Pre-compute in Power Query or SQL with spatial functions |
| `LENGTH(geometry, 'units')` | Number | Geodetic path length of line geometry | Pre-compute in SQL or custom column |
| `BUFFER(point, distance, 'units')` | Geometry | Polygon of radius around a point | No native equivalent; use ArcGIS Maps buffer analysis |
| `INTERSECTS(geometry1, geometry2)` | Boolean | True if two geometries overlap | No native DAX; pre-compute in SQL using spatial joins |
| `COLLECT(spatial)` | Geometry | Aggregates spatial values | No equivalent |
| `HEXBINX(x, y)` | Number | X of nearest hex bin | Pre-compute in Power Query or use a custom visual |
| `HEXBINY(x, y)` | Number | Y of nearest hex bin | Pre-compute in Power Query or use a custom visual |

---

#### Type Conversion Functions

| Tableau Syntax | Output | Description | Power BI DAX Equivalent | Power Query M |
|----------------|--------|-------------|------------------------|---------------|
| `STR(expression)` | String | Converts to string | `FORMAT([col], "General")` | `Text.From([col])` |
| `INT(expression)` | Integer | Converts to integer (truncates) | `INT([col])` | `Int64.From([col])` |
| `FLOAT(expression)` | Decimal | Converts to float | `CONVERT([col], DOUBLE)` | `Number.From([col])` |
| `DATE(expression)` | Date | Converts to date | `DATEVALUE([col])` | `Date.From([col])` |
| `DATETIME(expression)` | Datetime | Converts to datetime | `CONVERT([col], DATETIME)` | `DateTime.From([col])` |

---

#### Power Query M Conversion Reference

For transformations applied at data-load time (equivalent to Tableau's Data Source / Extract-level preparation):

| Tableau Transformation | Power Query M Equivalent |
|------------------------|--------------------------|
| `Text.Trim([col])` → `TRIM` | `Text.Trim([col])` |
| `Text.Upper([col])` → `UPPER` | `Text.Upper([col])` |
| `Text.Lower([col])` → `LOWER` | `Text.Lower([col])` |
| `Text.Proper([col])` → `PROPER` | `Text.Proper([col])` |
| `Text.Replace([col], "a", "b")` → `REPLACE` | `Text.Replace([col], "a", "b")` |
| `Text.Start([col], n)` → `LEFT` | `Text.Start([col], n)` |
| `Text.End([col], n)` → `RIGHT` | `Text.End([col], n)` |
| `Text.Middle([col], s, n)` → `MID` | `Text.Middle([col], s, n)` |
| `Text.Length([col])` → `LEN` | `Text.Length([col])` |
| `Text.Split([col], delim){n}` → `SPLIT` | `Text.Split([col], delim){n}` |
| Combine columns → `col1 & " " & col2` | `[col1] & " " & [col2]` |
| `DateTime.Date([col])` → Date from datetime | `DateTime.Date([col])` |
| `Date.Year([col])` → Year | `Date.Year([col])` |
| `Date.Month([col])` → Month number | `Date.Month([col])` |
| `Date.ToText([col], "MMM")` → Month name | `Date.ToText([col], "MMM")` |
| `"Q" & Text.From(Date.QuarterOfYear([col]))` → Quarter | `"Q" & Text.From(Date.QuarterOfYear([col]))` |
| `Date.DayOfWeek([col])` → Weekday number | `Date.DayOfWeek([col])` |
| `Table.AddColumn(src, "col", each ...)` → Calculated column | `Table.AddColumn(src, "col", each ...)` |
| `Table.TransformColumns(src, {{"col", fn, type}})` → Column transform | `Table.TransformColumns(src, {{"col", fn, type}})` |
| `Table.Pivot(...)` | `Table.Pivot(...)` |
| `Table.Unpivot(...)` | `Table.Unpivot(...)` |
| `Table.SplitColumn(...)` | `Table.SplitColumn(...)` |
| `Number.RoundDown([val] / bin) * bin` → Binning | `Number.RoundDown([val] / binSize) * binSize` |

---

## Troubleshooting

### LOD Expressions Are Complex to Migrate
Tableau Level of Detail (LOD) expressions are among the hardest things to replicate. Always validate the DAX equivalent produces the same row-level numbers against Tableau's output before finalizing.

- For `FIXED` LODs: use `CALCULATE` with `ALL()` to remove the filter context for non-fixed dimensions
- For nested LODs: consider materializing the intermediate result as a calculated table

### Tableau Blending vs. Power BI Relationships
Tableau data blending (linking on a common dimension between primary and secondary data sources) is **not the same** as a Power BI relationship. In Power BI, you must:
1. Load both sources as tables in the semantic model
2. Create an explicit relationship in `relationships.tmdl`
3. Note that blending uses `ATTR()` — in Power BI, use `MIN()` or `MAX()` for single-value columns

### Table Calculations Cannot Be Pre-Aggregated
Tableau table calculations (RUNNING_SUM, RANK, WINDOW_AVG, etc.) are computed post-aggregation in the viz. In Power BI, DAX measures are evaluated per visual context but without windowing functions by default. Use `RANKX`, `EARLIER`, or `CALCULATE` + `FILTER(ALLSELECTED(...))` patterns. Test carefully with complex calculations.

### Continuous vs. Discrete Dates
Tableau's date pill can be set to continuous (axis) or discrete (header). In Power BI:
- **Continuous date axis** → Set `axisType: Continuous` in the visual's category axis object
- **Discrete date header** → Use a text column like `"MMM YYYY"` as the axis field

### Dual-Axis Charts
Tableau supports dual-axis natively on any pair of measures. In Power BI, use `lineClusteredColumnComboChart` or `lineStackedColumnComboChart`. You are limited to one line and one bar measure type per combo chart — more complex dual-axis combinations may require separate stacked visuals.

### Custom SQL in Tableau → Custom Query in Power BI
Tableau custom SQL connections:
```xml
<connection>
  <relation name='Custom SQL' type='text'>
    <text>SELECT ... FROM ... WHERE ...</text>
  </relation>
</connection>
```
Equivalent in Power Query M:
```
source = Value.NativeQuery(Sql.Database("<server>", "<database>"),
    "SELECT ... FROM ... WHERE ...", null, [EnableFolding=true])
```

### Sets and Groups
- **Tableau Sets** (IN/OUT binary): Create a calculated column in Power BI with `IF(condition, "IN", "OUT")`
- **Tableau Groups** (renamed members): Create a calculated column with `SWITCH()` in DAX or a mapping table
- **Tableau Bins** (numeric ranges): Use `SWITCH(TRUE(), [value] < 10, "0-9", [value] < 20, "10-19", "20+")` in DAX or `Number.RoundDown([value] / <bin_size>) * <bin_size>` in M

### Actions / Cross-Filter Behavior
Tableau **filter actions** (click on a mark to filter another sheet) are analogous to Power BI's built-in **cross-filtering**. Enable cross-filter between visuals in Power BI's Format pane under "Edit interactions." Power BI cross-filtering is on by default; no manual action setup needed.

Tableau **highlight actions** → Power BI does not have a direct highlight action equivalent; cross-highlighting (keeping all marks visible but dimming unselected ones) is the default Power BI interaction.

### Validation Checklist

Before publishing:
- [ ] All Tableau calculated fields have DAX equivalents verified against sample data
- [ ] All data source connections point to the correct server/database/table
- [ ] All model relationships have the correct cardinality and cross-filter direction
- [ ] All LOD expressions validated row-by-row against Tableau output
- [ ] Report pages match dashboard layout (visual positions, sizes, titles)
- [ ] Slicers cover all Tableau quick filters and relevant parameters
- [ ] Conditional formatting replicates Tableau color encoding where used
- [ ] Date hierarchies behave consistently (year → quarter → month → day)
- [ ] Null handling matches Tableau (ZN / ISNULL → ISBLANK / IF)
- [ ] Theme colors match the Tableau palette

---

## Migration Checklist

### Before Migration
- [ ] Export Tableau workbook as `.twb` (unpackaged XML) for inspection
- [ ] List all data sources and their connection types
- [ ] Document all calculated fields, parameters, and sets
- [ ] Screenshot each dashboard and worksheet for visual reference
- [ ] Note all filters, sorts, and top-N settings

### During Migration
- [ ] Create Power BI project folder structure
- [ ] Create `model.tmdl` with culture settings
- [ ] Create table `.tmdl` files for each data source table
- [ ] Port all calculated fields to DAX measures or calculated columns
- [ ] Define relationships in `relationships.tmdl`
- [ ] Create page `.json` files for each dashboard
- [ ] Map each worksheet to a Power BI visual with correct projections
- [ ] Create slicers for each Tableau quick filter and parameter
- [ ] Apply theme JSON with Tableau color palette
- [ ] Configure conditional formatting where Tableau used color/size encoding

### After Migration
- [ ] Validate all KPIs match between Tableau and Power BI on the same data slice
- [ ] Test all slicer/filter interactions
- [ ] Test cross-filter behavior between visuals
- [ ] Confirm date hierarchies drill down correctly
- [ ] Publish to Power BI Service
- [ ] Configure data refresh schedule

---

## Complete Example: Superstore Sales Migration

This example shows a complete migration of the Tableau Superstore sample workbook.

**Source Workbook:** `tableau_source/Superstore.twb`

### Data Source Analysis

| Tableau Data Source | Connection | Tables | Has Calculated Fields | Power BI Action |
|---------------------|-----------|--------|----------------------|----------------|
| `Sample - Superstore` | Excel / SQL | Orders, Returns, People | Yes (Profit Ratio, Days to Ship, Segment Group) | **Import from SQL** |

### Table Analysis

| Table | Source | Has Transformations | Power BI Action |
|-------|--------|---------------------|----------------|
| `Orders` | `<server>.<database>.dbo.Orders` | Yes (calculated columns: Profit Ratio, Days to Ship) | **Create with M query + DAX measures** |
| `Returns` | `<server>.<database>.dbo.Returns` | No | **Direct import** |
| `People` | `<server>.<database>.dbo.People` | No | **Direct import** |
| `DimDate` | Calculated (date scaffold from Orders) | N/A — fully calculated | **Power Query date table** |
| `_Measures` | N/A | N/A | **DAX measures table** |

### Calculated Field Migration

| Tableau Field | Formula | Power BI DAX |
|---------------|---------|--------------|
| `Profit Ratio` | `SUM([Profit])/SUM([Sales])` | `DIVIDE(SUM('Orders'[Profit]), SUM('Orders'[Sales]))` |
| `Days to Ship` | `DATEDIFF('day',[Order Date],[Ship Date])` | Calculated column: `DATEDIFF('Orders'[Order Date], 'Orders'[Ship Date], DAY)` |
| `Returned Orders` | `IF ISNULL([Returned]) THEN "No" ELSE [Returned] END` | Calculated column: `IF(ISBLANK(RELATED('Returns'[Returned])), "No", "Yes")` |
| `Sales per Customer` | `SUM([Sales]) / COUNTD([Customer ID])` | `DIVIDE(SUM('Orders'[Sales]), DISTINCTCOUNT('Orders'[Customer ID]))` |

### Dashboard / Page Analysis

| Tableau Dashboard | Power BI Page | Sheets Included |
|------------------|--------------|----------------|
| Overview | Overview | Sales by Region (bar), Profit Over Time (line), KPIs (cards) |
| Product Analysis | Products | Category Bar, Sub-Category Table, Profit Ratio Scatter |
| Customer Analysis | Customers | Customer Count Card, Sales by Segment, Top Customers Table |

### Visuals Analysis

| Worksheet | Tableau Type | Power BI Visual | Dimensions | Measures |
|-----------|-------------|-----------------|------------|----------|
| Sales by Category | bar | clusteredColumnChart | Category | SUM(Sales) |
| Profit Over Time | line | lineChart | Order Date (Month) | SUM(Profit) |
| Sales by Region | filled map | filledMap | State | SUM(Sales) |
| KPI: Total Sales | text (BAN) | card | — | SUM(Sales) |
| KPI: Profit Ratio | text (BAN) | card | — | Profit Ratio |
| Customer Detail | text (crosstab) | tableEx | Customer Name, Segment | SUM(Sales), SUM(Profit) |

### Theme Colors (from Superstore default Tableau palette)

- dataColors[0]: `#4E79A7` (blue)
- dataColors[1]: `#F28E2B` (orange)
- dataColors[2]: `#E15759` (red)
- dataColors[3]: `#76B7B2` (teal)
- dataColors[4]: `#59A14F` (green)
- dataColors[5]: `#EDC948` (yellow)
- dataColors[6]: `#B07AA1` (purple)
- dataColors[7]: `#FF9DA7` (pink)
- dataColors[8]: `#9C755F` (brown)
- dataColors[9]: `#BAB0AC` (gray)

### Files Created

```
├── SuperstoreSales.pbip
├── SuperstoreSales.SemanticModel/
│   └── definition/
│       ├── model.tmdl
│       ├── relationships.tmdl
│       └── tables/
│           ├── Orders.tmdl
│           ├── Returns.tmdl
│           ├── People.tmdl
│           ├── DimDate.tmdl
│           └── _Measures.tmdl
├── SuperstoreSales.Report/
│   └── definition/
│       ├── report.json
│       └── pages/
│           ├── Overview/page.json
│           ├── Products/page.json
│           └── Customers/page.json
└── StaticResources/
    └── RegisteredResources/
        └── SuperstoreTheme.json
```

### Sample Orders.tmdl

```
table Orders
    lineageTag: a1b2c3d4-e5f6-7890-abcd-ef1234567890

    partition Orders = m
        mode: import
        source =
            let
                Source  = Sql.Database("<server>", "<database>"),
                Orders  = Source{[Schema="dbo", Item="Orders"]}[Data],
                #"Added Days to Ship" = Table.AddColumn(Orders, "Days to Ship",
                    each Duration.Days([Ship Date] - [Order Date]), Int64.Type)
            in
                #"Added Days to Ship"

    column Order ID
        dataType: string
        lineageTag: b2c3d4e5-...
        summarizeBy: none
        sourceColumn: Order ID

    column Customer ID
        dataType: string
        lineageTag: c3d4e5f6-...
        summarizeBy: none
        sourceColumn: Customer ID

    column Sales
        dataType: double
        lineageTag: d4e5f6a7-...
        summarizeBy: sum
        sourceColumn: Sales

    column Profit
        dataType: double
        lineageTag: e5f6a7b8-...
        summarizeBy: sum
        sourceColumn: Profit

    column Order Date
        dataType: dateTime
        lineageTag: f6a7b8c9-...
        summarizeBy: none
        sourceColumn: Order Date

    column Days to Ship
        dataType: int64
        lineageTag: a7b8c9d0-...
        summarizeBy: average
        sourceColumn: Days to Ship

    measure 'Sales Amount' = SUM('Orders'[Sales])
        lineageTag: b8c9d0e1-...
        formatString: \$#,0.00

    measure 'Profit Ratio' = DIVIDE(SUM('Orders'[Profit]), SUM('Orders'[Sales]))
        lineageTag: c9d0e1f2-...
        formatString: 0.00%

    measure 'Customer Count' = DISTINCTCOUNT('Orders'[Customer ID])
        lineageTag: d0e1f2a3-...
        formatString: #,0

    measure 'Sales per Customer' = DIVIDE([Sales Amount], [Customer Count])
        lineageTag: e1f2a3b4-...
        formatString: \$#,0.00
```

### Sample relationships.tmdl

```
relationship 'Orders_Returns'
    fromColumn: 'Orders'[Order ID]
    toColumn:   'Returns'[Order ID]

relationship 'Orders_People'
    fromColumn: 'Orders'[Region]
    toColumn:   'People'[Region]

relationship 'Orders_DimDate'
    fromColumn: 'Orders'[Order Date]
    toColumn:   'DimDate'[Date]
```
