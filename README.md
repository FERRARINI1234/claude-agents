# claude-agents

Framework de agentes especializados para Data Engineering com **Microsoft Fabric**, **dbt** e **Pentaho**.
Inclui 27 agentes, 4 skills, Knowledge Base sobre Fabric/Delta Lake, workflow SDD de 6 fases e 15 comandos slash.

---

## Instalacao

### Via plugin do Claude Code (recomendado)

Em qualquer projeto, execute dentro do Claude Code:

```
/plugin marketplace add https://github.com/FERRARINI1234/claude-agents
/plugin install claude-agents@claude-agents
```

Isso instala automaticamente todos os agentes, commands e skills em `~/.claude/`.

### Manual (fallback)

```bash
git clone https://github.com/FERRARINI1234/claude-agents.git
cp -r claude-agents/plugin/agents   ~/.claude/agents
cp -r claude-agents/plugin/commands ~/.claude/commands
cp -r claude-agents/plugin/skills   ~/.claude/skills
cp -r claude-agents/plugin/kb       ~/.claude/kb
```

### Via symlink (unica fonte da verdade)

```bash
git clone https://github.com/FERRARINI1234/claude-agents.git ~/.claude-agents
ln -s ~/.claude-agents/plugin/agents   ~/.claude/agents
ln -s ~/.claude-agents/plugin/commands ~/.claude/commands
ln -s ~/.claude-agents/plugin/skills   ~/.claude/skills
ln -s ~/.claude-agents/plugin/kb       ~/.claude/kb
```

Atualizacoes: `cd ~/.claude-agents && git pull`

---

## Agentes disponiveis

### Code Quality

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `code-reviewer` | Revisao especializada de codigo: qualidade, seguranca e manutenibilidade | Apos escrever ou modificar codigo significativo |
| `python-developer` | Arquiteto de codigo Python com dataclasses, type hints e generators | Ao escrever parsers e pipelines de dados |
| `code-cleaner` | Remove comentarios excessivos, aplica DRY e moderniza codigo Python | Ao limpar, refatorar ou modernizar codigo |
| `code-documenter` | Cria documentacao completa e production-ready (README, API docs) | Ao precisar de documentacao ou README |
| `claude-md-manager` | Especialista em criar e manter arquivos CLAUDE.md | Ao criar ou sincronizar o contexto do projeto |
| `python-tester-especialist` | Analisa codigo, executa testes e reporta resultados (unit + integration) | Ao testar, validar ou verificar codigo Python |

### Communication

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `adaptive-explainer` | Adapta explicacoes para qualquer audiencia com analogias e visualizacoes | Ao explicar conceitos tecnicos para stakeholders |
| `meeting-analyst` | Transforma notas de reuniao em documentacao estruturada e acionavel | Ao analisar transcricoes ou threads de Slack |
| `the-planner` | Arquiteto estrategico que cria planos de implementacao com validacao MCP | Ao planejar tarefas complexas ou decisoes de arquitetura |

### Data Engineering

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `medallion-architect` | Especialista em Medallion Architecture com 10+ anos de experiencia em lakehouse | Ao projetar arquiteturas de dados ou planejar camadas |

### Data Quality

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `soda-tests-especialist` | Cria arquivos de teste Soda Core para validacao de dados | Ao precisar de testes de qualidade para tabelas `dim_*` ou `fato_*` |

### Exploration

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `kb-architect` | Cria secoes completas de Knowledge Base do zero com validacao MCP | Ao criar dominios KB, auditar ou adicionar conceitos |

### Microsoft Fabric

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `fabric-notebook-specialist` | Cria e mantem notebooks Fabric com estrutura 100% correta (.platform + notebook-content.py) | Ao criar, corrigir ou revisar notebooks Fabric |

### IA / Data Engineering

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `adr-writer` | Cria, lista e mantem ADRs versionadas em `docs/adr/` | Ao documentar decisoes tecnicas arquiteturais |

### dbt

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `dbt-especialist` | Orquestrador de artefatos dbt: source.yml, staging, intermediate e marts via Skill dbt | Ao criar modelos dbt, source.yml ou staging em lote |

### Pentaho

| Agente | Descricao | Quando usar |
|--------|-----------|-------------|
| `pentaho-especialist` | Analista somente leitura de projetos PDI: mapeia .ktr, .kjb, conexoes e dependencias | Ao analisar, entender ou mapear artefatos Pentaho |
| `pentaho-business-especialist` | Extrai regras de negocio de steps IfNull, StringOperations e Calculator em .ktr e gera `business-role.md` | Ao documentar tratamentos de nulos, padronizacoes de texto e calculos |
| `pentaho-insert-update-especialist` | Extrai mapeamento de tabela, chave primaria e colunas de steps InsertUpdate em .ktr e gera .yml | Ao extrair mapeamento de carga (insert/update) de transformacoes |
| `pentaho-select-values-especialist` | Extrai mapeamento de-para (rename e type cast) de steps SelectValues em .ktr e gera .yml | Ao mapear renomeacoes e conversoes de tipos de colunas |
| `pentaho-table-input-especialist` | Extrai SQL de steps TableInput em .ktr e gera um .yml por step | Ao extrair SQLs de transformacoes Pentaho |
| `pentaho-table-mapping` | Le SQLs de .yml, .ktr ou .sql e gera `schema.yml` com mapeamento de tabelas e colunas | Ao mapear dependencias de dados a partir de SQLs de transformacoes |

### Workflow SDD (Spec-Driven Development)

| Agente | Fase | Descricao |
|--------|------|-----------|
| `brainstorm-agent` | Fase 0 | Exploracao colaborativa com requisitos vagos |
| `define-agent` | Fase 1 | Extracao e validacao de requisitos estruturados |
| `design-agent` | Fase 2 | Arquitetura e especificacao tecnica |
| `build-agent` | Fase 3 | Executor de implementacao a partir do DESIGN |
| `iterate-agent` | Fase 4 | Atualizacao cross-phase com consciencia de cascata |
| `ship-agent` | Fase 5 | Arquivamento de features e licoes aprendidas |

---

## Skills disponiveis

| Skill | Descricao |
|-------|-----------|
| `pyspark-fabric` | Desenvolvimento PySpark para Microsoft Fabric Lakehouse (ETL, Bronze/Silver/Gold) |
| `delta-lake-fabric` | Operacoes Delta Lake: MERGE/UPSERT, Time Travel, SCD Type 1 e 2 |
| `frontend-design` | Interfaces frontend production-grade (web components, dashboards, React) |
| `dbt` | Geracao de artefatos dbt: source.yml, staging, intermediate e marts seguindo Medallion Architecture |

---

## Comandos slash disponiveis

| Comando | Descricao |
|---------|-----------|
| `/adr` | Cria, lista ou atualiza Architecture Decision Records |
| `/git:create-pr` | Cria Pull Request no GitHub via `gh` CLI |
| `/git:create-branch` | Cria branch seguindo convencoes semanticas |
| `/azure-devops:create-pr` | Cria Pull Request no Azure DevOps via `az repos` |
| `/knowledge:create-kb` | Cria novo dominio na Knowledge Base |
| `/workflow:brainstorm` | Inicia Fase 0 do workflow SDD |
| `/workflow:define` | Executa Fase 1 — documento DEFINE |
| `/workflow:design` | Executa Fase 2 — documento DESIGN |
| `/workflow:build` | Executa Fase 3 — implementacao |
| `/workflow:iterate` | Executa Fase 4 — iteracao e cascata |
| `/workflow:ship` | Executa Fase 5 — arquivamento |
| `/dbt:create-source` | Cria `source.yml` para a camada staging de um projeto dbt |
| `/dbt:create-staging` | Cria modelos `stg_<tabela>.sql` a partir de um `source.yml` existente |
| `/pentaho:analyze-project` | Gera overview de um projeto Pentaho: .ktr, steps TableInput, SelectValues e DimensionLookup |
| `/pentaho:extract-sql` | Extrai SQLs de steps TableInput, gera .yml por step e `schema.yml` de mapeamento |

---

## Knowledge Base incluida

- **microsoft-fabric/concepts**: Lakehouse, Delta Lake, Notebooks, Medallion Architecture, PySpark Patterns, Fabric Deployment
- **microsoft-fabric/patterns**: Hash Change Detection, Merge-Upsert, Repository Pattern, SCD Implementation
- **Templates**: Concept e Pattern para criar novos dominios de KB

---

## Workflow SDD

O workflow Spec-Driven Development inclui 6 fases com templates padronizados:

```
Fase 0: BRAINSTORM  → explora requisitos vagos
Fase 1: DEFINE      → extrai requisitos com pontuacao de clareza
Fase 2: DESIGN      → arquitetura + manifesto de arquivos
Fase 3: BUILD       → implementacao + relatorio de build
Fase 4: ITERATE     → atualizacoes com consciencia de cascata
Fase 5: SHIP        → arquivamento + licoes aprendidas
```

Templates em `plugin/sdd/templates/`.

---

## Atualizando o plugin

Apos instalar via plugin:

```bash
# Atualiza para a versao mais recente
/plugin update claude-agents
```

Ou via symlink:

```bash
cd ~/.claude-agents && git pull
```

---

## Contribuindo

Para adicionar novos agentes, siga o template em `plugin/agents/_template.md.example`.
# claude-agents
