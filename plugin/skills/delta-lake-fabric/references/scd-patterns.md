# SCD Patterns com Delta Lake

## SCD Type 1 (Overwrite)

Sobrescreve valores antigos. Usado para dados onde historico nao e necessario.

### Implementacao

```python
from delta.tables import DeltaTable

def apply_scd1(spark, df_source, target_table, key_column):
    """
    Aplica SCD Type 1 (overwrite) em uma tabela Delta.

    Args:
        df_source: DataFrame com registros novos/atualizados
        target_table: Nome completo da tabela (ex: 'gold.dim_tipo')
        key_column: Coluna chave para match
    """
    delta_table = DeltaTable.forName(spark, target_table)

    delta_table.alias("target").merge(
        df_source.alias("source"),
        f"target.{key_column} = source.{key_column}"
    ).whenMatchedUpdateAll() \
     .whenNotMatchedInsertAll() \
     .execute()
```

### Exemplo Completo

```python
# 1. Ler do Silver com dedup
df = repository.read(sql="""
    SELECT *,
        ROW_NUMBER() OVER (
            PARTITION BY cd_tipo
            ORDER BY dt_ingestao DESC
        ) as index
    FROM bronze.tipos_movimento
""")
df = df.filter(col("index") == 1).drop("index")

# 2. Transformar e gerar chave surrogada
df = df.withColumn("id_dim_tipo_movimento",
    md5(col("cd_tipo").cast("string")))

# 3. Aplicar SCD Type 1
if table_exists(spark, "gold.dim_tipo_movimento"):
    apply_scd1(spark, df, "gold.dim_tipo_movimento", "id_dim_tipo_movimento")
else:
    df.write.format("delta").saveAsTable("gold.dim_tipo_movimento")
```

## SCD Type 2 (History Tracking)

Preserva historico completo de mudancas. Cada alteracao cria nova versao.

### Colunas Obrigatorias

| Coluna | Tipo | Descricao | Valor Inicial |
|--------|------|-----------|---------------|
| `valid_from` | timestamp | Inicio vigencia | `current_timestamp()` |
| `valid_to` | timestamp | Fim vigencia | `9999-12-31 23:59:59` |
| `is_valid` | boolean | Versao corrente? | `True` |
| `hash_key` | string | Hash das colunas de negocio | SHA1 |

### Implementacao Completa

```python
from pyspark.sql.functions import (
    col, lit, when, current_timestamp, to_timestamp,
    sha1, concat_ws, md5, expr
)
from delta.tables import DeltaTable

def apply_scd2(spark, df_source, target_table, natural_key, business_cols):
    """
    Aplica SCD Type 2 em uma tabela Delta.

    Args:
        df_source: DataFrame com registros do Silver
        target_table: Nome completo da tabela Gold
        natural_key: Coluna de chave natural (ex: 'chave')
        business_cols: Lista de colunas de negocio para hash
    """

    # 1. Adicionar hash de comparacao ao source
    df_source = df_source.withColumn(
        "hash_key",
        sha1(concat_ws("-", *[col(c).cast("string") for c in business_cols]))
    )

    # 2. Verificar se tabela existe
    try:
        delta_table = DeltaTable.forName(spark, target_table)
        df_target = delta_table.toDF()
        table_exists = True
    except:
        table_exists = False

    if not table_exists:
        # Primeira carga: criar tabela com colunas SCD2
        df_initial = df_source \
            .withColumn("valid_from", current_timestamp()) \
            .withColumn("valid_to", to_timestamp(lit("9999-12-31 23:59:59"))) \
            .withColumn("is_valid", lit(True))

        df_initial.write.format("delta").saveAsTable(target_table)
        return

    # 3. Obter registros validos atuais
    df_current = df_target.filter(col("is_valid") == True)

    # 4. Identificar novos e alterados
    df_changes = df_source.alias("s").join(
        df_current.alias("t"),
        col(f"s.{natural_key}") == col(f"t.{natural_key}"),
        how="left"
    ).filter(
        col(f"t.{natural_key}").isNull() |  # Novo registro
        (col("s.hash_key") != col("t.hash_key"))  # Registro alterado
    )

    if df_changes.count() == 0:
        return  # Nada a processar

    # 5. Criar novas versoes
    surrogate_key_col = f"id_dim_{target_table.split('.')[-1].replace('dim_', '')}"

    df_new_versions = df_changes.select(
        md5(concat_ws("-", col(f"s.{natural_key}"), current_timestamp()))
            .alias(surrogate_key_col),
        *[col(f"s.{c}") for c in df_source.columns if c != surrogate_key_col],
    ).withColumn("valid_from", current_timestamp()) \
     .withColumn("valid_to", to_timestamp(lit("9999-12-31 23:59:59"))) \
     .withColumn("is_valid", lit(True))

    # 6. Expirar versoes antigas
    df_to_expire = df_current.alias("t").join(
        df_new_versions.select(natural_key).alias("n"),
        col(f"t.{natural_key}") == col(f"n.{natural_key}"),
        how="inner"
    ).select("t.*") \
     .withColumn("valid_to", expr("current_timestamp() - INTERVAL 1 SECOND")) \
     .withColumn("is_valid", lit(False))

    # 7. Executar MERGE para expirar registros antigos
    if df_to_expire.count() > 0:
        delta_table.alias("target").merge(
            df_to_expire.alias("source"),
            f"target.{surrogate_key_col} = source.{surrogate_key_col}"
        ).whenMatchedUpdate(
            set={
                "valid_to": "source.valid_to",
                "is_valid": "source.is_valid"
            }
        ).execute()

    # 8. Inserir novas versoes
    df_new_versions.write.format("delta").mode("append").saveAsTable(target_table)
```

### Exemplo de Uso

```python
# Definir colunas de negocio (excluindo chaves e audit)
business_cols = ["nome", "endereco", "telefone", "email", "status"]

# Ler mudancas incrementais
last_version = repository.get_last_data_version(spark, "silver.clientes")
df = spark.sql(f"""
    SELECT * FROM table_changes('silver.clientes', {last_version})
""")

# Aplicar SCD Type 2
apply_scd2(
    spark=spark,
    df_source=df,
    target_table="gold.dim_cliente",
    natural_key="chave",
    business_cols=business_cols
)
```

## Consultas em Tabelas SCD Type 2

### Versao Atual

```python
df_atual = spark.read.table("gold.dim_cliente") \
    .filter(col("is_valid") == True)
```

### Versao em Data Especifica

```python
from pyspark.sql.functions import to_timestamp

data_referencia = to_timestamp(lit("2024-06-15 12:00:00"))

df_historico = spark.read.table("gold.dim_cliente") \
    .filter(
        (col("valid_from") <= data_referencia) &
        (col("valid_to") > data_referencia)
    )
```

### Historico Completo de um Registro

```python
df_historico = spark.read.table("gold.dim_cliente") \
    .filter(col("chave") == "123456") \
    .orderBy(col("valid_from").desc())
```

## Tabela de Referencia: Tipo de SCD por Entidade

| Entidade | SCD Type | Motivo |
|----------|----------|--------|
| `dim_cliente` | 2 | Historico de mudancas importante |
| `dim_conta_corrente` | 2 | Auditoria de alteracoes |
| `dim_tipo_movimento` | 1 | Dados de referencia estaticos |
| `dim_agencia` | 1 | Mudancas raras, sem necessidade de historico |
| `dim_calendario` | 1 | Tabela de referencia |
| `fato_*` | 1 | Fatos sao imutaveis ou corrigidos |

## Boas Praticas

1. **Sempre usar hash** para detectar mudancas - evita comparar coluna por coluna
2. **Excluir colunas de audit** do hash (processing_date, valid_from, etc.)
3. **Usar chave surrogada unica** por versao em SCD Type 2
4. **Particionar por is_valid** se tabela for muito grande
5. **Indexar natural_key** para joins eficientes
