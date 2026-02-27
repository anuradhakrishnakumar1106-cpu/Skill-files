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

| Tableau Mark Type | Tableau Context | Power BI Visual Type |
|-------------------|----------------|----------------------|
| `bar` (vertical) | Dimension on Columns, Measure on Rows | `clusteredColumnChart` |
| `bar` (horizontal) | Measure on Columns, Dimension on Rows | `clusteredBarChart` |
| `bar` (stacked) | Color encoding | `stackedBarChart` / `stackedColumnChart` |
| `bar` (100% stacked) | Percent of total color | `hundredPercentStackedBarChart` |
| `line` | Date on Columns | `lineChart` |
| `line` (dual axis) | Two measure rows | `lineClusteredColumnComboChart` |
| `area` (stacked) | Multiple series | `stackedAreaChart` |
| `area` (non-stacked) | Single series | `areaChart` |
| `circle` / `shape` | Two measures + dimension detail | `scatterChart` |
| `pie` | Color + Angle (size) | `pieChart` |
| `pie` (donut via size) | Color + size | `donutChart` |
| `text` (flat) | Dimensions + measures, no pivoting | `tableEx` |
| `text` (pivot/crosstab) | Row + Column shelves both used | `matrix` |
| `map` (filled) | Geographic dimension | `filledMap` |
| `map` (symbol) | Lat/Long or geo dimension + size | `map` |
| `gantt` | Date range bars | `clusteredBarChart` (approximation) |
| `square` / `circle` (highlight table) | Color on text mark | `matrix` with conditional formatting |

### Calculated Field Conversion

| Tableau Function | Power BI DAX Equivalent |
|-----------------|------------------------|
| `SUM([Sales])` | `SUM('Table'[Sales])` |
| `AVG([Sales])` | `AVERAGE('Table'[Sales])` |
| `COUNT([OrderID])` | `COUNT('Table'[OrderID])` |
| `COUNTD([CustomerID])` | `DISTINCTCOUNT('Table'[CustomerID])` |
| `MIN([Date])` | `MIN('Table'[Date])` |
| `MAX([Date])` | `MAX('Table'[Date])` |
| `MEDIAN([Sales])` | `MEDIAN('Table'[Sales])` |
| `PERCENTILE([Sales], 0.9)` | `PERCENTILEX.INC(ALL('Table'), 'Table'[Sales], 0.9)` |
| `ZN([Measure])` | `IF(ISBLANK([Measure]), 0, [Measure])` |
| `ISNULL([Field])` | `ISBLANK('Table'[Field])` |
| `IFNULL([Field], 0)` | `IF(ISBLANK('Table'[Field]), 0, 'Table'[Field])` |
| `IF cond THEN a ELSE b END` | `IF(cond, a, b)` |
| `IIF(cond, a, b)` | `IF(cond, a, b)` |
| `CASE [Field] WHEN 'A' THEN 1 ELSE 2 END` | `SWITCH('Table'[Field], "A", 1, 2)` |
| `STR([Number])` | `FORMAT([Number], "General")` |
| `INT([Float])` | `INT([Float])` or `TRUNC([Float], 0)` |
| `FLOAT([Int])` | `CONVERT([Int], DOUBLE)` |
| `LEFT([String], n)` | `LEFT([String], n)` |
| `RIGHT([String], n)` | `RIGHT([String], n)` |
| `MID([String], s, n)` | `MID([String], s, n)` |
| `LEN([String])` | `LEN([String])` |
| `UPPER([String])` | `UPPER([String])` |
| `LOWER([String])` | `LOWER([String])` |
| `TRIM([String])` | `TRIM([String])` |
| `CONTAINS([String], 'x')` | `CONTAINSSTRING([String], "x")` |
| `REPLACE([String], 'a', 'b')` | `SUBSTITUTE([String], "a", "b")` |
| `TODAY()` | `TODAY()` |
| `NOW()` | `NOW()` |
| `YEAR([Date])` | `YEAR([Date])` |
| `MONTH([Date])` | `MONTH([Date])` |
| `DAY([Date])` | `DAY([Date])` |
| `DATETRUNC('month', [Date])` | `DATE(YEAR([Date]), MONTH([Date]), 1)` |
| `DATEPART('weekday', [Date])` | `WEEKDAY([Date])` |
| `DATEDIFF('day', [Start], [End])` | `DATEDIFF([Start], [End], DAY)` |
| `DATEADD('month', 1, [Date])` | `EDATE([Date], 1)` |
| `RUNNING_SUM(SUM([Sales]))` | `CALCULATE(SUM(...), FILTER(ALLSELECTED(...), date <= MAX(date)))` |
| `RANK(SUM([Sales]))` | `RANKX(ALL('Table'), [Measure])` |
| `WINDOW_AVG(SUM([Sales]), -2, 0)` | Rolling average DAX using DATESINPERIOD |
| `TOTAL(SUM([Sales]))` | `CALCULATE([Measure], ALL('Table'[GroupDim]))` |
| `FIXED [Dim] : AGG` (LOD) | `CALCULATE([Measure], ALL(<other dims>))` — see note |
| `INCLUDE [Dim] : AGG` (LOD) | Add `[Dim]` to visual, or `SUMMARIZE` |
| `EXCLUDE [Dim] : AGG` (LOD) | `CALCULATE([Measure], ALL('Table'[Dim]))` |
| `{ FIXED : MAX([Date]) }` (grand total LOD) | `CALCULATE(MAX('Table'[Date]), ALL('Table'))` |

### Power Query M Conversion (for transformations)

| Tableau (M / data prep) | Power Query M Equivalent |
|------------------------|--------------------------|
| `Text.Trim([col])` | `Text.Trim([col])` |
| `Text.Upper([col])` | `Text.Upper([col])` |
| `Text.Lower([col])` | `Text.Lower([col])` |
| `Text.Replace([col], "a", "b")` | `Text.Replace([col], "a", "b")` |
| `Date from datetime` | `DateTime.Date([col])` |
| `Year from date` | `Date.Year([col])` |
| `Month number` | `Date.Month([col])` |
| `Month name` | `Date.ToText([col], "MMM")` |
| `Quarter` | `"Q" & Text.From(Date.QuarterOfYear([col]))` |
| `Day of week` | `Date.DayOfWeek([col])` |
| `Combine columns` | `[col1] & " " & [col2]` |
| `Split by delimiter` | `Table.SplitColumn(...)` |
| `Pivot` | `Table.Pivot(...)` |
| `Unpivot` | `Table.Unpivot(...)` |

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
