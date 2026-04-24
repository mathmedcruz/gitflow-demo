# Cenário 3 — Hotfix em produção 🔥

**Situação:** `master` está em `v0.2.0`, bug crítico reportado em prod, precisa consertar **agora**. `develop` já tem features da próxima release (0.3.0) em andamento — você **não** quer arrastá-las pra prod.

Filosofia: **hotfix parte de `master`** (não de `develop`), corrige, volta pra `master` com tag patch, e **obrigatoriamente** faz back-merge em `develop`.

```
develop  ●──●──●──●──●──●──●────────────────────●──●──  (F3, F4, F5 continuam)
master   ●──────────────────────────●─────────●──        v0.2.0 ── v0.2.1
                                    │ hotfix  │
                                    └─────────┘
                                              │ back-merge obrigatório
                                              └─► develop
```

**Diferente do GitLab Flow**: aqui não tem cherry-pick. É merge em `master` + merge em `develop` (2 merges distintos). O fluxo é mais parecido com o de release branch — só que mais rápido e feito sob pressão.

---

## 1) Hotfix branch parte de `master`

> ⚠️ **Sempre de `master`**, nunca de `develop`. Se sair de `develop`, o fix vai junto com features incompletas que **não estão em produção** — é o cenário clássico de "fix acidentalmente trouxe feature meio pronta pra prod".

```bash
git checkout master
git pull --rebase origin master
git checkout -b hotfix/0.2.1
```

**Nomenclatura:** `hotfix/X.Y.Z` (a próxima patch da tag atual em master — se master está em `v0.2.0`, hotfix é `0.2.1`).

Correção mínima:

```bash
git commit -am "fix(app): corrige payload do endpoint /version"
git push -u origin hotfix/0.2.1
```

---

## 2) PR `hotfix/0.2.1 → master` — review acelerado, **merge commit**

- Review acelerado (1 aprovador + CI verde).
- **Merge commit** (NÃO squash) — pelo mesmo motivo da release branch: preservar o ponto do hotfix no histórico.
- 🔒 Deploy em production pausa pra aprovação.

---

## 3) Tag em `master`

```bash
git checkout master
git pull --rebase origin master
git tag -a v0.2.1 -m "Hotfix 0.2.1 — corrige /version"
git push origin --tags
```

- 🟢 Deploy em production com `APP_VERSION=v0.2.1`.

---

## 4) Back-merge `hotfix/0.2.1 → develop` (OBRIGATÓRIO)

Sem esse passo, o fix só existe em `master`. Na próxima release (`release/0.3.0`), o bug volta — porque `develop` não tem a correção.

```bash
gh pr create -B develop -H hotfix/0.2.1 \
  --title "chore: back-merge hotfix/0.2.1 → develop" \
  --body "Back-merge do hotfix 0.2.1 pra develop."
# ... merge pela UI (merge commit) ...
```

⚠️ **Pode dar conflito** se as features de `develop` (F3, F4, F5) tocaram o mesmo código do fix. Resolve manualmente — a intenção é que o fix "valha" também em `develop` convivendo com as features. Se ficar ambíguo, valida com quem escreveu as features.

---

## 5) Release branch aberta? Back-merge também pra ela.

Se no momento do hotfix existir uma `release/0.3.0` em estabilização, você **também** precisa back-mergear o hotfix nela:

```
develop            ●──●──●──●──●──●──hotfix──●──
master             ●─────────────────●───●─────── v0.2.0 ── v0.2.1
                                     │   │
                                     hotfix (PR → master + tag + back-merge pra develop)
                                                        │
release/0.3.0      ●──●──●──●──●──────────────────merge─┘
                                                     (também recebe o hotfix)
```

Senão a `release/0.3.0` vai pra prod como `v0.3.0` **sem o hotfix** — regressão clássica. Checklist:

- [ ] PR hotfix → master + tag
- [ ] PR hotfix → develop
- [ ] PR hotfix → release/0.3.0 (se existir release branch aberta)

---

## 6) Limpeza

```bash
git push origin --delete hotfix/0.2.1
git branch -d hotfix/0.2.1
```

---

## Estado final

| Ambiente      | Branch           | Versão                 | Tem o fix? |
| ------------- | ---------------- | ---------------------- | ---------- |
| develop       | `develop`        | `v0.2.0-7-g...`        | ✅ (back-merge)    |
| staging (RC)  | `release/0.3.0`  | `0.3.0-rc`             | ✅ (back-merge — se existir) |
| production    | `master`         | **`v0.2.1`** (tag)     | ✅ (merge + tag) |

---

## ❗ Anti-padrões

- ❌ **Hotfix partindo de `develop`** — arrasta features incompletas pra prod.
- ❌ **Squash merge no PR do hotfix** — perde o merge commit como "ponto de hotfix" no histórico.
- ❌ **Esquecer o back-merge em `develop`** — bug volta na próxima release. É **a armadilha #1** do Git Flow (ver [docs/05-armadilhas](05-armadilhas-e-faq.md)).
- ❌ **Esquecer o back-merge em `release/*` aberta** — deploy da próxima release reintroduz o bug em prod.
- ❌ **Hotfix mexendo em schema de banco** — idem GitLab Flow: migration destrutiva + back-merge conflituoso = desastre. Se precisa mudar schema, abra uma release.

---

## 🆘 Conflito no back-merge?

Esperado se `develop` divergiu bastante. Resolve no PR:

```bash
git checkout hotfix/0.2.1
git pull origin develop     # traz develop pra local
# ... resolve conflitos ...
git commit
git push
# PR é atualizado automaticamente
```

Conflito grande = `develop` ficou muito à frente de `master` desde a última release. Prevenção: cadência curta de releases.

Próximo doc: [04 — Configuração GitHub](04-configuracao-github.md).
