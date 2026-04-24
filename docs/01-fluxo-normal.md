# Cenário 1 — Fluxo normal (feature → develop → release → main)

O **caminho feliz** do Git Flow:

```
feature/x ──PR──► develop ──(release/0.2.0)──► main + tag v0.2.0
                                              │
                                              └─ back-merge ──► develop
```

- Feature entra em `develop` via **PR + Squash and merge**.
- Quando acumulou escopo suficiente, abre `release/X.Y.Z` a partir de `develop`.
- Estabiliza na release branch (fixes de bugs descobertos em QA/staging).
- Fecha com **dois merges**: `release/X.Y.Z → main` (+ tag) e `release/X.Y.Z → develop` (back-merge).

---

## 1) Feature parte de `develop`

```bash
git checkout develop
git pull --rebase origin develop
# nome da branch: <tipo>/<page-id-8-chars>-<slug>   (ver docs/07-tracking-notion.md)
git checkout -b feature/1a2b3c4d-saudacao-pt-br
```

Implementa, commit em Conventional Commits **com trailer `Notion:`**, push:

```bash
git add src/main.py
git commit -m "feat(app): Melhora mensagem de boas-vindas" \
           -m "Notion: https://www.notion.so/revertai/Boas-vindas-1a2b3c4d5e6f7890abcdef1234567890"
git push -u origin feature/1a2b3c4d-saudacao-pt-br
```

> 📎 O trailer `Notion:` é o que a integração oficial [Notion ↔ GitHub](https://www.notion.so/help/github) usa pra linkar o commit à página. Ver [07-tracking-notion.md](07-tracking-notion.md).

---

## 2) PR `feature/* → develop` — Squash and merge

- Abra o PR com base em `develop` (`gh pr create -B develop`).
- CI verde + review + **Squash and merge** → 1 commit em `develop`.
- 🟢 Workflow **Deploy • develop** dispara — a feature já é visível no ambiente `develop`.

Limpeza:

```bash
git checkout develop && git pull --rebase origin develop
git branch -d feature/saudacao-pt-br
```

> ℹ️ `develop` é "o próximo release em formação" — acumula features até você decidir abrir uma release branch.

---

## 3) Abrir `release/X.Y.Z` quando estiver pronto

Quando acumulou features suficientes e você quer **fechar um release**:

```bash
git checkout develop
git pull --rebase origin develop
git checkout -b release/0.2.0
git push -u origin release/0.2.0
```

- 🟢 Workflow **Deploy • staging** dispara — um **release candidate** (`0.2.0-rc`) vai pra staging.
- A partir daqui, **features novas continuam indo pra `develop`** — elas ficam pra próxima release. **Não entram em `release/0.2.0`.**
- Bugs descobertos na release branch **são corrigidos ali mesmo** (e viram back-merge em `develop` depois).

Ver detalhes em [02-release-branch.md](02-release-branch.md).

---

## 4) QA valida em staging, fixes de estabilização entram na release branch

Se QA achar bug **específico da release**:

```bash
git checkout release/0.2.0
git pull --rebase origin release/0.2.0
git checkout -b bugfix/ajusta-validacao
git commit -am "fix(checkout): corrige validação de CEP"
git push -u origin bugfix/ajusta-validacao
# PR bugfix/ajusta-validacao → release/0.2.0 (base = release/0.2.0)
```

- PR aprovado, Squash and merge em `release/0.2.0`.
- 🟢 Novo RC em staging.
- QA revalida.

> ⚠️ **Não corrige em `develop` e espera cherry-pick** — no Git Flow, a release branch é a que está sendo estabilizada; corrigir direto nela mantém o fluxo natural. O back-merge final leva o fix pra `develop` depois.

---

## 5) Fechar release: merge em `main` + tag + back-merge em `develop`

Quando QA aprovar:

```bash
# 1. merge em main (PR release/0.2.0 → main, com MERGE COMMIT, não squash)
gh pr create -B main -H release/0.2.0 \
  --title "release: 0.2.0" \
  --body "Release 0.2.0"
# ... aprovação + merge pela UI (Create a merge commit) ...

# 2. tag em main (localmente, ou via release do GitHub)
git checkout main
git pull --rebase origin main
git tag -a v0.2.0 -m "Release 0.2.0"
git push origin main --tags

# 3. BACK-MERGE: release/0.2.0 → develop (para develop receber os fixes de estabilização)
gh pr create -B develop -H release/0.2.0 \
  --title "chore: back-merge release/0.2.0 → develop" \
  --body "Back-merge dos fixes de estabilização da 0.2.0"
# ... merge pela UI ...

# 4. deleta a release branch
git push origin --delete release/0.2.0
```

- 🔒 Workflow **Deploy • production** pausa esperando aprovação no Environment.
- 🟢 Após aprovação → deploy em prod.
- 🏷️ Tag `v0.2.0` fica visível em **Releases** no GitHub.

> 💡 **Usa merge commit (não squash) nos 2 merges da release.** É essencial: `main` precisa ter o merge commit como "ponto de release" no histórico, e o back-merge em `develop` tem que preservar a linhagem da release branch. Squash quebraria essa rastreabilidade.

---

## 6) Por que tag git é a versão (e não bump no manifest)?

No Git Flow *tradicional* (Driessen), a release branch é onde você edita `package.json` / `pyproject.toml` e faz `chore(release): bump para 0.2.0`. Aqui **pulamos esse passo** porque:

- Zero commit de "chore: bump" poluindo histórico.
- Zero conflito de merge em arquivo de versão entre release branches simultâneas.
- Uma única fonte de verdade: a **tag anotada em `main`**.

A versão em runtime vem de `APP_VERSION` (env var injetada no deploy — `git describe --tags --exact-match` no workflow). Ver [02-release-branch.md](02-release-branch.md#runtime-de-versão).

**Quando você PRECISARIA bumpar manifest:** se o projeto é publicado em PyPI/npm/crates. Não é o caso aqui — este é um serviço deployado por tag.

---

## ✅ Checagem final

| Ambiente      | Branch          | Versão (`git describe`)        |
| ------------- | --------------- | ------------------------------ |
| develop       | `develop`       | `v0.1.0-7-gabc123`             |
| staging (RC)  | `release/0.2.0` | `0.2.0-rc` (resolvido do nome) |
| production    | `main`        | **`v0.2.0`** (tag exata)       |

Depois da release fechada e back-merge feito, `develop` também contém o merge da `release/0.2.0`. Linhagem preservada.

---

## 📅 Cadência

- Features entram em `develop` a qualquer momento.
- Abrir `release/X.Y.Z` em **dia fixo** (ex: quinta de manhã).
- Fechar a release (merge em main + tag) quando QA aprovar (tipicamente mesmo dia ou próximo dia útil).

---

## ➡️ Próximos cenários

- [02 — Release branch em detalhe](02-release-branch.md)
- [03 — Hotfix em produção](03-hotfix.md)
- [05 — Armadilhas comuns e FAQ](05-armadilhas-e-faq.md)
- [06 — Git Flow vs GitLab Flow](06-gitflow-vs-gitlab-flow.md)
- [07 — Rastreamento Notion ↔ GitHub](07-tracking-notion.md)
