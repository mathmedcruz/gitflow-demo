# Guia — Configurando o GitHub para Git Flow

Este guia configura o repositório para **Git Flow profissional**: proteção em `master` e `develop`, environments para gate de production, CODEOWNERS, Conventional Commits no título do PR.

> **Resumo:** `master` só aceita PR vindo de `release/*` ou `hotfix/*`. `develop` aceita PR de `feature/*`, `bugfix/*`. Ninguém pushea direto. Production exige aprovação manual.

---

## 📋 Checklist

- [ ] 1. Criar a branch `develop`
- [ ] 2. Proteger `master` e `develop` (Rulesets)
- [ ] 3. Configurar Environments (`develop`, `staging`, `production`)
- [ ] 4. CODEOWNERS
- [ ] 5. Padronizar título de PR
- [ ] 6. Merge settings (squash pra feature; merge commit pra release/hotfix)
- [ ] 7. Recursos de segurança
- [ ] 8. (Opcional) Proteger tags `v*`

---

## 1) Criar `develop`

```bash
git checkout master
git pull
git checkout -b develop
git push -u origin develop
```

Em **Settings → Branches**, set `develop` como *default branch* (opcional — mas faz sentido, já que é onde o trabalho do dia-a-dia acontece).

---

## 2) Rulesets

**Settings → Rules → Rulesets → New branch ruleset**

### Ruleset #1 — "Master protection"

| Campo | Valor |
| --- | --- |
| **Name** | `Master protection` |
| **Enforcement** | Active |
| **Target branches** | Include by pattern: `master` |

Regras:

- ✅ **Restrict deletions**
- ✅ **Require a pull request before merging**
  - Required approvals: `1`
  - ✅ Dismiss stale approvals
  - ✅ Require review from Code Owners
  - ✅ Require conversation resolution
  - **Allowed merge methods:** apenas **Merge commit** (squash e rebase desmarcados pra master)
- ✅ **Restrict updates** — apenas bypass list pode atualizar diretamente
- ✅ **Require status checks**
  - `CI / Lint & Testes`
  - `PR title lint / Valida título do PR (Conventional Commits)`
- ✅ **Block force pushes**

**Extra (opcional mas útil):** use a UI de rulesets pra exigir que o PR pra master só possa vir de branches que casem com `release/**` ou `hotfix/**`. Isso evita `feature/* → master` acidental. Se o ruleset não suporta essa regra diretamente, cubra com review manual do release manager.

### Ruleset #2 — "Develop protection"

| Campo | Valor |
| --- | --- |
| **Name** | `Develop protection` |
| **Enforcement** | Active |
| **Target branches** | Include by pattern: `develop` |

Regras:

- ✅ **Restrict deletions**
- ✅ **Require a pull request before merging**
  - Required approvals: `1`
  - ✅ Require conversation resolution
  - **Allowed merge methods:** **Squash** (pra feature/bugfix) E **Merge commit** (pra back-merges de release/hotfix). Rebase desmarcado.
- ✅ **Require status checks** — `CI / Lint & Testes` + `PR title lint`
- ✅ **Block force pushes**

### Resumo visual

| Branch     | Exige PR? | Merge method aceito        | Force-push? |
| ---------- | --------- | -------------------------- | ----------- |
| `master`   | ✅        | **Merge commit** apenas    | ❌          |
| `develop`  | ✅        | Squash (feature) + Merge commit (back-merge) | ❌ |
| `release/*`| ❌        | N/A (push direto permitido em bugfixes dela via PR pra ela) | ❌ |
| `hotfix/*` | ❌        | idem                       | ❌          |

---

## 3) Environments

**Settings → Environments → New environment**

| Ambiente      | Required reviewers | Deployment branches                 |
| ------------- | ------------------ | ----------------------------------- |
| `develop`     | —                  | `develop`                           |
| `staging`     | — (ou 1)           | `release/*`                         |
| `production`  | **1 ou 2 pessoas** | `master` (e opcionalmente `refs/tags/v*`) |

Secrets e variáveis por ambiente separados (`DATABASE_URL` de prod não aparece em develop).

---

## 4) CODEOWNERS

Já criamos `.github/CODEOWNERS`. Ajuste os handles pra equipe real.

---

## 5) Título de PR

Já criamos `.github/workflows/pr-lint.yml` que valida o prefixo (`feat`, `fix`, `hotfix`, `release`, `chore`, etc).

**Settings → General → Pull Requests:**

- ✅ **Allow squash merging** — default pra feature/bugfix. Commit message = *Pull request title*.
- ✅ **Allow merge commits** — NECESSÁRIO pra release/hotfix merges e back-merges. Commit message = *Pull request title + commit details*.
- ❌ **Allow rebase merging** — desmarque.
- ✅ **Automatically delete head branches** — limpa feature/bugfix/hotfix branches após merge.

> 💡 O Git Flow *exige* merge commits em release/hotfix pra preservar o "ponto de release" no grafo de `master`. Squash em release quebraria isso.

---

## 6) Política de merge — resumo

| Ação                                       | Via            | Estratégia       |
| ------------------------------------------ | -------------- | ---------------- |
| `feature/* → develop`                      | PR (UI)        | **Squash merge** |
| `bugfix/* → develop`                       | PR (UI)        | **Squash merge** |
| `bugfix/* → release/X.Y.Z` (estabilização) | PR (UI)        | **Squash merge** |
| `release/X.Y.Z → master`                   | PR (UI)        | **Merge commit** |
| `release/X.Y.Z → develop` (back-merge)     | PR (UI)        | **Merge commit** |
| `hotfix/X.Y.Z → master`                    | PR (UI)        | **Merge commit** |
| `hotfix/X.Y.Z → develop` (back-merge)      | PR (UI)        | **Merge commit** |

---

## 7) Recursos de segurança

**Settings → Code security and analysis:**

- ✅ **Dependency graph**
- ✅ **Dependabot alerts** (passivo)
- ✅ **Secret scanning**
- ✅ **Push protection**
- ✅ **Code scanning** (CodeQL)

> ℹ️ Sem `dependabot.yml` aqui — projeto didático não precisa de PRs automáticos de bump. Adicionar quando a dor de dependências desatualizadas aparecer.

---

## 8) Tags (opcional)

**Settings → Rules → Rulesets → New tag ruleset**

- Target tags: `v*`
- ✅ Restrict deletions, ✅ Restrict updates — tags imutáveis.

---

## 🎓 Teste de fumaça

1. Push direto em `master` → **rejeitado**.
2. Push direto em `develop` → **rejeitado**.
3. PR `feature/x → develop` com título inválido → **check falha**.
4. PR `feature/x → develop` válido → squash merge → `deploy-develop` dispara → ambiente `develop` atualizado.
5. Criar `release/0.2.0`, push → `deploy-release` dispara → RC em staging.
6. PR `release/0.2.0 → master` (merge commit) → deploy prod pausa pra aprovação.
7. Tag `v0.2.0` em master → deploy roda com APP_VERSION correto.
8. PR `release/0.2.0 → develop` (back-merge) → develop atualizado.

Se os 8 passos passam, Git Flow está configurado. 🎉

---

## 📎 Referências

- [A successful Git branching model — Vincent Driessen](https://nvie.com/posts/a-successful-git-branching-model/)
- [git-flow CLI extension (opcional)](https://github.com/nvie/gitflow)
- [Rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0/)
