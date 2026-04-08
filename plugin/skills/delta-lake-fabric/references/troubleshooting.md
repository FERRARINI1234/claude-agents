# Troubleshooting Delta Lake

## Erros Comuns

### AnalysisException: Table not found

**Erro:**
```
AnalysisException: Table or view 'gold.dim_cliente' not found
```

**Causas:**
1. Tabela nao existe
2. Lakehouse nao vinculado
3. Nome incorreto (case sensitive)

**Solucoes:**
```python
# Verificar tabelas disponiveis
spark.catalog.listTables("gold")

# Verificar se tabela existe
spark.catalog.tableExists("gold.dim_cliente")

# Listar databases
spark.catalog.listDatabases()
```

### ConcurrentAppendException

**Erro:**
```
ConcurrentAppendException: Files were added by concurrent update
```

**Causa:** Dois jobs tentando escrever na mesma tabela simultaneamente.

**Solucoes:**
```python
# 1. Usar isolation level serializable
spark.conf.set("spark.databricks.delta.isolationLevel", "Serializable")

# 2. Retry com backoff
from time import sleep
import random

def write_with_retry(df, table, max_retries=3):
    for attempt in range(max_retries):
        try:
            df.write.format("delta").mode("append").saveAsTable(table)
            return
        except Exception as e:
            if "ConcurrentAppendException" in str(e) and attempt < max_retries - 1:
                sleep(random.uniform(1, 5))
            else:
                raise
```

### ConcurrentDeleteReadException

**Erro:**
```
ConcurrentDeleteReadException: This transaction attempted to read files that were deleted
```

**Causa:** VACUUM executado enquanto query longa rodava.

**Solucoes:**
```python
# 1. Aumentar retencao do VACUUM
spark.sql("VACUUM gold.dim_cliente RETAIN 168 HOURS")

# 2. Nao executar VACUUM durante janelas de processamento
```

### ProtocolChangedException

**Erro:**
```
ProtocolChangedException: The protocol version of the Delta table has been changed
```

**Causa:** Upgrade de protocolo Delta durante operacao.

**Solucao:**
```python
# Recriar DeltaTable reference
delta_table = DeltaTable.forName(spark, "gold.dim_cliente")
```

### SchemaEnforcementException

**Erro:**
```
AnalysisException: A schema mismatch detected when writing to the Delta table
```

**Solucoes:**
```python
# 1. Habilitar merge de schema
df.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .saveAsTable("gold.dim_cliente")

# 2. Ou sobrescrever schema (CUIDADO!)
df.write.format("delta") \
    .option("overwriteSchema", "true") \
    .mode("overwrite") \
    .saveAsTable("gold.dim_cliente")
```

## Problemas de Performance

### Queries Lentas

**Diagnostico:**
```python
# Ver plano de execucao
df.explain(True)

# Procurar por:
# - FileScan com muitos arquivos
# - Shuffle (Exchange) excessivo
# - BroadcastNestedLoopJoin (ineficiente)
```

**Solucoes:**
```python
# 1. OPTIMIZE + ZORDER
spark.sql("""
    OPTIMIZE gold.fato_movimento
    ZORDER BY (id_dim_cliente)
""")

# 2. Verificar predicates pushdown
df.filter(col("data") == "2024-01-15").explain()
# Deve mostrar: PushedFilters: [IsNotNull(data), EqualTo(data,2024-01-15)]

# 3. Estatisticas da tabela
spark.sql("ANALYZE TABLE gold.dim_cliente COMPUTE STATISTICS")
```

### Muitos Arquivos Pequenos

**Diagnostico:**
```python
spark.sql("DESCRIBE DETAIL gold.fato_movimento").select("numFiles").show()
```

**Solucoes:**
```python
# 1. OPTIMIZE
spark.sql("OPTIMIZE gold.fato_movimento")

# 2. Habilitar auto compaction
spark.sql("""
    ALTER TABLE gold.fato_movimento
    SET TBLPROPERTIES (delta.autoOptimize.autoCompact = true)
""")

# 3. Coalesce antes de escrever
df.coalesce(10).write.format("delta").mode("append").saveAsTable("gold.fato")
```

### MERGE Lento

**Causas:**
1. Tabela muito grande sem particionamento
2. Muitos arquivos (fragmentacao)
3. Condicao de match complexa

**Solucoes:**
```python
# 1. Particionar tabela por coluna de filtro comum
# Recriar tabela com particionamento

# 2. OPTIMIZE antes do MERGE
spark.sql("OPTIMIZE gold.dim_cliente")

# 3. Filtrar source para reduzir volume
df_source_filtered = df_source.filter(col("data") >= "2024-01-01")

# 4. Usar hint de broadcast se source pequeno
from pyspark.sql.functions import broadcast
delta_table.alias("t").merge(
    broadcast(df_small_source).alias("s"),
    "t.chave = s.chave"
).whenMatchedUpdateAll().execute()
```

## Problemas de CDF (Change Data Feed)

### table_changes() Retorna Vazio

**Causas:**
1. CDF nao habilitado
2. Versao incorreta

**Solucoes:**
```python
# Verificar se CDF esta habilitado
spark.sql("SHOW TBLPROPERTIES gold.dim_cliente").filter(
    col("key") == "delta.enableChangeDataFeed"
).show()

# Habilitar CDF
spark.sql("""
    ALTER TABLE silver.contas
    SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
""")

# Verificar versoes disponiveis
DeltaTable.forName(spark, "silver.contas").history().show()
```

### table_changes() Erro de Versao

**Erro:**
```
AnalysisException: start version X is not available
```

**Causa:** VACUUM removeu arquivos da versao solicitada.

**Solucao:**
```python
# Obter versao mais antiga disponivel
history = DeltaTable.forName(spark, "silver.contas").history()
min_version = history.agg({"version": "min"}).collect()[0][0]

# Usar versao disponivel
df = spark.sql(f"SELECT * FROM table_changes('silver.contas', {min_version})")
```

## Debug de Dados

### Verificar Integridade

```python
# Contar registros por versao (SCD2)
spark.sql("""
    SELECT is_valid, COUNT(*) as qtd
    FROM gold.dim_cliente
    GROUP BY is_valid
""").show()

# Verificar duplicatas na chave
spark.sql("""
    SELECT chave, COUNT(*) as qtd
    FROM gold.dim_cliente
    WHERE is_valid = true
    GROUP BY chave
    HAVING COUNT(*) > 1
""").show()
```

### Comparar Versoes

```python
# Versao atual
df_atual = spark.read.table("gold.dim_cliente")

# Versao anterior
df_anterior = spark.read.format("delta") \
    .option("versionAsOf", 5) \
    .table("gold.dim_cliente")

# Diferencas
df_diff = df_atual.subtract(df_anterior)
print(f"Registros diferentes: {df_diff.count()}")
```

### Restaurar Versao Anterior

```python
# RESTORE para versao especifica
spark.sql("RESTORE TABLE gold.dim_cliente TO VERSION AS OF 5")

# RESTORE para timestamp
spark.sql("""
    RESTORE TABLE gold.dim_cliente
    TO TIMESTAMP AS OF '2024-01-15 10:00:00'
""")
```

## Checklist de Debug

```text
ERRO DE ESCRITA
[ ] Tabela existe?
[ ] Schema compativel?
[ ] Lakehouse vinculado?
[ ] Permissoes corretas?

PERFORMANCE
[ ] OPTIMIZE executado recentemente?
[ ] Muitos arquivos pequenos?
[ ] Predicates sendo pushed down?
[ ] Particionamento adequado?

CONCORRENCIA
[ ] Multiplos jobs escrevendo?
[ ] VACUUM durante queries longas?
[ ] Isolation level configurado?

CDF
[ ] CDF habilitado na tabela?
[ ] Versao solicitada existe?
[ ] VACUUM preservou versao?
```

## Comandos Uteis

```python
# Detalhes da tabela
spark.sql("DESCRIBE DETAIL gold.dim_cliente").show()

# Historico
spark.sql("DESCRIBE HISTORY gold.dim_cliente").show()

# Propriedades
spark.sql("SHOW TBLPROPERTIES gold.dim_cliente").show()

# Schema
spark.sql("DESCRIBE gold.dim_cliente").show()

# Estatisticas
spark.sql("DESCRIBE EXTENDED gold.dim_cliente").show()
```
