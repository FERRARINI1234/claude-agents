# Repository Pattern

> **MCP Validated:** 2026-02-09

## Problem

Each medallion layer (Bronze, Silver, Gold) has distinct read/write semantics (hash-dedup vs append vs MERGE). Without a shared interface, every notebook would duplicate boilerplate for error handling, metadata management, and write operations.

## Solution

An Abstract Base Class (`Repository`) defines the interface. Each layer implements a concrete subclass with layer-specific behavior. An `ErrorFactory` / `Error` pair centralizes error logging.

### Class Hierarchy

```
Repository (ABC)                      repository/01_bronze/base_repository.Notebook
  |-- GravaBronze                     repository/01_bronze/bronze_repository.Notebook
  |-- GravaSilver                     repository/02_silver/silver_repository.Notebook
  |-- GravaGold                       repository/03_gold/gold_repository.Notebook

ErrorFactory (ABC)                    repository/05_errors/error_factory.Notebook
  |-- Error                           repository/05_errors/Handlers.Notebook
```

### Base Repository (ABC)

```python
class Repository(ABC):
    def __init__(self, spark):
        self.spark = spark

    @abstractmethod
    def add_metadados(self, **kwargs): pass

    @abstractmethod
    def read(self, **kwargs): pass

    @abstractmethod
    def save(self, **kwargs): pass

    def computa_coluna_hash(self, nome_tabela, colunas_ignorar): ...
    def adiciona_hash_diff(self, tabela, expressao_hash): ...
    def salva_tabela_bronze(self, dataframe_origem, tabela_destino, coluna_hash, tabela_existe): ...
```

### Layer Implementations

| Class | Layer | Key Methods |
|-------|-------|-------------|
| `GravaBronze` | Bronze | `add_metadados()`, `read(tabela=)`, `save(dataframe=, format=, mode=, table_name=)`, `computa_coluna_hash()`, `adiciona_hash_diff()`, `salva_tabela_bronze()` |
| `GravaSilver` | Silver | `read(sql=)`, `trata_colunas()`, `padroniza_flags()`, `salva_tabela_silver()`, `salva_tabela_parametros()` |
| `GravaGold` | Gold | `read(sql=)`, `cria_chaves()`, `merge_tabela_gold()`, `get_last_version()`, `get_last_data_version()`, `salva_tabela_gold()` |

### Error Handling

All repositories compose an `Error` instance for centralized logging:

```python
class GravaSilver(Repository):
    def __init__(self, spark):
        super().__init__(spark)
        self.error_handler = Error(spark)

    def read(self, **kwargs):
        try:
            return self.spark.sql(kwargs.get('sql'))
        except Exception as ex:
            self.error_handler.throw_exception(
                error_item="read_bronze_table",
                error_log=str(ex),
                obs=f"Erro ao executar a instrucao sql: {instrucao}")
```

The `Error` class logs to `error_mart.log_error`:

```python
class Error(ErrorFactory):
    def throw_exception(self, **kwargs):
        self.spark.sql("""
            CREATE TABLE IF NOT EXISTS error_mart.log_error (
                ERROR_ITEM STRING, ERROR_LOG STRING,
                OBS STRING, ERROR_TIMESTAMP STRING
            )""")
        self.spark.sql(f"""
            INSERT INTO error_mart.log_error
            VALUES ('{error_item}', '{error_log}', '{error_obs}', '{error_timestamp}')
        """)
```

## Code Example

How a transformation notebook uses the pattern:

```python
# 1. Import via %run (loads Repository ABC + layer class + Error)
%run silver_repository

# 2. Instantiate
repository = GravaSilver(spark=spark)

# 3. Read from upstream
df = repository.read(sql="SELECT ... FROM bronze.sbcc_ccorrente")

# 4. Transform
df = repository.trata_colunas(df, columns)
df = repository.padroniza_flags(df, columns)

# 5. Save
repository.salva_tabela_silver(df, 'contas_correntes', 'silver')
```

## When to Apply

- Every new transformation notebook MUST use the corresponding repository class.
- Do not write directly to Delta tables outside the repository methods.
- Always instantiate with `spark=spark` (the global Spark session).

## Trade-offs

**Pros:**
- Consistent error handling across all notebooks
- Write semantics encapsulated per layer (dedup, append, merge)
- Single place to modify write behavior (e.g., adding audit columns)

**Cons:**
- `%run` imports are not true module imports -- no IDE autocomplete
- Error class uses string interpolation for SQL, which could be fragile with special characters
- The ABC defines Bronze-specific methods (`computa_coluna_hash`, `salva_tabela_bronze`) that are not abstract, mixing concerns
