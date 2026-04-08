# Criar source.yml dbt

> Cria um arquivo `source.yml` para um projeto dbt, coletando parametros do adapter configurado e usando arquivos existentes como base quando disponivel.

## Uso

```bash
/dbt:create-source
/dbt:create-source [caminho/para/arquivo-base]
```

## Exemplos

```bash
# Criacao interativa (recomendado)
/dbt:create-source

# A partir de um arquivo base existente
/dbt:create-source models/staging/schema.yml

# A partir de um schema de referencia exportado do banco
/dbt:create-source /opt/projeto/exports/schema_erp.yml
```

---

## Visao Geral

Este comando faz parte do fluxo de criacao da camada staging no dbt:

```text
Etapa 1: /dbt:create-source  → source.yml         (ESTE COMANDO)
Etapa 2: /dbt:create-staging → stg_<tabela>.sql
```

---

## O Que Este Comando Faz

1. **Investigar** — Busca `schema.yml` e `source.yml` existentes no projeto
2. **Questionar** — Coleta `name`, `database` e `schema` do adapter dbt configurado
3. **Verificar** — Pergunta se ha arquivo base para importar tabelas
4. **Delegar** — Aciona o agente `dbt-especialist` para gerar o artefato via Skill dbt

---

## Opcoes

| Comando | Acao |
|---------|------|
| `/dbt:create-source` | Criacao interativa completa |
| `/dbt:create-source <arquivo>` | Usa arquivo informado como base |

---

## Processo

### Passo 1: Investigar Projeto dbt

Antes de qualquer interacao com o usuario, executar:

```text
1. Localizar raiz do projeto dbt
   Glob: src/dbt/**/dbt_project.yml

2. Identificar adapter configurado
   Read: src/dbt/**/profiles.yml  (ou dbt_project.yml)
   → Identificar: dremio | trino | redshift | snowflake | bigquery | spark

3. Verificar source.yml existente
   Glob: src/dbt/**/models/**/source.yml
   Glob: src/dbt/**/models/**/_sources.yml

4. Buscar schema.yml no projeto
   Glob: src/dbt/**/models/**/schema.yml
   Glob: src/dbt/**/schema.yml
```

### Passo 2: Comunicar Descobertas e Questionar Arquivo Base

Se `schema.yml` for encontrado:

```markdown
Encontrei um arquivo schema.yml em: <caminho>

Posso usar esse arquivo como base para extrair as tabelas do source.yml?
  (a) Sim — usar schema.yml como base
  (b) Nao — tenho outro arquivo para indicar
  (c) Nao — vou informar os dados manualmente
```

Se `schema.yml` NAO for encontrado:

```markdown
Voce possui algum arquivo para usar como base na criacao do source.yml?
  (ex: schema.yml, exports de banco, arquivo de documentacao de tabelas)

  (a) Sim — informarei o caminho do arquivo
  (b) Nao — vou informar os dados manualmente
```

### Passo 3: Coletar Parametros OBRIGATORIOS do Adapter

**SEMPRE** perguntar os tres parametros abaixo, mesmo que haja arquivo base.
Os valores sao especificos do adapter dbt configurado no projeto.

```markdown
Informe os parametros do source dbt:

  name:
    Identificador do source no dbt — aparece em {{ source('NAME', 'TABELA') }}
    Exemplos: CONSINCO, ERP_SYS, CRM, SAP, OMIE

  database:
    Catalogo ou banco de dados no warehouse
    Depende do adapter:
      • Dremio    → espaco de catalogo (ex: CONSINCO, ANALYTICS)
      • Trino     → catalogo (ex: hive, iceberg, postgresql)
      • Snowflake → database (ex: PROD_DB, RAW_DATA)
      • Redshift  → database (ex: analytics, raw)
      • BigQuery  → project (ex: meu-projeto-gcp)

  schema:
    Schema ou namespace onde as tabelas estao
    Exemplos: CONSINCO.dbo, public, raw_erp, dbo
```

### Passo 4: Resolver Pasta de Destino pelo Database

A pasta de destino do `source.yml` e sempre derivada do valor de `database` informado pelo usuario:

```text
src/dbt/lakehouse/models/staging/<database>/source.yml
```

**Exemplos:**
- `database: consinco`  → `models/staging/consinco/source.yml`
- `database: TRUCKS`    → `models/staging/TRUCKS/source.yml`
- `database: ERP_SYS`   → `models/staging/ERP_SYS/source.yml`

**Logica:**
- Se a pasta `staging/<database>/` ja existir → usar a pasta existente
- Se a pasta `staging/<database>/` NAO existir → criar a pasta e salvar o `source.yml` dentro

Verificar se ja existe `source.yml` na pasta resolvida:

```markdown
Ja existe um source.yml em: staging/<database>/source.yml

O que deseja fazer?
  (a) Mesclar — adicionar as novas tabelas ao source.yml existente
  (b) Substituir — criar um novo source.yml (o existente sera sobrescrito)
  (c) Cancelar — manter o arquivo atual sem alteracoes
```

### Passo 5: Acionar dbt-especialist

Com todos os parametros coletados, acionar o agente `dbt-especialist` passando:

- Acao: criar source.yml
- `name` coletado do usuario
- `database` coletado do usuario
- `schema` coletado do usuario
- Arquivo base (se houver): caminho informado ou `schema.yml` encontrado
- Decisao sobre source.yml existente (mesclar / substituir / novo)

O agente `dbt-especialist` invocara a Skill dbt com o comando adequado:

```text
# Com arquivo base:
/dbt source from-file <caminho_do_arquivo>

# Sem arquivo base (manual):
/dbt source create <name> <database> <schema>
```

---

## Loop de Execucao

```text
┌─────────────────────────────────────────────────────────┐
│               LOOP DO CREATE-SOURCE                     │
├─────────────────────────────────────────────────────────┤
│  1. Investigar projeto (Glob + Read)                    │
│  2. Apresentar descobertas ao usuario                   │
│  3. Coletar: name, database, schema (OBRIGATORIO)       │
│  4. Coletar: arquivo base (opcional)                    │
│  5. Acionar dbt-especialist                             │
│     └─ Se FALHAR → reportar erro e sugerir alternativa  │
│  6. Exibir caminho do source.yml gerado                 │
│  7. Sugerir proximo passo: /dbt:create-staging          │
└─────────────────────────────────────────────────────────┘
```

---

## Saida

| Artefato | Localizacao |
|----------|-------------|
| **source.yml** | `src/dbt/lakehouse/models/staging/<database>/source.yml` |

**Formato exato do `source.yml` gerado:**

```yaml
version: "2"

sources:
  - name: ERP_SYS
    database: TRUCKS
    schema: ERP_SYS.dbo

    tables:
      - name: CLIENTE
      - name: DIREITOS
      - name: DOCDEVOLUCAO
      - name: DOCUMENTOFISCAL
      - name: EMPRESA
```

> **Importante:** o `source.yml` lista **apenas tabelas** — colunas nunca devem ser incluidas.

**Proximo Passo:** `/dbt:create-staging <tabela>` (quando pronto)

---

## Controle de Qualidade

Antes de marcar como concluido, verificar:

```text
[ ] name, database e schema foram informados pelo usuario (nao presumidos)
[ ] Adapter dbt identificado e parametros conferem com o tipo de adapter
[ ] schema.yml pesquisado e usuario consultado sobre uso
[ ] source.yml existente verificado antes de criar novo
[ ] Skill dbt invocada (nao gerou YAML manualmente)
[ ] source.yml contem APENAS tabelas — sem colunas
[ ] Pasta de destino e staging/<database>/ (derivada do database informado)
[ ] Pasta criada automaticamente se nao existia
[ ] Apenas 1 source.yml na pasta de destino
[ ] Caminho do arquivo gerado informado ao usuario
[ ] Proximo passo indicado (/dbt:create-staging)
```

---

## Lidando com Problemas

| Problema | Acao |
|----------|------|
| `profiles.yml` nao encontrado | Perguntar ao usuario qual adapter esta sendo usado |
| `schema.yml` com formato desconhecido | Ler e exibir preview; perguntar se pode ser usado |
| source.yml existente com conflito de name | Alertar o usuario e perguntar como proceder |
| Adapter nao reconhecido | Pedir ao usuario que informe manualmente o tipo de warehouse |
| Bloqueio na Skill dbt | Parar e informar ao usuario |

---

## Premissas

- **Pasta = database** — o `source.yml` e sempre salvo dentro de `models/staging/<database>/`; o nome da pasta e exatamente o valor do `database` informado pelo usuario
- **Pasta existente** — se `staging/<database>/` ja existir, usar sem perguntar; se nao existir, criar automaticamente
- **1 source.yml por pasta** — cada pasta de database tem exatamente um `source.yml`; nunca criar multiplos arquivos de source na mesma pasta
- **Apenas tabelas, sem colunas** — o `source.yml` lista somente as tabelas; colunas NAO devem ser incluidas

---

## Restricoes

- NUNCA presumir `name`, `database` ou `schema` sem confirmar com o usuario
- NUNCA sobrescrever `source.yml` existente sem confirmacao explicita
- NUNCA gerar YAML de source manualmente — sempre usar a Skill dbt via `dbt-especialist`
- NUNCA incluir colunas no source.yml — apenas o nome das tabelas
- NUNCA criar mais de um source.yml na mesma pasta de destino
- NUNCA usar pasta fixa para o source.yml — a pasta SEMPRE deriva do valor de `database`
- SEMPRE criar a pasta `staging/<database>/` se nao existir, sem perguntar ao usuario
- SEMPRE verificar se `schema.yml` existe antes de perguntar sobre arquivo base
- SEMPRE identificar o adapter dbt para contextualizar os exemplos de `database` e `schema`

---

## Dicas

1. **Adapter primeiro** — identificar o adapter antes de pedir `database` e `schema` permite dar exemplos especificos e evitar confusao
2. **schema.yml como atalho** — quando encontrado, acelera muito a criacao pois ja lista as tabelas
3. **name = alias do sistema** — usar o nome do sistema de origem (ex: CONSINCO, SAP) facilita a leitura do codigo dbt

---

## Referencias

- Agente: `.claude/agents/dbt/dbt-especialist.md`
- Skill: `.claude/skills/dbt/SKILL.md`
- Proximo comando: `.claude/commands/dbt/create-staging.md`
