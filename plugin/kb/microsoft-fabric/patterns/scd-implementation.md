# SCD Implementation

> **MCP Validated:** 2026-02-09

## Problem

Business entities change over time. Some changes should overwrite old values (SCD Type 1), while others must preserve history with validity tracking (SCD Type 2). The project needs both strategies depending on the entity.

## Solution

### SCD Type 1 (Overwrite)

Used for entities where history is not needed (e.g., `tipo_movimento_conta_corrente`, `escritura_agcr`). The Silver layer deduplicates using `ROW_NUMBER()`, and the Gold layer uses `MERGE` to update existing records in place.

```python
# Silver: Dedup to get latest record
df = repository.read(sql="""
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY CD_POSTO, NR_CONTA
            ORDER BY DT_INGESTAO DESC
        ) as index
    FROM bronze.sbcc_historico
""")
df = df.filter(col('index') == 1)

# Gold: MERGE (update matched, insert new)
repository.merge_tabela_gold(
    df, 'dim_tipo_movimento', 'id_dim_tipo_movimento', 'id_dim_tipo_movimento', 'gold')
```

### SCD Type 2 (History Tracking)

Used for entities that require full change history (e.g., `contas_correntes`, `canais_digitais`, `conta_corrente`). The Gold layer maintains validity windows.

**Step 1: Read incremental changes from Silver using Delta CDF**

```python
last_version = repository.get_last_data_version(spark, 'silver.contas_correntes')
df = repository.read(sql=f"""
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY chave ORDER BY processing_date DESC
        ) as index
    FROM table_changes('silver.contas_correntes', {last_version})
""")
df = df.filter(col('index') == 1).drop('index')
```

**Step 2: Compute hash for change detection**

```python
hash_cols = [c for c in df.columns if c not in ['chave', 'processing_date']]
df_source = df.withColumn('hash_key', sha1(concat_ws('-', *hash_cols)))
```

**Step 3: Compare source against current valid records in Gold**

```python
df_target = DeltaTable.forName(spark, 'gold.dim_conta_corrente').toDF()
target = df_target.filter(col("is_valid") == True)

df_new_versions = (
    df_source.alias("s")
    .join(target.alias("t"), on="chave", how="left")
    .filter(
        col("t.chave").isNull() |                  # new record
        (col("s.hash_key") != col("t.hash_key"))   # changed record
    )
)
```

**Step 4: Create new version records**

```python
df_new_versions = (
    df_new_versions.select(
        md5(concat_ws('-', col("s.chave"), current_timestamp())).alias('id_dim_conta_corrente'),
        col('s.data_abertura'), ...
    )
    .withColumn('valid_from', current_timestamp())
    .withColumn('valid_to', to_timestamp(lit('9999-12-31 23:59:59')))
    .withColumn('is_valid', lit(True))
)
```

**Step 5: Expire old records**

```python
df_hist = (
    target.alias("t").join(df_new_versions.alias("n"), on="chave", how="inner")
    .select(col('t.id_dim_conta_corrente'), col('t.valid_from'), ...)
    .withColumn('valid_to', expr("current_timestamp() - INTERVAL 1 DAY"))
    .withColumn('is_valid', lit(False))
)
```

**Step 6: Write both expired and new records**

```python
# Expire old versions via MERGE
repository.merge_tabela_gold(df_hist, table, key, key, 'gold')

# Append new versions
df_new_versions.write.format("delta").mode("append").saveAsTable(f"gold.{table}")
```

## SCD Type Assignment

Defined in `docs/table_registry_new.yaml`:

| Entity | SCD Type |
|--------|----------|
| `conta_corrente` | 2 |
| `canais_digitais` | 2 |
| `componente_conta_corrente` | 2 |
| `proposta_credito` | 2 |
| `movimento_conta_corrente` | 1 |
| `tipo_movimento_conta_corrente` | 1 |
| `escritura_agcr` | 1 |
| `debito_automatico` | 1 |

## SCD Type 2 Columns

| Column | Purpose |
|--------|---------|
| `valid_from` | Timestamp when the record version became active |
| `valid_to` | Timestamp when the record version expired (`9999-12-31 23:59:59` = current) |
| `is_valid` | Boolean flag (`True` = current version, `False` = expired) |
| `hash_key` | SHA1 hash of all business columns for change detection |

## When to Apply

- Check `table_registry_new.yaml` for the entity's `scd_type` before implementing.
- SCD Type 2 requires `valid_from`, `valid_to`, `is_valid`, and `hash_key` columns in Gold.
- SCD Type 1 uses straightforward MERGE with `whenMatchedUpdateAll()`.

## Trade-offs

**SCD Type 1:** Simpler, less storage, no history. Good for reference data.
**SCD Type 2:** Full audit trail, more storage, more complex queries (must filter `is_valid = True`).
