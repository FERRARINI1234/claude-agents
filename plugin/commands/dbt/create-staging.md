# Criar Camada Staging dbt

> Cria modelos `stg_<tabela>.sql` na camada staging de um projeto dbt lendo as tabelas de um `source.yml` existente e salvando os arquivos na mesma pasta.

## Uso

```bash
/dbt:create-staging
/dbt:create-staging <caminho/para/source.yml>
```

## Exemplos

```bash
# Criacao interativa (pergunta o source.yml)
/dbt:create-staging

# Apontando diretamente para o source.yml
/dbt:create-staging src/dbt/lakehouse/models/staging/consinco/source.yml

# Caminho absoluto
/dbt:create-staging /opt/TAF-etl-n3-dw/src/dbt/lakehouse/models/staging/consinco/source.yml
```

---

## Visao Geral

Este comando faz parte do fluxo de criacao da camada staging no dbt:

```text
Etapa 1: /dbt:create-source  → source.yml
Etapa 2: /dbt:create-staging → stg_<tabela>.sql   (ESTE COMANDO)
```

---

## O Que Este Comando Faz

1. **Localizar** — Resolve o caminho do `source.yml` (argumento ou interativo)
2. **Ler** — Extrai `source name` e lista de tabelas do `source.yml`
3. **Questionar** — Pergunta quais tabelas criar e se serao incrementais
4. **Delegar** — Aciona o agente `dbt-especialist` para gerar os `.sql` via Skill dbt
5. **Salvar** — Arquivos criados na mesma pasta do `source.yml`

---

## Opcoes

| Comando | Acao |
|---------|------|
| `/dbt:create-staging` | Interativo — pergunta o source.yml e as tabelas |
| `/dbt:create-staging <source.yml>` | Usa source.yml informado, pergunta apenas as tabelas |

---

## Processo

### Passo 1: Resolver o source.yml

Se `<caminho/para/source.yml>` foi passado como argumento, usar diretamente e confirmar:

```
source.yml encontrado: <caminho>
```

Se nao foi passado, buscar automaticamente antes de perguntar:

```text
Glob: src/dbt/**/models/**/source.yml
```

- Se encontrar **um unico** arquivo → confirmar com o usuario se e o correto
- Se encontrar **multiplos** → listar e perguntar qual usar
- Se **nao encontrar nenhum** → solicitar o caminho ao usuario:

```
Nenhum source.yml encontrado.
Informe o caminho do source.yml:
(ex: src/dbt/lakehouse/models/staging/consinco/source.yml)
```

### Passo 2: Ler o source.yml e Extrair Dados

Usar `Read` para carregar o `source.yml`. Extrair obrigatoriamente:

| Campo extraido | Origem no YAML | Usado como |
|----------------|----------------|------------|
| `source_name`  | `sources[0].name` | `sistema_origem` na skill dbt |
| `tabelas`      | `sources[0].tables[].name` | lista de tabelas a processar |

A **pasta de destino** dos `.sql` e sempre a mesma pasta do `source.yml`:

```text
pasta_destino = dirname(caminho_do_source_yml)
```

Exemplo: `source.yml` em `.../staging/consinco/` → `.sql` salvos em `.../staging/consinco/`

### Passo 3: Apresentar Tabelas e Selecionar

Exibir as tabelas encontradas e perguntar ao usuario quais criar:

```markdown
source.yml: staging/consinco/source.yml
Source name: consinco
Tabelas encontradas (12):

  1. GE_PESSOA
  2. MAD_CLIENTEEND
  3. MAD_PEDVENDA
  ...

Quais tabelas deseja criar?
  (a) Todas
  (b) Selecionar especificas (informe os numeros: ex: 1,3,5)
```

### Passo 4: Perguntar sobre Incremental

Para cada tabela selecionada, perguntar:

```
A tabela <TABELA> sera incremental?
  (a) Nao — full refresh (PADRAO)
  (b) Sim — incremental
```

Se **Sim**, coletar tambem:
- `unique_key` — coluna chave unica (ex: `SEQPEDVENDA`)
- `coluna_data` — coluna de data usada no filtro incremental (ex: `DTAINCLUSAO`)

> **Regra:** nao presumir incremental — sempre perguntar. Se houver duvida, usar full refresh.

### Passo 5: Acionar dbt-especialist

Com todos os dados coletados, acionar o agente `dbt-especialist` passando:

- Lista de tabelas a processar (com configuracao de cada uma)
- `source_name` extraido do `source.yml`
- `pasta_destino` = pasta do `source.yml` (NAO perguntar ao agente — ja esta definida)
- Configuracao de incremental por tabela (quando aplicavel)

Instrucao ao agente:

```
Para cada tabela abaixo, use a Skill dbt para gerar o modelo staging:

  Skill: /dbt staging <TABELA> <SOURCE_NAME>
  Pasta de destino: <PASTA_DESTINO> (nao perguntar — usar esta pasta diretamente)
  Source name: <SOURCE_NAME>

  Tabelas:
    - <TABELA_1>: full refresh
    - <TABELA_2>: incremental | unique_key=<KEY> | coluna_data=<COL>
    ...

Nomeacao dos arquivos: stg_<nomedatabela_em_minusculo>.sql
```

### Passo 6: Validar Saida

Verificar que os arquivos foram criados corretamente:

```bash
ls <pasta_destino>/stg_*.sql
```

---

## Loop de Execucao

```text
┌─────────────────────────────────────────────────────────────┐
│               LOOP DO CREATE-STAGING                        │
├─────────────────────────────────────────────────────────────┤
│  1. Resolver source.yml (argumento ou Glob ou usuario)      │
│  2. Ler source.yml → extrair source_name + tabelas          │
│  3. Perguntar: todas as tabelas ou selecionar especificas   │
│  4. Para cada tabela selecionada:                           │
│     └─ Perguntar: incremental? (unique_key + coluna_data)   │
│  5. Acionar dbt-especialist com pasta_destino definida      │
│     └─ Se FALHAR em uma tabela → registrar e continuar      │
│  6. Exibir relatorio final com arquivos criados             │
│  7. Sugerir: dbt run --select staging                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Saida

| Artefato | Localizacao |
|----------|-------------|
| **stg_\<tabela\>.sql** | mesma pasta do `source.yml` |

**Formato do arquivo gerado (full refresh):**

```sql
{% set nome_tabela = 'NOME_TABELA' %}
{% set sistema_origem = "'SOURCE_NAME'" %}
{% set formato_arquivo = "'iceberg'" %}

SELECT
    *,
    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') as DATA_INGESTAO,
    {{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}
FROM
{{ source('SOURCE_NAME', nome_tabela) }}
```

**Formato do arquivo gerado (incremental):**

```sql
{{
    config(
        materialized='incremental',
        unique_key='<UNIQUE_KEY>',
        on_schema_change='append_new_columns'
    )
}}

{% set nome_tabela = 'NOME_TABELA' %}
{% set sistema_origem = "'SOURCE_NAME'" %}
{% set formato_arquivo = "'iceberg'" %}

SELECT
    *,
    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') as DATA_INGESTAO,
    {{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}
FROM
{{ source('SOURCE_NAME', nome_tabela) }}
{% if is_incremental() %}
WHERE CAST(<COLUNA_DATA> AS DATE) >= (select max(CAST(<COLUNA_DATA> AS DATE)) from {{ this }})
{% endif %}
```

**Relatorio final:**

```markdown
## Staging criado

**source.yml:** staging/consinco/source.yml
**Source name:** consinco
**Pasta de destino:** staging/consinco/
**Tabelas processadas:** N

| # | Tabela | Arquivo | Tipo |
|---|--------|---------|------|
| 1 | GE_PESSOA | stg_ge_pessoa.sql | full refresh |
| 2 | MAD_PEDVENDA | stg_mad_pedvenda.sql | incremental |

### Proximo Passo
dbt run --select staging
```

---

## Controle de Qualidade

Antes de marcar como concluido, verificar:

```text
[ ] source.yml lido e source_name extraido corretamente
[ ] Lista de tabelas extraida do source.yml (nao digitada manualmente)
[ ] Pasta de destino = mesma pasta do source.yml
[ ] Perguntado sobre incremental para cada tabela (nunca presumido)
[ ] Skill dbt invocada para cada tabela (nao gerou SQL manualmente)
[ ] Arquivos nomeados como stg_<tabela_em_minusculo>.sql
[ ] Relatorio final exibido com lista de arquivos criados
```

---

## Lidando com Problemas

| Problema | Acao |
|----------|------|
| `source.yml` nao encontrado | Solicitar caminho ao usuario |
| `source.yml` sem tabelas listadas | Informar ao usuario e encerrar |
| source.yml com multiplos sources | Perguntar qual source usar |
| Tabela ja possui `stg_*.sql` na pasta | Alertar usuario e perguntar se substitui |
| Falha em uma tabela especifica | Registrar erro, continuar com as demais e reportar no resumo |
| Bloqueio na Skill dbt | Parar e informar ao usuario |

---

## Premissas

- **Pasta de destino = pasta do source.yml** — os `.sql` sao sempre salvos na mesma pasta do `source.yml` usado como entrada; nunca perguntar ao usuario sobre o destino
- **source_name vem do source.yml** — o campo `sources[0].name` do `source.yml` e o `sistema_origem` usado na skill dbt; nunca presumir
- **Nomeacao em minusculo** — `stg_<tabela_em_minusculo>.sql` (ex: `stg_mad_pedvenda.sql`)
- **Incremental nunca presumido** — sempre perguntar; padrao e full refresh

---

## Restricoes

- NUNCA gerar SQL de staging manualmente — sempre usar a Skill dbt via `dbt-especialist`
- NUNCA presumir `source_name` — ler sempre do campo `name` do `source.yml`
- NUNCA perguntar a pasta de destino — ela e sempre a mesma pasta do `source.yml`
- NUNCA presumir que uma tabela e incremental sem confirmar com o usuario
- SEMPRE processar todas as tabelas selecionadas, mesmo que uma falhe
- SEMPRE nomear os arquivos em minusculo: `stg_<tabela>.sql`

---

## Dicas

1. **source_name como sistema_origem** — o `name:` do source.yml e exatamente o valor passado para `{{ source('NAME', tabela) }}` e para a skill `/dbt staging <tabela> <NAME>`
2. **Em lote** — ao selecionar "todas as tabelas", o agente processa uma por vez mas entrega o relatorio consolidado no final
3. **Incremental em lote** — se varias tabelas forem incrementais com a mesma coluna de data, informar de uma vez ao agente para agilizar

---

## Referencias

- Agente: `.claude/agents/dbt/dbt-especialist.md`
- Skill: `.claude/skills/dbt/SKILL.md`
- Comando anterior: `.claude/commands/dbt/create-source.md`