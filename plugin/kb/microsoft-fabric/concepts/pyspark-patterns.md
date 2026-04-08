# PySpark Patterns

> **MCP Validated:** 2026-02-09

## What It Is

This document describes the common PySpark patterns used across all layers of the Credicoamo data platform. These are not generic Spark patterns -- they reflect the specific idioms and conventions adopted in this codebase.

## Reading Data

### SQL-Based Reading (Silver and Gold)

Silver and Gold layers read from upstream via `spark.sql()`:

```python
df = repository.read(
    sql="""
        SELECT
            CAST(CD_POSTO AS INTEGER) AS CD_POSTO,
            CAST(NR_CONTA AS INTEGER) AS NR_CONTA,
            TP_CONTA,
            ROW_NUMBER() OVER (
                PARTITION BY CD_POSTO, NR_CONTA
                ORDER BY DT_INGESTAO DESC
            ) as index
        FROM bronze.sbcc_ccorrente
    """)
```

### Table-Based Reading (Bronze)

Bronze reads from staging using `spark.read.table()`:

```python
df = self.spark.read.table(tabela)
```

## Deduplication with ROW_NUMBER

The most common pattern for getting the latest record per key:

```python
# In the SQL query
ROW_NUMBER() OVER (
    PARTITION BY CD_POSTO, NR_CONTA
    ORDER BY DT_INGESTAO DESC
) as index

# Then filter in PySpark
df = df.filter(col('index') == 1)
```

## Column Transformations

### Type Casting

```python
col('CD_POSTO').cast('integer')
col('NR_CONTA').cast('string')
to_date(col('DT_ABERTURA'), 'dd/MM/yyyy').alias('data_abertura')
```

### Conditional Logic with `when`

Used extensively for business rule mapping:

```python
when(col('TP_CONTA') == '1', 'Individual')
.when(col('TP_CONTA') == '2', 'Conjunta Solidaria')
.when(col('TP_CONTA') == '3', 'Conjunta Nao Solidaria')
.when(col('TP_CONTA') == '4', 'Pessoa Juridica')
.otherwise('N/D').alias('tipo_conta')
```

### NULL Handling by Type (`trata_colunas`)

```python
# Integers: NULL -> 0
when(col(column).isNull(), 0).otherwise(col(column))

# Strings: NULL -> 'N/I', then initcap(lower())
when(col(column).isNull(), lit('N/I')).otherwise(initcap(lower(col(column))))

# Doubles: NULL -> 0
when(col(column).isNull(), 0).otherwise(col(column))

# Dates: epoch sentinel -> NULL
when(col(column) == lit('1970-01-01'), None).otherwise(col(column))
```

### Flag Normalization (`padroniza_flags`)

```python
when(upper(col(column)) == 'T', 'Sim')
.when(upper(col(column)) == 'F', 'Nao')
.when(col(column).isNull(), 'N/I')
.otherwise(col(column))
```

## Hash Generation

### SHA1 for Change Detection (Bronze)

```python
# Build expression dynamically from table columns
expressao_concat = f"sha1(concat_ws('-', {', '.join(lista_colunas)}))"

# Add as column
df = df.withColumn('hash_diff', expr(expressao_concat))
```

### MD5 for Surrogate Keys (Gold)

```python
# From chave_* columns to id_dim_*
df = df.withColumn(c, when(col(c).isNull(), None).otherwise(md5(col(c).cast('string'))))

# From date columns to id_data_*
df = df.withColumn(
    f"id_{c}",
    md5(date_format(to_date(col(c), 'dd/MM/yyyy'), 'yyyyMMdd'))
)

# Composite key generation
md5(concat_ws('-', col("s.chave"), current_timestamp())).alias('id_dim_conta_corrente')
```

## Join Patterns

### Multi-Table Joins with Aliases

```python
df_main = df_main.alias('main')
df_lookup = df_lookup.alias('lookup')

result = (
    df_main.filter(col('index') == 1)
    .join(df_lookup.filter(col('index') == 1),
          (col('main.CD_POSTO') == col('lookup.CD_POSTO')) &
          (col('main.NR_CONTA') == col('lookup.NR_CONTA')),
          how='left')
)
```

### Anti-Join for Deduplication (Bronze)

```python
df_novos = dataframe_origem.alias("novos").join(
    df_tabela_destino.alias("existentes"),
    on=coluna_hash,
    how="anti"
)
```

## Composite Key Construction

Silver tables build natural keys using `concat`:

```python
concat(
    col('ccorrente.CD_POSTO').cast('string'),
    lit('|'),
    col('ccorrente.NR_CONTA').cast('string')
).alias('chave')
```

## Audit Column

All Silver and Gold tables include a `processing_date` timestamp:

```python
data_processamento = datetime.now().strftime('%d/%m/%Y %H:%M')
df = df.withColumn('processing_date',
    to_timestamp(lit(data_processamento), 'dd/MM/yyyy HH:mm'))
```

## Gotchas

- Always use `col()` from `pyspark.sql.functions`, not string column references, for join conditions.
- Date parsing uses `dd/MM/yyyy` (Brazilian format) -- not `MM/dd/yyyy`.
- `initcap(lower(col))` is applied to all string columns in Silver to standardize casing.
- `concat_ws` uses `'-'` as separator for hash inputs; natural keys use `'|'`.

## Related

- [Medallion Architecture](medallion-architecture.md) -- where each pattern is used
- [Hash Change Detection](../patterns/hash-change-detection.md) -- SHA1 pattern detail
- [Merge Upsert](../patterns/merge-upsert.md) -- Gold layer write pattern
