# Comando Design

> Criar arquitetura e especificacao tecnica em uma unica passagem (Fase 2)

## Uso

```bash
/design <arquivo-define>
```

## Exemplos

```bash
/design .claude/sdd/features/DEFINE_FUNCOES_CLOUD_RUN.md
/design DEFINE_AUTH_USUARIO.md
/design .claude/sdd/features/DEFINE_EXTRACAO_NOTAS.md
```

---

## Visao Geral

Esta e a **Fase 2** do fluxo de trabalho AgentSpec de 5 fases:

```text
Fase 0: /brainstorm → .claude/sdd/features/BRAINSTORM_{FEATURE}.md (opcional)
Fase 1: /define    → .claude/sdd/features/DEFINE_{FEATURE}.md
Fase 2: /design    → .claude/sdd/features/DESIGN_{FEATURE}.md (ESTE COMANDO)
Fase 3: /build     → Codigo + .claude/sdd/reports/BUILD_REPORT_{FEATURE}.md
Fase 4: /ship      → .claude/sdd/archive/{FEATURE}/SHIPPED_{DATA}.md
```

O comando `/design` combina o que costumava ser Plan + Spec + ADRs em um unico documento com decisoes de arquitetura inline.

---

## O Que Este Comando Faz

1. **Analisar** - Entender requisitos do DEFINE
2. **Arquitetar** - Projetar solucao de alto nivel com diagramas
3. **Decidir** - Documentar decisoes-chave com justificativa (ADRs inline)
4. **Especificar** - Criar manifesto de arquivos e padroes de codigo
5. **Planejar Testes** - Definir estrategia de testes

---

## Processo

### Passo 1: Carregar Contexto

```markdown
Read(.claude/sdd/features/DEFINE_{FEATURE}.md)
Read(.claude/sdd/templates/DESIGN_TEMPLATE.md)
Read(.claude/CLAUDE.md)

# Explorar codebase por padroes:
Glob(**/*.py) | head -20
Grep("class |def ") | sample
```

### Passo 2: Criar Arquitetura

Projetar a solucao:

| Componente | Conteudo |
|------------|----------|
| **Visao Geral** | Diagrama ASCII do sistema |
| **Componentes** | Lista de modulos/servicos |
| **Fluxo de Dados** | Como os dados fluem pelo sistema |
| **Pontos de Integracao** | Dependencias externas |

### Passo 3: Documentar Decisoes (ADRs Inline)

Para cada escolha significativa:

```markdown
### Decisao: {Nome}

| Atributo | Valor |
|----------|-------|
| **Status** | Aceita |
| **Data** | AAAA-MM-DD |

**Contexto:** Por que esta decisao foi necessaria

**Escolha:** O que estamos fazendo

**Justificativa:** Por que esta abordagem

**Alternativas Rejeitadas:**
1. Opcao A - rejeitada porque X
2. Opcao B - rejeitada porque Y

**Consequencias:**
- Trade-off que aceitamos
- Beneficio que ganhamos
```

### Passo 4: Criar Manifesto de Arquivos

Listar todos os arquivos a criar/modificar:

| # | Arquivo | Acao | Proposito | Dependencias |
|---|---------|------|-----------|--------------|
| 1 | `caminho/para/arquivo.py` | Criar | Handler principal | Nenhuma |
| 2 | `caminho/para/config.yaml` | Criar | Configuracao | Nenhuma |
| 3 | `caminho/para/handler.py` | Criar | Handler de requisicoes | 1, 2 |

### Passo 5: Definir Padroes de Codigo

Fornecer trechos de codigo prontos para copiar-colar para padroes-chave.

### Passo 6: Planejar Estrategia de Testes

| Tipo de Teste | Escopo | Ferramentas |
|---------------|--------|-------------|
| Unitario | Funcoes | pytest |
| Integracao | API | pytest + requests |
| E2E | Fluxo completo | Manual/automatizado |

### Passo 7: Salvar

```markdown
Write(.claude/sdd/features/DESIGN_{NOME_FEATURE}.md)
```

---

## Saida

| Artefato | Localizacao |
|----------|-------------|
| **DESIGN** | `.claude/sdd/features/DESIGN_{NOME_FEATURE}.md` |

**Proximo Passo:** `/build .claude/sdd/features/DESIGN_{NOME_FEATURE}.md`

---

## Controle de Qualidade

Antes de salvar, verificar:

```text
[ ] Diagrama de arquitetura esta claro
[ ] Todas as decisoes importantes documentadas com justificativa
[ ] Manifesto de arquivos esta completo (todos os arquivos listados)
[ ] Padroes de codigo estao prontos para copiar-colar
[ ] Estrategia de testes cobre os requisitos
[ ] Sem dependencias circulares na arquitetura
```

---

## Dicas

1. **Diagrama Primeiro** - Arte ASCII clarifica o pensamento
2. **Decisoes Sao Permanentes** - Documente o "por que" nao apenas o "o que"
3. **Arquivos Auto-Contidos** - Cada arquivo deve funcionar independentemente
4. **Config Sobre Codigo** - Use YAML para configuracoes, nao valores hardcoded
5. **Teste Cedo** - Projete para testabilidade desde o inicio

---

## Referencias

- Agente: `.claude/agents/workflow/design-agent.md`
- Template: `.claude/sdd/templates/DESIGN_TEMPLATE.md`
- Contratos: `.claude/sdd/architecture/WORKFLOW_CONTRACTS.yaml`
- Proxima Fase: `.claude/commands/workflow/build.md`
