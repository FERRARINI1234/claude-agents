# Merge Upsert Pattern

> **MCP Validated:** 2026-02-09

## Problem

Gold layer tables need to be updated incrementally: new records should be inserted, and existing records should be updated with new values. A full overwrite would be wasteful and would lose SCD Type 2 history.

## Solution

The `merge_tabela_gold()` method in `GravaGold` uses Delta Lake's MERGE operation to perform upserts based on a key column match.

### Core Implementation

```python
def merge_tabela_gold(self, dataframe, tabela_destino, chave_tabela_origem,
                      chave_tabela_destino, layer):
    caminho = f"{layer}.{tabela_destino}"
    gold_table = DeltaTable.forName(self.spark, caminho)

    gold_table.alias('gold').merge(
        dataframe.alias('silver'),
        f'gold.{chave_tabela_destino} = silver.{chave_tabela_origem}'
    ).whenMatchedUpdateAll()\
     .whenNotMatchedInsertAll()\
     .execute()
```

### Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `dataframe` | Source DataFrame with new/updated records | `df_new_versions` |
| `tabela_destino` | Target Gold table name | `'dim_conta_corrente'` |
| `chave_tabela_origem` | Key column in source DataFrame | `'id_dim_conta_corrente'` |
| `chave_tabela_destino` | Key column in target table | `'id_dim_conta_corrente'` |
| `layer` | Lakehouse prefix | `'gold'` |

### Surrogate Key Generation

Before MERGE, surrogate keys are generated from natural keys using MD5:

```python
# Business key columns (chave_*) are hashed to id_dim_*
for c in df.columns:
    if c.lower().startswith('chave_'):
        df = df.withColumn(c,
            when(col(c).isNull(), None)
            .otherwise(md5(col(c).cast('string')))
        )

# Then renamed
for c in colunas:
    if 'chave_' in c.lower():
        novo_nome = c.lower().replace('chave_', 'id_dim_')
        df = df.withColumnRenamed(c, novo_nome)

# Date columns get their own dimension keys
for c in df.columns:
    if c.lower().startswith('data_'):
        df = df.withColumn(
            f"id_{c.lower()}",
            md5(date_format(to_date(col(c), 'dd/MM/yyyy'), 'yyyyMMdd'))
        )
```

## Code Example

Full Gold notebook flow using MERGE:

```python
# 1. Get last data version from Silver
last_version = repository.get_last_data_version(spark, 'silver.contas_correntes')

# 2. Read incremental changes
df = repository.read(sql=f"""
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY chave ORDER BY processing_date DESC
    ) as index
    FROM table_changes('silver.contas_correntes', {last_version})
""")
df = df.filter(col('index') == 1).drop('index')

# 3. Check if table exists
try:
    df_target = DeltaTable.forName(spark, f'gold.{table}').toDF()
    tabela_existe = True
except:
    tabela_existe = False

# 4. If table exists: MERGE expired records, then APPEND new ones
if tabela_existe:
    if df_hist.count() > 0:
        repository.merge_tabela_gold(df_hist, table, key, key, 'gold')
    if df_new.count() > 0:
        df_new.write.format("delta").mode("append").saveAsTable(f"gold.{table}")
else:
    # First run: just save directly
    df.write.saveAsTable(f'gold.{table}')
```

## When to Apply

- Use MERGE for all Gold dimension updates (SCD Type 1 and Type 2 expiration).
- Use direct APPEND for new version records in SCD Type 2 (new records get a new surrogate key).
- The first time a Gold table is created, skip MERGE and use `saveAsTable()` directly.

## Trade-offs

**Pros:**
- Atomic operation -- either the entire merge succeeds or nothing changes
- `whenMatchedUpdateAll()` avoids listing every column explicitly
- Efficient -- only processes changed records via `table_changes()`

**Cons:**
- `whenMatchedUpdateAll()` updates ALL columns, including audit columns -- verify this is desired
- Key column names must match between source and target, or be mapped explicitly
- MERGE on very large tables can be slow -- partitioning strategy matters
