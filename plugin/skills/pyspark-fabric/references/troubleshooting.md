# Troubleshooting PySpark no Fabric

## Erros Comuns

### AnalysisException: Table or view not found

**Erro:**
```
AnalysisException: Table or view not found: bronze.tabela
```

**Causas:**
1. Tabela nao existe no lakehouse
2. Lakehouse nao esta vinculado ao notebook
3. Nome do lakehouse incorreto

**Solucoes:**
```python
# Verificar tabelas disponiveis
spark.catalog.listTables("bronze")

# Verificar se tabela existe
spark.catalog.tableExists("bronze.tabela")

# Listar lakehouses disponiveis
spark.catalog.listDatabases()
```

### Column not found

**Erro:**
```
AnalysisException: cannot resolve 'coluna' given input columns
```

**Solucoes:**
```python
# Verificar colunas do DataFrame
df.printSchema()
df.columns

# Verificar case sensitivity
df.select([col(c.lower()) for c in df.columns])
```

### Type Mismatch em Join

**Erro:**
```
AnalysisException: cannot resolve due to data type mismatch
```

**Solucao:**
```python
# Garantir mesmo tipo nas colunas de join
df1 = df1.withColumn("chave", col("chave").cast("string"))
df2 = df2.withColumn("chave", col("chave").cast("string"))

df_join = df1.join(df2, "chave")
```

### NULL em Aggregate Functions

**Problema:** Agregacoes ignoram NULLs silenciosamente.

**Solucao:**
```python
from pyspark.sql.functions import coalesce, lit

# Tratar NULL antes de agregar
df = df.withColumn("valor", coalesce(col("valor"), lit(0)))
```

### Cartesian Product Warning

**Erro:**
```
WARN: Detected implicit cartesian product
```

**Causa:** Join sem condicao ou condicao avaliando sempre true.

**Solucao:**
```python
# Verificar condicao de join
# ERRADO:
df1.join(df2, df1.col1 == df2.col1 or df1.col1.isNull())

# CORRETO:
df1.join(df2, (df1.col1 == df2.col1) | (df1.col1.isNull() & df2.col1.isNull()))
```

## Problemas de Performance

### Shuffle Excessivo

**Sintomas:** Jobs lentos, muitos stages.

**Diagnostico:**
```python
# Ver plano de execucao
df.explain(True)

# Procurar por "Exchange" (shuffle)
```

**Solucoes:**
```python
# 1. Broadcast para tabelas pequenas
from pyspark.sql.functions import broadcast
df_result = df_grande.join(broadcast(df_pequena), "chave")

# 2. Reparticionar antes de joins pesados
df = df.repartition(200, "chave_join")

# 3. Coalesce para reduzir particoes
df = df.coalesce(10)
```

### Out of Memory (OOM)

**Sintomas:** Job falha com erro de memoria.

**Solucoes:**
```python
# 1. Persistir DataFrames intermediarios
df_intermediario.persist()
# ... usar df_intermediario ...
df_intermediario.unpersist()

# 2. Aumentar particoes
spark.conf.set("spark.sql.shuffle.partitions", 400)

# 3. Evitar collect() em DataFrames grandes
# ERRADO:
lista = df.collect()
# CORRETO:
df.write.format("delta").save(path)
```

### Skew de Dados

**Sintomas:** Algumas tasks demoram muito mais que outras.

**Diagnostico:**
```python
# Verificar distribuicao de chaves
df.groupBy("chave_join").count().orderBy(col("count").desc()).show(20)
```

**Solucoes:**
```python
# 1. Salting para chaves desbalanceadas
from pyspark.sql.functions import concat, lit, rand

# Adicionar salt
df1 = df1.withColumn("salt", (rand() * 10).cast("int"))
df1 = df1.withColumn("chave_salt", concat(col("chave"), lit("_"), col("salt")))

# 2. AQE (Adaptive Query Execution)
spark.conf.set("spark.sql.adaptive.enabled", True)
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", True)
```

## Debug de Dados

### Verificar Qualidade

```python
# Contar NULLs por coluna
from pyspark.sql.functions import count, when, isnan

df.select([
    count(when(col(c).isNull() | isnan(col(c)), c)).alias(c)
    for c in df.columns
]).show()
```

### Verificar Duplicatas

```python
# Contar duplicatas
df.groupBy("chave").count().filter(col("count") > 1).show()

# Ver registros duplicados
df.groupBy("chave").count().filter(col("count") > 1) \
    .join(df, "chave").show()
```

### Verificar Ranges

```python
# Estatisticas basicas
df.describe().show()

# Min/Max especifico
df.agg(
    min("data_movimento").alias("data_min"),
    max("data_movimento").alias("data_max"),
    min("valor").alias("valor_min"),
    max("valor").alias("valor_max")
).show()
```

## Configuracoes Uteis

```python
# Mostrar mais linhas no output
spark.conf.set("spark.sql.repl.eagerEval.maxNumRows", 100)

# Aumentar shuffle partitions
spark.conf.set("spark.sql.shuffle.partitions", 200)

# Habilitar AQE
spark.conf.set("spark.sql.adaptive.enabled", True)

# Mostrar plano fisico
df.explain("formatted")

# Cache inteligente
spark.conf.set("spark.sql.inMemoryColumnarStorage.compressed", True)
```

## Logs e Monitoramento

### Ver Logs do Spark

No Fabric, os logs estao disponiveis na interface do notebook em:
- Spark UI > Executors > Logs

### Monitorar Jobs

```python
# Adicionar descricao ao job
spark.sparkContext.setJobDescription("Processando fato_movimento")

# Ver jobs ativos
spark.sparkContext.statusTracker().getActiveJobIds()
```

## Checklist de Debug

```text
[ ] Tabela existe no lakehouse correto?
[ ] Colunas existem com nomes corretos (case)?
[ ] Tipos de dados compativeis nos joins?
[ ] Condicoes de join estao corretas?
[ ] DataFrame tem dados (nao vazio)?
[ ] NULLs estao sendo tratados?
[ ] Particoes estao balanceadas?
[ ] Memoria suficiente para operacao?
```
