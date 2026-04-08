# Comando Ship

> Arquivar funcionalidade concluida com licoes aprendidas (Fase 4)

## Uso

```bash
/ship <arquivo-define>
```

## Exemplos

```bash
/ship .claude/sdd/features/DEFINE_FUNCOES_CLOUD_RUN.md
/ship DEFINE_AUTH_USUARIO.md
```

---

## Visao Geral

Esta e a **Fase 4** do fluxo de trabalho AgentSpec de 5 fases:

```text
Fase 0: /brainstorm → .claude/sdd/features/BRAINSTORM_{FEATURE}.md (opcional)
Fase 1: /define    → .claude/sdd/features/DEFINE_{FEATURE}.md
Fase 2: /design    → .claude/sdd/features/DESIGN_{FEATURE}.md
Fase 3: /build     → Codigo + .claude/sdd/reports/BUILD_REPORT_{FEATURE}.md
Fase 4: /ship      → .claude/sdd/archive/{FEATURE}/SHIPPED_{DATA}.md (ESTE COMANDO)
```

O comando `/ship` arquiva todos os artefatos da funcionalidade e captura licoes aprendidas.

---

## O Que Este Comando Faz

1. **Verificar** - Confirmar que todos os artefatos existem e o build passou
2. **Arquivar** - Mover documentos da funcionalidade para pasta de arquivo
3. **Documentar** - Criar resumo SHIPPED com licoes aprendidas
4. **Limpar** - Remover arquivos de trabalho da pasta features

---

## Processo

### Passo 1: Verificar Conclusao

```markdown
Read(.claude/sdd/features/DEFINE_{FEATURE}.md)
Read(.claude/sdd/features/DESIGN_{FEATURE}.md)
Read(.claude/sdd/reports/BUILD_REPORT_{FEATURE}.md)

# Verificar se relatorio de build mostra sucesso
```

### Passo 2: Criar Pasta de Arquivo

```bash
mkdir -p .claude/sdd/archive/{NOME_FEATURE}/
```

### Passo 3: Copiar Artefatos para Arquivo

```bash
cp .claude/sdd/features/DEFINE_{FEATURE}.md .claude/sdd/archive/{FEATURE}/
cp .claude/sdd/features/DESIGN_{FEATURE}.md .claude/sdd/archive/{FEATURE}/
cp .claude/sdd/reports/BUILD_REPORT_{FEATURE}.md .claude/sdd/archive/{FEATURE}/
```

### Passo 4: Gerar Documento SHIPPED

Criar resumo com:

| Secao | Conteudo |
|-------|----------|
| **Resumo** | O que foi construido |
| **Cronograma** | Datas de inicio → entrega |
| **Metricas** | Linhas de codigo, arquivos criados |
| **Licoes Aprendidas** | O que funcionou bem, o que melhorar |
| **Artefatos** | Lista de todos os documentos arquivados |

### Passo 5: Atualizar Status dos Documentos

Atualizar documentos arquivados para status "Entregue":

```markdown
Edit: archive/{FEATURE}/DEFINE_{FEATURE}.md
  - Status: → "✅ Entregue"
  - Adicionar revisao: "Entregue e arquivado"

Edit: archive/{FEATURE}/DESIGN_{FEATURE}.md
  - Status: → "✅ Entregue"
  - Adicionar revisao: "Entregue e arquivado"
```

### Passo 6: Limpar Arquivos de Trabalho

```bash
rm .claude/sdd/features/DEFINE_{FEATURE}.md
rm .claude/sdd/features/DESIGN_{FEATURE}.md
rm .claude/sdd/reports/BUILD_REPORT_{FEATURE}.md
```

### Passo 7: Salvar Documento SHIPPED

```markdown
Write(.claude/sdd/archive/{FEATURE}/SHIPPED_{DATA}.md)
```

---

## Saida

| Artefato | Localizacao |
|----------|-------------|
| **SHIPPED** | `.claude/sdd/archive/{FEATURE}/SHIPPED_{DATA}.md` |
| **DEFINE** | `.claude/sdd/archive/{FEATURE}/DEFINE_{FEATURE}.md` |
| **DESIGN** | `.claude/sdd/archive/{FEATURE}/DESIGN_{FEATURE}.md` |
| **BUILD_REPORT** | `.claude/sdd/archive/{FEATURE}/BUILD_REPORT_{FEATURE}.md` |

**Proximo Passo:** Iniciar nova funcionalidade com `/define`

---

## Controle de Qualidade

Antes de entregar, verificar:

```text
[ ] BUILD_REPORT mostra todas as tarefas concluidas
[ ] Sem problemas criticos no relatorio de build
[ ] Todos os testes passando
[ ] Codigo implantado (se aplicavel)
```

---

## Quando Entregar

Entregue quando:
- Todos os testes de aceitacao do DEFINE passam
- Relatorio de build mostra 100% de conclusao
- Nenhum problema bloqueador permanece

---

## Categorias de Licoes Aprendidas

Documente licoes nestas areas:

| Categoria | Exemplo |
|-----------|---------|
| **Processo** | "Dividir tarefas em partes menores ajudou" |
| **Tecnico** | "Arquivos de config funcionam melhor que variaveis de ambiente" |
| **Comunicacao** | "Esclarecimento antecipado evitou retrabalho" |
| **Ferramentas** | "Usar biblioteca X simplificou Y" |

---

## Dicas

1. **Nao Pule Isso** - Licoes aprendidas previnem erros futuros
2. **Seja Honesto** - Documente o que nao funcionou tambem
3. **Seja Especifico** - "Melhor planejamento" → "Criar diagrama de arquitetura antes de codificar"
4. **Arquive Tudo** - Voce do futuro vai agradecer o voce do presente

---

## Referencias

- Agente: `.claude/agents/workflow/ship-agent.md`
- Template: `.claude/sdd/templates/SHIPPED_TEMPLATE.md`
- Contratos: `.claude/sdd/architecture/WORKFLOW_CONTRACTS.yaml`
- Fase Anterior: `.claude/commands/workflow/build.md`
