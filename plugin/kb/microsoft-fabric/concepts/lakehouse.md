# Lakehouse

> **MCP Validated:** 2026-02-09

## What It Is

A Microsoft Fabric Lakehouse is a unified data platform that combines the flexibility of a data lake with the structured querying of a data warehouse. It stores data in Delta Lake format on OneLake and exposes tables via a SQL analytics endpoint. In this project (Credicoamo), six lakehouses form the backbone of the data platform, each corresponding to a layer or function in the medallion architecture.

## Lakehouses in This Project

| Lakehouse | Purpose | Layer |
|-----------|---------|-------|
| `staging` | Landing zone for raw CSV files from source systems | Staging |
| `bronze` | Raw data with hash-based change tracking | Bronze |
| `silver` | Cleaned, typed, business-named entities | Silver |
| `gold` | Star schema dimensions (`dim_*`) and facts (`fato_*`) | Gold |
| `base_parametro` | Reference/parameter tables (overwrite mode) | Parameters |
| `error_mart` | Centralized error logging (`log_error` table) | Cross-cutting |

## How Lakehouse References Work

Every Fabric notebook declares its lakehouse dependencies in a `METADATA` block at the top of the file. This JSON-like structure specifies the default lakehouse, its workspace, and any additional known lakehouses the notebook can access.

```python
# METADATA ********************
# META {
# META   "kernel_info": { "name": "synapse_pyspark" },
# META   "dependencies": {
# META     "lakehouse": {
# META       "default_lakehouse": "61a29c0b-54cf-44a1-8338-a036035e5fd5",
# META       "default_lakehouse_name": "silver",
# META       "default_lakehouse_workspace_id": "1ea45368-8a68-4eb7-b7b6-b20fae91527f",
# META       "known_lakehouses": [
# META         { "id": "61a29c0b-54cf-44a1-8338-a036035e5fd5" }
# META       ]
# META     }
# META   }
# META }
```

## DEV vs PROD Lakehouse IDs

Lakehouse IDs differ between environments. The mapping is stored in `CICD/config.yml`:

- **DEV workspace:** "Mentors" (`1ea45368-8a68-4eb7-b7b6-b20fae91527f`)
- **PROD workspace:** "Dados" (`6a3b7d2d-6c8c-443f-9c86-76556276dc7c`)

Each lakehouse has a distinct GUID per environment. The CI/CD pipeline (`update_lakehouse_to_prod.py`) performs regex-based replacement to swap DEV IDs for PROD IDs in all notebook METADATA blocks during deployment.

## When to Use

- Reference `CICD/config.yml` when you need lakehouse IDs for either environment.
- When creating a new notebook, always include the correct METADATA block with the target lakehouse.
- Cross-lakehouse queries use the format `lakehouse_name.table_name` (e.g., `bronze.sbcc_ccorrente`).

## Gotchas

- Lakehouse IDs in METADATA are GUIDs -- they are not human-readable. Always check `config.yml` for the correct mapping.
- If a notebook references multiple lakehouses, all must appear in the `known_lakehouses` array.
- The `default_lakehouse` is the one used for unqualified table references (e.g., `SELECT * FROM my_table`).
- Connection IDs (Fabric, Uniface, TM1Coamo) are shared across DEV and PROD.

## Related

- [Notebooks](notebooks.md) -- METADATA block structure
- [Fabric Deployment](fabric-deployment.md) -- DEV-to-PROD ID replacement
- [Medallion Architecture](medallion-architecture.md) -- how lakehouses map to layers
