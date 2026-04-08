# Delta Lake Operations

## MERGE/UPSERT (Gold Layer)

Pattern padrao para upsert incremental:

```python
from delta.tables import DeltaTable

# Obter tabela Delta existente
delta_table = DeltaTable.forName(spark, f"{lakehouse}.{table_name}")

# Executar MERGE
(delta_table.alias("target")
    .merge(
        df_source.alias("source"),
        "target.chave = source.chave"
    )
    .whenMatchedUpdateAll()
    .whenNotMatchedInsertAll()
    .execute())
```

### MERGE com Condicoes Customizadas

```python
(delta_table.alias("t")
    .merge(
        df_source.alias("s"),
        "t.id_dim_cliente = s.id_dim_cliente"
    )
    .whenMatchedUpdate(
        condition="t.hash_diff != s.hash_diff",
        set={
            "nome": "s.nome",
            "valor": "s.valor",
            "processing_date": "s.processing_date"
        }
    )
    .whenNotMatchedInsert(
        values={
            "id_dim_cliente": "s.id_dim_cliente",
            "nome": "s.nome",
            "valor": "s.valor",
            "processing_date": "s.processing_date"
        }
    )
    .execute())
```

### MERGE com Delete

```python
(delta_table.alias("t")
    .merge(df_source.alias("s"), "t.chave = s.chave")
    .whenMatchedDelete(condition="s.flag_deletado = 'S'")
    .whenMatchedUpdateAll(condition="s.flag_deletado = 'N'")
    .whenNotMatchedInsertAll(condition="s.flag_deletado = 'N'")
    .execute())
```

## Modos de Escrita

### Append (Bronze/Silver)

```python
df.write.format("delta") \
    .mode("append") \
    .saveAsTable(f"{lakehouse}.{table_name}")
```

### Overwrite (Parametros)

```python
df.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .saveAsTable(f"{lakehouse}.{table_name}")
```

### Particao por Data

```python
df.write.format("delta") \
    .mode("append") \
    .partitionBy("data_referencia") \
    .saveAsTable(f"{lakehouse}.{table_name}")
```

## Time Travel

### Ler Versao Especifica

```python
# Por versao
df = spark.read.format("delta") \
    .option("versionAsOf", 5) \
    .table(f"{lakehouse}.{table_name}")

# Por timestamp
df = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-15 10:00:00") \
    .table(f"{lakehouse}.{table_name}")
```

### Consultar Historico

```python
delta_table = DeltaTable.forName(spark, f"{lakehouse}.{table_name}")
history = delta_table.history()
history.select("version", "timestamp", "operation").show()
```

### Obter Ultima Versao

```python
def get_last_version(spark, table_name):
    delta_table = DeltaTable.forName(spark, table_name)
    return delta_table.history(1).select("version").collect()[0][0]
```

## Otimizacao

### OPTIMIZE

```python
spark.sql(f"OPTIMIZE {lakehouse}.{table_name}")
```

### OPTIMIZE com ZORDER

```python
spark.sql(f"""
    OPTIMIZE {lakehouse}.{table_name}
    ZORDER BY (id_dim_cliente, data_movimento)
""")
```

### VACUUM

```python
# Remove arquivos antigos (>7 dias por padrao)
spark.sql(f"VACUUM {lakehouse}.{table_name}")

# Com retencao customizada
spark.sql(f"VACUUM {lakehouse}.{table_name} RETAIN 168 HOURS")
```

## Schema Evolution

### Adicionar Colunas

```python
df.write.format("delta") \
    .mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable(f"{lakehouse}.{table_name}")
```

### Alterar Schema

```python
spark.sql(f"""
    ALTER TABLE {lakehouse}.{table_name}
    ADD COLUMN nova_coluna STRING
""")
```

## Validacoes

### Verificar se Tabela Existe

```python
def table_exists(spark, database, table):
    return spark.catalog.tableExists(f"{database}.{table}")
```

### Verificar se Tabela Esta Vazia

```python
def is_empty(spark, table_name):
    return spark.read.table(table_name).count() == 0
```
