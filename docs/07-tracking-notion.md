# Cenário 7 — Rastreamento Notion ↔ GitHub

Este doc define **o padrão de commits, branches e PRs** para que o trabalho seja rastreado no Notion.

A premissa: toda mudança **não trivial** nasce de uma página (card/ticket/doc) no Notion. O link pro Notion tem que aparecer no PR **e** no commit final que fica em `main`/`develop`, pra que a integração oficial [Notion ↔ GitHub](https://www.notion.so/help/github) faça o link bidirecional automaticamente.

---

## 🔗 Como a integração do Notion descobre o link

A integração oficial faz sync quando encontra uma **URL `notion.so`** em:

1. **Título do PR** (opcional — só se couber sem ficar feio).
2. **Corpo do PR** (✅ **aqui é obrigatório** — é onde colamos).
3. **Mensagem de commit** (opcional, mas recomendado no commit de merge pra ficar no histórico de `main`).

Qualquer URL do formato abaixo funciona:

```
https://www.notion.so/<workspace>/<slug>-<page-id>
https://www.notion.so/<page-id>
```

O **page-id** é o hash de 32 caracteres (ou com traços) no final da URL. É ele que a integração usa pra casar PR ↔ página.

---

## 📝 Padrão de commits

Seguimos **[Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0/)** (já validado no PR lint) e adicionamos um **trailer git** pro Notion.

### Formato

```
<tipo>(<escopo opcional>): <assunto em Pt-BR, começa com maiúscula>

<corpo opcional explicando o porquê>

Notion: https://www.notion.so/<workspace>/<slug>-<page-id>
```

- `Notion:` é um **git trailer** (linha chave-valor no fim da mensagem, separada por linha em branco). Igual ao `Co-Authored-By:` / `Signed-off-by:`.
- Um commit pode ter **múltiplos** trailers `Notion:` se a mudança fecha mais de uma página.
- Pra **fechar** a página no Notion quando o PR mergear, prefixe com `Closes` no corpo do PR (ver seção de PR abaixo). No commit, o trailer `Notion:` serve só pro link.

### Exemplos

✅ Bom:

```
feat(api): Adiciona endpoint /status

Expõe status detalhado pra dashboards de observabilidade consumirem
sem precisar parsear /health.

Notion: https://www.notion.so/revertai/Endpoint-status-1a2b3c4d5e6f7890abcdef1234567890
```

✅ Bom (commit curto sem corpo):

```
fix(checkout): Corrige validação de CEP com hífen

Notion: https://www.notion.so/revertai/CEP-hifen-1a2b3c4d5e6f7890abcdef1234567890
```

❌ Ruim (sem link do Notion — não aparece nos relatórios):

```
feat(api): Adiciona endpoint /status
```

❌ Ruim (link colado no meio do corpo — trailer não é reconhecido como tal pelo `git interpret-trailers`):

```
feat(api): Adiciona endpoint /status

Referente a https://www.notion.so/... esta mudança adiciona...
```

### Quando o trailer `Notion:` é opcional

- Commits em branches temporárias durante desenvolvimento (squash apaga tudo depois).
- `chore: back-merge release/X.Y.Z → develop` (é mecânico, não tem página no Notion).
- `chore(release): v0.2.0` (tag de release — a página Notion é do épico, não da tag).

O que importa é que o **commit final em `develop` ou `main`** (pós-squash/merge) tenha o trailer.

---

## 🌱 Padrão de branches

```
<tipo>/<page-id-curto>-<slug-descritivo>
```

- `<tipo>`: `feature`, `bugfix`, `release`, `hotfix`.
- `<page-id-curto>`: primeiros **8 caracteres** do page-id da URL do Notion. Suficiente pra buscar no Notion e não polui o nome da branch.
- `<slug-descritivo>`: kebab-case, curto (≤ 40 chars), descreve a mudança.

### Exemplos

```
feature/1a2b3c4d-endpoint-status
bugfix/9f8e7d6c-validacao-cep
hotfix/0.2.1-payment-timeout        # hotfix usa versão, page-id no PR
release/0.3.0                        # release usa versão, sem page-id
```

> **Por que só 8 caracteres?** O page-id completo tem 32 e polui visualmente. 8 caracteres dão ~4 bilhões de combinações — colisão no mesmo repo é improvável, e o link completo fica no PR/commit.

---

## 🔀 Padrão de PRs

### Título

**Conventional Commits**, igual ao commit. O job `pr-lint` já valida.

```
feat(api): Adiciona endpoint /status
```

### Corpo (obrigatório)

O template já traz uma seção **📎 Notion** — preencha com a URL completa. O workflow `pr-lint` falha se **nenhuma URL `notion.so`** for encontrada no corpo.

```markdown
## 📎 Notion

- https://www.notion.so/revertai/Endpoint-status-1a2b3c4d5e6f7890abcdef1234567890

Closes: https://www.notion.so/revertai/Endpoint-status-1a2b3c4d5e6f7890abcdef1234567890
```

> **`Closes:` vs link solto:** a integração do Notion fecha a página automaticamente quando o PR mergeia se você escrever `Closes: <url-notion>` (equivalente ao `Closes #123` do GitHub pra issues).

### Quando múltiplas páginas

Liste todas. Só quem tem `Closes:` fecha no merge — as outras ficam linkadas mas abertas.

```markdown
## 📎 Notion

Closes: https://www.notion.so/.../Endpoint-status-1a2b3c4d...
Refs: https://www.notion.so/.../Epic-Observabilidade-abcdef12...
```

---

## 🚦 Exceções (PRs que não precisam de Notion)

O workflow `pr-lint` **pula a validação** quando:

- O PR tem a label `no-notion` (adicione manualmente se a mudança é infra/tooling interno sem ticket).
- A branch de origem é `release/*` ou `hotfix/*` **indo pra `main`** (o link do Notion já está nos commits da feature/bugfix que formaram a release; o PR de release não precisa repetir).
- O PR é de back-merge (`chore: back-merge ...`).

Pra todo resto — **feature → develop**, **bugfix → develop**, **bugfix → release/***  — é obrigatório.

---

## ⚙️ Como extrair o page-id da URL do Notion

Cole a URL da página e pegue o hash no final:

```
https://www.notion.so/revertai/Endpoint-status-1a2b3c4d5e6f7890abcdef1234567890
                                               └────────── page-id ──────────┘
```

Algumas URLs vêm com traços (`1a2b3c4d-5e6f-7890-abcd-ef1234567890`). Ambos formatos funcionam na integração. Pro nome da branch, use os **8 primeiros caracteres sem traços**.

Um `git alias` ajuda:

```bash
git config --global alias.feature '!f() { \
  id=$(echo "$1" | grep -oE "[a-f0-9]{32}" | cut -c1-8); \
  slug="$2"; \
  git checkout -b "feature/${id}-${slug}"; \
}; f'

# uso:
git feature "https://www.notion.so/.../Endpoint-status-1a2b3c4d5e6f7890abcdef1234567890" "endpoint-status"
```

---

## ✅ Checklist mental antes de abrir o PR

- [ ] Nome da branch segue `<tipo>/<page-id-curto>-<slug>`.
- [ ] Commits têm `Notion:` trailer (pelo menos o commit principal — após squash, é o que fica).
- [ ] Título do PR é Conventional Commits.
- [ ] Corpo do PR tem URL `notion.so` na seção **📎 Notion**.
- [ ] Se a mudança **fecha** a página: usou `Closes: <url>` no corpo.

---

## ➡️ Relacionados

- [01 — Fluxo normal](01-fluxo-normal.md) — exemplos de commit com trailer Notion
- [04 — Configuração GitHub](04-configuracao-github.md) — ativar integração oficial Notion↔GitHub
