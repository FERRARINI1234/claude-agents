# Notebooks

> **MCP Validated:** 2026-02-09

## What It Is

Microsoft Fabric notebooks are the primary compute artifact in this project. Each notebook is a PySpark program that runs on the Synapse Spark engine. Notebooks are stored in the repository as directories with a `.Notebook` suffix containing a `notebook-content.py` file.

## File Structure

```
my_notebook.Notebook/
  notebook-content.py    # The actual code
```

The `.Notebook` directory is a Fabric artifact convention. The code lives entirely in `notebook-content.py`.

## METADATA Blocks

Every notebook starts with a file-level METADATA block declaring the Spark kernel and lakehouse dependencies:

```python
# Fabric notebook source

# METADATA ********************
# META {
# META   "kernel_info": { "name": "synapse_pyspark" },
# META   "dependencies": {
# META     "lakehouse": {
# META       "default_lakehouse": "61a29c0b-54cf-44a1-8338-a036035e5fd5",
# META       "default_lakehouse_name": "silver",
# META       "default_lakehouse_workspace_id": "1ea45368-...",
# META       "known_lakehouses": [
# META         { "id": "61a29c0b-54cf-44a1-8338-a036035e5fd5" }
# META       ]
# META     }
# META   }
# META }
```

Each cell also has its own METADATA block specifying the language:

```python
# METADATA ********************
# META {
# META   "language": "python",
# META   "language_group": "synapse_pyspark"
# META }
```

## Cell Delimiters

Cells are separated by comment lines. Three types exist:

| Delimiter | Purpose |
|-----------|---------|
| `# CELL ********************` | Code cell boundary |
| `# METADATA ********************` | Cell metadata boundary |
| `# MARKDOWN ********************` | Markdown cell boundary |

## `%run` Directives

Notebooks import shared code using Fabric's `%run` magic command. This executes another notebook in the same Spark session, making its classes and functions available:

```python
# CELL ********************
%run silver_repository

# CELL ********************
%run errors_repository
```

Key `%run` chains in this project:

| Notebook | Runs | Gets Access To |
|----------|------|----------------|
| Silver notebooks | `silver_repository` | `GravaSilver` class (which itself runs `base_silver_repository` and `errors_repository`) |
| Gold notebooks | `gold_repository` | `GravaGold` class (which itself runs `base_gold_repository` and `errors_repository`) |
| Bronze notebooks | `bronze_repository` | `GravaBronze` class |

## Standard Notebook Pattern

Every transformation notebook follows this structure:

```python
# 1. Import repository via %run
%run silver_repository

# 2. Instantiate repository
repository = GravaSilver(spark=spark)

# 3. Read from upstream layer
df = repository.read(sql="SELECT ... FROM bronze.table_name")

# 4. Transform (joins, casts, column selection, flag normalization)
df = repository.trata_colunas(df, columns)
df = repository.padroniza_flags(df, columns)

# 5. Save to current layer
repository.salva_tabela_silver(df, 'table_name', 'silver')
```

## `mssparkutils` Usage

Gold notebooks use `mssparkutils` to detect the current workspace:

```python
from notebookutils import mssparkutils
workspace_name = mssparkutils.env.getWorkspaceName()
```

This is used to differentiate behavior between DEV ("Mentors") and PROD ("Dados").

## Gotchas

- The `%run` directive resolves notebooks by name within the same workspace. Notebook names must be unique per workspace.
- METADATA blocks use `# META` comment prefixes -- they are parsed by Fabric but are plain Python comments.
- When creating a new notebook, always include both the file-level and cell-level METADATA blocks.
- Lakehouse IDs in METADATA are environment-specific. The CI/CD pipeline replaces them during deployment.
- The first line of every notebook file must be `# Fabric notebook source`.

## Related

- [Lakehouse](lakehouse.md) -- METADATA lakehouse references
- [Repository Pattern](../patterns/repository-pattern.md) -- how `%run` enables the pattern
- [Fabric Deployment](fabric-deployment.md) -- METADATA block ID replacement
