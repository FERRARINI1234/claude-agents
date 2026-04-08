# Advanced PySpark Patterns

## SCD Type 2 (Slowly Changing Dimension)

### Implementacao Completa

```python
from pyspark.sql.functions import col, lit, when, current_timestamp, md5

def apply_scd2(spark, df_new, target_table, key_cols, compare_cols):
    """
    Aplica SCD Type 2 em uma dimensao.

    Args:
        df_new: DataFrame com registros novos
        target_table: Nome da tabela destino
        key_cols: Lista de colunas que formam a chave natural
        compare_cols: Lista de colunas para detectar mudancas
    """
    from delta.tables import DeltaTable

    # Adicionar colunas de controle
    df_new = df_new \
        .withColumn("dt_inicio_vigencia", current_timestamp()) \
        .withColumn("dt_fim_vigencia", lit(None).cast("timestamp")) \
        .withColumn("fl_ativo", lit("S"))

    # Criar hash de comparacao
    hash_expr = md5(concat_ws("-", *[col(c).cast("string") for c in compare_cols]))
    df_new = df_new.withColumn("hash_registro", hash_expr)

    delta_table = DeltaTable.forName(spark, target_table)

    # Merge condition
    merge_cond = " AND ".join([f"t.{c} = s.{c}" for c in key_cols])
    merge_cond += " AND t.fl_ativo = 'S'"

    # Executar merge
    (delta_table.alias("t")
        .merge(df_new.alias("s"), merge_cond)
        # Fechar registro antigo se hash mudou
        .whenMatchedUpdate(
            condition="t.hash_registro != s.hash_registro",
            set={
                "dt_fim_vigencia": "s.dt_inicio_vigencia",
                "fl_ativo": "'N'"
            }
        )
        .execute())

    # Inserir novos registros (incluindo versoes novas de registros alterados)
    df_to_insert = df_new.alias("s").join(
        delta_table.toDF().filter(col("fl_ativo") == "S").alias("t"),
        on=[col(f"s.{c}") == col(f"t.{c}") for c in key_cols],
        how="left_anti"
    ).union(
        df_new.alias("s").join(
            delta_table.toDF().filter(col("fl_ativo") == "S").alias("t"),
            on=[col(f"s.{c}") == col(f"t.{c}") for c in key_cols] +
               [col("s.hash_registro") != col("t.hash_registro")],
            how="inner"
        ).select("s.*")
    )

    df_to_insert.write.format("delta").mode("append").saveAsTable(target_table)
```

## Window Functions

### Running Total

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import sum as spark_sum

window_spec = Window \
    .partitionBy("id_dim_cliente") \
    .orderBy("data_movimento") \
    .rowsBetween(Window.unboundedPreceding, Window.currentRow)

df = df.withColumn("saldo_acumulado", spark_sum("valor_movimento").over(window_spec))
```

### Lag/Lead

```python
from pyspark.sql.functions import lag, lead

window_spec = Window.partitionBy("id_dim_cliente").orderBy("data_movimento")

df = df \
    .withColumn("valor_anterior", lag("valor", 1).over(window_spec)) \
    .withColumn("valor_proximo", lead("valor", 1).over(window_spec))
```

### Ranking

```python
from pyspark.sql.functions import rank, dense_rank, row_number

window_spec = Window.partitionBy("categoria").orderBy(col("valor").desc())

df = df \
    .withColumn("rank", rank().over(window_spec)) \
    .withColumn("dense_rank", dense_rank().over(window_spec)) \
    .withColumn("row_num", row_number().over(window_spec))
```

### Moving Average

```python
window_spec = Window \
    .partitionBy("id_dim_cliente") \
    .orderBy("data_movimento") \
    .rowsBetween(-6, 0)  # Ultimos 7 dias

df = df.withColumn("media_movel_7d", avg("valor").over(window_spec))
```

## Pivot e Unpivot

### Pivot (Linhas para Colunas)

```python
df_pivot = df.groupBy("id_dim_cliente") \
    .pivot("tipo_produto", ["A", "B", "C"]) \
    .agg(sum("valor"))
```

### Unpivot (Colunas para Linhas)

```python
from pyspark.sql.functions import expr

df_unpivot = df.select(
    "id_dim_cliente",
    expr("stack(3, 'produto_a', produto_a, 'produto_b', produto_b, 'produto_c', produto_c) as (tipo_produto, valor)")
)
```

## Broadcast Join

Para tabelas pequenas (< 10MB):

```python
from pyspark.sql.functions import broadcast

df_result = df_fato.join(
    broadcast(df_dim_pequena),
    df_fato.id_dim == df_dim_pequena.id_dim,
    how="left"
)
```

## UDFs (User Defined Functions)

### UDF Simples

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType

@udf(returnType=StringType())
def formata_cpf(cpf):
    if cpf and len(cpf) == 11:
        return f"{cpf[:3]}.{cpf[3:6]}.{cpf[6:9]}-{cpf[9:]}"
    return cpf

df = df.withColumn("cpf_formatado", formata_cpf(col("cpf")))
```

### Pandas UDF (Melhor Performance)

```python
from pyspark.sql.functions import pandas_udf
import pandas as pd

@pandas_udf(StringType())
def normaliza_texto(s: pd.Series) -> pd.Series:
    return s.str.strip().str.lower().str.title()

df = df.withColumn("nome_normalizado", normaliza_texto(col("nome")))
```

## Agregacoes Condicionais

```python
from pyspark.sql.functions import sum, when, count

df_agg = df.groupBy("id_dim_cliente").agg(
    sum("valor").alias("total"),
    sum(when(col("tipo") == "CREDITO", col("valor")).otherwise(0)).alias("total_creditos"),
    sum(when(col("tipo") == "DEBITO", col("valor")).otherwise(0)).alias("total_debitos"),
    count(when(col("status") == "ATIVO", True)).alias("qtd_ativos")
)
```

## Distinct Count Aproximado

Para grandes volumes (mais rapido que countDistinct):

```python
from pyspark.sql.functions import approx_count_distinct

df_agg = df.groupBy("regiao").agg(
    approx_count_distinct("id_cliente", rsd=0.05).alias("clientes_unicos_aprox")
)
```

## Explode (Arrays e Maps)

### Explode Array

```python
from pyspark.sql.functions import explode, explode_outer

# Cria uma linha para cada elemento do array
df = df.withColumn("produto", explode(col("lista_produtos")))

# Mantem linhas mesmo se array vazio/null
df = df.withColumn("produto", explode_outer(col("lista_produtos")))
```

### Explode Map

```python
from pyspark.sql.functions import explode

df = df.select(
    col("id"),
    explode(col("atributos")).alias("chave", "valor")
)
```

## Sampling

```python
# Amostra aleatoria de 10%
df_sample = df.sample(fraction=0.1, seed=42)

# Amostra estratificada
df_sample = df.sampleBy("categoria", fractions={"A": 0.1, "B": 0.2, "C": 0.15}, seed=42)
```
