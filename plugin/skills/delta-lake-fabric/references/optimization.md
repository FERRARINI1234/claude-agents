# Otimizacao Delta Lake

## OPTIMIZE

Compacta arquivos pequenos em arquivos maiores para melhor performance de leitura.

### Sintaxe Basica

```python
# Via SQL
spark.sql("OPTIMIZE gold.dim_cliente")

# Via DeltaTable API
from delta.tables import DeltaTable
delta_table = DeltaTable.forName(spark, "gold.dim_cliente")
delta_table.optimize().executeCompaction()
```

### OPTIMIZE com ZORDER

Ordena dados por colunas frequentemente filtradas para melhor data skipping.

```python
# Via SQL
spark.sql("""
    OPTIMIZE gold.fato_movimento
    ZORDER BY (id_dim_cliente, data_movimento)
""")

# Via API
delta_table.optimize().executeZOrderBy("id_dim_cliente", "data_movimento")
```

### Quando usar ZORDER

| Cenario | Colunas Recomendadas |
|---------|---------------------|
| Filtros frequentes | Colunas do WHERE |
| Joins | Colunas de join |
| Agregacoes | Colunas do GROUP BY |
| Time series | Coluna de data |

**Limite:** Maximo 4 colunas no ZORDER (ordem importa).

## VACUUM

Remove arquivos antigos que nao sao mais necessarios.

### Sintaxe

```python
# Padrao: remove arquivos > 7 dias
spark.sql("VACUUM gold.dim_cliente")

# Com retencao customizada (em horas)
spark.sql("VACUUM gold.dim_cliente RETAIN 168 HOURS")  # 7 dias

# Dry run (apenas lista arquivos)
spark.sql("VACUUM gold.dim_cliente DRY RUN")
```

### Configuracao de Retencao

```python
# Permitir retencao menor que 7 dias (CUIDADO!)
spark.conf.set("spark.databricks.delta.retentionDurationCheck.enabled", False)

# Executar vacuum com 24 horas
spark.sql("VACUUM gold.dim_cliente RETAIN 24 HOURS")
```

### Boas Praticas VACUUM

| Regra | Motivo |
|-------|--------|
| Minimo 7 dias | Permite time travel e jobs longos |
| Agendar semanalmente | Evita acumulo de arquivos |
| Executar apos OPTIMIZE | Limpa arquivos do compaction |
| Monitorar espaco | VACUUM libera storage |

## Particionamento

### Criar Tabela Particionada

```python
df.write.format("delta") \
    .partitionBy("ano", "mes") \
    .saveAsTable("gold.fato_movimento")
```

### Quando Particionar

| Particionar | Nao Particionar |
|-------------|-----------------|
| Tabelas > 1TB | Tabelas < 100GB |
| Filtros frequentes na coluna | Cardinalidade muito alta |
| Cardinalidade baixa-media | Muitas particoes pequenas |
| Dados historicos | Dados sempre lidos por completo |

### Evitar Over-Partitioning

```python
# RUIM: muitas particoes pequenas
df.write.partitionBy("data")  # 365 particoes/ano

# BOM: particoes maiores
df.write.partitionBy("ano_mes")  # 12 particoes/ano

# Verificar tamanho das particoes
spark.sql("""
    DESCRIBE DETAIL gold.fato_movimento
""").select("numFiles", "sizeInBytes").show()
```

## Auto Optimize

Habilita compactacao automatica apos cada write.

```python
# Habilitar para tabela especifica
spark.sql("""
    ALTER TABLE gold.dim_cliente
    SET TBLPROPERTIES (
        delta.autoOptimize.optimizeWrite = true,
        delta.autoOptimize.autoCompact = true
    )
""")

# Habilitar globalmente (para novas tabelas)
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", True)
spark.conf.set("spark.databricks.delta.autoCompact.enabled", True)
```

## Caching

### Cache de DataFrame

```python
# Cache em memoria
df.cache()
# ou
df.persist()

# Usar o DataFrame...
df.count()
df.filter(...).show()

# Liberar cache
df.unpersist()
```

### Delta Cache (Fabric)

```python
# Habilitar delta cache
spark.conf.set("spark.databricks.io.cache.enabled", True)
```

## Metricas e Monitoramento

### Estatisticas da Tabela

```python
# Detalhes da tabela
spark.sql("DESCRIBE DETAIL gold.dim_cliente").show()

# Campos retornados:
# - numFiles: quantidade de arquivos
# - sizeInBytes: tamanho total
# - numPartitions: numero de particoes
```

### Historico de Operacoes

```python
delta_table = DeltaTable.forName(spark, "gold.dim_cliente")
history = delta_table.history()

# Ver metricas das operacoes
history.select(
    "version",
    "operation",
    "operationMetrics.numOutputRows",
    "operationMetrics.numOutputBytes"
).show()
```

### Verificar Fragmentacao

```python
# Arquivos por particao
spark.sql("""
    SELECT
        _metadata.file_path,
        _metadata.file_size
    FROM gold.dim_cliente
""").groupBy().agg(
    count("*").alias("num_files"),
    avg("file_size").alias("avg_size"),
    min("file_size").alias("min_size"),
    max("file_size").alias("max_size")
).show()
```

## Configuracoes de Performance

```python
# Shuffle partitions (padrao 200)
spark.conf.set("spark.sql.shuffle.partitions", 100)

# Adaptive Query Execution
spark.conf.set("spark.sql.adaptive.enabled", True)
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", True)

# Broadcast threshold (10MB padrao)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 50 * 1024 * 1024)  # 50MB

# Delta specific
spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", True)
spark.conf.set("spark.databricks.delta.autoCompact.enabled", True)
```

## Checklist de Otimizacao

```text
ANTES DE PRODUCAO
[ ] Particionamento adequado (se tabela grande)
[ ] ZORDER nas colunas de filtro/join
[ ] Auto optimize habilitado
[ ] Retencao de vacuum definida

MANUTENCAO PERIODICA
[ ] OPTIMIZE semanal em tabelas grandes
[ ] VACUUM apos OPTIMIZE
[ ] Monitorar fragmentacao
[ ] Revisar query plans

TROUBLESHOOTING
[ ] Verificar numero de arquivos
[ ] Verificar tamanho das particoes
[ ] Analisar shuffle nas queries
[ ] Verificar skew de dados
```
