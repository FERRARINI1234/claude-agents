# Criar Branch

Crie uma nova branch seguindo as convencoes de nomenclatura do projeto.

## Instrucoes

1. **Perguntar o numero da atividade**
   - Pergunte ao usuario: "Qual o numero da atividade? (ex: 1234, TASK-567, ou 'nenhum')"
   - Se informado, incluir no nome da branch: `<tipo>/<numero>-<descricao-curta>`
   - Se nao houver atividade, seguir sem numero: `<tipo>/<descricao-curta>`

2. **Verificar estado atual**
   - Execute `git status` para garantir que nao ha mudancas pendentes
   - Execute `git branch` para ver a branch atual

3. **Determinar o tipo da branch**

   | Prefixo      | Uso                                    |
   |--------------|----------------------------------------|
   | `feat/`      | Nova funcionalidade                    |
   | `fix/`       | Correcao de bug                        |
   | `refactor/`  | Refatoracao de codigo                  |
   | `docs/`      | Alteracoes em documentacao             |
   | `chore/`     | Tarefas de manutencao                  |
   | `test/`      | Adicao ou modificacao de testes        |

4. **Criar a branch**
   ```bash
   git checkout main
   git pull origin main
   # Com numero de atividade:
   git checkout -b <tipo>/<numero>-<descricao-curta>
   # Sem numero de atividade:
   git checkout -b <tipo>/<descricao-curta>
   ```

5. **Convencoes de nome**
   - Usar letras minusculas
   - Separar palavras com hifen `-`
   - Ser descritivo mas conciso
   - Exemplos com numero de atividade:
     - `#1234-pipeline-notas-fiscais`
     - `#5678-corrigir-merge-dim-pessoa`
   - Exemplos sem numero de atividade:
     - `refactor/padronizar-notebooks-gold`
     - `test/checks-soda-dimensoes`

## Exemplo de Fluxo

```bash
# Com numero de atividade
git checkout main
git pull origin main
git checkout -b #1234-pipeline-extracao-agcr

# Sem numero de atividade
git checkout main
git pull origin main
git checkout -b feat/pipeline-extracao-agcr
```

## NUNCA
- Criar branches a partir de branches de feature (sempre a partir da main)
- Usar espacos ou caracteres especiais no nome
- Criar branch sem pull da main atualizada