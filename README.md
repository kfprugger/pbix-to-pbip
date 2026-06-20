# pbix-to-pbip

A reusable GitHub Copilot skill for converting Power BI `.pbix` / `.pbip` assets into source-controlled PBIP projects, optionally migrating semantic models to Microsoft Fabric Direct Lake, publishing report/model items to Fabric workspaces, and validating parity against the source report.

This repository intentionally contains **generic** guidance only. It should not include customer names, tenant names, workspace IDs, server names, secrets, or environment-specific identifiers.

## What this skill does

The skill in `.github/copilot/skills/pbix-to-pbip/SKILL.md` helps an agent:

1. Convert or work with native PBIP folder structures.
2. Audit a semantic model before Direct Lake migration.
3. Materialize unsupported Import/M/native-query shaping upstream in Fabric.
4. Convert model metadata and partitions to Direct Lake.
5. Publish the semantic model and report to Fabric service.
6. Place the report in a configured report folder and the semantic model in a configured model folder.
7. Validate the deployed service report with storage-mode evidence, refresh/framing evidence, DAX regression, and screenshots/exports.

## Repository layout

```text
.github/copilot/skills/pbix-to-pbip/SKILL.md  # Copilot skill instructions
README.md                                    # Usage and configuration
LICENSE
```

## Install/use with GitHub Copilot skills

In a repository where GitHub Copilot custom skills are enabled, keep the skill at:

```text
.github/copilot/skills/pbix-to-pbip/SKILL.md
```

Then ask Copilot/your coding agent for tasks such as:

```text
Use the pbix-to-pbip skill to convert ./Reports/Sales.pbix to PBIP and publish it to Fabric.
```

```text
Use the pbix-to-pbip skill to migrate ./Reports/Sales.pbip to Direct Lake and publish it as Sales - DL.
```

```text
Use the pbix-to-pbip skill to validate that the published Direct Lake report matches the source report.
```

## Required project inputs

Before publishing, provide or configure these values for your environment:

| Input | Description |
|---|---|
| Source PBIX/PBIP path | Local path to the report/model source. |
| Default workspace ID | Fabric workspace for normal reports. |
| Restricted workspace ID | Fabric workspace for sensitive reports, if used. |
| Report folder name or ID | Folder for Report items. |
| Model folder name or ID | Folder for SemanticModel items. |
| Fabric Warehouse/Lakehouse SQL endpoint | SQL endpoint used to discover/materialize Direct Lake source tables. |
| Warehouse/Lakehouse database name | Database/item name used by the Direct Lake model source expression. |
| Authentication method | Azure CLI, PowerShell, service principal, or delegated user token. |
| Baseline validation source | Source report screenshots, exported files, or DAX output. |

Do not commit real tenant IDs, workspace IDs, server names, customer names, API keys, access tokens, passwords, connection strings with credentials, or screenshots containing sensitive data to this generic repository.

## Direct Lake migration expectations

Direct Lake migration is not a simple connector replacement. It changes where table shaping happens and can affect DAX behavior.

The skill requires a readiness audit before publishing:

- Power Query/M transformations
- native SQL queries
- calculated columns and calculated tables
- DAX measures used by visuals
- relationships and cross-filter directions
- data types and key uniqueness
- date tables and auto date hierarchies
- field parameters
- calculation groups
- hierarchies
- RLS/OLS rules
- DirectQuery fallback behavior

Unsupported Import-mode logic should be moved upstream into Fabric Warehouse/Lakehouse physical tables before Direct Lake binding.

## Publishing convention

For Direct Lake deployments, publish with a `- DL` suffix:

```text
<original report name> - DL
```

Use the same display name for the semantic model unless your workspace has a stricter convention.

If sensitivity routing is configured, the skill asks:

```text
Is this published report sensitive?
```

Then it routes:

- sensitive reports to the restricted workspace
- non-sensitive reports to the default/shared workspace

In either workspace:

- the Report item must be in the configured report folder
- the SemanticModel item must be in the configured model folder
- leaving items in the workspace root is not final-complete

## Validation requirements

A deployment is complete only after current service evidence exists:

1. Fabric item metadata confirms expected names, IDs, workspace, and folders.
2. Semantic model definition confirms Direct Lake partitions and no unexpected Import partitions.
3. Direct Lake refresh/framing succeeds or any fallback/exception is explicitly documented.
4. DAX regression validates representative metrics for every report page.
5. A published-service report export or screenshots confirm the report renders.
6. Source scans confirm no environment-specific source strings or generic-repo-forbidden identifiers are embedded.

Final output should include:

```text
Report URL: https://app.powerbi.com/groups/{workspaceId}/reports/{reportId}
Workspace: <name> (<workspaceId>)
Report: <name> (<reportId>) in <report-folder>
Semantic model: <name> (<semanticModelId>) in <model-folder>
Direct Lake evidence: <summary>
Validation evidence: <summary>
Known exceptions: <none or documented list>
```

## Local development checklist

Before committing changes to this repository:

```bash
# Inspect changed files
git status --short

# Check for accidental environment/customer references
grep -R "<customer-or-tenant-name>" .
```

Also search for real workspace IDs, tenant IDs, access tokens, server names, and customer-specific URLs before publishing this repository.

## License

See [LICENSE](LICENSE).
