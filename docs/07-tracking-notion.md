# Cenário 7 — Rastreamento Notion ↔ GitHub

Este doc define **o padrão de commits, branches e PRs** para que o trabalho seja rastreado no Notion.

A premissa: toda mudança **não trivial** nasce de uma página (card/ticket/doc) no Notion. O identificador da página tem que aparecer no PR **e** no commit final que fica em `main`/`develop`, pra que a integração oficial [Notion ↔ GitHub](https://www.notion.so/help/github) faça o link bidirecional automaticamente.

---

## 🔗 Como a integração do Notion descobre o link

A integração oficial reconhece **dois mecanismos** no corpo/título do PR e nas mensagens de commit:

### 1. Unique ID + magic word (preferido)

Padrão estilo Jira/Linear. Exige que o database do Notion tenha a propriedade **Unique ID** habilitada (a integração cria automaticamente ao conectar "GitHub Pull Requests" na primeira vez).

```
<magic-word> <UNIQUE-ID>
```

**Magic words que *fecham* a página no merge:** `close`, `closes`, `closed`, `fix`, `fixes`, `fixed`, `resolve`, `resolves`, `resolved`, `complete`, `completes`, `completed`, `completing`.

**Magic words que apenas *linkam* (sem fechar):** `ref`, `references`, `part of`, `related to`, `contributes to`, `towards`.

```
fixes GFD-13
closes GFD-42
refs GFD-7
part of GFD-100
```

### 2. URL `notion.so` (fallback)

Qualquer URL do formato abaixo funciona (usamos quando o card ainda não tem Unique ID configurado, ou quando queremos um link clicável com preview):

```
https://www.notion.so/<workspace>/<slug>-<page-id>
https://www.notion.so/<page-id>
```

> **Regra prática:** use **Unique ID** em branch, commit trailer e `closes/refs` do PR. URL só em comentários longos onde um preview clicável ajude.

---

## 📝 Padrão de commits

Seguimos **[Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0/)** (já validado no PR lint) e adicionamos um **trailer git** pro Notion.

### Formato

```
<tipo>(<escopo opcional>): <assunto em Pt-BR, começa com maiúscula>

<corpo opcional explicando o porquê>

Notion: GFD-42
```

- `Notion:` é um **git trailer** (linha chave-valor no fim da mensagem, separada por linha em branco). Igual ao `Co-Authored-By:` / `Signed-off-by:`.
- Um commit pode ter **múltiplos** trailers `Notion:` se a mudança toca mais de uma página (`Notion: GFD-42` + `Notion: GFD-43`).
- Pra **fechar** a página quando o PR mergeia, use a magic word no corpo do PR (ver seção de PR abaixo). O trailer `Notion:` no commit só garante o link bidirecional.
- URL ainda é aceita no trailer (`Notion: https://www.notion.so/...`) como fallback pra quando não tem Unique ID.

### Exemplos

✅ Bom (ID):

```
feat(api): Adiciona endpoint /status

Expõe status detalhado pra dashboards de observabilidade consumirem
sem precisar parsear /health.

Notion: GFD-42
```

✅ Bom (commit curto sem corpo):

```
fix(checkout): Corrige validação de CEP com hífen

Notion: GFD-87
```

✅ Bom (múltiplas páginas):

```
refactor(auth): Extrai provider de sessão

Notion: GFD-120
Notion: GFD-121
```

✅ Aceitável (URL como fallback, quando database não tem Unique ID):

```
feat(api): Adiciona endpoint /status

Notion: https://www.notion.so/revertai/Endpoint-status-1a2b3c4d5e6f7890abcdef1234567890
```

❌ Ruim (sem link do Notion — não aparece nos relatórios):

```
feat(api): Adiciona endpoint /status
```

❌ Ruim (link colado no meio do corpo — trailer não é reconhecido como tal pelo `git interpret-trailers`):

```
feat(api): Adiciona endpoint /status

Referente a GFD-42 esta mudança adiciona...
```

### Quando o trailer `Notion:` é opcional

- Commits em branches temporárias durante desenvolvimento (squash apaga tudo depois).
- `chore: back-merge release/X.Y.Z → develop` (é mecânico, não tem página no Notion).
- `chore(release): v0.2.0` (tag de release — a página Notion é do épico, não da tag).

O que importa é que o **commit final em `develop` ou `main`** (pós-squash/merge) tenha o trailer.

---

## 🌱 Padrão de branches

```
<tipo>/<UNIQUE-ID>/<slug-descritivo>
```

- `<tipo>`: `feature`, `bugfix`, `release`, `hotfix`.
- `<UNIQUE-ID>`: Unique ID do card no Notion (ex.: `GFD-42`). Pode ser copiado direto da UI do Notion ou do badge do card.
- `<slug-descritivo>`: kebab-case, curto (≤ 40 chars), descreve a mudança.

> **Por que a barra `/` entre ID e slug?** Git e vários clients (GitHub UI, lazygit, tig) tratam `/` como "pasta" e agrupam. Assim, duas frentes do mesmo ticket ficam visualmente juntas:
> ```
> feature/GFD-42/endpoint-status
> feature/GFD-42/spike-cache
> ```

### Exemplos

```
feature/GFD-42/endpoint-status
bugfix/GFD-87/validacao-cep
hotfix/0.2.1-payment-timeout        # hotfix usa versão; ID vai no PR/commit
release/0.3.0                        # release usa versão, sem ID
```

> **Por que não usar URL/hash?** O Unique ID é **estável** (nunca muda se renomearem a página), **curto** (6-10 chars), e **pesquisável** no Notion. O hash de 32 chars ou os 8 primeiros eram opacos — não dá pra buscar `1a2b3c4d` no Notion e achar a página.

> **Atenção com rulesets/patterns:** como a branch agora tem 2 níveis (`feature/GFD-42/slug`), use `feature/**` (duplo asterisco) em rulesets e workflows que casem por glob. `feature/*` só pega 1 nível.

### Fallback (sem Unique ID configurado)

Enquanto o database do Notion não tiver a propriedade Unique ID habilitada, use os 8 primeiros caracteres do page-id no lugar do ID:

```
feature/1a2b3c4d/endpoint-status
```

Migre pro padrão com ID assim que possível — é uma mudança de 1 clique no database do Notion.

---

## 🔀 Padrão de PRs

### Título

**Começa com o Unique ID entre brackets**, seguido de uma descrição curta em Pt-BR:

```
[GFD-42] Adiciona endpoint /status
[GFD-87] Corrige validação de CEP com hífen
[GFD-120] Extrai provider de sessão
```

> **Nota:** o título do PR **não** exige Conventional Commits (o job de título no `pr-lint` foi desligado). Os **commits** ainda seguem Conventional Commits — é lá que mora a automação de SemVer. No squash merge, a mensagem final pode ser ajustada pra Conventional Commits na hora do merge se quiser manter a `main`/`develop` limpas.

### Exceções (sem card no Notion)

- PRs com label `no-notion`, back-merges e `release/* → main` / `hotfix/* → main` não precisam do ID no título. Use descrição direta:
  ```
  Release 0.3.0
  Back-merge release/0.3.0 → develop
  chore(ci): Atualiza action do semantic-pr
  ```

### Corpo (obrigatório)

O template já traz uma seção **📎 Notion** — preencha com magic word + Unique ID (ou URL como fallback). O workflow `pr-lint` falha se nada for encontrado no corpo.

```markdown
## 📎 Notion

Closes GFD-42
```

> **`Closes <ID>` vs link solto:** a integração do Notion fecha a página automaticamente quando o PR mergeia se você usar uma magic word de fechamento (`closes`, `fixes`, `resolves`...). Um ID solto ou URL "pelada" também linka, mas **não fecha** no merge.

### Quando múltiplas páginas

Liste todas com a magic word correta — **só quem tem magic word de fechamento fecha no merge**; o resto fica linkado mas aberto:

```markdown
## 📎 Notion

Closes GFD-42
Refs GFD-100
Part of GFD-7
```

### Fallback (URL)

Se o database não tem Unique ID, use URL com `Closes:` / `Refs:`:

```markdown
## 📎 Notion

Closes: https://www.notion.so/revertai/Endpoint-status-1a2b3c4d5e6f7890abcdef1234567890
Refs: https://www.notion.so/revertai/Epic-Observabilidade-abcdef1234567890abcdef1234567890
```

---

## 🚦 Exceções (PRs que não precisam de Notion)

O workflow `pr-lint` **pula a validação** quando:

- O PR tem a label `no-notion` (adicione manualmente se a mudança é infra/tooling interno sem ticket).
- A branch de origem é `release/*` ou `hotfix/*` **indo pra `main`** (o link do Notion já está nos commits da feature/bugfix que formaram a release; o PR de release não precisa repetir).
- O PR é de back-merge (`chore: back-merge ...`).

Pra todo resto — **feature → develop**, **bugfix → develop**, **bugfix → release/***  — é obrigatório.

---

## ⚙️ Como obter o Unique ID de um card do Notion

1. Abra a página no Notion.
2. O Unique ID aparece como badge (ex.: `GFD-42`) no cabeçalho da página **se** o database tiver a propriedade habilitada.
3. Se não aparecer: vá no database → **⋯** → **Properties** → **+ Add a property** → **Unique ID** → defina prefixo (ex.: `GFD`). A partir dali, todo card ganha ID incremental.

> A integração "GitHub Pull Requests" do Notion cria essa propriedade automaticamente na primeira conexão — então normalmente já existe.

### Um `git alias` ajuda a criar branch a partir do ID:

```bash
git config --global alias.feature '!f() { \
  id="$1"; \
  slug="$2"; \
  git checkout -b "feature/${id}/${slug}"; \
}; f'

# uso:
git feature GFD-42 endpoint-status
# → checkout em feature/GFD-42/endpoint-status
```

---

## ✅ Checklist mental antes de abrir o PR

- [ ] Nome da branch segue `<tipo>/<UNIQUE-ID>/<slug>` (ou versão, no caso de release/hotfix).
- [ ] Commits têm `Notion: <UNIQUE-ID>` trailer (pelo menos o commit principal — após squash, é o que fica).
- [ ] Título do PR começa com `[UNIQUE-ID]` (ex.: `[GFD-42] Adiciona endpoint /status`).
- [ ] Corpo do PR tem magic word + ID na seção **📎 Notion** (ex.: `Closes GFD-42`).
- [ ] Se a mudança **fecha** a página: usou `Closes` / `Fixes` / `Resolves` (não só `Refs`).

---

## ➡️ Relacionados

- [01 — Fluxo normal](01-fluxo-normal.md) — exemplos de commit com trailer Notion
- [04 — Configuração GitHub](04-configuracao-github.md) — ativar integração oficial Notion↔GitHub
