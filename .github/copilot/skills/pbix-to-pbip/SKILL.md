---
name: pbix-to-pbip
description: Convert Power BI Desktop reports from PBIX/PBIP into source-controlled PBIP artifacts, optionally migrate semantic models to Fabric Direct Lake, publish report and semantic model items to Microsoft Fabric workspaces, and validate visual/metric parity before and after deployment.
---

# PBIX to PBIP conversion and Fabric Direct Lake deployment

Use this skill when the user asks to convert a `.pbix` to PBIP, convert an existing PBIP for Fabric, migrate a semantic model to Direct Lake, publish a report/model pair to Fabric, or validate that a deployed report behaves like the source report.

## Core outcomes

- Produce a valid native PBIP folder structure for the report and semantic model.
- Preserve report pages, visuals, slicers, bookmarks, measures, relationships, formatting, and model metadata unless a documented migration constraint requires a change.
- When Direct Lake is requested, treat it as a semantic-model migration, not a connector-only replacement.
- Publish using a `- DL` suffix when deploying a Direct Lake version.
- Route the report and semantic model to the correct workspace/folders.
- Validate with evidence: storage mode, row counts, DAX measure regression, screenshots/exports, and service item metadata.

## Required inputs

Collect or infer these before changing files or publishing:

| Input | Purpose |
|---|---|
| Source PBIX or PBIP path | Source report/model to convert. |
| Target report display name | Used for PBIP names and Fabric item names. |
| Target Fabric workspace IDs | At least default and restricted/sensitive workspace IDs if sensitivity routing is used. |
| Report folder name or ID | Folder where the Fabric Report item must live. |
| Semantic model folder name or ID | Folder where the Fabric SemanticModel item must live. |
| Target Fabric Warehouse/Lakehouse item and SQL endpoint | Required for Direct Lake source binding and upstream materialization. |
| Sensitivity answer | Ask whether the published report is sensitive if workspace routing depends on sensitivity. |
| Baseline comparison source | Original report/model, exported screenshots, or baseline DAX results. |

If the user configured default and restricted workspaces, ask only: **"Is this published report sensitive?"** Route sensitive content to the restricted workspace; otherwise route to the default/shared workspace.

## PBIP structure

Expected native PBIP layout:

```text
<ReportName>.pbip
<ReportName>.Report/
  definition.pbir
  definition/
    report.json
    version.json
    pages/
      pages.json
      <pageId>/page.json
      <pageId>/visuals/<visualId>/visual.json
  StaticResources/
<ReportName>.SemanticModel/
  definition.pbism
  definition/
    database.tmdl
    model.tmdl
    relationships.tmdl
    expressions.tmdl
    tables/<TableName>.tmdl
    cultures/<culture>.tmdl
```

Publishing via Fabric item APIs needs definition parts with paths relative to the report or semantic model root.

## Direct Lake migration gate

Do not publish a Direct Lake version until this audit is complete.

### 1. Inventory model objects

Inventory every:

- table and partition
- M/Power Query transformation
- native SQL query
- calculated column
- calculated table
- measure used by report visuals
- relationship, including cross-filter direction and inactive relationships
- date table / auto date hierarchy
- field parameter
- calculation group
- hierarchy
- sort-by column
- display folder / formatting string
- RLS/OLS rule
- unsupported data type risk

Classify tables as:

- physical Warehouse/Lakehouse table
- SQL/native-query-shaped table
- M-shaped table
- calculated table
- disconnected/helper table
- auto local date table

### 2. Materialize unsupported shaping upstream

Direct Lake tables must bind to physical Delta-backed Fabric entities. SQL views, Import M partitions, and arbitrary native SQL shaping are not enough for `directLakeOnly` models.

For every shaped Import table:

1. Create a physical Fabric Warehouse/Lakehouse table with equivalent output columns.
2. Move calculated column logic upstream into that physical table where Direct Lake on SQL/Warehouse cannot support it.
3. Preserve display names in TMDL while mapping `sourceColumn` to the materialized source column.
4. Validate row counts and sample rows against the pre-migration source.

Examples of logic that often needs upstream materialization:

- concatenated customer/item keys
- parsed document numbers from free-text fields
- route/day derived attributes
- item grouping/category flags
- date dimension rows for auto date hierarchy replacement
- M-added columns or type transforms

### 3. Convert model metadata to Direct Lake

Minimum semantic-model changes:

```tmdl
database
	compatibilityLevel: 1604
```

```tmdl
model Model
	culture: en-US
	defaultPowerBIDataSourceVersion: powerBI_V3
	sourceQueryCulture: en-US
	directLakeBehavior: directLakeOnly
```

Add a Fabric source expression:

```tmdl
expression DatabaseSource =
		let
			database = Sql.Database("<fabric-sql-endpoint>", "<warehouse-or-lakehouse-sql-db-name>")
		in
			database
	lineageTag: <guid>
```

Convert Import partitions:

```tmdl
partition <TableName> = entity
	mode: directLake
	source
		entityName: <physical-fabric-table>
		schemaName: <schema>
		expressionSource: DatabaseSource
```

Acceptance checks:

- all analytical table partitions are `mode: directLake`
- no `mode: import` remains unless explicitly documented as an approved composite-model exception
- no unsupported calculated columns/tables remain for the selected Direct Lake mode
- service metadata confirms Direct Lake after publish

### 4. Date handling

Auto-generated local date tables are a common Direct Lake migration failure.

Recommended safe path:

- Materialize role-playing date tables in the Fabric source, one per date role if necessary.
- Preserve visual references and date hierarchies by either:
  - keeping existing `LocalDateTable_*` table names but converting them to Direct Lake entity partitions backed by physical date tables, or
  - replacing visual JSON and TMDL references consistently with new physical date table names.
- Bound date tables to the actual source date min/max when possible to avoid confusing future years in slicers.
- Preserve hierarchy names if visuals refer to `Date Hierarchy`.

### 5. DAX regression

Measures remain DAX, but context and storage assumptions can change in Direct Lake. Build a validation matrix:

| Page | Visual | Measure/field | Baseline query/value | Direct Lake query/value | Status |
|---|---|---|---|---|---|

Include at least:

- one representative metric per report page
- each high-risk business measure
- measures affected by date intelligence
- measures affected by bidirectional or inactive relationships
- measures over materialized calculated columns
- count/row-grain checks for major fact tables

Use the same data snapshot. If the source has changed, use a documented cutoff filter and state the reason.

## Publishing workflow

### Workspace and folder routing

If sensitivity routing is configured:

1. Ask: **"Is this published report sensitive?"**
2. Select the restricted workspace when sensitive; select the default/shared workspace when not sensitive.
3. Publish the Report item to the configured report folder.
4. Publish the SemanticModel item to the configured model folder.
5. Verify item metadata after publish includes the expected `workspaceId` and `folderId`.

Do not leave final items in the workspace root.

### Naming

For Direct Lake deployments:

- Report: `<original report name> - DL`
- Semantic model: `<original semantic model/report name> - DL`

Do not overwrite the original baseline/import report unless the user explicitly asks.

### Fabric item API pattern

Create/update semantic model first, then create/update report bound to that semantic model.

Report `definition.pbir` for service binding:

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
  "version": "4.0",
  "datasetReference": {
    "byConnection": {
      "connectionString": "Data Source=powerbi://api.powerbi.com/v1.0/myorg/<workspaceId>;Initial Catalog=<semantic-model-display-name>;semanticModelId=<semantic-model-id>"
    }
  }
}
```

Use `folderId` when creating items, or move items after update:

```http
POST https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/items/{itemId}/move

{ "targetFolderId": "<folderId>" }
```

### Direct Lake framing

After creating or updating a Direct Lake semantic model, trigger and verify refresh/framing. Evidence can come from refresh history showing `DirectLakeFraming` completed, successful DAX queries, and service metadata confirming `directLake` partitions.

## Validation workflow

Before final delivery, collect current-state evidence:

1. Fabric item metadata:
   - workspace ID
   - report ID/name/folder
   - semantic model ID/name/folder
2. Service semantic model definition:
   - Direct Lake partition count
   - zero unexpected Import partitions
   - zero unsupported calculated-column/table markers
3. Refresh/framing history:
   - latest relevant Direct Lake framing status
4. DAX regression output:
   - page-level metric checks
   - row counts for key tables
5. Visual snapshot:
   - exported PDF/PNG or browser screenshots from the published service report
6. Source scan:
   - no source-specific on-prem server references remain in the Direct Lake model
   - no tenant/customer names or secrets are embedded in generic artifacts

## Final response requirements

Include:

- Published report URL: `https://app.powerbi.com/groups/{workspaceId}/reports/{reportId}`
- Workspace name/ID
- Report item ID and folder
- Semantic model item ID and folder
- Direct Lake storage-mode evidence
- Refresh/framing evidence
- DAX/page validation summary
- Any documented exceptions or known deltas

Never claim completion without current evidence from the published service items.
