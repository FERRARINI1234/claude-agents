---
name: pyspark-fabric
description: |
  Especialista em desenvolvimento PySpark para Microsoft Fabric Lakehouse.
  Use quando precisar: (1) criar notebooks PySpark para transformacao de dados,
  (2) implementar pipelines ETL nas camadas Bronze/Silver/Gold,
  (3) trabalhar com Delta Lake e operacoes MERGE/UPSERT,
  (4) aplicar patterns de deduplicacao, hash, joins e type casting,
  (5) debugar erros de Spark ou otimizar queries.
  Segue convencoes do projeto Credicoamo: repositorios por camada, hash SHA1/MD5,
  flags T/F para Sim/Nao, datas em formato brasileiro dd/MM/yyyy.
---

# PySpark Fabric

Skill para desenvolvimento PySpark no Microsoft Fabric seguindo os padroes do projeto Credicoamo.

## Estrutura de Notebooks

Todo notebook Fabric segue este padrao:

```python
# Fabric notebook source

# METADATA ********************
# META {
# META   "kernel_info": { "name": "synapse_pyspark" },
# META   "dependencies": {
# META     "lakehouse": {
# META       "default_lakehouse": "{lakehouse_id}",
# META       "default_lakehouse_name": "{layer}",
# META       "default_lakehouse_workspace_id": "{workspace_id}"
# META     }
# META   }
# META }

# CELL ********************
%run {layer}_repository

# CELL ********************
# Codigo PySpark
```

## Workflow por Camada

### Bronze (Ingestao)

```
1. Ler dados do staging via spark.read.table()
2. Adicionar hash SHA1 para change detection
3. Fazer anti-join com tabela existente (dedup)
4. Salvar em modo append
```

### Silver (Limpeza)

```
1. Ler do Bronze via spark.sql() com ROW_NUMBER()
2. Aplicar filtro index == 1 (latest record)
3. Fazer type casting e tratar NULLs
4. Normalizar flags T/F -> Sim/Nao
5. Adicionar processing_date
6. Salvar em modo append
```

### Gold (Agregacao)

```
1. Ler do Silver via spark.sql()
2. Fazer joins com dimensoes
3. Criar chaves surrogadas MD5 (id_dim_*)
4. Executar MERGE/UPSERT
5. Adicionar processing_date
```

## Imports Essenciais

```python
from pyspark.sql.functions import (
    col, lit, when, concat, concat_ws,
    upper, lower, initcap, trim,
    to_date, to_timestamp, date_format, current_timestamp,
    md5, sha1, expr,
    row_number, coalesce
)
from pyspark.sql.window import Window
from pyspark.sql.types import (
    StringType, IntegerType, DoubleType, DateType, TimestampType
)
from datetime import datetime
```

## Patterns Rapidos

### Deduplicacao com ROW_NUMBER

```python
df = repository.read(sql="""
    SELECT *, ROW_NUMBER() OVER (
        PARTITION BY cd_posto, nr_conta
        ORDER BY dt_ingestao DESC
    ) as index
    FROM bronze.tabela
""")
df = df.filter(col('index') == 1)
```

### Type Casting

```python
col('CD_POSTO').cast('integer')
col('VALOR').cast('double')
to_date(col('DT_ABERTURA'), 'dd/MM/yyyy').alias('data_abertura')
```

### Tratamento de NULLs

```python
# Inteiros: NULL -> 0
when(col(c).isNull(), 0).otherwise(col(c))

# Strings: NULL -> 'N/I' + initcap
when(col(c).isNull(), lit('N/I')).otherwise(initcap(lower(col(c))))

# Datas: epoch -> NULL
when(col(c) == lit('1970-01-01'), None).otherwise(col(c))
```

### Flags T/F -> Sim/Nao

```python
when(upper(col(c)) == 'T', 'Sim')
.when(upper(col(c)) == 'F', 'Nao')
.when(col(c).isNull(), 'N/I')
.otherwise(col(c))
```

### Hash para Change Detection (Bronze)

```python
expressao = f"sha1(concat_ws('-', {', '.join(colunas)}))"
df = df.withColumn('hash_diff', expr(expressao))
```

### Chave Surrogada MD5 (Gold)

```python
df = df.withColumn('id_dim_cliente',
    when(col('chave_cliente').isNull(), None)
    .otherwise(md5(col('chave_cliente').cast('string'))))
```

### Join com Alias

```python
df_main = df_main.alias('m')
df_lookup = df_lookup.alias('l')

result = df_main.join(
    df_lookup,
    (col('m.cd_posto') == col('l.cd_posto')) &
    (col('m.nr_conta') == col('l.nr_conta')),
    how='left'
)
```

### Chave Natural Composta

```python
concat(
    col('cd_posto').cast('string'),
    lit('|'),
    col('nr_conta').cast('string')
).alias('chave')
```

### Coluna de Auditoria

```python
data_proc = datetime.now().strftime('%d/%m/%Y %H:%M')
df = df.withColumn('processing_date',
    to_timestamp(lit(data_proc), 'dd/MM/yyyy HH:mm'))
```

## Referencias Detalhadas

- **Operacoes Delta Lake**: Ver [references/delta-operations.md](references/delta-operations.md)
- **Patterns Avancados**: Ver [references/advanced-patterns.md](references/advanced-patterns.md)
- **Troubleshooting**: Ver [references/troubleshooting.md](references/troubleshooting.md)

## Convencoes do Projeto

| Item | Bronze | Silver | Gold |
|------|--------|--------|------|
| Colunas | UPPERCASE | snake_case | snake_case |
| Chaves | - | chave_* | id_dim_* |
| Hash | SHA1 | - | MD5 |
| Datas | string dd/MM/yyyy | DateType | DateType |
| NULLs string | - | 'N/I' | 'N/I' |
| NULLs int | - | 0 | 0 |
| Flags | T/F | Sim/Nao | Sim/Nao |

## Gotchas

- Sempre usar `col()` para join conditions, nunca strings
- Datas em formato brasileiro: `dd/MM/yyyy`
- Strings: `initcap(lower(col))` no Silver
- Hash: `concat_ws('-', ...)` para SHA1, `concat_ws('|', ...)` para chaves naturais
- Sempre adicionar `processing_date` em Silver e Gold
