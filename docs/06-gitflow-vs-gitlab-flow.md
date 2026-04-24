# Git Flow vs GitLab Flow — quando usar cada um

Os dois modelos resolvem o mesmo problema — **como organizar o trabalho do time entre desenvolvimento, validação e produção** — mas com filosofias diferentes. Esse doc compara os dois lado a lado pra te ajudar a escolher.

---

## 🎯 Resumo em 1 parágrafo

- **Git Flow (2010, Vincent Driessen)** — pensado pra produtos com **releases planejadas**, múltiplas versões suportadas em paralelo, e time de QA dedicado. Usa branches permanentes (`main`, `develop`) + branches de fluxo (`release/*`, `hotfix/*`). Merges pra tudo.
- **GitLab Flow (2014, GitLab)** — pensado pra produtos com **deploy contínuo** ou quase contínuo, poucas versões em paralelo. Usa `main` + branches de ambiente (`staging`, `production`). Cherry-pick pra hotfix.

---

## 📊 Comparação direta

| Aspecto                      | Git Flow                                            | GitLab Flow (env branches)                          |
| ---------------------------- | --------------------------------------------------- | --------------------------------------------------- |
| **Branches permanentes**     | 2 (`main`, `develop`)                             | 3 (`main`, `staging`, `production`)                 |
| **Onde features entram**     | `develop`                                           | `main`                                              |
| **Produção está em**         | `main` (com tag)                                  | `production` (com tag)                              |
| **Staging está em**          | `release/X.Y.Z` (temporária) → deploy em staging    | branch `staging` (permanente)                       |
| **Hotfix: operação git**     | Merge em `main` + merge em `develop` (back-merge) | Cherry-pick pra `staging` e `production`            |
| **Número de PRs no hotfix**  | 2-3 (main, develop, release aberta se houver)     | 1 (pra `main`) + 2 cherry-picks sem PR              |
| **Release branch**           | Sim — `release/X.Y.Z` congela escopo                | Não — `main` é promovido quando pronto              |
| **Back-merge obrigatório?**  | Sim — release → develop, hotfix → develop           | Não — upstream-first faz o trabalho naturalmente    |
| **Conflitos no dia-a-dia**   | Concentrados no back-merge                          | Concentrados no cherry-pick                         |
| **Histórico em `main`/`production`** | Merge commits (visível cada release como "bolha") | Fast-forward / cherry-pick de commits individuais |
| **Complexidade de fluxo**    | Mais alta (5 tipos de branch, 2 back-merges típicos) | Mais baixa (3 branches, operações diretas)         |
| **Tempo até prod**           | Mais lento (release branch estabiliza por dias)     | Mais rápido (main → staging → production)           |
| **Suporte a versões paralelas** | Bom (`support/X.Y.Z` no Git Flow estendido)       | Ruim — um canal só                                  |
| **Adequado pra trunk-based** | Não (filosoficamente oposto)                        | Quase (se você encurtar staging)                    |

---

## ✅ Use Git Flow quando…

- Seu produto tem **releases versionadas** (ex: `v1.0`, `v2.0`) que o cliente percebe — software empacotado, SaaS com API versionada, firmware, bibliotecas.
- Você **suporta múltiplas versões em produção simultaneamente** (cliente A na `v1.x`, cliente B na `v2.x`).
- Tem **time de QA dedicado** que valida release candidates em staging por dias.
- A próxima release tem escopo **planejado com antecedência** (sprint planning, roadmap trimestral).
- Ciclo de release é **semanal ou mais longo**.

**Produtos típicos:** Firefox, bibliotecas open-source, ERP, produto SaaS vendido com SLAs de "nova versão a cada N meses".

---

## ✅ Use GitLab Flow quando…

- Seu produto faz **deploy contínuo** ou quase — múltiplos deploys por semana, idealmente por dia.
- Só uma versão vive em prod por vez — **não** há "cliente A na v1, cliente B na v2".
- QA é mais **smoke tests automatizados + observabilidade** do que validação manual em staging.
- Time é menor ou o processo prioriza **velocidade de entrega** sobre cerimônia.
- Você quer **cherry-pick cirúrgico** pra hotfix em vez de gerenciar back-merges.

**Produtos típicos:** SaaS modernos (Notion, Linear, GitHub), produtos internos, micro-serviços, APIs em evolução contínua.

---

## ⚖️ Os tradeoffs principais

### Git Flow: segurança > velocidade

Release branch é uma **janela de estabilização**: `develop` pode avançar pra próxima versão sem travar a validação da atual. Preço: cerimônia (2 merges no release, 2 no hotfix), fluxo mais lento, e o back-merge vira o ponto mais crítico — **esquecer o back-merge é o bug clássico do Git Flow**.

### GitLab Flow: velocidade > cerimônia

Cherry-pick é cirúrgico e rápido. Sem release branch, sem back-merge. Preço: menos flexibilidade pra escopo congelado (se alguma feature está meio pronta em `main`, você adia a promoção inteira ou faz revert), e cherry-pick em env branches muito atrasadas **conflita**.

---

## 🔄 Como migrar entre os modelos

### De Git Flow pra GitLab Flow

1. Consolide `develop` em `main` — GitLab Flow usa um único branch de integração.
2. Crie `staging` e `production` a partir de `main`.
3. Pare de criar `release/*` — promova `main → staging → production` via `git merge --no-ff` local.
4. Troque `hotfix/* → main + develop` por `hotfix/* → main` com cherry-pick.
5. Atualize proteções de branch (ruleset novo).

Ver o projeto irmão [gitlab-flow-demo](../../gitlab-flow-demo) pra como fica o setup final.

### De GitLab Flow pra Git Flow

1. Renomeie `main → develop` (ou mantenha `main` e adicione `develop` — mas idealmente um só fluxo entra).
2. Crie `main` a partir do último commit em `production` (ou da última tag).
3. Deprecate a branch `staging` — passa a usar `release/*` como staging.
4. Deprecate a branch `production` — `main` vira o que era `production`.
5. Treine o time pra "release branch + back-merge" — é o hábito novo.

Raro migrar nessa direção (normalmente é o contrário), mas possível.

---

## 🧪 Qual é "melhor"?

Nenhum é melhor em absoluto — são ferramentas pra problemas diferentes. A pergunta certa é:

1. **Quantas versões diferentes podem estar em prod ao mesmo tempo?** >1 → Git Flow. Apenas 1 → GitLab Flow.
2. **Com que frequência sai release?** Menor que semanal → Git Flow. Semanal ou mais → GitLab Flow.
3. **Tem time de QA que valida RC por dias?** Sim → Git Flow. Não / smoke tests automatizados → GitLab Flow.
4. **O cliente percebe "versão"?** Sim → Git Flow. Não (SaaS) → GitLab Flow.

Se 3 ou mais respostas puxam pra um lado, esse é o modelo certo. Meio-a-meio significa que você pode escolher o mais simples (GitLab Flow) e subir complexidade se aparecer dor.

---

## 📎 Referências

- [Vincent Driessen — A successful Git branching model (Git Flow, 2010)](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitLab Flow — docs oficiais](https://docs.gitlab.com/ee/topics/gitlab_flow.html)
- [GitHub Flow](https://docs.github.com/en/get-started/using-github/github-flow) — terceira alternativa, ainda mais simples
- [Trunk-based development](https://trunkbaseddevelopment.com/) — quarta alternativa, extremamente simples, exige feature flags pesados
