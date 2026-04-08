---
name: delta-lake-fabric
description: |
  Especialista em Delta Lake para Microsoft Fabric Lakehouse.
  Use quando precisar: (1) implementar operacoes MERGE/UPSERT em tabelas Delta,
  (2) trabalhar com Time Travel e Change Data Feed (CDF),
  (3) implementar SCD Type 1 ou Type 2 em dimensoes,
  (4) otimizar tabelas com OPTIMIZE e VACUUM,
  (5) gerenciar schema evolution e versionamento,
  (6) debugar problemas de concorrencia ou performance em Delta.
  Segue convencoes do projeto Credicoamo: Bronze append com hash, Silver append com dedup,
  Gold MERGE para dimensoes, CDF para leitura incremental.
---

# Delta Lake Fabric

Skill para operacoes Delta Lake no Microsoft Fabric seguindo os padroes do projeto Credicoamo.

## Conceitos Fundamentais

Delta Lake e o formato de armazenamento para todas as tabelas no Fabric Lakehouse. Fornece:
- **ACID Transactions**: Operacoes atomicas e consistentes
- **Schema Enforcement**: Validacao automatica de schema
- **Time Travel**: Acesso a versoes anteriores dos dados
- **Change Data Feed**: Captura de mudancas incrementais
- **MERGE**: Upserts eficientes

## Modos de Escrita por Camada

| Camada | Modo | Metodo | Quando Usar |
|--------|------|--------|-------------|
| Bronze | `append` | `salva_tabela_bronze()` | Ingestao com anti-join hash |
| Silver | `append` | `salva_tabela_silver()` | Dados limpos, dedup na leitura |
| Silver (params) | `overwrite` | `salva_tabela_parametros()` | Tabelas de referencia |
| Gold | `MERGE` | `merge_tabela_gold()` | Dimensoes e fatos |

## Import Essencial

```python
from delta.tables import DeltaTable
from pyspark.sql.functions import col, lit, when, current_timestamp
```

## Operacoes Basicas

### Ler Tabela Delta

```python
# Via spark.read
df = spark.read.table("gold.dim_cliente")

# Via DeltaTable API
delta_table = DeltaTable.forName(spark, "gold.dim_cliente")
df = delta_table.toDF()
```

### Escrever Tabela Delta

```python
# Append
df.write.format("delta").mode("append").saveAsTable("silver.tabela")

# Overwrite
df.write.format("delta").mode("overwrite").saveAsTable("parametros.tabela")

# Com merge de schema
df.write.format("delta").mode("append") \
    .option("mergeSchema", "true") \
    .saveAsTable("silver.tabela")
```

### Verificar se Tabela Existe

```python
def table_exists(spark, full_table_name):
    try:
        DeltaTable.forName(spark, full_table_name)
        return True
    except:
        return False
```

## MERGE/UPSERT

### MERGE Basico (SCD Type 1)

```python
delta_table = DeltaTable.forName(spark, "gold.dim_cliente")

delta_table.alias("target").merge(
    df_source.alias("source"),
    "target.id_dim_cliente = source.id_dim_cliente"
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()
```

### MERGE com Condicoes

```python
delta_table.alias("t").merge(
    df_source.alias("s"),
    "t.chave = s.chave"
).whenMatchedUpdate(
    condition="t.hash_diff != s.hash_diff",
    set={
        "nome": "s.nome",
        "valor": "s.valor",
        "processing_date": "s.processing_date"
    }
).whenNotMatchedInsert(
    values={
        "chave": "s.chave",
        "nome": "s.nome",
        "valor": "s.valor",
        "processing_date": "s.processing_date"
    }
).execute()
```

### MERGE com Delete

```python
delta_table.alias("t").merge(
    df_source.alias("s"),
    "t.chave = s.chave"
).whenMatchedDelete(condition="s.flag_deletado = 'S'") \
 .whenMatchedUpdateAll(condition="s.flag_deletado = 'N'") \
 .whenNotMatchedInsertAll(condition="s.flag_deletado = 'N'") \
 .execute()
```

## Time Travel

### Ler Versao Especifica

```python
# Por numero de versao
df = spark.read.format("delta") \
    .option("versionAsOf", 5) \
    .table("gold.dim_cliente")

# Por timestamp
df = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-15 10:00:00") \
    .table("gold.dim_cliente")
```

### Consultar Historico

```python
delta_table = DeltaTable.forName(spark, "gold.dim_cliente")
history = delta_table.history()
history.select("version", "timestamp", "operation", "operationMetrics").show()
```

### Obter Ultima Versao

```python
def get_last_version(spark, table_name):
    delta_table = DeltaTable.forName(spark, table_name)
    return delta_table.history(1).select("version").collect()[0][0]
```

### Obter Ultima Versao de Dados (ignora OPTIMIZE/VACUUM)

```python
def get_last_data_version(spark, table_name):
    delta_table = DeltaTable.forName(spark, table_name)
    history = delta_table.history()

    data_ops = ["WRITE", "MERGE", "UPDATE", "DELETE", "INSERT",
                "CREATE TABLE AS SELECT", "REPLACE TABLE AS SELECT",
                "STREAMING UPDATE"]

    result = (history
        .filter(col("operation").isin(data_ops))
        .orderBy(col("version").desc())
        .select("version")
        .limit(1)
        .collect())

    return result[0][0] if result else None
```

## Change Data Feed (CDF)

### Ler Mudancas Incrementais

```python
last_version = get_last_data_version(spark, "silver.contas")

df = spark.sql(f"""
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY chave ORDER BY processing_date DESC
    ) as index
    FROM table_changes('silver.contas', {last_version})
""")
df = df.filter(col("index") == 1).drop("index")
```

### Habilitar CDF em Tabela Existente

```python
spark.sql("""
    ALTER TABLE silver.contas
    SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
""")
```

## Referencias Detalhadas

- **SCD Implementation**: Ver [references/scd-patterns.md](references/scd-patterns.md)
- **Otimizacao**: Ver [references/optimization.md](references/optimization.md)
- **Troubleshooting**: Ver [references/troubleshooting.md](references/troubleshooting.md)

## Convencoes do Projeto

### Fluxo de Dados Gold

```
1. Obter ultima versao de dados do Silver
2. Ler mudancas incrementais via table_changes()
3. Deduplicar com ROW_NUMBER()
4. Verificar se tabela Gold existe
5. Se existe: MERGE + APPEND
6. Se nao existe: saveAsTable()
```

### Colunas SCD Type 2

| Coluna | Tipo | Descricao |
|--------|------|-----------|
| `valid_from` | timestamp | Inicio da vigencia |
| `valid_to` | timestamp | Fim da vigencia (9999-12-31 = atual) |
| `is_valid` | boolean | True = versao corrente |
| `hash_key` | string | SHA1 das colunas de negocio |

## Gotchas

- `table_changes()` requer CDF habilitado na tabela fonte
- `get_last_data_version()` filtra OPTIMIZE/VACUUM para encontrar ultima mudanca real
- Silver usa `append` - deduplicacao acontece via `ROW_NUMBER()` na leitura
- Primeira escrita em Gold usa `saveAsTable()`, nao MERGE
- MERGE em tabelas muito grandes pode ser lento - considere particionamento
