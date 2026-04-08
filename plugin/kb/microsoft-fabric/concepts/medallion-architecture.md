# Medallion Architecture

> **MCP Validated:** 2026-02-09

## What It Is

The Medallion Architecture is the data layering strategy used in this project. Data flows strictly through four layers: Staging, Bronze, Silver, and Gold. Each layer has a dedicated Fabric Lakehouse, a repository class, and distinct transformation responsibilities.

## Layer Overview

```
CSV Files (Uniface, base_parametros)
    |
    v
[Staging] -- Raw CSV ingestion, adds metadata (DT_INGESTAO, CAMADA)
    |
    v
[Bronze]  -- Hash-based dedup, preserves raw data, UPPERCASE columns
    |
    v
[Silver]  -- Business names, type casting, NULL handling, flags, SCD
    |
    v
[Gold]    -- Star schema (dim_*/fato_*), surrogate keys (MD5), MERGE
```

## Layer Details

### Staging

- **Location:** `core/notebooks/01_staging/` (single notebook)
- **Lakehouse:** `staging`
- **Function:** Ingests raw CSV files from source systems. Adds metadata columns: `DT_INGESTAO` (ingestion timestamp), `CAMADA` (layer name), source system info.
- **Write mode:** Overwrite

### Bronze

- **Location:** `core/notebooks/02_bronze/` (single notebook)
- **Lakehouse:** `bronze`
- **Repository:** `GravaBronze`
- **Function:** Preserves raw data from staging. Adds `hash_diff` column (SHA1 of all columns) for change detection. Only inserts records whose hash is not already present (anti-join).
- **Table naming:** Source system prefix: `sbcc_ccorrente`, `sbfi_proposta`, `sbsc_adescnldig`
- **Column naming:** UPPERCASE (as received from source)
- **Write mode:** Append (deduplicated via hash)
- **Extraction types:** `full` (entire dataset) or `daily` (incremental)

### Silver

- **Location:** `core/notebooks/04_silver/` (~30 notebooks, one per entity)
- **Lakehouse:** `silver`
- **Repository:** `GravaSilver`
- **Function:** Reads from Bronze via SQL. Applies:
  - Type casting (`CAST(CD_POSTO AS INTEGER)`)
  - `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY DT_INGESTAO DESC)` for deduplication
  - NULL handling via `trata_colunas()`: strings to `'N/I'`, integers to `0`, doubles to `0.0`, epoch dates to `NULL`
  - String standardization: `initcap(lower(col))`
  - Flag normalization via `padroniza_flags()`: `'T'/'F'` to `'Sim'/'Nao'`
  - `processing_date` audit column
- **Table naming:** Business names: `contas_correntes`, `propostas`, `canais_digitais`
- **Column naming:** lowercase snake_case
- **Write mode:** Append
- **SCD support:** Type 1 (overwrite) and Type 2 (history) per table config in `table_registry_new.yaml`

### Gold

- **Location:** `core/notebooks/05_gold/` (~30 notebooks, one per dim/fact)
- **Lakehouse:** `gold`
- **Repository:** `GravaGold`
- **Function:** Reads from Silver using `table_changes()` for incremental processing. Builds star schema:
  - Surrogate keys: `chave_*` columns hashed with MD5 to become `id_dim_*`
  - Date keys: `data_*` columns formatted as `yyyyMMdd` then hashed with MD5 to become `id_data_*`
  - SCD Type 2 tracking: `valid_from`, `valid_to`, `is_valid`, `hash_key` columns
  - MERGE for updating expired records
- **Table naming:** `dim_conta_corrente`, `dim_agencia`, `fato_movimento_conta`, `fato_agcr`
- **Write mode:** MERGE (upsert) + Append

## Data Flow Example

```
uniface.csv --> staging.sbcc_ccorrente
            --> bronze.sbcc_ccorrente (+ hash_diff)
            --> silver.contas_correntes (typed, cleaned, lowercase)
            --> gold.dim_conta_corrente (surrogate keys, SCD2)
```

## Key Conventions

| Convention | Bronze | Silver | Gold |
|-----------|--------|--------|------|
| Column case | UPPERCASE | lowercase snake_case | lowercase snake_case |
| Primary keys | Source composite keys | `chave` (natural key) | `id_dim_*` (MD5 surrogate) |
| Date format | `dd/MM/yyyy` strings | `DateType` | `DateType` + `id_data_*` key |
| NULL strings | Raw | `'N/I'` | `'N/I'` |
| NULL integers | Raw | `0` | `0` |
| Audit column | `DT_INGESTAO` | `processing_date` | `processing_date` |

## Business Domains

Tables are organized into domains: Associados, Captacoes, Credito, Dimensoes. Domain ownership and dependencies are defined in `docs/table_registry_new.yaml`.

## Related

- [Lakehouse](lakehouse.md) -- one lakehouse per layer
- [Repository Pattern](../patterns/repository-pattern.md) -- one repository class per layer
- [PySpark Patterns](pyspark-patterns.md) -- transformation techniques used across layers
