## 📌 O quê

<!-- Resumo curto do que este PR faz. 1–3 frases. -->

## 🤔 Por quê

<!-- Qual o problema ou necessidade que motivou esta mudança? -->

## 📎 Notion

<!--
Use uma "magic word" do Notion + Unique ID do card (preferido):
  Closes GFD-42          → fecha a página no merge
  Fixes GFD-42           → idem (alias)
  Refs GFD-42            → só linka (não fecha)
  Part of GFD-100        → idem

Fallback (quando o database não tem Unique ID ainda):
  Closes: https://www.notion.so/<workspace>/<slug>-<page-id>

Padrão completo: docs/07-tracking-notion.md

Exceções (use label `no-notion` ou pule esta seção):
- PR `release/* → main`, `hotfix/* → main` ou back-merge.
- Mudanças internas de tooling sem card no Notion.
-->

Closes GFD-

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

- [ ] O título do PR começa com `[UNIQUE-ID]` do Notion (ex.: `[GFD-42] Adiciona endpoint /status`)
- [ ] A seção **📎 Notion** tem magic word + ID (ex.: `Closes GFD-42`) — ou URL de fallback, ou label `no-notion`
- [ ] Commits têm trailer `Notion: <UNIQUE-ID>` (ao menos o commit principal após squash)
- [ ] Atualizei `CHANGELOG.md` (se a mudança for relevante pro usuário)
- [ ] Atualizei documentação em `docs/` (se aplicável)
- [ ] CI está verde
