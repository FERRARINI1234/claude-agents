# Comando ADR

> Criar e gerenciar Architecture Decision Records (ADRs) do projeto Credicoamo.

## Uso

```bash
/adr <subcomando> [argumentos]
```

## Subcomandos

| Subcomando | Descrição | Exemplo |
|------------|-----------|---------|
| `criar` | Cria nova ADR numerada com template | `/adr criar "Padrão de chaves surrogate"` |
| `listar` | Lista todas as ADRs com status atual | `/adr listar` |
| `status` | Atualiza o status de uma ADR | `/adr status ADR-0001 "Aceita"` |
| `indice` | Regenera o README.md do diretório adr/ | `/adr indice` |

---

## Visão Geral

ADRs (Architecture Decision Records) são documentos que registram **decisões técnicas significativas**, capturando:
- O **contexto** que motivou a decisão
- A **decisão** tomada
- As **alternativas** consideradas e rejeitadas
- As **consequências** e trade-offs aceitos

Todas as ADRs ficam versionadas em `docs/adr/` e são automaticamente indexadas em `docs/adr/README.md`.

---

## Processo por Subcomando

### `/adr criar "Título"`

Inicia o fluxo guiado de criação de uma nova ADR:

1. Determina o próximo número sequencial (ADR-0001, ADR-0002...)
2. Carrega o template de `.claude/sdd/templates/ADR_TEMPLATE.md`
3. Guia o usuário pelas seções:
   - Contexto do problema
   - Decisão tomada
   - Alternativas consideradas
   - Camadas impactadas (Bronze/Silver/Gold/Repository/CI-CD)
   - Responsável e revisores
4. Salva em `docs/adr/ADR-XXXX-titulo-em-kebab-case.md`
5. Atualiza o índice em `docs/adr/README.md`

```bash
/adr criar "Uso de MD5 como algoritmo de chave surrogate na Gold"
```

### `/adr listar`

Exibe todas as ADRs existentes em formato tabular:

```
| ADR | Título | Status | Data |
|-----|--------|--------|------|
| ADR-0001 | ... | Aceita | 12/03/2026 |
| ADR-0002 | ... | Proposta | 15/03/2026 |
```

### `/adr status ADR-XXXX "Novo Status"`

Atualiza o status de uma ADR específica e regenera o índice.

Valores válidos: `Proposta` | `Em Discussão` | `Aceita` | `Rejeitada` | `Depreciada` | `Substituída`

```bash
/adr status ADR-0001 "Aceita"
/adr status ADR-0003 "Substituída"
```

### `/adr indice`

Regenera o arquivo `docs/adr/README.md` lendo todos os arquivos `ADR-*.md` e reconstruindo o índice com título, status, data e responsável de cada ADR.

Útil após edições manuais ou para corrigir inconsistências no índice.

---

## Saída

| Artefato | Localização |
|----------|-------------|
| **ADR individual** | `docs/adr/ADR-XXXX-titulo-kebab.md` |
| **Índice atualizado** | `docs/adr/README.md` |

---

## Controle de Qualidade

Antes de finalizar uma ADR, verificar:

```text
[ ] Título descreve uma decisão, não uma tarefa
[ ] Contexto explica o problema motivador
[ ] Decisão usa linguagem imperativa ("Vamos usar...", "Adotaremos...")
[ ] Pelo menos uma alternativa documentada com motivo de rejeição
[ ] Seção Impacto preenche todas as camadas Medallion
[ ] Status inicial = "Proposta"
[ ] Índice README.md foi atualizado
```

---

## Dicas

1. **Decisão vs Tarefa** — Uma ADR é sobre o "por que", não o "o que fazer". "Implementar cache Redis" é tarefa. "Usar Redis como cache de sessão" é decisão arquitetural.
2. **Documente enquanto decide** — ADRs escritas depois perdem contexto e alternativas descartadas
3. **Alternativas importam** — Quem lê no futuro precisa saber o que foi considerado e descartado
4. **Status vivo** — Deprecie ou substitua ADRs antigas em vez de excluir

---

## Referências

- Agente: `.claude/agents/communication/adr-writer.md`
- Template: `.claude/sdd/templates/ADR_TEMPLATE.md`
- Diretório ADRs: `docs/adr/`
- Índice: `docs/adr/README.md`
- Design do sistema: `.claude/sdd/features/DESIGN_ADR_SYSTEM.md`
