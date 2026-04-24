# Cenário 2 — Release branch em detalhe 📦

A **release branch** é o coração do Git Flow. É onde a release **existe como entidade** enquanto é estabilizada — separada do fluxo de features novas que continua em `develop`.

## Anatomia de uma release branch

```
develop  ─●──●──●──●──●──●───────────────●──────●──●──  (features continuam entrando)
                        │                │       └─ volta aqui (back-merge)
                        │                │
                        └─► release/0.2.0 ─► main + tag v0.2.0
                                 │               (via PR com merge commit)
                                 └─ fixes de estabilização aqui
```

Três momentos:

1. **Abertura** — corta de `develop` quando o escopo da release está definido.
2. **Estabilização** — só bugfixes entram, deploy em staging via workflow.
3. **Fechamento** — dois merges: em `main` (+ tag) e back-merge em `develop`.

---

## 1) Quando abrir?

Abre quando **todas as features da próxima versão estão em `develop`** e você quer parar de aceitar escopo novo nessa release. Critérios típicos:

- Ciclo de 2 semanas acabou.
- Sprint fechou, product owner aprovou o escopo.
- Acumulou X features + Y bugfixes desde a última tag.

**Não abra release enquanto features críticas estão meio prontas** — ou segura a release, ou revert as features incompletas em `develop` antes.

---

## 2) Nomenclatura: `release/X.Y.Z`

```bash
git checkout develop
git pull --rebase origin develop
git checkout -b release/0.2.0
git push -u origin release/0.2.0
```

A versão está **no nome da branch**. O workflow [`deploy-release.yml`](../.github/workflows/deploy-release.yml) extrai `0.2.0` de `release/0.2.0` via `${GITHUB_REF_NAME#release/}` e deploya como `0.2.0-rc` em staging.

**SemVer pelas Conventional Commits desde a última tag em `main`:**

- só `fix:` / `refactor:` / `chore:` → **PATCH** (`release/0.2.1`)
- algum `feat:` → **MINOR** (`release/0.3.0`)
- breaking change → **MAJOR** (`release/1.0.0`)

---

## 3) Enquanto a release branch existe: duas regras

### Regra 1: `develop` continua recebendo features

A partir do momento que `release/0.2.0` existe, **features novas entram em `develop` normalmente**, mas ficam pra próxima release (`0.3.0`). Isso é o grande ganho do Git Flow — você pode estabilizar 0.2.0 **sem bloquear** o desenvolvimento de 0.3.0.

### Regra 2: só **bugfixes de estabilização** entram em `release/0.2.0`

Fixes descobertos em staging durante a validação da RC:

```bash
git checkout release/0.2.0
git pull --rebase origin release/0.2.0
git checkout -b bugfix/ajusta-validacao-cep
# ... correção + commit ...
git push -u origin bugfix/ajusta-validacao-cep
# PR bugfix/ajusta-validacao-cep → release/0.2.0 (base = release/0.2.0!)
```

**NÃO faça:**

- ❌ Cherry-pick de commits de `develop` pra `release/0.2.0` — se a feature estabilizada não estava no escopo, ela vai junto por engano.
- ❌ Merge `develop → release/0.2.0` — idem, trouxe coisa nova pra uma release que deveria estar congelando.

Se uma feature **tem** que entrar na release tardiamente (raro), documente no PR do back-merge e justifique — mas o padrão é: escopo congela quando a branch nasce.

---

## 4) Deploy da release branch (staging)

A cada push em `release/0.2.0`, o workflow [`deploy-release.yml`](../.github/workflows/deploy-release.yml) publica um **release candidate** em staging:

```
staging: 0.2.0-rc (commit abc1234)
```

QA valida a RC. Bug encontrado → bugfix PR → merge em `release/0.2.0` → novo RC → revalida.

---

## 5) Fechamento: os dois merges (e é aqui que o Git Flow "doi")

### Merge 1: `release/0.2.0 → main` (com **merge commit**, não squash)

```bash
gh pr create -B main -H release/0.2.0 \
  --title "release: 0.2.0" \
  --body "Release 0.2.0 — resumo das mudanças"
```

- Aprovação + merge pela UI com **"Create a merge commit"** (NÃO squash).
- O merge commit em `main` marca o "ponto de release" — é o que a tag vai referenciar.
- 🔒 Deploy em production pausa pra aprovação no Environment.

### Taggear

```bash
git checkout main
git pull --rebase origin main
git tag -a v0.2.0 -m "Release 0.2.0"
git push origin --tags
```

O workflow [`deploy-main.yml`](../.github/workflows/deploy-main.yml) dispara no push (ou na criação da tag) e resolve `APP_VERSION` via `git describe --tags --exact-match`.

### Merge 2: back-merge `release/0.2.0 → develop`

**OBRIGATÓRIO.** Sem isso, todos os fixes de estabilização aplicados em `release/0.2.0` **não existem em `develop`**. Features futuras vão reintroduzir os bugs.

```bash
gh pr create -B develop -H release/0.2.0 \
  --title "chore: back-merge release/0.2.0 → develop" \
  --body "Back-merge dos fixes de estabilização."
# merge pela UI (merge commit, não squash)
```

### Deletar a release branch

```bash
git push origin --delete release/0.2.0
git branch -d release/0.2.0
```

---

## 6) Runtime de versão

Esse projeto não usa bump de manifest — a tag é a versão. Injeção em runtime:

**Build-time (recomendado):**

```dockerfile
FROM python:3.12-slim
ARG VERSION=dev
ENV APP_VERSION=$VERSION
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

No workflow (`deploy-main.yml`):

```yaml
- id: v
  run: echo "version=$(git describe --tags --exact-match)" >> "$GITHUB_OUTPUT"
- run: docker build --build-arg VERSION=${{ steps.v.outputs.version }} -t myapp:${{ steps.v.outputs.version }} .
```

No código:

```python
# src/main.py
APP_VERSION = os.getenv("APP_VERSION", "dev")

@app.get("/version")
def version():
    return {"version": APP_VERSION}
```

Para RC em staging (`release/*`), o workflow injeta `APP_VERSION=0.2.0-rc` — distinguível de `v0.2.0` da tag em main.

---

## 7) Resumo visual

```
1. git checkout -b release/0.2.0 (corta de develop)
2. Deploy RC em staging automaticamente
3. QA → bug? → bugfix/* → PR pra release/0.2.0 → novo RC
4. QA aprova
5. PR release/0.2.0 → main (merge commit)
6. git tag -a v0.2.0 → push --tags (deploy prod)
7. PR release/0.2.0 → develop (back-merge)
8. Delete release/0.2.0
```

Demora? Sim, mais que GitLab Flow. Por quê? Porque a release branch permite **desenvolvimento paralelo** da próxima versão em `develop` sem travar a estabilização atual. Esse é o valor.

Próximo cenário: [03 — Hotfix](03-hotfix.md).
