# 📦 Estudo de Automação com GitHub Projects + Actions

Este repositório é dedicado a **explorar as funcionalidades do GitHub Projects (v2)** e suas possibilidades de **automação com GitHub Actions**. A ideia central é testar até onde conseguimos automatizar a gestão de projetos diretamente na plataforma do GitHub, sem depender de ferramentas externas.

---

## 🚀 Objetivo do Projeto

- Testar integrações com `gh` CLI
- Criar workflows automatizados para manipular campos customizados de um projeto GitHub
- Simular um ambiente ágil com colunas, status, itens e vinculação de sub-issues
- Automatizar preenchimento de campos com base em eventos do GitHub

---

## 🧠 O que já foi automatizado?

Atualmente, temos um workflow que **detecta automaticamente quando uma issue é criada com `[Bug]` no título** e então **preenche o campo `Work Item` como `Bug`** no GitHub Project.

---

## ⚙️ Workflow: `Setar Work Item como 'Bug'`

Este workflow é acionado sempre que uma nova issue é **aberta**. Se o título da issue começar com `[Bug]`, ele irá automaticamente localizar o item correspondente no GitHub Projects e preencher o campo `"Work Item"` com o valor `"Bug"`.

### 📄 Arquivo: `.github/workflows/setar-workitem-bug.yml`

```yaml
name: Setar Work Item como 'Bug'

on:
  issues:
    types: [opened]

jobs:
  setar_workitem_bug:
    runs-on: ubuntu-latest
    permissions:
      issues: read
      contents: read

    steps:
      - name: Verifica se é bug pelo título
        id: checar
        run: |
          if [[ "${{ github.event.issue.title }}" == "[Bug]"* ]]; then
            echo "IS_BUG=true" >> $GITHUB_ENV
          fi
```
💡 Esse step verifica se o título da issue começa com [Bug]. Se sim, define a variável de ambiente IS_BUG=true para os próximos passos do fluxo.

```yaml
      - name: Obter project-id pelo número do projeto
        if: env.IS_BUG == 'true'
        run: |
          echo "🔎 Buscando project-id..."
          PROJECT_ID=$(gh project list --owner Leo-Fernandes-sh3 --format json \
            | jq -r '.projects[] | select(.number == 5) | .id')

          echo "::notice::PROJECT_ID=$PROJECT_ID"

          if [ -z "$PROJECT_ID" ]; then
            echo "::error::PROJECT_ID não encontrado"
            exit 1
          fi

          echo "PROJECT_ID=$PROJECT_ID" >> $GITHUB_ENV
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
```
🔍 Localiza o project-id usando o número do projeto (neste caso 5) e salva na variável PROJECT_ID.

```yaml
      - name: Obter item-id da issue no projeto
        if: env.IS_BUG == 'true'
        run: |
          ISSUE_NUMBER=${{ github.event.issue.number }}
          ITEM_ID=$(gh project item-list 5 --owner Leo-Fernandes-sh3 --format json \
            | jq -r --arg num "$ISSUE_NUMBER" '.items[] | select(.content.number == ($num | tonumber)) | .id')

          echo "::notice::ITEM_ID=$ITEM_ID"

          if [ -z "$ITEM_ID" ]; then
            echo "::error::Item da issue não encontrado no projeto"
            exit 1
          fi

          echo "ITEM_ID=$ITEM_ID" >> $GITHUB_ENV
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
```
🔄 Busca o item-id correspondente à issue aberta dentro do projeto, baseado no número da issue.

```yaml
      - name: Obter field-id do campo "Work Item"
        if: env.IS_BUG == 'true'
        run: |
          FIELD_ID=$(gh project field-list 5 --owner Leo-Fernandes-sh3 --format json \
            | jq -r '.fields[] | select(.name == "Work Item") | .id')
          echo "FIELD_ID=$FIELD_ID" >> $GITHUB_ENV
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
```
🏷️ Localiza o field-id do campo "Work Item" dentro do projeto para poder preenchê-lo depois.

```yaml
      - name: Atualizar campo Work Item como 'Bug'
        if: env.IS_BUG == 'true'
        run: |
          gh project item-edit \
            --id "$ITEM_ID" \
            --project-id "$PROJECT_ID" \
            --field-id "$FIELD_ID" \
            --single-select-option-id 40c1efaf
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
```
📝 Por fim, preenche o campo "Work Item" com o valor "Bug" usando o option-id correspondente (40c1efaf).

## 🔐 Autenticação com Token Pessoal (PAT)

Para que o workflow funcione corretamente e tenha permissão para interagir com o GitHub Projects, você precisa configurar um **Personal Access Token (PAT)** com os escopos corretos e adicioná-lo como secret no repositório.

### 🛠️ Passo a passo para criar o token

1. Acesse: [https://github.com/settings/tokens](https://github.com/settings/tokens)
2. Clique em **"Generate new token (classic)"**
3. Dê um nome ao token e selecione uma validade (por exemplo: 90 dias)
4. Selecione os seguintes escopos:
   - ✅ `repo`
   - ✅ `read:project`
   - ✅ `write:project`
5. Clique em **"Generate token"** e **copie o token gerado** (ele será mostrado apenas uma vez)

---

### 🔐 Como adicionar o token como secret no repositório

1. Vá até o seu repositório no GitHub
2. Acesse: `Settings` → `Secrets and variables` → `Actions`
3. Clique em **"New repository secret"**
4. Preencha os campos:
   - **Name**: `GH_PAT`
   - **Value**: _cole o token gerado_
5. Clique em **"Add secret"**

---

### ✅ Utilização no workflow

No seu workflow YAML, o token será utilizado automaticamente por meio da variável de ambiente `GH_TOKEN`:

```yaml
env:
  GH_TOKEN: ${{ secrets.GH_PAT }}



