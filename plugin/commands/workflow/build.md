# Comando Build

> Executar implementacao com geracao de tarefas on-the-fly (Fase 3)

## Uso

```bash
/build <arquivo-design>
```

## Exemplos

```bash
/build .claude/sdd/features/DESIGN_FUNCOES_CLOUD_RUN.md
/build DESIGN_AUTH_USUARIO.md
```

---

## Visao Geral

Esta e a **Fase 3** do fluxo de trabalho AgentSpec de 5 fases:

```text
Fase 0: /brainstorm → .claude/sdd/features/BRAINSTORM_{FEATURE}.md (opcional)
Fase 1: /define    → .claude/sdd/features/DEFINE_{FEATURE}.md
Fase 2: /design    → .claude/sdd/features/DESIGN_{FEATURE}.md
Fase 3: /build     → Codigo + .claude/sdd/reports/BUILD_REPORT_{FEATURE}.md (ESTE COMANDO)
Fase 4: /ship      → .claude/sdd/archive/{FEATURE}/SHIPPED_{DATA}.md
```

O comando `/build` executa a implementacao, gerando tarefas on-the-fly a partir do manifesto de arquivos.

---

## O Que Este Comando Faz

1. **Analisar** - Extrair manifesto de arquivos do DESIGN
2. **Priorizar** - Ordenar arquivos por dependencias
3. **Executar** - Criar cada arquivo com verificacao
4. **Validar** - Executar testes apos cada mudanca significativa
5. **Relatar** - Gerar relatorio de build

---

## Processo

### Passo 1: Carregar Contexto

```markdown
Read(.claude/sdd/features/DESIGN_{FEATURE}.md)
Read(.claude/sdd/features/DEFINE_{FEATURE}.md)
Read(.claude/CLAUDE.md)
```

### Passo 2: Extrair Tarefas do Manifesto de Arquivos

Converter o manifesto de arquivos em uma lista de tarefas:

```markdown
Do manifesto de arquivos do DESIGN:
| Arquivo | Acao | Proposito |

Gerar:
- [ ] Criar/Modificar {arquivo1}
- [ ] Criar/Modificar {arquivo2}
- [ ] ...
```

### Passo 3: Ordenar por Dependencias

Analisar imports e dependencias para determinar ordem de execucao.

### Passo 4: Executar Cada Tarefa

Para cada arquivo:

1. **Escrever** - Criar o arquivo seguindo padroes de codigo do DESIGN
2. **Verificar** - Executar comando de verificacao (lint, type check, teste de import)
3. **Marcar Completo** - Atualizar progresso

### Passo 5: Executar Validacao Completa

Apos todos os arquivos criados:

```bash
# Verificacao de lint
ruff check .

# Verificacao de tipos (se aplicavel)
mypy .

# Executar testes
pytest
```

### Passo 6: Gerar Relatorio de Build

```markdown
Write(.claude/sdd/reports/BUILD_REPORT_{FEATURE}.md)
```

---

## Saida

| Artefato | Localizacao |
|----------|-------------|
| **Codigo** | Conforme especificado no manifesto de arquivos do DESIGN |
| **Relatorio de Build** | `.claude/sdd/reports/BUILD_REPORT_{FEATURE}.md` |

**Proximo Passo:** `/ship .claude/sdd/features/DEFINE_{FEATURE}.md` (quando pronto)

---

## Loop de Execucao

O agente de build segue este loop para cada tarefa:

```text
┌─────────────────────────────────────────────────────┐
│                    EXECUTAR TAREFA                    │
├─────────────────────────────────────────────────────┤
│  1. Ler tarefa do manifesto                          │
│  2. Escrever codigo seguindo padroes do DESIGN       │
│  3. Executar comando de verificacao                  │
│     └─ Se FALHAR → Corrigir e tentar novamente (max 3)│
│  4. Marcar tarefa como completa                      │
│  5. Passar para proxima tarefa                       │
└─────────────────────────────────────────────────────┘
```

---

## Controle de Qualidade

Antes de marcar como completo, verificar:

```text
[ ] Todos os arquivos do manifesto criados
[ ] Todos os comandos de verificacao passam
[ ] Verificacao de lint passa
[ ] Testes passam (se aplicavel)
[ ] Sem comentarios TODO deixados no codigo
[ ] Relatorio de build gerado
```

---

## Dicas

1. **Siga o DESIGN** - Nao improvise, use os padroes de codigo
2. **Verifique Incrementalmente** - Teste apos cada arquivo, nao no final
3. **Corrija para Frente** - Se algo quebrar, corrija imediatamente
4. **Auto-Contido** - Cada arquivo deve ser funcionalmente independente
5. **Sem Comentarios** - Codigo deve ser auto-documentado

---

## Lidando com Problemas Durante o Build

Se encontrar problemas:

| Problema | Acao |
|----------|------|
| Requisito faltando | Use `/iterate` para atualizar DEFINE |
| Problema de arquitetura | Use `/iterate` para atualizar DESIGN |
| Bug simples | Corrija imediatamente e continue |
| Bloqueio maior | Pare e relate no relatorio de build |

---

## Referencias

- Agente: `.claude/agents/workflow/build-agent.md`
- Template: `.claude/sdd/templates/BUILD_REPORT_TEMPLATE.md`
- Contratos: `.claude/sdd/architecture/WORKFLOW_CONTRACTS.yaml`
- Proxima Fase: `.claude/commands/workflow/ship.md`
