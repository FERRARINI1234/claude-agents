# Comando Define

> Capturar requisitos e valida-los em uma unica passagem (Fase 1)

## Uso

```bash
/define <entrada>
```

## Exemplos

```bash
# A partir de um documento BRAINSTORM (recomendado apos /brainstorm)
/define .claude/sdd/features/BRAINSTORM_PROCESSAMENTO_NOTAS.md

# A partir de notas de reuniao ou entrada bruta
/define notas/notas-reuniao.md
/define "Construir funcoes Cloud Run para processamento de notas fiscais"
/define docs/email-stakeholder.txt
```

---

## Visao Geral

Esta e a **Fase 1** do fluxo de trabalho AgentSpec de 5 fases:

```text
Fase 0: /brainstorm → .claude/sdd/features/BRAINSTORM_{FEATURE}.md (opcional)
Fase 1: /define    → .claude/sdd/features/DEFINE_{FEATURE}.md (ESTE COMANDO)
Fase 2: /design    → .claude/sdd/features/DESIGN_{FEATURE}.md
Fase 3: /build     → Codigo + .claude/sdd/reports/BUILD_REPORT_{FEATURE}.md
Fase 4: /ship      → .claude/sdd/archive/{FEATURE}/SHIPPED_{DATA}.md
```

O comando `/define` combina o que costumava ser Intake + PRD + Refine em uma unica fase iterativa. Quando alimentado com um documento BRAINSTORM, ele extrai requisitos pre-validados com minima necessidade de esclarecimento.

---

## O Que Este Comando Faz

1. **Extrair** - Obter requisitos de qualquer entrada (notas, emails, conversas)
2. **Estruturar** - Organizar em problema, usuarios, objetivos, criterios de sucesso
3. **Validar** - Pontuacao de clareza integrada (deve atingir 12/15 para prosseguir)
4. **Esclarecer** - Fazer perguntas direcionadas para quaisquer lacunas

---

## Processo

### Passo 1: Carregar Contexto

```markdown
Read(.claude/sdd/templates/DEFINE_TEMPLATE.md)
Read(.claude/CLAUDE.md)

# Se arquivo fornecido:
Read(<arquivo-entrada>)
```

### Passo 2: Classificar Entrada

Identificar o tipo de entrada para guiar a extracao:

| Tipo de Entrada | Padrao | Foco |
|-----------------|--------|------|
| `documento_brainstorm` | BRAINSTORM_*.md do /brainstorm | Pre-validado, extrair diretamente |
| `notas_reuniao` | Bullet points, itens de acao | Decisoes, requisitos |
| `thread_email` | Re:, Enc:, assinaturas | Solicitacoes, restricoes |
| `conversa` | Linguagem informal | Problema central, usuarios |
| `requisito_direto` | Solicitacao estruturada | Todos os elementos presentes |
| `fontes_mistas` | Multiplos formatos | Consolidar, deduplicar |

**Nota:** Quando a entrada e um documento BRAINSTORM, a extracao e simplificada porque:
- Perguntas de descoberta ja foram respondidas
- Abordagens foram avaliadas
- YAGNI foi aplicado
- Usuario validou a direcao

### Passo 3: Extrair Entidades

Extrair estes elementos da entrada:

| Elemento | Padroes de Extracao |
|----------|---------------------|
| **Problema** | "Estamos lutando com...", "O problema e...", "Ponto de dor:" |
| **Usuarios** | "Para a equipe...", "Clientes querem...", "Usuarios precisam..." |
| **Objetivos** | "Precisamos...", "O objetivo e...", "Sucesso parece..." |
| **Criterios de Sucesso** | "Sucesso significa...", "Saberemos quando...", "Medido por..." |
| **Testes de Aceitacao** | "Dado/Quando/Entao", "Caso de teste:", "Cenario:" |
| **Restricoes** | "Deve funcionar com...", "Nao pode mudar...", "Limitado por..." |
| **Fora do Escopo** | "Nao incluindo...", "Adiado para...", "Excluido:" |

### Passo 4: Calcular Pontuacao de Clareza

Pontuar cada elemento (0-3 pontos):

| Elemento | Pontuacao | Significado |
|----------|-----------|-------------|
| Problema | 0-3 | Claro, especifico, acionavel |
| Usuarios | 0-3 | Identificados com pontos de dor |
| Objetivos | 0-3 | Resultados mensuraveis |
| Sucesso | 0-3 | Criterios testaveis |
| Escopo | 0-3 | Limites explicitos |

**Guia de Pontuacao:**
- 0 = Completamente ausente
- 1 = Vago ou incompleto
- 2 = Claro mas faltando detalhes
- 3 = Cristalino, acionavel

**Minimo para prosseguir:** 12/15 (80%)

### Passo 5: Preencher Lacunas (se necessario)

Se pontuacao < 12, usar `AskUserQuestion` com opcoes especificas:

```markdown
Exemplos de perguntas:
- "Quem e o usuario principal: (a) equipe interna, (b) clientes, (c) ambos?"
- "Qual e o prazo: (a) este sprint, (b) este trimestre, (c) sem prazo?"
```

### Passo 6: Gerar Documento

Escrever o documento estruturado seguindo o template, depois salvar:

```markdown
Write(.claude/sdd/features/DEFINE_{NOME_FEATURE}.md)
```

---

## Saida

| Artefato | Localizacao |
|----------|-------------|
| **DEFINE** | `.claude/sdd/features/DEFINE_{NOME_FEATURE}.md` |

**Proximo Passo:** `/design .claude/sdd/features/DEFINE_{NOME_FEATURE}.md`

---

## Controle de Qualidade

Antes de salvar, verificar:

```text
[ ] Declaracao do problema e clara e especifica
[ ] Pelo menos uma persona de usuario identificada
[ ] Criterios de sucesso sao mensuraveis
[ ] Testes de aceitacao sao testaveis
[ ] Fora do escopo e explicito
[ ] Pontuacao de Clareza >= 12/15
```

---

## Dicas

1. **Seja Especifico** - "Melhorar desempenho" → "Reduzir latencia da API para <200ms"
2. **Use Numeros** - "Lidar com muitos usuarios" → "Suportar 1000 usuarios simultaneos"
3. **Teste Criterios** - Se voce nao pode testar, nao esta claro o suficiente
4. **Defina Escopo Rigorosamente** - O que esta FORA e tao importante quanto o que esta DENTRO

---

## Referencias

- Agente: `.claude/agents/workflow/define-agent.md`
- Template: `.claude/sdd/templates/DEFINE_TEMPLATE.md`
- Contratos: `.claude/sdd/architecture/WORKFLOW_CONTRACTS.yaml`
- Fase Anterior: `.claude/commands/workflow/brainstorm.md` (opcional)
- Proxima Fase: `.claude/commands/workflow/design.md`
