# ADR-XXXX — {Título Descritivo}

## Metadata

| Atributo | Valor |
|----------|-------|
| **Status** | Proposta |
| **Data** | dd/MM/yyyy |
| **Responsável** | {Nome} |
| **Revisores** | {Nome1}, {Nome2} |

---

## Contexto

{O problema ou situação que motiva esta decisão. Descreva o cenário atual,
as forças em jogo e por que uma decisão é necessária neste momento.

Exemplo: "Atualmente as datas são armazenadas como strings no formato dd/MM/yyyy
na camada Bronze. A conversão para DateType ocorre em cada notebook Silver
individualmente, sem padronização, gerando inconsistências entre entidades."}

---

## Decisão

{A solução escolhida. Seja claro e direto sobre o que estamos fazendo.
Use linguagem imperativa: "Vamos usar...", "Adotaremos...", "Implementaremos..."}

---

## Justificativa

{Por que esta opção foi escolhida em vez das alternativas. Explique os
critérios de avaliação e como a decisão os atende.}

---

## Alternativas Consideradas

### Alternativa 1: {Nome}

**Descrição:** {O que seria feito}

**Rejeitada porque:** {Motivo da rejeição}

### Alternativa 2: {Nome}

**Descrição:** {O que seria feito}

**Rejeitada porque:** {Motivo da rejeição}

---

## Consequências

### Positivas

- {Benefício 1}
- {Benefício 2}

### Negativas

- {Trade-off 1}
- {Trade-off 2}

### Riscos

- {Risco identificado}: {como mitigar}

---

## Impacto

| Camada/Módulo | Afetado? | Descrição do Impacto |
|---------------|----------|----------------------|
| Staging | Sim/Não | {descrição ou "—"} |
| Bronze | Sim/Não | {descrição ou "—"} |
| Silver | Sim/Não | {descrição ou "—"} |
| Gold | Sim/Não | {descrição ou "—"} |
| Repository | Sim/Não | {descrição ou "—"} |
| CI/CD | Sim/Não | {descrição ou "—"} |
| Documentação | Sim/Não | {descrição ou "—"} |

---

## Relacionamentos

| Tipo | ADR | Descrição |
|------|-----|-----------|
| Substituída por | — | {se depreciada, qual ADR a substitui} |
| Substitui | — | {se esta ADR substitui outra} |
| Relacionada a | — | {ADRs com decisões complementares} |

---

## Notas Adicionais

{Informações extras, links para documentação externa, diagramas, referências, etc.
Remova esta seção se não houver conteúdo adicional.}
