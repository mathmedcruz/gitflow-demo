# Armadilhas comuns e FAQ — Git Flow

Pega-ratão específico do Git Flow (que o GitLab Flow não tem) e perguntas do dia-a-dia.

---

## 🪤 Armadilhas comuns

### 1. Esquecer o back-merge em `develop`

**Sintoma:** release (ou hotfix) foi pra main com tag, saiu em prod. Semanas depois, a próxima release reintroduz o bug que você tinha corrigido.

**Consequência:** regressão. Você já consertou isso uma vez — agora precisa consertar de novo, ou entender por que voltou.

**Causa raiz:** o fix só foi pra `main`. `develop` não recebeu. Feature nova em `develop` sobrescreveu o arquivo e o bug voltou.

**Defesa:**
- **Checklist no template de PR de release/hotfix**: `[ ] Back-merge em develop`, `[ ] Back-merge em release/* aberta (se houver)`.
- Automação possível: GitHub Action que, ao fechar PR `release/* → main`, abre automaticamente o PR de back-merge pra `develop`.

---

### 2. Esquecer o back-merge em `release/*` aberta durante hotfix

**Sintoma:** hotfix sai em prod como `v0.2.1`. Próxima release (`v0.3.0`) sai duas semanas depois… e o bug que foi corrigido no hotfix **volta em prod**.

**Consequência:** mesma regressão da #1, mas por um caminho diferente. A `release/0.3.0` já estava aberta quando o hotfix foi feito — ela **não recebeu** o fix do hotfix, só o `develop` recebeu. Quando ela mergeou em main, sobrescreveu o hotfix.

**Defesa:**
- No momento do hotfix, verificar: `git branch -a --list 'origin/release/*'`. Se tiver branch aberta, PR do hotfix **também pra ela**.
- Template de PR de hotfix explicita isso.

---

### 3. Squash merge em `release/* → main`

**Sintoma:** aparentemente funciona. Deploy sai. Tag é criada.

**Consequência (descoberta semanas depois):** o histórico de `main` perdeu o merge commit da release. `git log --graph main` fica linear, sem o "ponto de release" visível. Pior: o back-merge pra `develop` fica estranho — o squash criou um novo SHA que não tem relação com os commits originais da release branch.

**Defesa:**
- No ruleset de `main`, aceite **apenas "Merge commit"** como método. Bloqueia o acidente na raiz.

---

### 4. Release branch aberta virando permanente

**Sintoma:** `release/0.2.0` foi aberta há 3 semanas. Continua aberta. Vão entrando bugfixes, nunca fecha.

**Consequência:** estabilização sem fim. QA cansa. `develop` vai acumulando 0.3.0 worth of features. Eventualmente alguém nota que `develop` está "muito à frente" e o back-merge final vai ser um inferno.

**Defesa:**
- **Time-box** pra release branch: 3-5 dias de estabilização. Se não fecha nesse prazo, algo está errado (escopo grande demais? QA subdimensionado?).
- Se o problema é "bugs demais pra estabilizar", considerar: abortar a release, voltar features problemáticas em `develop`, reabrir release depois.

---

### 5. Feature entrando direto em `main`

**Sintoma:** alguém com permissão push-a feature em `main` ou abre PR `feature/* → main`.

**Consequência:** viola o fluxo. A feature nunca passou por `develop` → `release/*` → QA. Foi direto pra prod.

**Defesa:**
- Ruleset em `main` exigindo PR + Code Owners review.
- Processo de review: release manager recusa PR que não vem de `release/*` ou `hotfix/*`.

---

### 6. Hotfix partindo de `develop`

**Sintoma:** alguém criou `hotfix/0.2.1` a partir de `develop` em vez de `main`.

**Consequência:** o hotfix arrasta features meio prontas de `develop` pra prod. Pode sair em prod código que não passou por staging.

**Defesa:**
- Treinamento: "hotfix SEMPRE de `main`". No template de PR de hotfix: `[ ] Branch saiu de main (confirme: git log --oneline main..HEAD mostra APENAS o commit do fix)`.

---

### 7. Conflito gigante no back-merge

**Sintoma:** PR `release/0.2.0 → develop` (back-merge) tem 50 arquivos em conflito.

**Consequência:** release manager perde horas resolvendo. Algum conflito é resolvido errado — introduz bug em `develop`.

**Causa:** `develop` e `release/0.2.0` divergiram muito. Ou a release foi aberta tarde, ou features estavam entrando em `develop` durante estabilização longa.

**Defesa:**
- Release branches curtas (armadilha #4).
- Se o fix na release for em arquivo "quente" (muito mexido em features de `develop`), considerar: fazer o mesmo fix direto em `develop` agora, pra reduzir divergência no back-merge.

---

## ❓ FAQ

### Por que preciso de `develop`? Por que não só `main` + feature branches?

Porque `main` = "o que está em prod". Se você mergeia features direto em `main`, perde a capacidade de **acumular múltiplas features** pra próxima release sem deploy imediato. `develop` é a área de integração contínua que alimenta a próxima `release/*`.

Se você não precisa dessa separação (seu produto faz deploy contínuo, cada merge vai pra prod), **você não precisa de Git Flow** — GitHub Flow ou trunk-based é mais barato.

---

### Posso usar Git Flow em produto com deploy contínuo?

Pode, mas é fricção. Git Flow foi desenhado pra produtos com **releases planejadas** (software empacotado, SaaS com versionamento, bibliotecas). Se você faz deploy de cada PR que entra em main, o `develop` vira redundante com `main`, e as release branches viram cerimônia sem valor.

Sinais de que Git Flow está "forçando" o fluxo:
- Releases saem a cada 1-2 dias.
- `release/*` branches vivem menos de 2 horas.
- Você não tem staging ou QA dedicado.

Nesses casos, **troque pra GitLab Flow ou GitHub Flow**.

---

### Como lidar com duas releases em paralelo?

Raro no Git Flow, mas possível: `release/0.2.0` (em estabilização) e `release/0.3.0` (já aberta porque o time quer começar a próxima).

Regra: **uma release por vez na prática**. Se realmente precisa de paralelismo, é sintoma de que as releases estão grandes demais.

Se inevitável: hotfix precisa de PR pra `main` + `develop` + `release/0.2.0` + `release/0.3.0` (4 PRs). Complexidade explode rápido.

---

### Posso pular a release branch pra releases pequenas?

Não é o espírito do Git Flow, mas alguns times fazem "mini Git Flow": direto `develop → main` via PR + tag, pulando `release/*`. Isso é essencialmente GitHub Flow / GitLab Flow. Se for seu caminho mais frequente, admita: você não quer Git Flow.

---

### E se descobrir um bug na release branch que **também** existe em main (prod)?

É simultaneamente um **hotfix** (precisa sair em prod) E um **bugfix de estabilização** (precisa entrar na `release/*`).

Ordem recomendada:
1. Trata como hotfix primeiro: `hotfix/0.2.1` de `main`, fix, merge em main + tag.
2. Back-merge em `develop`.
3. Back-merge em `release/0.3.0` (pra garantir que a próxima release não reintroduza).

---

### Quando uso `git cherry-pick` no Git Flow?

Raramente. O Git Flow usa **merges** pra tudo — feature → develop (squash), release → main/develop (merge commit), hotfix → main/develop (merge commit). Cherry-pick aparece apenas em casos excepcionais tipo "preciso levar um commit específico de uma release branch abortada pra outra", e mesmo aí é sinal de desvio do fluxo.

O GitLab Flow, ao contrário, usa cherry-pick como operação primária (hotfix) — é uma diferença de filosofia.

---

### Git Flow exige usar a extensão `git-flow` CLI?

Não. A extensão (`brew install git-flow-avh`) automatiza os comandos (`git flow feature start x`, `git flow release finish 0.2.0`), mas faz exatamente o que um script bash faria. Use se gostar; sem ela, o fluxo é o mesmo com `git checkout -b` / `git merge`.

---

### Por que merge commit (e não squash) em release/hotfix?

Duas razões:

1. **Rastreabilidade em `main`:** `git log --graph main` mostra cada release como um "bolha" do merge, fácil de identificar. Squash linearizaria, perdendo o marco.
2. **Back-merge funciona limpo:** o merge commit tem **dois pais** (release branch + main). O back-merge pra develop usa esses pais pra resolver conflitos de forma previsível. Squash cria um SHA "órfão" sem essa parentalidade.

---

### Qual o resumo de uma linha?

> **Features viram em `develop`. Release branches entregam em `main` com tag e voltam pra `develop`. Hotfix sai de `main`, volta pra `main` com tag e pra `develop` (+ `release/*` aberta se houver). Back-merge é lei.**

Se essa frase parece complexa demais pro seu produto: provavelmente [GitLab Flow](06-gitflow-vs-gitlab-flow.md) serve melhor.
