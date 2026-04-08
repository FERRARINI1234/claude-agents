# Extract SQL

> Extrai os SQLs de steps TableInput em arquivos .ktr do Pentaho, gera arquivos `.yml` por step e em seguida produz um `schema.yml` com o mapeamento de tabelas e colunas referenciadas.

## Uso

```bash
/extract-sql <caminho-entrada> [caminho-destino]
```

## Exemplos

```bash
# Extrair SQLs de uma pasta inteira (perguntara o destino)
/extract-sql /etl/pentaho/

# Extrair SQLs informando origem e destino diretamente
/extract-sql /etl/pentaho/ /opt/TAF-etl-n3-dw/src/tables

# Extrair de um arquivo especifico com destino definido
/extract-sql /etl/pentaho/carga_clientes.ktr /opt/TAF-etl-n3-dw/src/inputs/fatos/07.01_fato_pedido

# Extrair de subpasta sem destino (perguntara onde salvar)
/extract-sql /etl/pentaho/fatos/
```

---

## Opcoes

| Forma de uso | Comportamento |
|---|---|
| `/extract-sql` | Pergunta entrada e destino |
| `/extract-sql <entrada>` | Usa entrada fornecida, pergunta destino |
| `/extract-sql <entrada> <destino>` | Usa ambos diretamente, sem perguntas |

---

## O Que Este Comando Faz

Este comando executa **duas fases em sequencia** apos resolver os argumentos:

| Fase | Agente | Responsabilidade | Saida |
|------|--------|-----------------|-------|
| **1** | `pentaho-table-input-especialist` | Localizar steps TableInput nos `.ktr` e extrair o SQL de cada um | Um `.yml` por step |
| **2** | `pentaho-table-mapping` | Ler os `.yml` gerados, analisar os SQLs e mapear tabelas e colunas | `schema.yml` consolidado |
| **3** | *(sugestao)* | Indicar o proximo passo para criacao da camada staging no dbt | `/dbt:create-source` |

---

## Processo

### Passo 1: Resolver o Caminho de Entrada

Se `<caminho-entrada>` foi passado como argumento, usar diretamente.
Caso contrario, SEMPRE perguntar ao usuario antes de prosseguir:

```
Informe o caminho do projeto Pentaho ou da pasta que sera analisada:
(ex: /etl/pentaho/, /opt/projeto/transformacoes/, arquivo.ktr)
```

### Passo 2: Resolver a Pasta de Destino

Se `[caminho-destino]` foi passado como argumento, usar diretamente — confirmar com o usuario exibindo o caminho resolvido.
Caso contrario, SEMPRE perguntar ao usuario antes de prosseguir:

```
Informe a pasta de destino onde os arquivos .yml serao salvos:
(ex: /opt/TAF-etl-n3-dw/src/tables, src/inputs/fatos/07.01_fato_pedido/)
```

> **Exemplo com ambos os argumentos:**
> `/extract-sql /etl/pentaho/ /opt/TAF-etl-n3-dw/src/tables`
> → entrada: `/etl/pentaho/` | destino: `/opt/TAF-etl-n3-dw/src/tables` — nenhuma pergunta necessaria.

### Passo 3: Carregar Fontes de Conhecimento

Antes de acionar os agentes, consultar as seguintes fontes de conhecimento:

- `.claude/CLAUDE.md` — convencoes e padroes do projeto
- `.claude/kb/pentaho/` — KB do dominio Pentaho (especialmente `concepts/table-input-step.md` e `patterns/sql-extraction.md`)

### Passo 4: Fase 1 — Invocar `pentaho-table-input-especialist`

Acionar o agente `pentaho-table-input-especialist` com a instrucao:

```
Extraia todos os steps TableInput do caminho: {caminho_informado}

Salve os arquivos .yml gerados em: {pasta_destino_informada}

Para cada step TableInput encontrado:
- Extraia: name, connection, sql, execute_each_row, variables_active
- Decodifique entidades XML no SQL (&lt; &gt; &amp; &apos; &#xd; &#xa;)
- Formate a indentacao do SQL para legibilidade no YAML (sem alterar logica)
- Gere um arquivo {name}.yml separado na pasta de destino

Consulte .claude/kb/pentaho/ para padroes do dominio.
NUNCA altere o SQL original — apenas formate a indentacao para apresentacao no YAML.
NUNCA modifique arquivos .ktr.
```

### Passo 5: Validar Saida da Fase 1

Verificar os arquivos `.yml` gerados antes de prosseguir para a Fase 2:

```text
[ ] Um arquivo .yml gerado por step TableInput
[ ] SQL preservado sem alteracao de logica
[ ] Entidades XML decodificadas corretamente
[ ] Nomes de arquivos sanitizados (sem caracteres invalidos)
[ ] Nenhum arquivo .ktr foi modificado
```

Se a Fase 1 nao gerou nenhum arquivo `.yml`, interromper e reportar ao usuario — nao acionar a Fase 2.

### Passo 6: Fase 2 — Invocar `pentaho-table-mapping`

Com os arquivos `.yml` gerados na Fase 1, acionar o agente `pentaho-table-mapping` com a instrucao:

```
Leia todos os arquivos .yml da pasta: {pasta_destino_informada}

Para cada arquivo .yml:
- Leia o campo `sql:` e extraia as tabelas e colunas referenciadas
- Identifique database e schema (confirme com o usuario se nao for identificavel no SQL)
- Registre as colunas com o nome original (antes do AS), sem aliases
- Nao registre CTEs como tabelas reais

Consolide tudo em um unico schema.yml:
- Pergunte ao usuario onde salvar o schema.yml
- Se ja existir um schema.yml no destino, faca merge (append sem duplicar)

Consulte .claude/kb/pentaho/ para padroes do dominio.
```

### Passo 7: Validar Saida da Fase 2

```text
[ ] schema.yml gerado ou atualizado com merge
[ ] Database e schema confirmados com o usuario (nao presumidos)
[ ] Aliases de coluna ignorados — apenas nomes originais registrados
[ ] CTEs nao registradas como tabelas
[ ] Nenhuma tabela ou coluna duplicada no schema.yml
[ ] Resumo com totais exibido (database, schemas, tabelas, colunas)
```

### Passo 8: Sugerir Proximo Passo

Apos a conclusao das duas fases, exibir a seguinte sugestao ao usuario:

```markdown
---

## Proximo Passo Sugerido

O `schema.yml` gerado pode ser usado como base para criar o `source.yml` do dbt.

Execute:

```bash
/dbt:create-source {caminho_do_schema_yml}
```

Este comando usara o `schema.yml` como arquivo base para gerar o `source.yml`
da camada staging no dbt, mapeando automaticamente as tabelas e colunas identificadas.

> Referencia: `.claude/commands/dbt/create-source.md`
```

---

## Loop de Execucao

```text
┌─────────────────────────────────────────────────────────────┐
│                  LOOP DO EXTRACT-SQL                        │
├─────────────────────────────────────────────────────────────┤
│  [FASE 1] pentaho-table-input-especialist                   │
│  1. Descobrir arquivos .ktr no caminho informado            │
│  2. Para cada .ktr → localizar steps TableInput             │
│  3. Para cada step → extrair e gerar .yml                   │
│     └─ Se FALHAR → registrar erro e continuar               │
│  4. Reportar resumo da Fase 1                               │
│                                                             │
│  [VALIDAR] Ha arquivos .yml gerados?                        │
│  └─ NAO → Interromper e reportar ao usuario                 │
│                                                             │
│  [FASE 2] pentaho-table-mapping                             │
│  5. Ler os .yml gerados e extrair tabelas e colunas dos SQL │
│  6. Confirmar database e schema com o usuario               │
│  7. Gerar ou atualizar schema.yml (com merge se existir)    │
│  8. Reportar resumo da Fase 2                               │
│                                                             │
│  [SUGESTAO] Indicar /dbt:create-source como proximo passo  │
└─────────────────────────────────────────────────────────────┘
```

---

## Saida

| Artefato | Fase | Localizacao |
|----------|------|-------------|
| **{step-name}.yml** | Fase 1 | `{pasta_destino_informada}/{step-name}.yml` |
| **schema.yml** | Fase 2 | `{pasta_informada_pelo_usuario}/schema.yml` |

**Formato de cada arquivo `.yml` gerado na Fase 1:**

```yaml
name: {name}
conexao: {connection}
execute_each_row: {N ou Y}
variables_active: {N ou Y}
sql: |
  {sql formatado com indentacao de 2 espacos}
```

**Formato do `schema.yml` gerado na Fase 2:**

```yaml
database: '<DATABASE>'
schema:
  - name: '<SCHEMA>'
    tables:
      - name: '<TABELA>'
        columns:
          - name: '<COLUNA_ORIGINAL>'
```

**Formato do relatorio final:**

```markdown
## Extracao e Mapeamento concluidos

### Fase 1 — Extracao de SQL
**Caminho analisado:** {caminho}
**Pasta de destino:** {pasta_destino}
**Arquivos .ktr processados:** {N}
**Steps TableInput encontrados:** {N}
**Arquivos .yml gerados:** {N}

| # | Arquivo .ktr | Step Name | Conexao | Arquivo .yml |
|---|--------------|-----------|---------|--------------|
| 1 | ...          | ...       | ...     | ...          |

### Fase 2 — Mapeamento de Dados
**Database:** {database}
**Schemas:** {schemas}
**Tabelas mapeadas:** {N}
**Colunas totais:** {N}
**schema.yml salvo em:** {caminho}

### Proximo Passo
/dbt:create-source {caminho_do_schema_yml}
```

---

## Controle de Qualidade

Antes de marcar como concluido, verificar:

```text
FASE 1 — EXTRACAO
[ ] Caminho de entrada validado e informado pelo usuario
[ ] Pasta de destino validada e informada pelo usuario
[ ] KB consultada antes de acionar o agente
[ ] Somente steps <type>TableInput</type> processados
[ ] SQL extraido sem alteracao de logica ou clausulas
[ ] Entidades XML decodificadas (&lt; &gt; &amp; &apos; &#xd; &#xa;)
[ ] Um arquivo .yml separado por step TableInput
[ ] Nenhum arquivo .ktr modificado
[ ] Resumo da Fase 1 exibido ao usuario

FASE 2 — MAPEAMENTO
[ ] Fase 1 concluiu com pelo menos um .yml gerado
[ ] Todos os .yml da pasta lidos e SQLs analisados
[ ] Database confirmado com o usuario (nao presumido)
[ ] Schema confirmado com o usuario (nao presumido)
[ ] Aliases de coluna ignorados
[ ] CTEs nao registradas como tabelas
[ ] schema.yml com merge correto (sem duplicatas)
[ ] Resumo da Fase 2 exibido ao usuario

SUGESTAO
[ ] /dbt:create-source sugerido com o caminho do schema.yml
```

---

## Lidando com Problemas

| Problema | Acao |
|----------|------|
| Caminho nao informado | Perguntar ao usuario antes de prosseguir |
| Pasta de destino nao informada | Perguntar ao usuario antes de prosseguir |
| Caminho nao existe | Informar erro e solicitar caminho correto |
| Nenhum `.ktr` encontrado | Informar e sugerir verificar o caminho |
| Nenhum step `TableInput` encontrado | Informar ao usuario e nao acionar Fase 2 |
| `.ktr` com XML invalido | Registrar no relatorio, pular e continuar |
| Pasta de destino nao existe | Perguntar ao usuario se deve criar |
| Permissao negada na escrita | Informar ao usuario e solicitar orientacao |
| Database/schema nao identificavel no SQL | Pausar Fase 2 e confirmar com o usuario |
| schema.yml ja existe | Fazer merge — nunca sobrescrever sem confirmacao |

---

## Restricoes

- NUNCA altere o SQL original — apenas indentacao para apresentacao no YAML e permitida
- NUNCA modifique arquivos `.ktr` ou qualquer codigo-fonte
- NUNCA gere os `.yml` sem ter a pasta de destino definida (argumento ou resposta do usuario)
- NUNCA concatene multiplos steps em um unico arquivo `.yml`
- NUNCA presuma database ou schema na Fase 2 — sempre confirmar com o usuario
- NUNCA acione a Fase 2 se a Fase 1 nao gerou nenhum `.yml`
- SEMPRE pergunte o caminho de entrada se nao fornecido como argumento
- SEMPRE pergunte a pasta de destino se nao fornecida como argumento

---

## Dicas

1. **Arquivo unico vs pasta** — o agente da Fase 1 suporta ambos; informe um `.ktr` especifico ou uma pasta inteira
2. **Entidades XML** — o agente decodifica automaticamente `&lt;`, `&gt;`, `&amp;`, `&apos;`, `&#xd;` e `&#xa;`
3. **Nomes de arquivos** — caracteres invalidos no nome do step sao substituidos por `_` no nome do `.yml`
4. **schema.yml como ponte** — o `schema.yml` gerado na Fase 2 e o insumo direto para o `/dbt:create-source`
5. **Merge inteligente** — se o `schema.yml` ja existir, a Fase 2 faz append sem duplicar tabelas ou colunas

---

## Referencias

- Agente Fase 1: `.claude/agents/pentaho/pentaho-table-input-especialist.md`
- Agente Fase 2: `.claude/agents/pentaho/pentaho-table-mapping.md`
- KB: `.claude/kb/pentaho/concepts/table-input-step.md`
- KB: `.claude/kb/pentaho/patterns/sql-extraction.md`
- KB: `.claude/kb/pentaho/specs/yml-output-format.yaml`
- Comando relacionado: `.claude/commands/pentaho/analyze-project.md`
- Comando relacionado: `.claude/commands/pentaho/convert-to-dbt.md`
- Proxima Fase: `.claude/commands/dbt/create-source.md`
