# Analisar Projeto Pentaho

> Gera um overview de um projeto ou pasta Pentaho, mapeando as transformacoes existentes e seus principais steps (TableInput, SelectValues, DimensionLookup).

## Uso

```bash
/analyze-project <caminho>
```

## Exemplos

```bash
# Analisar pasta completa do projeto
/analyze-project /etl/pentaho/

# Analisar uma subpasta especifica
/analyze-project /etl/pentaho/dimensoes/

# Analisar um unico arquivo .ktr
/analyze-project /etl/pentaho/carga_clientes.ktr
```

---

## O Que Este Comando Faz

1. **Descobrir** — localizar todos os arquivos `.ktr` no caminho informado
2. **Inspecionar** — para cada `.ktr`, identificar e contar steps por tipo
3. **Classificar** — mapear quais transformacoes tem leitura (TableInput), renomeacao (SelectValues) e dimensao (DimensionLookup)
4. **Consolidar** — montar o overview do projeto com totais e distribuicao por arquivo
5. **Relatar** — exibir o resumo estruturado com navegacao facilitada

---

## Processo

### Passo 1: Receber o Caminho

Se nao fornecido como argumento, perguntar:

```
Informe o caminho do projeto ou pasta Pentaho a analisar:
(ex: /etl/pentaho/, /opt/project/transformacoes/, arquivo.ktr)
```

### Passo 2: Invocar o Agente

Acionar o agente `pentaho-especialist` com a instrucao:

```
Analise o caminho: {caminho}

Para cada arquivo .ktr encontrado, identifique e conte os seguintes tipos de step:
- TableInput      → leitura de dados via SQL
- SelectValues    → renomeacao, reordenacao e type casting de colunas
- DimensionLookup → lookup e atualizacao de dimensoes (SCD)

Gere um overview consolidado conforme o formato de saida descrito.
```

### Passo 3: Descoberta dos Arquivos

O agente ira:
1. Usar `Glob` para descobrir todos os `.ktr` no caminho (`**/*.ktr`)
2. Reportar o total de transformacoes encontradas antes de iniciar a analise
3. Processar cada `.ktr` em sequencia

### Passo 4: Inspecao por Transformacao

Para cada `.ktr`, o agente ira extrair:

| Campo | Origem no XML | Descricao |
|-------|---------------|-----------|
| Nome do arquivo | nome do `.ktr` | Identificador da transformacao |
| Steps `TableInput` | `<type>TableInput</type>` | Leituras SQL (nome + conexao) |
| Steps `SelectValues` | `<type>SelectValues</type>` | Mapeamentos de colunas (nome) |
| Steps `DimensionLookup` | `<type>DimensionLookup</type>` | Dimensoes gerenciadas (nome + tabela alvo) |

### Passo 5: Montar o Relatorio de Overview

Consolidar e exibir o resultado no formato da secao **Saida** abaixo.

---

## Saida

### Visao Consolidada do Projeto

```markdown
## Overview do Projeto Pentaho
Caminho: {caminho}
Transformacoes encontradas: {N} arquivo(s) .ktr

### Distribuicao de Steps
| Tipo de Step     | Total no Projeto |
|------------------|-----------------|
| TableInput       | {N}             |
| SelectValues     | {N}             |
| DimensionLookup  | {N}             |
```

### Detalhamento por Transformacao

```markdown
### Transformacoes

| Arquivo .ktr | TableInput | SelectValues | DimensionLookup |
|--------------|:----------:|:------------:|:---------------:|
| carga_cliente.ktr | 2 | 1 | 0 |
| carga_pedido.ktr  | 3 | 2 | 1 |
| ...               | ... | ... | ... |
| **TOTAL**     | **N** | **N** | **N** |
```

### Detalhes por Step (quando relevante)

```markdown
#### {arquivo.ktr}

**TableInput**
| Step Name | Conexao |
|-----------|---------|
| Busca Clientes | DB_ERP |

**SelectValues**
| Step Name |
|-----------|
| Mapeia Campos Cliente |

**DimensionLookup**
| Step Name | Tabela Alvo |
|-----------|-------------|
| Dim Cliente | DW.DIM_CLIENTE |
```

---

## Opcoes de Profundidade

| Cenario | Comportamento |
|---------|---------------|
| Pasta com muitos `.ktr` (> 10) | Exibir visao consolidada primeiro, detalhes sob demanda |
| Pasta com poucos `.ktr` (<= 10) | Exibir consolidado + detalhes completos de uma vez |
| Arquivo `.ktr` unico | Exibir detalhes completos diretamente |

---

## Lidando com Problemas

| Problema | Acao |
|----------|------|
| Caminho nao existe | Informar erro e solicitar caminho correto |
| Nenhum `.ktr` encontrado | Informar e sugerir verificar o caminho ou extensao |
| `.ktr` com XML invalido ou corrompido | Registrar no relatorio e continuar com os demais |
| Step sem nome definido | Usar `(sem nome)` como placeholder no relatorio |
| Pasta muito grande (> 50 `.ktr`) | Avisar o usuario sobre o volume e confirmar antes de prosseguir |

---

## Restricoes

- NUNCA altere, mova ou modifique qualquer arquivo `.ktr` analisado (somente leitura)
- NUNCA pule arquivos sem relatar — todo `.ktr` deve aparecer no resumo, mesmo que vazio
- SEMPRE exibir os totais consolidados no inicio do relatorio

---

## Referencias

- Agente: `.claude/agents/pentaho/pentaho-especialist.md`
- KB: `.claude/kb/pentaho/concepts/step-types.md`
- KB: `.claude/kb/pentaho/concepts/ktr-structure.md`
- KB: `.claude/kb/pentaho/quick-reference.md`
- Comando relacionado: `.claude/commands/pentaho/extract-sql.md`
