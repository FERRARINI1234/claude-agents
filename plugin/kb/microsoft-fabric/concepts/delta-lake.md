# Delta Lake

> **MCP Validated:** 2026-02-09

## What It Is

Delta Lake is the storage format for all tables in this Microsoft Fabric project. It provides ACID transactions, schema enforcement, time travel, and efficient upserts (MERGE). Every table across all lakehouses (staging, bronze, silver, gold, base_parametro, error_mart) is stored as Delta.

## How It Works in This Project

### Write Modes

| Layer | Write Mode | Method |
|-------|-----------|--------|
| Bronze | `append` (deduplicated) | `salva_tabela_bronze()` with anti-join on `hash_diff` |
| Silver | `append` | `salva_tabela_silver()` via `saveAsTable()` |
| Silver (params) | `overwrite` | `salva_tabela_parametros()` for reference tables |
| Gold | `append` + `MERGE` | `merge_tabela_gold()` for upserts, direct append for new records |

### Reading with DeltaTable API

The Gold layer uses the `DeltaTable` class from the `delta` package for advanced operations:

```python
from delta import DeltaTable

# Get a reference to an existing Delta table
delta_table = DeltaTable.forName(spark, 'gold.dim_conta_corrente')

# Read as DataFrame
df = delta_table.toDF()
```

### Time Travel and Change Data Feed

Gold notebooks use `table_changes()` to read only records added since the last data-modifying version:

```python
last_version = repository.get_last_data_version(spark, 'silver.contas_correntes')

df = repository.read(
    sql=f"""
        SELECT *, ROW_NUMBER() OVER (
            PARTITION BY chave ORDER BY processing_date DESC
        ) as index
        FROM table_changes('silver.contas_correntes', {last_version})
    """)
```

### Version History

Two methods retrieve version information from Delta history:

```python
# Get the absolute latest version (any operation)
def get_last_version(self, spark_session, table):
    deltaTable = DeltaTable.forName(spark_session, table)
    df = deltaTable.history(1)
    return df.select(col("version")).collect()[0][0]

# Get the last version that modified data (filters out OPTIMIZE, etc.)
def get_last_data_version(self, spark_session, table):
    delta_table = DeltaTable.forName(spark_session, table)
    history_df = delta_table.history()
    data_ops = ["WRITE", "MERGE", "UPDATE", "DELETE", "INSERT",
                "CREATE TABLE AS SELECT", "REPLACE TABLE AS SELECT",
                "STREAMING UPDATE"]
    last_data_version = (
        history_df.filter(col("operation").isin(data_ops))
        .orderBy(col("version").desc())
        .select("version").limit(1).collect()
    )
    return last_data_version[0][0] if last_data_version else None
```

### MERGE Operations

The Gold layer uses Delta MERGE for upserts:

```python
gold_table = DeltaTable.forName(self.spark, 'gold.dim_conta_corrente')
gold_table.alias('gold').merge(
    dataframe.alias('silver'),
    'gold.id_dim_conta_corrente = silver.id_dim_conta_corrente'
).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()
```

## Gotchas

- `table_changes()` requires Delta Lake Change Data Feed (CDF) to be enabled on the source table.
- `get_last_data_version()` filters out maintenance operations (OPTIMIZE, VACUUM) to find the last actual data change.
- Silver tables use `append` mode, meaning historical records accumulate -- deduplication happens via `ROW_NUMBER()` at read time.
- When a Gold table does not exist yet, the first write uses `saveAsTable()` directly instead of MERGE.

## Related

- [Merge Upsert Pattern](../patterns/merge-upsert.md) -- full MERGE implementation
- [Hash Change Detection](../patterns/hash-change-detection.md) -- Bronze dedup with Delta
- [SCD Implementation](../patterns/scd-implementation.md) -- how Delta enables SCD Type 2
