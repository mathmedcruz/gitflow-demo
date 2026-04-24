# Git Flow Demo 🌿

Projeto didático para demonstrar **Git Flow** (Vincent Driessen, 2010) usando **GitHub Actions** para simular deploys em **develop**, **staging** e **production**.

O foco não é a aplicação em si (um FastAPI bem simples), e sim **como o código atravessa o ritual do Git Flow** — `feature` → `develop` → `release` → `master`, mais hotfixes em emergências — e **como lidar com bugs em cada fase**.

---

## 🎯 Objetivo

1. Entender as **5 famílias de branches** do Git Flow: `master`, `develop`, `feature/*`, `release/*`, `hotfix/*`.
2. Praticar o **caminho feliz**: feature → develop → release → master + tag.
3. Praticar **hotfix** em produção sem perder o fix em develop (back-merge obrigatório).
4. Enxergar os tradeoffs vs. GitLab Flow (tem um doc dedicado).

---

## 🌳 Modelo de branches

```
                           ┌────────────────────────┐
                           │   master (produção)    │ ← tag v0.2.0 vive aqui
                           └──────▲────────▲────────┘
                                  │ merge   │ merge (hotfix)
                                  │ release │
                           ┌──────┴───┐ ┌───┴────────┐
                           │release/*│ │ hotfix/*   │
                           └──────▲───┘ └───▲────────┘
                                  │         │ (parte de master)
                           ┌──────┴─────────┴────────┐
                           │        develop          │ ← deploy contínuo no ambiente develop
                           └──────▲──────────────────┘
                                  │
                           ┌──────┴──────┐
                           │  feature/*  │
                           └─────────────┘
```

| Branch       | Permanente? | Sai de        | Volta pra                 | Deploy?                    |
| ------------ | ----------- | ------------- | ------------------------- | -------------------------- |
| `master`     | ✅ sim      | —             | —                         | **production** (via tag)   |
| `develop`    | ✅ sim      | `master` (1x)| —                         | **develop** (contínuo)     |
| `feature/*`  | ❌ não      | `develop`     | `develop` (PR, squash)    | — (roda CI em PR)          |
| `release/*`  | ❌ não      | `develop`     | `master` + `develop`      | **staging** (RC)           |
| `hotfix/*`   | ❌ não      | `master`      | `master` + `develop`      | **staging** (verificação)  |

**Regras de ouro:**

1. **Features NUNCA mergeiam direto em `master`** — passam por `develop` → `release/*` → `master`.
2. **`master` só recebe merge de `release/*` ou `hotfix/*`** — nada mais.
3. **Todo merge em `master` é taggeado** (`vX.Y.Z`) — a tag é a versão.
4. **Depois de mergear em `master`, sempre back-merge em `develop`** — senão o fix "some" em features futuras.

---

## 🏗️ Estrutura do projeto

```
.
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                    # Testes em PRs e pushes
│   │   ├── pr-lint.yml               # Valida título do PR
│   │   ├── deploy-develop.yml        # Deploy em develop (push develop)
│   │   ├── deploy-release.yml        # Deploy RC em staging (push release/*)
│   │   └── deploy-master.yml         # Deploy em production (push master + tag)
│   ├── ISSUE_TEMPLATE/
│   ├── CODEOWNERS
│   └── pull_request_template.md
├── src/main.py                       # App FastAPI minimalista
├── tests/test_main.py                # Testes (pytest + TestClient)
├── docs/
│   ├── 01-fluxo-normal.md            # feature → develop → release → master
│   ├── 02-release-branch.md          # Como abrir, estabilizar e fechar release/X.Y.Z
│   ├── 03-hotfix.md                  # Hotfix em master + back-merge obrigatório
│   ├── 04-configuracao-github.md     # Rulesets para master e develop
│   ├── 05-armadilhas-e-faq.md        # Pega-ratão + comparação com GitLab Flow
│   ├── 06-gitflow-vs-gitlab-flow.md  # Comparação direta
│   └── 07-tracking-notion.md         # Padrão de commits/branches/PRs para Notion
├── CHANGELOG.md
├── requirements.txt
├── requirements-dev.txt
└── README.md
```

> **Versão = tag git.** Não há `pyproject.toml` com campo `version`. Diferente do Git Flow *tradicional* (que bumpa manifest na release branch), aqui a tag anotada em `master` é a única fonte de verdade — ver [docs/02-release-branch.md](docs/02-release-branch.md) pra detalhes.

---

## 🧪 Rodando localmente

```bash
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt

pytest                               # roda os testes
APP_ENV=local uvicorn src.main:app --port 3000 --reload   # http://localhost:3000
```

Endpoints:

- `GET /` — mensagem + ambiente
- `GET /health` — health check
- `GET /version` — versão (lê `APP_VERSION` da env, fallback `"dev"`)

---

## ⚙️ Configuração do repositório no GitHub

Passo a passo completo (rulesets para `master` e `develop`, environments, CODEOWNERS…) está em **[docs/04-configuracao-github.md](docs/04-configuracao-github.md)**.

Versão curta:

1. Criar a branch `develop` a partir de `master`.
2. Em **Settings → Rules → Rulesets**, criar:
   - **"Master protection"** → target `master` → exige PR vindo de `release/*` ou `hotfix/*`, CI verde, Code Owners, block force-push, restrict delete.
   - **"Develop protection"** → target `develop` → exige PR, CI verde, block force-push, restrict delete.
3. Em **Settings → Environments**, criar `develop`, `staging`, `production` — marcar *Required reviewers* em `production`.
4. Em **Settings → General**, habilitar **squash merging** (default pra features) e **merge commits** (pra release/hotfix, que precisam preservar merge commit por causa do back-merge).

---

## 📚 Cenários (leia nesta ordem)

1. **[Fluxo normal — feature → develop → release → master](docs/01-fluxo-normal.md)**
2. **[Release branch: estabilização + merge em master](docs/02-release-branch.md)**
3. **[Hotfix: bug em produção + back-merge](docs/03-hotfix.md)**
4. **[Configuração profissional do GitHub](docs/04-configuracao-github.md)**
5. **[Armadilhas comuns e FAQ](docs/05-armadilhas-e-faq.md)**
6. **[Git Flow vs GitLab Flow — quando usar cada um](docs/06-gitflow-vs-gitlab-flow.md)**
7. **[Rastreamento Notion ↔ GitHub](docs/07-tracking-notion.md)** — padrão de commits, branches e PRs

---

## 🧠 As 4 regras do Git Flow

1. **`master` é "o que está em produção".** Só recebe merge de `release/*` ou `hotfix/*`, e cada merge vira uma tag.
2. **`develop` é "o próximo release".** Feature branches partem daqui e voltam pra cá via PR.
3. **`release/*` estabiliza** — depois que abriu, só fixes de estabilização; features novas esperam na `develop`.
4. **`hotfix/*` é a única exceção** ao "tudo passa por develop" — sai direto de `master`, volta em `master` (+ tag) **e** em `develop` (back-merge).

### Resumo de uma linha

> **Feature parte de `develop`. Release estabiliza a partir de `develop`, entrega em `master` com tag, e volta pra `develop`. Hotfix parte de `master`, fecha em `master` com tag, e volta pra `develop`. Back-merge é lei.**

---

## 📝 Licença

MIT — veja `LICENSE` (ou adicione um, se for publicar).
