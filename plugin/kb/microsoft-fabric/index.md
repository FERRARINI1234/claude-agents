# Microsoft Fabric Knowledge Base

> **MCP Validated:** 2026-02-09

Knowledge base for the Credicoamo data lakehouse platform built on Microsoft Fabric.

## Concepts

| Concept | Description | File |
|---------|-------------|------|
| [Lakehouse](concepts/lakehouse.md) | Fabric Lakehouses: staging, bronze, silver, gold, base_parametro, error_mart | concepts/lakehouse.md |
| [Delta Lake](concepts/delta-lake.md) | Delta format, ACID transactions, time travel, MERGE, version history | concepts/delta-lake.md |
| [Notebooks](concepts/notebooks.md) | Fabric notebook structure, METADATA blocks, cell delimiters, %run directives | concepts/notebooks.md |
| [Medallion Architecture](concepts/medallion-architecture.md) | Four-layer data flow: Staging, Bronze, Silver, Gold | concepts/medallion-architecture.md |
| [PySpark Patterns](concepts/pyspark-patterns.md) | SQL reads, ROW_NUMBER dedup, type casting, hash generation, joins | concepts/pyspark-patterns.md |
| [Fabric Deployment](concepts/fabric-deployment.md) | Azure DevOps CI/CD, DEV-to-PROD ID replacement, auto-docs | concepts/fabric-deployment.md |

## Patterns

| Pattern | Description | File |
|---------|-------------|------|
| [Repository Pattern](patterns/repository-pattern.md) | ABC hierarchy: Repository, GravaBronze, GravaSilver, GravaGold, Error | patterns/repository-pattern.md |
| [SCD Implementation](patterns/scd-implementation.md) | SCD Type 1 (overwrite) and Type 2 (history) in Silver/Gold layers | patterns/scd-implementation.md |
| [Merge Upsert](patterns/merge-upsert.md) | Gold layer Delta MERGE with MD5 surrogate keys | patterns/merge-upsert.md |
| [Hash Change Detection](patterns/hash-change-detection.md) | Bronze layer SHA1 dedup via anti-join on hash_diff | patterns/hash-change-detection.md |

## Quick Navigation

- **New to the project?** Start with [Medallion Architecture](concepts/medallion-architecture.md)
- **Creating a notebook?** See [Notebooks](concepts/notebooks.md) and [Repository Pattern](patterns/repository-pattern.md)
- **Working on Bronze?** See [Hash Change Detection](patterns/hash-change-detection.md)
- **Working on Silver?** See [PySpark Patterns](concepts/pyspark-patterns.md) and [SCD Implementation](patterns/scd-implementation.md)
- **Working on Gold?** See [Merge Upsert](patterns/merge-upsert.md) and [Delta Lake](concepts/delta-lake.md)
- **Deploying?** See [Fabric Deployment](concepts/fabric-deployment.md) and [Lakehouse](concepts/lakehouse.md)
