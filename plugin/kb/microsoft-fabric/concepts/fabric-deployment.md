# Fabric Deployment

> **MCP Validated:** 2026-02-09

## What It Is

The deployment process moves notebooks from the DEV workspace ("Mentors") to the PROD workspace ("Dados") in Microsoft Fabric. This is automated via an Azure DevOps pipeline that replaces environment-specific IDs in notebook METADATA blocks, updates workspace references, and regenerates documentation.

## Pipeline Overview

The pipeline is defined in `azure-pipelines.yml` and triggers on merges to the `master` branch (excluding changes under `CICD/`).

```yaml
trigger:
  branches:
    include: [master]
  paths:
    exclude: [CICD/**]
```

### Pipeline Stages

```
1. update_lakehouse_to_prod.py  -- Replace DEV lakehouse/workspace IDs with PROD IDs
2. update_workspace_to_prod.py  -- Update workspace references in pipeline artifacts
3. generate_docs.py             -- Regenerate data catalog from table_registry_new.yaml
4. Git commit with [skip ci]    -- Auto-commit changes, prevent recursive trigger
```

## DEV-to-PROD ID Replacement

`CICD/update_lakehouse_to_prod.py` performs regex-based replacement across all notebook files:

1. Reads `CICD/config.yml` to build a mapping of lakehouse names to PROD IDs.
2. Scans all `.Notebook/notebook-content.py` files in these directories:
   - `core/notebooks/01_staging/`
   - `core/notebooks/02_bronze/`
   - `core/notebooks/03_parametros/01_parametros_ingestao/`
   - `core/notebooks/03_parametros/02_parametros_negocios/`
   - `core/notebooks/04_silver/`
   - `core/notebooks/05_gold/`
3. For each notebook, finds the `default_lakehouse_name` and replaces:
   - `default_lakehouse` ID with the PROD lakehouse ID
   - `default_lakehouse_workspace_id` with the PROD workspace ID
   - First `known_lakehouses` entry ID with the PROD lakehouse ID

The regex patterns used:

```python
# Replace default_lakehouse ID
re.sub(r'(# META\s+"default_lakehouse":\s+)"[^"]+"', ...)

# Replace workspace ID
re.sub(r'(# META\s+"default_lakehouse_workspace_id":\s+)"[^"]+"', ...)

# Replace known_lakehouses ID
re.sub(r'(# META\s+"known_lakehouses":.*?"id":\s*)"[^"]+"', ..., flags=re.MULTILINE)
```

## Environment Configuration

`CICD/config.yml` structure:

```yaml
dev:
  workspace_name: Mentors
  workspace_id: 1ea45368-8a68-4eb7-b7b6-b20fae91527f
  lakehouses:
    - name: staging
      id: e8c3068c-...
    - name: bronze
      id: 188b7ae0-...
    # ... (6 lakehouses total)

prod:
  workspace_name: Dados
  workspace_id: 6a3b7d2d-6c8c-443f-9c86-76556276dc7c
  lakehouses:
    - name: staging
      id: a3718e49-...
    - name: bronze
      id: e8157ad4-...
    # ... (6 lakehouses total)
```

Connection IDs (Fabric, Uniface, TM1Coamo) are shared between environments.

## Auto-Documentation

`CICD/generate_docs.py` reads `docs/table_registry_new.yaml` and generates:
- Mermaid lineage diagrams showing table dependencies
- Domain-specific documentation pages under `docs/`

## Git Workflow

- **`develop`** -- Development branch (work happens here)
- **`master`** -- Production branch (CI/CD trigger)
- PRs flow from `develop` to `master`
- The pipeline auto-commits with `[skip ci]` to prevent recursive triggers
- Commit message includes source commit hash and build ID

## Running CI/CD Scripts Locally

```bash
pip install -r CICD/requirements.txt    # pyyaml, requests
python CICD/update_lakehouse_to_prod.py
python CICD/generate_docs.py
```

## Gotchas

- Never commit notebooks with PROD IDs to the `develop` branch -- the pipeline handles this.
- The `[skip ci]` tag in auto-commits prevents infinite pipeline loops.
- If a new lakehouse is added, update `config.yml` with both DEV and PROD IDs before deploying.
- The pipeline runs on `ubuntu-latest` with Python 3.11.

## Related

- [Lakehouse](lakehouse.md) -- DEV/PROD lakehouse ID mapping
- [Notebooks](notebooks.md) -- METADATA blocks that get modified
