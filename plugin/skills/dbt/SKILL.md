# Skill: dbt

> Especialista dbt para projetos Mentorstec. Gera modelos dbt seguindo padrões Medallion Architecture.
> **OBRIGATORIO:** Consultar o agente `medallion-architect` antes de criar ou modificar qualquer modelo intermediate ou marts.
> Use este skill sempre que precisar criar ou manipular modelos dbt em qualquer camada.

## Uso

```bash
/dbt                                                        # modo interativo — pergunta tudo
/dbt staging CLIENTE ERP_SYS                                # cria modelo staging
/dbt staging <tabela> <sistema>
/dbt intermediate <nome_modelo>                             # cria modelo intermediate
/dbt marts <tipo> <nome_tabela>                             # cria modelo marts (tipo: fato ou dim)
/dbt source create <source_name> <database> <schema>        # cria source.yml
/dbt source add table <tabela>                              # adiciona tabela a source.yml existente
/dbt source from-file <path_to_schema.yml>                  # cria source.yml a partir de arquivo schema
```

---

## Sub-comandos

| Comando | Descrição |
|---------|-----------|
| `/dbt staging` | Cria modelo da camada Staging (`stg_<nomedatabela>.sql`) |
| `/dbt intermediate` | Cria modelo da camada Intermediate (`int_<nomedomodelo>.sql`) |
| `/dbt marts` | Cria modelo da camada Marts (`fato_<nomedatabela>.sql` ou `dim_<nomedatabela>.sql`) |
| `/dbt source create` | Cria um novo arquivo `source.yml` |
| `/dbt source add table` | Adiciona uma tabela a um `source.yml` existente |
| `/dbt source from-file` | Lê um `schema.yml` e gera `source.yml` com todas as tabelas |

Se nenhum sub-comando for passado, perguntar ao usuário os parâmetros necessários.

---

## Processo de Execução — `source create`

### Passo 1 — Coletar parâmetros do source

Se não fornecidos como argumento, perguntar:

- `source_name` — nome do source dbt (ex: `ERP_SYS`, `CRM`)
- `database` — nome do banco de dados (ex: `TRUCKS`, `ANALYTICS`)
- `schema` — schema completo (ex: `ERP_SYS.dbo`, `CRM.public`)

### Passo 2 — Localizar pasta de destino

**Obrigatório:** perguntar ao usuário onde salvar o arquivo.

Sugerir locais comuns encontrados via `Glob`:

```
models/staging/
models/01_staging/
models/sources/
```

Se o usuário não indicar a pasta, **não prosseguir** — solicitar o caminho.

### Passo 3 — Verificar se já existe source.yml na pasta

Usar `Glob` para verificar se `source.yml` ou `_sources.yml` já existe na pasta indicada.

- Se existir: avisar o usuário e perguntar se deseja substituir ou usar `/dbt source add table` para acrescentar tabelas.
- Se não existir: criar o arquivo com o template padrão (lista de tabelas vazia).

### Passo 4 — Gerar o arquivo `source.yml`

Aplicar o template da seção **Template source.yml**.

### Passo 5 — Confirmar saída

Exibir o conteúdo gerado e o caminho do arquivo.

---

## Processo de Execução — `source add table`

### Passo 1 — Coletar parâmetros

Se não fornecido como argumento, perguntar:

- `tabela` — nome da tabela a adicionar (ex: `CLIENTE`, `PEDIDO`)

### Passo 2 — Localizar o `source.yml` existente

Usar `Glob` para encontrar arquivos `source.yml` ou `_sources.yml` no projeto.

- Se encontrar **um único arquivo**: confirmar com o usuário se é o correto.
- Se encontrar **múltiplos arquivos**: listar e perguntar qual usar.
- Se **não encontrar nenhum**: perguntar ao usuário:
  ```
  Nenhum source.yml foi encontrado.
    (a) O caminho está errado? Informe o caminho correto.
    (b) Deseja criar um novo source.yml? Use /dbt source create
  ```

### Passo 3 — Ler o arquivo existente

Usar `Read` para carregar o conteúdo atual do `source.yml`.

### Passo 4 — Verificar duplicidade

Checar se a tabela já existe na lista `tables:` do source.

- Se já existir: avisar o usuário e não duplicar.
- Se não existir: adicionar `- name: <TABELA>` na lista `tables:`, mantendo a ordem alfabética quando possível.

### Passo 5 — Salvar e confirmar

Usar `Edit` para atualizar o arquivo. Exibir o diff da alteração.

---

## Processo de Execução — `source from-file`

Permite criar ou popular um `source.yml` dbt extraindo todas as tabelas de um arquivo `schema.yml` de referência.

### Formato aceito do schema.yml de entrada

```yaml
database: 'CONSINCO'
schema:
  - name: 'CONSINCO'
    tables:
      - name: 'MLF_NOTAFISCAL'
        columns:
          - name: 'DTAHORLANCTO'
      - name: 'CLIENTE'
        columns:
          - name: 'SEQPESSOA'
```

> Campos relevantes extraídos: `database` (raiz), `schema[].name` (usado como `source_name` e `schema`), `schema[].tables[].name` (lista de tabelas).

### Passo 1 — Receber o caminho do arquivo

Se não fornecido como argumento, perguntar:

```
Informe o caminho do arquivo schema.yml (ex: /opt/TAF-etl-n3-dw/src/tables/schema.yml):
```

### Passo 2 — Ler e parsear o schema.yml

Usar `Read` para carregar o arquivo. Extrair os valores candidatos:

- `database` — valor do campo raiz `database:`
- Para cada entrada em `schema:`:
  - `source_name` = `schema[i].name`
  - `tables` = lista de `schema[i].tables[].name`

Se o arquivo não existir ou não seguir o formato esperado, informar o erro e encerrar.

### Passo 3 — OBRIGATÓRIO: Coletar e confirmar parâmetros do source com o usuário

**NUNCA presumir os valores extraídos do arquivo.** Mesmo que o schema.yml contenha `database`, `name` e `schema`, esses valores dependem do adapter dbt configurado e podem diferir do arquivo de referência.

Usar `AskUserQuestion` para coletar os três parâmetros obrigatórios:

```
Os seguintes valores foram extraídos do arquivo:
  → name:     <valor_extraido>
  → database: <valor_extraido>
  → schema:   <valor_extraido>

Confirme ou corrija os parâmetros do source dbt:

  name:     nome do source (ex: CONSINCO, ERP_SYS, CRM)
            → Aparecerá em {{ source('NAME', 'TABELA') }}

  database: nome do banco/catálogo no warehouse (ex: CONSINCO, ANALYTICS)
            → Depende do adapter: Spark usa catálogo, Snowflake usa database, etc.

  schema:   schema completo (ex: CONSINCO.dbo, ERP_SYS.public, dbo)
            → Depende do banco de dados de origem
```

**Só prosseguir após o usuário confirmar ou fornecer os três valores.**

### Passo 4 — Exibir resumo e confirmar

Exibir um resumo antes de gerar os arquivos:

```
Arquivo: /opt/TAF-etl-n3-dw/src/tables/schema.yml
Database: <confirmado_pelo_usuario>
Sources encontrados:
  → <name>: 10 tabelas (MLF_NOTAFISCAL, MFL_DOCTOFISCAL, MAD_PEDVENDA ...)
Deseja prosseguir? (s/n)
```

### Passo 5 — Localizar pasta de destino

Perguntar ao usuário onde salvar o `source.yml`. Sugerir locais comuns encontrados via `Glob`:

```
models/staging/
models/01_staging/
models/sources/
```

### Passo 6 — Verificar se já existe source.yml

Usar `Glob` para checar se `source.yml` ou `_sources.yml` já existe na pasta indicada.

- Se existir: perguntar se deseja **substituir** ou **mesclar** (acrescentar tabelas novas sem duplicar).
- Se não existir: criar o arquivo com todas as tabelas extraídas.

### Passo 7 — Gerar o arquivo source.yml

Para cada source encontrado, gerar a seção correspondente seguindo o **Template source.yml**, usando os valores **confirmados pelo usuário** no Passo 3. Tabelas em ordem alfabética dentro de cada source.

### Passo 8 — Confirmar saída

Exibir o conteúdo gerado, o caminho do arquivo e o total de tabelas incluídas.

---

## Processo de Execução — `staging`

### Passo 1 — Coletar parâmetros base + SQL opcional

Se não fornecidos como argumento, perguntar:

- `nome_tabela` — ex: `CLIENTE`, `PEDIDO`, `RECEBPAGTS`
- `sistema_origem` — ex: `ERP_SYS`, `CRM`, `SAP`
- `sql_base` *(opcional)* — SQL de referência para extrair colunas (ex: SELECT de uma query existente)

> **Se o usuário não fornecer SQL → usar `SELECT *` (comportamento padrão).**

---

### Passo 2 — Analisar SQL e extrair tabela/colunas (apenas se SQL fornecida)

Se o usuário forneceu uma SQL como base, analisar o conteúdo para extrair:

1. **Nome da tabela** — identificar a cláusula `FROM` principal da query.
   - Ignorar aliases, schemas e sub-queries; usar apenas o nome da tabela principal.
   - Se identificado, usar como `nome_tabela` (sobrepõe o valor do Passo 1, confirmar com o usuário).

2. **Lista de colunas** — extrair todas as colunas listadas na cláusula `SELECT`.
   - Preservar aliases quando existirem (ex: `SEQPESSOA AS id_pessoa` → alias `id_pessoa`).
   - Ignorar expressões complexas; incluir coluna bruta com comentário `-- verificar`.
   - Excluir colunas de metadados (`DATA_INGESTAO`, `add_metadata`) se já estiverem na SQL.

**Se a SQL não puder ser parseada** (ex: formato inválido) → avisar o usuário e usar `SELECT *`.

---

### Passo 3 — Verificar se staging já existe

Usar `Glob` para verificar se o arquivo `stg_<nome_tabela>.sql` já existe na pasta de staging do projeto:

```
models/staging/stg_<nome_tabela>.sql
models/01_staging/stg_<nome_tabela>.sql
```

**Cenário A — Arquivo JÁ existe:**

1. Usar `Read` para carregar o conteúdo atual do arquivo.
2. Identificar o bloco `SELECT` existente.
3. Inserir as colunas extraídas no Passo 2 dentro do `SELECT`, **antes** de `DATA_INGESTAO` e `add_metadata`.
4. Não duplicar colunas que já estejam presentes.
5. Usar `Edit` para atualizar o arquivo com as novas colunas.
6. Exibir o diff da alteração e encerrar (pular Passos 4–6).

**Cenário B — Arquivo NÃO existe:**

Continuar para o Passo 4 (criação de novo arquivo).

---

### Passo 4 — Perguntar sobre Incremental *(apenas para criação de novo arquivo)*

**SEMPRE** perguntar ao usuário:

```
A tabela será incremental?
  (a) Não — tabela full refresh (PADRÃO)
  (b) Sim — tabela incremental
```

**Se não houver resposta ou dúvida → NÃO será incremental.**

Se a resposta for **Sim**, coletar também:

- `unique_key` — coluna chave única (ex: `RPVID`, `CLIVID`)
- `coluna_data_incremental` — coluna de data usada no filtro (ex: `RPVDATAPAGAMENTO`, `CREATED_AT`)

---

### Passo 5 — Localizar pasta de destino

**Se `pasta_destino` foi fornecida pelo chamador (ex: comando `/dbt:create-staging`) → usar diretamente, sem perguntar ao usuário.**

Caso contrário, usar `Glob` para encontrar a estrutura do projeto dbt:

```
models/staging/
models/01_staging/
```

Se não encontrar via Glob, perguntar ao usuário onde salvar.

---

### Passo 6 — Gerar o arquivo SQL

Aplicar o template correspondente (ver seções abaixo):

- **Com SQL fornecida** → usar template com colunas explícitas extraídas no Passo 2.
- **Sem SQL** → usar template `SELECT *` padrão.

---

### Passo 7 — Confirmar saída

Exibir o conteúdo gerado, o caminho do arquivo e os próximos passos dbt.

---

## Template source.yml (Padrão Mentorstec)

**Arquivo:** `models/staging/source.yml` *(ou pasta indicada pelo usuário)*

```yaml
version: "2"

sources:
  - name: <SOURCE_NAME>
    database: <DATABASE>
    schema: <SOURCE_NAME>.dbo

    tables:
      - name: <TABELA>
```

> **Regras:**
> - `name` do source: exatamente como aparece no sistema de origem (ex: `ERP_SYS`)
> - `database`: nome do banco de dados no warehouse (ex: `TRUCKS`)
> - `schema`: padrão `<SOURCE_NAME>.dbo` salvo indicação contrária do usuário
> - Cada tabela é um item `- name:` dentro de `tables:`
> - Manter tabelas em ordem alfabética

### Exemplo com múltiplas tabelas

```yaml
version: "2"

sources:
  - name: ERP_SYS
    database: TRUCKS
    schema: ERP_SYS.dbo

    tables:
      - name: CLIENTE
      - name: DIREITOS
      - name: PEDIDO
      - name: RECEBPAGTS
```

---

## Templates Staging (Padrão Mentorstec)

### Staging — Full Refresh sem SQL (padrão)

Usado quando **nenhuma SQL é fornecida** como base.

**Arquivo:** `models/staging/stg_<nomedatabela>.sql`

```sql
{% set nome_tabela = '<NOME_TABELA>' %}
{% set sistema_origem = "'<SISTEMA_ORIGEM>'" %}
{% set formato_arquivo = "'iceberg'" %}

SELECT
    *,
    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') as DATA_INGESTAO,
    {{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}
FROM
{{ source('<SISTEMA_ORIGEM>', nome_tabela) }}
```

---

### Staging — Full Refresh com SQL (colunas explícitas)

Usado quando **uma SQL é fornecida** e as colunas são extraídas dela.

**Arquivo:** `models/staging/stg_<nomedatabela>.sql`

```sql
{% set nome_tabela = '<NOME_TABELA>' %}
{% set sistema_origem = "'<SISTEMA_ORIGEM>'" %}
{% set formato_arquivo = "'iceberg'" %}

SELECT
    <COLUNA_A>,
    <COLUNA_B>,
    <COLUNA_C>,
    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') as DATA_INGESTAO,
    {{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}
FROM
{{ source('<SISTEMA_ORIGEM>', nome_tabela) }}
```

> **Nota:** `nome_tabela` em MAIÚSCULAS sem aspas externas. `sistema_origem` com aspas simples internas. O macro `add_metadata` adiciona colunas de metadados padronizadas.
> Quando colunas são extraídas de SQL, **não usar `*`** — listar as colunas explicitamente na mesma ordem que aparecem na SQL original.

---

### Staging — Incremental sem SQL (padrão)

**Arquivo:** `models/staging/stg_<nomedatabela>.sql`

```sql
{{
    config(
        materialized='incremental',
        unique_key='<UNIQUE_KEY>',
        on_schema_change='append_new_columns'
    )
}}

{% set nome_tabela = '<NOME_TABELA>' %}
{% set sistema_origem = "'<SISTEMA_ORIGEM>'" %}
{% set formato_arquivo = "'iceberg'" %}

SELECT
    *,
    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') as DATA_INGESTAO,
    {{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}
FROM
{{ source('<SISTEMA_ORIGEM>', nome_tabela) }}
{% if is_incremental() %}
WHERE CAST(<COLUNA_DATA> AS DATE) >= (select max(CAST(<COLUNA_DATA> AS DATE)) from {{ this }})
{% endif %}
```

---

### Staging — Incremental com SQL (colunas explícitas)

**Arquivo:** `models/staging/stg_<nomedatabela>.sql`

```sql
{{
    config(
        materialized='incremental',
        unique_key='<UNIQUE_KEY>',
        on_schema_change='append_new_columns'
    )
}}

{% set nome_tabela = '<NOME_TABELA>' %}
{% set sistema_origem = "'<SISTEMA_ORIGEM>'" %}
{% set formato_arquivo = "'iceberg'" %}

SELECT
    <COLUNA_A>,
    <COLUNA_B>,
    <COLUNA_C>,
    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') as DATA_INGESTAO,
    {{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}
FROM
{{ source('<SISTEMA_ORIGEM>', nome_tabela) }}
{% if is_incremental() %}
WHERE CAST(<COLUNA_DATA> AS DATE) >= (select max(CAST(<COLUNA_DATA> AS DATE)) from {{ this }})
{% endif %}
```

---

## Template Intermediate (Padrão Mentorstec)

**Arquivo:** `models/intermediate/int_<nomedomodelo>.sql`

```sql
with <nome_cte> as (
    SELECT
        -- colunas com cast e transformações de negócio
        CAST(<COLUNA_CHAVE> AS INT) as <alias_chave>,
        UPPER(<COLUNA_TEXTO>) as <alias_texto>,
        CAST(<COLUNA_DATA> AS DATE) AS <alias_data>,
        CASE
            WHEN <CONDICAO> THEN <VALOR_A>
            ELSE <VALOR_B>
        END <alias_calculado>,
        -- chaves de relacionamento
        CAST(<COLUNA_FK> AS VARCHAR(100)) AS <alias_chave_fk>,
        CAST(COALESCE(TO_CHAR(<COLUNA_DATA>, 'yyyyMMdd'), '0') AS VARCHAR(10)) AS <alias_chave_data>,
        from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') AS processing_date
    FROM {{ ref('<stg_modelo_origem>') }}
),
<nome_cte_2> as (
    SELECT
        a.*,
        b.<coluna_join>
    FROM <nome_cte> a
    INNER JOIN {{ ref('<outro_modelo>') }} b
        ON b.<coluna_b> BETWEEN a.<data_inicio> AND a.<data_fim>
)
SELECT
    <chave_composta> AS chave,
    <col_1>,
    <col_2>,
    -- demais colunas
    processing_date
FROM <nome_cte_2>
```

> **Regras Intermediate:**
> - Contém lógica de negócio: cálculos, joins, transformações de tipo
> - Referencia sempre modelos staging via `{{ ref('stg_...') }}`
> - Inclui `processing_date` com timezone `America/Sao_Paulo`
> - Chaves de relacionamento em `VARCHAR` com `CAST` explícito
> - Chaves de data no formato `yyyyMMdd` com `TO_CHAR`
> - **Nomenclatura de colunas (OBRIGATÓRIO):**
>   - Todos os aliases de coluna em **minúsculo**: `AS chave`, `AS descricao`, nunca `AS CHAVE` ou `AS DESCRICAO`
>   - **Proibido** trailing underscore: `AS chave_` → corrigir para `AS chave`
>   - Underscores internos são permitidos: `AS chave_regiao`, `AS cod_representante`
>   - Nomes de modelos em **plural**: `int_marcas`, `int_representantes`, `int_rotas`
>   - **Proibido** prefixo `dim_` ou `fato_` na camada intermediate: usar somente `int_<plural>`
> - **DECODE Oracle:** substituir `DECODE(col, v1, r1, v2, r2)` pela macro `{{ decode('col', "'v1'", "'r1'", "'v2'", "'r2'") }}`

---

## Template Marts (Padrão Mentorstec)

### Marts — Dimensão

**Arquivo:** `models/marts/<dominio>/dim_<nomedatabela>.sql`

```sql
SELECT
    -- 1ª coluna: ID surrogate (chave primária da dimensão) — SEMPRE primeiro
    md5(CAST(<col_id_origem> AS VARCHAR)) AS id_dim_<nome>,

    -- atributos descritivos da dimensão
    <col_situacao>,
    <col_codigo>,
    <col_nome>,
    -- demais atributos descritivos...

    -- IDs de outras dimensões (snowflake) — SEMPRE no final, antes de processing_date
    md5(CAST(<col_fk_dim> AS VARCHAR)) AS id_dim_<outra_dimensao>,

    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') AS processing_date
FROM {{ ref('<modelo_intermediate_ou_staging>') }}
```

### Marts — Fato

**Arquivo:** `models/marts/<dominio>/fato_<nomedatabela>.sql`

> **Domínio:** Sempre perguntar ao usuário antes de criar o arquivo.
> Exemplos: `comercial`, `estoque`, `financeiro`, `logistica`
> NUNCA usar o nome do banco/sistema de origem como pasta (ex: não usar `consinco`, `erp_sys`).

```sql
SELECT
    -- 1ª coluna: chave primária da fato — SEMPRE primeiro
    <col_chave> AS chave,

    -- métricas e medidas
    <col_metrica_1>,
    <col_metrica_2>,
    -- demais métricas...

    -- IDs de dimensões (chaves estrangeiras) — SEMPRE no final, antes de processing_date
    md5(CAST(<col_fk_dim_1> AS VARCHAR)) AS id_dim_<dimensao_1>,
    md5(CAST(<col_fk_dim_2> AS VARCHAR)) AS id_dim_<dimensao_2>,

    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') AS processing_date
FROM {{ ref('<modelo_intermediate>') }}
```

> **Regras Marts:**
> - Apenas SELECTs simples — sem lógica de negócio (deve estar no intermediate)
> - Referencia sempre modelos intermediate via `{{ ref('int_...') }}`
> - Inclui `processing_date` com timezone `America/Sao_Paulo`
> - **Dimensões:** 1ª coluna = `id_dim_<nome>` gerado via `md5(CAST(<col> AS VARCHAR))` (chave primária); atributos no meio; IDs snowflake (FKs para outras dimensões) no final antes de `processing_date`
> - **Fatos:** 1ª coluna = `chave` (chave primária da fato); métricas no meio; todos os `id_dim_*` (FKs para dimensões) no final antes de `processing_date`
> - Surrogate key: sempre `md5(CAST(<col_origem> AS VARCHAR))` — nunca `hex()` ou sequência

---

## Convenções Mentorstec

| Convenção | Regra |
|-----------|-------|
| Nome arquivo Staging | `stg_<nomedatabela>.sql` |
| Nome arquivo Intermediate | `int_<nomedomodelo>.sql` |
| Nome arquivo Marts Dimensão | `marts/<dominio>/dim_<nomedatabela>.sql` — domínio definido pelo usuário |
| Nome arquivo Marts Fato | `marts/<dominio>/fato_<nomedatabela>.sql` — domínio definido pelo usuário |
| Domínio Marts | Perguntar SEMPRE ao usuário (ex: `comercial`, `estoque`, `financeiro`, `logistica`) — NUNCA usar nome do banco/sistema |
| `nome_tabela` no Jinja | Sempre em MAIÚSCULAS, sem aspas externas: `'CLIENTE'` |
| `sistema_origem` no Jinja | Com aspas simples internas: `"'ERP_SYS'"` |
| `formato_arquivo` no Jinja | Com aspas simples internas: `"'iceberg'"` |
| Macro de metadados | `{{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}` |
| Coluna de auditoria Staging | `DATA_INGESTAO` com timezone `America/Sao_Paulo` |
| Coluna de auditoria Int/Marts | `processing_date` com timezone `America/Sao_Paulo` |
| Staging usa SELECT * | Padrão quando sem SQL base — se SQL fornecida, listar colunas explicitamente |
| SQL como base | Quando fornecida: extrair nome da tabela e colunas do SELECT; verificar se staging já existe antes de criar |
| Staging existente | Se `stg_<tabela>.sql` já existir: adicionar colunas extraídas (sem duplicar) em vez de recriar |
| Incremental padrão | `NÃO` — perguntar sempre, mas não presumir |
| Filtro incremental | `CAST(<coluna_data> AS DATE) >= max(CAST(<coluna_data> AS DATE))` |
| Surrogate key (Marts) | `md5(CAST(<col_origem> AS VARCHAR))` — nunca `hex()` ou sequência |
| Ordem colunas Dimensão | 1º `id_dim_*` (PK) → atributos descritivos → IDs snowflake → `processing_date` |
| Ordem colunas Fato | 1º `chave` (PK) → métricas/medidas → `id_dim_*` (FKs) → `processing_date` |

---

## Arquitetura de Camadas (Medallion dbt)

> **REGRA OBRIGATORIA:** Consultar `medallion-architect` antes de criar intermediate ou marts.

```
sources/      → definições YAML das fontes externas

staging/      → Bronze/Silver: SELECT * com metadados, 1:1 com fonte
                SEM regras de negócio, SEM COALESCE, SEM CASE WHEN
                stg_<nomedatabela>.sql

intermediate/ → Silver agnóstica: JOINs, deduplicação, seleção de colunas
                SEM calculos, SEM CASE WHEN, SEM COALESCE de negócio
                Passa NULLs raw — conformação estrutural apenas
                int_<nomedomodelo>.sql

marts/        → Gold: toda lógica de negócio via CTEs organizados
                CASE WHEN, COALESCE, cálculos, surrogate keys md5
                Estrutura obrigatória de CTEs:
                  base → medidas → regras_negocio → chaves_dimensionais
                marts/<dominio>/fato_<tabela>.sql
                marts/<dominio>/dim_<tabela>.sql
```

---

## Comandos dbt Úteis

```bash
# Execução
dbt run                                  # todos os modelos
dbt run --select staging                 # apenas camada staging
dbt run --select stg_cliente             # modelo específico
dbt run --full-refresh                   # rebuild de todos os incrementais

# Execução com dependências
dbt run --select +stg_cliente            # modelo + upstream
dbt run --select stg_cliente+            # modelo + downstream

# Testes
dbt test                                 # todos os testes
dbt test --select stg_cliente            # testa modelo específico
dbt build                                # run + test na ordem do DAG

# Documentação
dbt docs generate
dbt docs serve

# Debug
dbt compile                              # compila sem executar
dbt debug                                # verifica conexão e config
dbt ls --select tag:staging              # lista modelos por tag
```

---

## Exemplos Completos

### Source Create: `ERP_SYS`, database=`TRUCKS`, schema=`ERP_SYS.dbo`

**Comando:** `/dbt source create ERP_SYS TRUCKS ERP_SYS.dbo`
**Arquivo:** `models/staging/source.yml`

```yaml
version: "2"

sources:
  - name: ERP_SYS
    database: TRUCKS
    schema: ERP_SYS.dbo

    tables:
```

---

### Source Add Table: adicionar `CLIENTE` ao source.yml existente

**Comando:** `/dbt source add table CLIENTE`

Resultado após edição do `source.yml`:

```yaml
version: "2"

sources:
  - name: ERP_SYS
    database: TRUCKS
    schema: ERP_SYS.dbo

    tables:
      - name: CLIENTE
```

---

### Source from-file: schema.yml com database `CONSINCO`

**Comando:** `/dbt source from-file /opt/TAF-etl-n3-dw/src/tables/schema.yml`

Dado o arquivo de entrada:

```yaml
database: 'CONSINCO'
schema:
  - name: 'CONSINCO'
    tables:
      - name: 'MLF_NOTAFISCAL'
      - name: 'MAD_PEDVENDA'
      - name: 'GE_PESSOA'
```

**Arquivo gerado:** `models/staging/source.yml`

```yaml
version: "2"

sources:
  - name: CONSINCO
    database: CONSINCO
    schema: CONSINCO.dbo

    tables:
      - name: GE_PESSOA
      - name: MAD_PEDVENDA
      - name: MLF_NOTAFISCAL
```

---

### Staging Full Refresh: `CLIENTE`, `ERP_SYS`

**Arquivo:** `models/staging/stg_cliente.sql`

```sql
{% set nome_tabela = 'CLIENTE' %}
{% set sistema_origem = "'ERP_SYS'" %}
{% set formato_arquivo = "'iceberg'" %}

SELECT
    *,
    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') as DATA_INGESTAO,
    {{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}
FROM
{{ source('ERP_SYS', nome_tabela) }}
```

---

### Staging Incremental: `RECEBPAGTS`, `ERP_SYS`, unique_key=`RPVID`, data=`RPVDATAPAGAMENTO`

**Arquivo:** `models/staging/stg_recebpagts.sql`

```sql
{{
    config(
        materialized='incremental',
        unique_key='RPVID',
        on_schema_change='append_new_columns'
    )
}}

{% set nome_tabela = 'RECEBPAGTS' %}
{% set sistema_origem = "'ERP_SYS'" %}
{% set formato_arquivo = "'iceberg'" %}

SELECT
    *,
    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') as DATA_INGESTAO,
    {{ add_metadata(nome_tabela, sistema_origem, formato_arquivo) }}
FROM
{{ source('ERP_SYS', nome_tabela) }}
{% if is_incremental() %}
WHERE CAST(RPVDATAPAGAMENTO AS DATE) >= (select max(CAST(RPVDATAPAGAMENTO AS DATE)) from {{ this }})
{% endif %}
```

---

### Intermediate: `int_atestados_vendedores.sql`

**Arquivo:** `models/intermediate/int_atestados_vendedores.sql`

```sql
with atestados as (
    SELECT
        CAST(svaCodigoVendedor AS INT) as cod_vendedor,
        UPPER(svaTipoAfastamento) as tipo_afastamento,
        CAST(svaDataInicio AS DATE) AS data_inicio,
        CAST(svaDataFim AS DATE) AS data_fim,
        CASE
            WHEN DATEDIFF(svaDataFim, svaDataInicio) = 0 THEN 1
            ELSE DATEDIFF(svaDataFim, svaDataInicio)
        END qtd_dias_afastamento,
        CAST(svaCodigoVendedor AS VARCHAR(100)) AS chave_consultor_venda,
        CAST(COALESCE(TO_CHAR(svaDataInicio, 'yyyyMMdd'), '0') AS VARCHAR(10)) AS chave_data_inicio,
        CAST(COALESCE(TO_CHAR(svaDataFim, 'yyyyMMdd'), '0') AS VARCHAR(10)) AS chave_data_fim,
        from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') AS processing_date
    FROM {{ ref('stg_tabseniorvendedorafastamento') }}
),
gera_sequencia_datas AS (
    SELECT
        a.*,
        d.data AS data_ocorrencia
    FROM atestados a
    INNER JOIN {{ ref('calendarios') }} d
        ON d.data BETWEEN a.data_inicio AND a.data_fim
)
SELECT
    CONCAT(chave_consultor_venda, '-', TO_CHAR(data_ocorrencia, 'yyyyMMdd')) AS chave,
    cod_vendedor,
    tipo_afastamento,
    data_inicio,
    data_fim,
    data_ocorrencia,
    qtd_dias_afastamento,
    chave_consultor_venda,
    chave_data_inicio,
    chave_data_fim,
    CAST(COALESCE(TO_CHAR(data_ocorrencia, 'yyyyMMdd'), '0') AS VARCHAR(10)) AS chave_data_ocorrencia,
    processing_date
FROM gera_sequencia_datas
```

---

### Marts Dimensão: `dim_gerentes.sql`

**Arquivo:** `models/marts/comercial/dim_gerentes.sql`

```sql
SELECT
    -- 1ª coluna: ID surrogate (chave primária)
    md5(CAST(id_dim_gerente AS VARCHAR)) AS id_dim_gerente,

    -- atributos descritivos
    situacao,
    codigo,
    nome,
    cpf_cnpj,
    cep,
    endereco,
    cidade,
    estado,
    telefone,
    celular,
    tipo_consultor,

    -- IDs snowflake (FKs para outras dimensões) viriam aqui, antes de processing_date

    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') AS processing_date
FROM {{ ref('gerentes') }}
```

### Marts Dimensão com Snowflake: `dim_produto.sql`

**Arquivo:** `models/marts/comercial/dim_produto.sql`

```sql
SELECT
    -- 1ª coluna: ID surrogate (chave primária)
    md5(CAST(id_dim_produto AS VARCHAR)) AS id_dim_produto,

    -- atributos descritivos
    data_alteracao,
    codigo,
    descricao,
    situacao,
    unid_medida,
    caracteristica,
    cod_tipo_produto,
    tipo_produto,
    cod_tipo_cadastro,
    tipo_cadastro,
    tipo_antena,
    qtd,
    peso,
    custo_medio,

    -- IDs snowflake (FKs para outras dimensões) — SEMPRE no final
    md5(CAST(id_dim_familia AS VARCHAR)) AS id_dim_familia,
    md5(CAST(id_dim_grupo AS VARCHAR)) AS id_dim_grupo,

    from_utc_timestamp(CURRENT_TIMESTAMP, 'America/Sao_Paulo') AS processing_date
FROM {{ ref('int_produto') }}
```

---

## Boas Práticas (dbt + Mentorstec)

**Fazer:**
- Usar incremental apenas em tabelas com alto volume (> 500k linhas)
- Testar `unique` e `not_null` nas chaves de todas as camadas
- Documentar as colunas no arquivo `_models.yml` correspondente
- Usar `CAST(<coluna> AS DATE)` no filtro incremental para garantir consistência de tipo
- Em staging: usar `SELECT *` com `add_metadata` — não mapear colunas individualmente
- Em intermediate: aplicar toda lógica de negócio, casts, joins e cálculos
- Em marts: apenas SELECT simples — lógica já deve estar no intermediate

**Evitar:**
- Pular camadas: raw → mart gera dívida técnica
- Hardcodar datas — use `{{ var('start_date') }}`
- Colocar lógica de negócio em marts — pertence ao intermediate
- Presumir que a tabela é incremental sem confirmar com o usuário
- Usar duplo underscore em nomes de arquivo (`stg_erp__cliente`) — padrão atual é `stg_cliente`

---

## Estratégias Incrementais Disponíveis

```sql
-- Delete+Insert (padrão Mentorstec)
{{ config(materialized='incremental', unique_key='id') }}

-- Merge (melhor para late-arriving data)
{{ config(
    materialized='incremental',
    unique_key='id',
    incremental_strategy='merge',
    merge_update_columns=['status', 'valor', 'updated_at']
) }}

-- Insert Overwrite (baseado em partição — Iceberg/Delta)
{{ config(
    materialized='incremental',
    incremental_strategy='insert_overwrite',
    partition_by={"field": "DATA_INGESTAO", "data_type": "date", "granularity": "day"}
) }}
```
