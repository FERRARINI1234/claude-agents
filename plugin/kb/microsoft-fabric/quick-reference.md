# Microsoft Fabric Quick Reference

> **MCP Validated:** 2026-02-09

## Layer Summary

| Layer | Lakehouse | Repository | Write Mode | Naming |
|-------|-----------|------------|------------|--------|
| Staging | `staging` | -- | overwrite | Source names |
| Bronze | `bronze` | `GravaBronze` | append (hash dedup) | `sbcc_*`, `sbfi_*` |
| Silver | `silver` | `GravaSilver` | append | `contas_correntes`, `propostas` |
| Gold | `gold` | `GravaGold` | MERGE + append | `dim_*`, `fato_*` |

## Standard Notebook Template

```python
%run <layer>_repository
repository = Grava<Layer>(spark=spark)
df = repository.read(sql="SELECT ... FROM <upstream>.<table>")
# ... transform ...
repository.salva_tabela_<layer>(df, 'table_name', 'layer')
```

## Key Methods Per Layer

**Bronze:** `computa_coluna_hash()`, `adiciona_hash_diff()`, `salva_tabela_bronze()`
**Silver:** `trata_colunas()`, `padroniza_flags()`, `salva_tabela_silver()`, `salva_tabela_parametros()`
**Gold:** `cria_chaves()`, `merge_tabela_gold()`, `get_last_version()`, `get_last_data_version()`

## NULL Conventions

| Type | NULL becomes | Code |
|------|-------------|------|
| String | `'N/I'` | `when(col(c).isNull(), lit('N/I'))` |
| Integer | `0` | `when(col(c).isNull(), 0)` |
| Double | `0.0` | `when(col(c).isNull(), 0)` |
| Date | `NULL` (if `1970-01-01`) | `when(col(c) == lit('1970-01-01'), None)` |

## Flag Normalization

`'T'` -> `'Sim'` | `'F'` -> `'Nao'` | `NULL` -> `'N/I'`

## Key Naming Conventions

| From | To | Method |
|------|----|--------|
| `chave_agencia` | `id_dim_agencia` | `md5(col('chave_agencia'))` |
| `data_abertura` | `id_data_abertura` | `md5(date_format(col, 'yyyyMMdd'))` |
| `CD_POSTO` (Bronze) | `cd_posto` (Silver) | Column aliasing in SQL |

## SCD Type 2 Columns

`valid_from` (timestamp), `valid_to` (`9999-12-31` = current), `is_valid` (boolean), `hash_key` (SHA1)

## Environments

| | DEV | PROD |
|--|-----|------|
| Workspace | Mentors | Dados |
| Config | `CICD/config.yml` -> `dev:` | `CICD/config.yml` -> `prod:` |
| Branch | `develop` | `master` |

## Error Logging

All errors go to `error_mart.log_error` with columns: `ERROR_ITEM`, `ERROR_LOG`, `OBS`, `ERROR_TIMESTAMP`.

## Files Reference

| What | Where |
|------|-------|
| Table registry | `docs/table_registry_new.yaml` |
| CI/CD config | `CICD/config.yml` |
| Pipeline | `azure-pipelines.yml` |
| Bronze repo | `repository/01_bronze/bronze_repository.Notebook/` |
| Silver repo | `repository/02_silver/silver_repository.Notebook/` |
| Gold repo | `repository/03_gold/gold_repository.Notebook/` |
| Error handler | `repository/05_errors/Handlers.Notebook/` |
