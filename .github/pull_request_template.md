## 📌 O quê

<!-- Resumo curto do que este PR faz. 1–3 frases. -->

## 🤔 Por quê

<!-- Qual o problema ou necessidade que motivou esta mudança? -->

## 📎 Notion

<!--
Cole a URL da página do Notion que originou esta mudança.
Use `Closes:` para que a página seja fechada automaticamente no merge.
Use `Refs:` para apenas linkar (sem fechar).
Padrão completo: docs/07-tracking-notion.md

Exceções (use label `no-notion` ou pule esta seção):
- PR `release/* → main`, `hotfix/* → main` ou back-merge.
- Mudanças internas de tooling sem card no Notion.
-->

Closes: https://www.notion.so/

## 🧪 Como testei

<!-- Passos para reproduzir o teste manual, evidências, screenshots. -->

- [ ] Rodei `pytest` localmente
- [ ] Testei manualmente em dev/develop
- [ ] Screenshots / logs anexados (se aplicável)

## 🚦 Tipo da mudança

- [ ] 🐛 Bugfix (corrige bug em `develop` — PR vai pra `develop`)
- [ ] ✨ Feature (PR `feature/* → develop`)
- [ ] ♻️ Refactor (sem mudança de comportamento)
- [ ] 📝 Docs
- [ ] 🔧 Chore / build / CI
- [ ] 🚀 Release (PR `release/X.Y.Z → main`)
- [ ] 🔥 Hotfix (PR `hotfix/X.Y.Z → main` + back-merge em `develop`)

## 🔄 Destino

- [ ] → `develop` (padrão pra features e fixes)
- [ ] → `main` (release ou hotfix)
- [ ] Preciso do back-merge em `develop` depois (release / hotfix)

## ✅ Checklist

- [ ] O título do PR segue [Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0/)
- [ ] A seção **📎 Notion** tem link (ou o PR tem label `no-notion`)
- [ ] Commits têm trailer `Notion: <url>` (ao menos o commit principal após squash)
- [ ] Atualizei `CHANGELOG.md` (se a mudança for relevante pro usuário)
- [ ] Atualizei documentação em `docs/` (se aplicável)
- [ ] CI está verde
