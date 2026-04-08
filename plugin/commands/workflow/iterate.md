# Comando Iterate

> Atualizar qualquer documento de fase quando requisitos ou design mudam (Inter-Fases)

## Uso

```bash
/iterate <arquivo> "<descricao-da-mudanca>"
```

## Exemplos

```bash
/iterate BRAINSTORM_CLOUD_RUN.md "Considerar processamento em lote ao inves de tempo real"
/iterate DEFINE_CLOUD_RUN.md "Adicionar suporte para PDFs, nao apenas TIFF"
/iterate DESIGN_CLOUD_RUN.md "Funcoes precisam ser auto-contidas, sem common/ compartilhado"
/iterate .claude/sdd/features/DEFINE_AUTH.md "Mudar de JWT para autenticacao baseada em sessao"
```

---

## Visao Geral

O comando `/iterate` trabalha com **fases de documentos** do fluxo de trabalho AgentSpec:

```text
Fase 0: /brainstorm → BRAINSTORM_{FEATURE}.md ← /iterate pode atualizar
Fase 1: /define    → DEFINE_{FEATURE}.md     ← /iterate pode atualizar
Fase 2: /design    → DESIGN_{FEATURE}.md     ← /iterate pode atualizar
Fase 3: /build     → (codigo)                ← Atualizar DESIGN, depois /build
Fase 4: /ship      → (arquivo)               ← N/A
```

Use `/iterate` quando descobrir algo que precisa mudar no meio do caminho.

**Importante:** Para mudar codigo durante a Fase 3, atualize o documento DESIGN primeiro. A cascata para o codigo dispara um rebuild via `/build`. Isso garante rastreabilidade.

---

## O Que Este Comando Faz

1. **Detectar Fase** - Identificar qual documento de fase esta sendo atualizado
2. **Analisar Impacto** - Determinar efeitos downstream
3. **Atualizar Documento** - Aplicar mudancas com rastreamento de versao
4. **Cascata** - Propagar mudancas para documentos downstream se necessario

---

## Processo

### Passo 1: Carregar Documento Alvo

```markdown
Read(<arquivo-alvo>)

# Identificar tipo de documento:
# - BRAINSTORM_*.md → Fase 0
# - DEFINE_*.md → Fase 1
# - DESIGN_*.md → Fase 2
```

### Passo 2: Analisar Mudanca

Determinar o tipo de mudanca:

| Tipo de Mudanca | Exemplo | Impacto |
|-----------------|---------|---------|
| **Aditiva** | "Tambem suportar PDF" | Baixo - adiciona ao existente |
| **Modificadora** | "Mudar de X para Y" | Medio - atualiza existente |
| **Removedora** | "Remover funcionalidade Z" | Medio - simplifica |
| **Arquitetural** | "Usar padrao diferente" | Alto - pode requerer redesign |

### Passo 3: Aplicar Mudancas

Atualizar o documento com:

1. **Mudanca Aplicada** - A modificacao real
2. **Bump de Versao** - Incrementar versao no historico de revisao
3. **Nota de Mudanca** - O que mudou e por que

### Passo 4: Avaliar Necessidade de Cascata

| Origem | Cascateia Para |
|--------|----------------|
| Mudanca DEFINE | Pode precisar atualizacao DESIGN |
| Mudanca DESIGN | Pode precisar atualizacao de codigo |

Determinar se documentos downstream precisam de atualizacoes baseado nas regras de cascata.

### Passo 5: Executar Cascata (se necessario)

Se cascata necessaria, perguntar ao usuario:

```markdown
"Esta mudanca no DEFINE afeta o DESIGN. Opcoes:
(a) Atualizar DESIGN automaticamente para corresponder
(b) Apenas atualizar DEFINE, eu lido com o DESIGN manualmente
(c) Mostre-me o que mudaria primeiro"
```

### Passo 6: Salvar Atualizacoes

```markdown
Write(<arquivo-alvo>)
# Se cascata:
Write(<documento-downstream>)
```

---

## Saida

| Artefato | Localizacao |
|----------|-------------|
| **Documento Atualizado** | Mesmo local da entrada |
| **Atualizacoes de Cascata** | Documentos downstream (se aplicavel) |

---

## Regras de Cascata

### Mudancas DEFINE → Impacto DESIGN

| Mudanca DEFINE | Impacto DESIGN |
|----------------|----------------|
| Novo requisito | Pode precisar novo componente |
| Criterio de sucesso alterado | Pode precisar abordagem diferente |
| Expansao de escopo | Precisa novas secoes |
| Reducao de escopo | Pode simplificar |
| Nova restricao | Deve ser acomodada |

### Mudancas DESIGN → Impacto Codigo

| Mudanca DESIGN | Impacto Codigo |
|----------------|----------------|
| Novo arquivo no manifesto | Criar arquivo |
| Arquivo removido | Deletar arquivo |
| Padrao alterado | Atualizar arquivos afetados |
| Nova decisao | Pode precisar refatoracao |
| Mudanca de arquitetura | Atualizacoes significativas necessarias |

---

## Rastreamento de Versao

Cada documento mantem historico de revisao:

```markdown
## Historico de Revisao

| Versao | Data | Autor | Mudancas |
|--------|------|-------|----------|
| 1.0 | 2026-01-25 | define-agent | Versao inicial |
| 1.1 | 2026-01-25 | iterate-agent | Adicionado suporte PDF por solicitacao do usuario |
```

---

## Dicas

1. **Itere Cedo** - Capture mudancas antes de comecar a codificar
2. **Seja Especifico** - "Adicionar X" e melhor que "melhorar"
3. **Verifique Cascata** - Mudancas se propagam downstream
4. **Mantenha Historico** - Rastreamento de versao mostra evolucao
5. **Nao Lute Contra** - Requisitos mudam, isso e normal

---

## Quando Usar /iterate vs Comecar do Zero

| Situacao | Acao |
|----------|------|
| < 30% de mudanca | `/iterate` |
| Adicionar/modificar funcionalidades | `/iterate` |
| Mudar restricoes | `/iterate` |
| > 50% diferente | Novo `/define` |
| Problema completamente diferente | Novo `/define` |
| Usuarios-alvo diferentes | Novo `/define` |

---

## Referencias

- Agente: `.claude/agents/workflow/iterate-agent.md`
- Contratos: `.claude/sdd/architecture/WORKFLOW_CONTRACTS.yaml`
