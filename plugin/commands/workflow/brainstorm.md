# Comando Brainstorm

> Exploracao colaborativa antes da captura de requisitos (Fase 0)

## Uso

```bash
/brainstorm <ideia-ou-solicitacao>
/brainstorm "Construir pipeline de extracao de notas fiscais"
/brainstorm notas/ideia-inicial.txt
```

## Exemplos

```bash
# A partir de uma ideia direta
/brainstorm "Quero automatizar o processamento de notas fiscais"

# A partir de um arquivo com anotacoes
/brainstorm docs/notas-reuniao.md

# A partir de uma declaracao de problema
/brainstorm "Nossa equipe gasta muito tempo com entrada manual de dados"
```

---

## Visao Geral

Esta e a **Fase 0** do fluxo de trabalho AgentSpec de 5 fases:

```text
Fase 0: /brainstorm → .claude/sdd/features/BRAINSTORM_{FEATURE}.md (ESTE COMANDO)
Fase 1: /define    → .claude/sdd/features/DEFINE_{FEATURE}.md
Fase 2: /design    → .claude/sdd/features/DESIGN_{FEATURE}.md
Fase 3: /build     → Codigo + .claude/sdd/reports/BUILD_REPORT_{FEATURE}.md
Fase 4: /ship      → .claude/sdd/archive/{FEATURE}/SHIPPED_{DATA}.md
```

O comando `/brainstorm` explora ideias atraves de dialogo antes de capturar requisitos formais.

---

## O Que Este Comando Faz

1. **Explorar** - Entender o contexto do projeto e padroes existentes
2. **Questionar** - Fazer uma pergunta por vez para esclarecer a intencao
3. **Coletar** - Reunir arquivos de exemplo, dados de referencia ou ground truth para fundamentacao do LLM
4. **Propor** - Apresentar 2-3 abordagens com trade-offs
5. **Simplificar** - Aplicar YAGNI para remover funcionalidades desnecessarias
6. **Validar** - Confirmar entendimento incrementalmente
7. **Documentar** - Gerar documento BRAINSTORM para /define

---

## Processo

### Passo 1: Reunir Contexto

```markdown
Read(.claude/CLAUDE.md)
Read(.claude/sdd/templates/BRAINSTORM_TEMPLATE.md)
Explorar estrutura do projeto, commits recentes, padroes existentes
```

### Passo 2: Perguntas de Descoberta

Fazer perguntas UMA POR VEZ:

| Tipo de Pergunta | Quando Usar |
|------------------|-------------|
| Multipla Escolha | Quando as opcoes sao claras (preferido) |
| Aberta | Quando explorando territorio desconhecido |
| Esclarecedora | Quando a resposta foi vaga |

**Minimo:** 3 perguntas antes de propor abordagens

### Passo 3: Coleta de Amostras (Fundamentacao LLM)

Perguntar sobre amostras disponiveis para melhorar a precisao da IA/LLM:

```markdown
"Voce tem alguma amostra que possa ajudar a fundamentar a solucao?
(a) Arquivos de entrada de exemplo
(b) Exemplos de saida esperada
(c) Ground truth / dados verificados
(d) Nenhum disponivel"
```

Se existirem amostras, analisar e documentar na saida do BRAINSTORM.

### Passo 4: Explorar Abordagens

Apresentar 2-3 abordagens distintas:

```markdown
### Abordagem A: {Nome} ⭐ Recomendada
**Por que:** {Raciocinio}
**Pros:** {Beneficios}
**Contras:** {Trade-offs}

### Abordagem B: {Nome}
**Por que nao recomendada:** {Raciocinio}
```

### Passo 5: Aplicar YAGNI

Para cada funcionalidade, perguntar:
- Precisamos disso para o MVP?
- Isso resolve o problema central?

Remover funcionalidades que nao passarem. Documentar o que foi removido e por que.

### Passo 6: Validar Incrementalmente

Apresentar o design em secoes (200-300 palavras cada):

```text
Secao → Verificar com usuario → Ajustar se necessario → Proxima secao
```

**Minimo:** 2 pontos de validacao

### Passo 7: Gerar Documento

```markdown
Write(.claude/sdd/features/BRAINSTORM_{FEATURE}.md)
```

---

## Saida

| Artefato | Localizacao |
|----------|-------------|
| **Documento Brainstorm** | `.claude/sdd/features/BRAINSTORM_{FEATURE}.md` |

**Proximo Passo:** `/define .claude/sdd/features/BRAINSTORM_{FEATURE}.md`

---

## Controle de Qualidade

Antes de marcar como completo:

```text
[ ] Minimo de 3 perguntas de descoberta feitas
[ ] Pergunta sobre coleta de amostras feita
[ ] Pelo menos 2 abordagens exploradas
[ ] YAGNI aplicado (funcionalidades removidas)
[ ] Minimo de 2 validacoes completadas
[ ] Usuario confirmou abordagem selecionada
[ ] Requisitos preliminares incluidos
```

---

## Estilo de Interacao

### Uma Pergunta por Vez

```markdown
BOM:
"Qual e o caso de uso principal?
(a) Relatorios internos
(b) Voltado para clientes
(c) Ambos"

RUIM:
"Qual e o caso de uso? Quem sao os usuarios? Qual e o prazo?"
```

### Liderar com Recomendacao

```markdown
BOM:
"Eu recomendo a Abordagem A porque [raciocinio].
Aqui estao as alternativas a considerar..."

RUIM:
"Aqui estao tres abordagens. Qual voce quer?"
```

### Estar Pronto para Voltar Atras

```markdown
BOM:
"Isso e diferente do que eu entendi. Deixe-me revisar..."

RUIM:
"Passando para a proxima secao..."
```

---

## Quando Usar /brainstorm vs /define

| Cenario | Usar |
|---------|------|
| Ideia vaga, precisa explorar | `/brainstorm` |
| Requisitos claros, pronto para capturar | `/define` diretamente |
| Documento BRAINSTORM existente | `/define <arquivo-brainstorm>` |
| Notas de reuniao com pedidos claros | `/define` diretamente |
| "Quero construir algo mas nao sei o que" | `/brainstorm` |

---

## Dicas

1. **Leve seu tempo** - Exploracao e sobre entender, nao velocidade
2. **Pergunte por que** - "Por que voce precisa disso?" revela requisitos verdadeiros
3. **Questione o escopo** - A maioria das funcionalidades nao e necessaria para o MVP
4. **Confie no usuario** - Eles conhecem seu dominio, voce conhece padroes
5. **Documente funcionalidades removidas** - Elas podem voltar depois

---

## Lidando com Diferentes Entradas

| Tipo de Entrada | Abordagem |
|-----------------|-----------|
| Ideia vaga | Comece com "Conte-me mais sobre..." |
| Solicitacao especifica | Valide entendimento, depois explore abordagens |
| Declaracao de problema | Foque nos pontos de dor, depois solucoes |
| Solicitacao de funcionalidade | Questione a necessidade, explore alternativas |
| Solicitacao de comparacao | Explore trade-offs, faca recomendacao |

---

## Referencias

- Agente: `.claude/agents/workflow/brainstorm-agent.md`
- Template: `.claude/sdd/templates/BRAINSTORM_TEMPLATE.md`
- Contratos: `.claude/sdd/architecture/WORKFLOW_CONTRACTS.yaml`
- Proxima Fase: `.claude/commands/workflow/define.md`
