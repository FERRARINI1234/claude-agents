# Hash Change Detection

> **MCP Validated:** 2026-02-09

## Problem

The Bronze layer receives full or daily extracts from source systems. Inserting all records on every run would create massive duplication. The system needs to detect which records have actually changed and only insert those.

## Solution

A SHA1 hash of all column values (the `hash_diff` column) is computed for both incoming and existing records. An anti-join on the hash identifies only genuinely new or changed records, which are then appended.

### Step 1: Compute Hash Expression

`computa_coluna_hash()` dynamically builds a SHA1 expression from all table columns, excluding specified columns:

```python
def computa_coluna_hash(self, nome_tabela, colunas_ignorar):
    colunas = self.spark.catalog.listColumns(tableName=nome_tabela)
    colunas_ignorar = colunas_ignorar or []

    lista_colunas = [col.name for col in colunas if col.name not in colunas_ignorar]

    expressao_concat = f"sha1(concat_ws('-', {', '.join(lista_colunas)}))"
    return expressao_concat
    # Example output: "sha1(concat_ws('-', CD_POSTO, NR_CONTA, TP_CONTA, ST_CONTA))"
```

### Step 2: Add Hash Column to DataFrame

`adiciona_hash_diff()` reads the staging table and adds the `hash_diff` column:

```python
def adiciona_hash_diff(self, tabela, expressao_hash):
    df = self.read(tabela=tabela)
    result = df.withColumn('hash_diff', expr(expressao_hash))
    return result
```

### Step 3: Save with Deduplication

`salva_tabela_bronze()` performs an anti-join to exclude records whose hash already exists:

```python
def salva_tabela_bronze(self, dataframe_origem, tabela_destino, coluna_hash, tabela_existe):
    if tabela_existe:
        # Read existing hashes from Bronze
        df_tabela_destino = self.spark.read.table(tabela_destino).select(col(coluna_hash))

        # Anti-join: keep only records whose hash is NOT in the existing table
        df_novos = dataframe_origem.alias("novos").join(
            df_tabela_destino.alias("existentes"),
            on=coluna_hash,
            how="anti"
        )

        df_novos.write.format('delta').mode('append').saveAsTable(tabela_destino)
    else:
        # First run: insert everything
        dataframe_origem.write.format('delta').mode('append').saveAsTable(tabela_destino)

    return True
```

## Code Example

Full Bronze notebook flow:

```python
repository = GravaBronze(spark=spark)

# Check if Bronze table already exists
tabela_existe = spark.catalog.tableExists("bronze.sbcc_ccorrente")

# Build hash expression (exclude metadata columns from hash)
coluna_hash = repository.computa_coluna_hash(
    nome_tabela="staging.sbcc_ccorrente",
    colunas_ignorar=["DT_INGESTAO", "CAMADA"]
)

# Add hash_diff column to staging data
df_com_hash = repository.adiciona_hash_diff(
    tabela="staging.sbcc_ccorrente",
    expressao_hash=coluna_hash
)

# Add metadata (ingestion timestamp, layer name)
df_com_metadados = repository.add_metadados(dataframe=df_com_hash, camada="bronze")

# Save: only inserts records with new hashes
repository.salva_tabela_bronze(
    dataframe_origem=df_com_metadados,
    tabela_destino="bronze.sbcc_ccorrente",
    coluna_hash="hash_diff",
    tabela_existe=tabela_existe
)
```

## When to Apply

- Applied to ALL Bronze table writes, both full and daily extraction types.
- The `colunas_ignorar` parameter should exclude metadata columns (`DT_INGESTAO`, `CAMADA`) that change on every run but do not indicate a real data change.
- For `full` extraction type tables: every record is re-extracted, so hash dedup prevents duplicating unchanged records.
- For `daily` extraction type tables: only new/changed records arrive, but hash still prevents re-inserting duplicates from overlapping date ranges.

## Trade-offs

**Pros:**
- Storage efficient -- only genuinely changed records are stored
- Simple logic -- one SHA1 comparison determines change
- Works with both full and incremental extractions
- Uses `concat_ws('-', ...)` which handles NULLs gracefully (treats as empty string)

**Cons:**
- SHA1 hash collisions are theoretically possible (extremely rare in practice)
- The anti-join reads ALL existing hashes from the Bronze table, which scales linearly with table size
- If source columns are reordered, the hash changes even though data has not -- column order matters in `concat_ws`
- `spark.catalog.listColumns()` requires the table to already exist in the metastore
