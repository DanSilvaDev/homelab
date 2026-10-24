---
name: create-pull-request
description: Cria um Pull Request no GitHub para este repositório (homelab), seguindo o template em .github/PULL_REQUEST_TEMPLATE.md, com descrição bilíngue (Português BR e Inglês), objetiva e fácil de entender. Use quando o usuário pedir para abrir/criar um PR, ou quando um conjunto de mudanças estiver pronto para revisão e precisar ser proposto via PR.
---

# Criar Pull Request (padrão homelab)

Esta skill documenta como abrir um Pull Request neste repositório usando o `gh` CLI, preenchendo a descrição no formato bilíngue exigido pelo template do projeto.

## Pré-requisitos antes de criar o PR

1. Confirme o estado do branch atual:
   ```bash
   git status
   git diff
   git log --oneline origin/main..HEAD
   ```
2. Garanta que o trabalho está commitado (nunca abra PR com mudanças não commitadas relevantes).
3. Se o branch ainda não foi enviado ao remoto, faça o push primeiro:
   ```bash
   git push -u origin <nome-do-branch>
   ```
4. Revise se algum `clusters/*/config.md` ou documento de troubleshooting precisa ser atualizado antes de abrir o PR (ver skill `create-troubleshooting-doc` se aplicável).

## Passo 1: Ler o template do projeto

Sempre baseie a descrição do PR no template existente em `.github/PULL_REQUEST_TEMPLATE.md`. Não invente uma estrutura diferente — leia o arquivo antes de montar o corpo do PR, para garantir que novas seções adicionadas ao template no futuro também sejam respeitadas.

## Passo 2: Montar a descrição (bilíngue, objetiva)

Regras de conteúdo, obrigatórias:
- **Duas seções de descrição**: `## Descrição (PT-BR)` e `## Description (EN)`, ambas presentes, com o mesmo conteúdo semântico (não precisa ser tradução palavra-por-palavra, mas deve comunicar a mesma coisa).
- **Objetiva e direta**: frases curtas, foco no "o quê" e "por quê" da mudança, sem narrar o processo de investigação (isso fica no troubleshooting doc, se houver um).
- **Sem jargão desnecessário**: escreva para alguém que vai revisar rapidamente, não para quem acompanhou a sessão inteira.
- Preencha também as seções do template: tipo de mudança, componentes afetados, como foi testado, e o checklist.

## Passo 3: Criar o PR via `gh`

Use um heredoc para o corpo, preservando a formatação Markdown, e termine com as linhas de atribuição exigidas pela sessão (se houver uma instrução de atribuição ativa no momento, ela tem precedência sobre este template):

```bash
gh pr create \
  --title "<título curto e descritivo, até ~70 caracteres>" \
  --body "$(cat <<'EOF'
## Descrição (PT-BR)

<resumo objetivo da mudança e motivação>

## Description (EN)

<objective summary of the change and motivation>

---

## Tipo de mudança / Type of change

- [x] <marcar o(s) item(ns) aplicável(is)>

## Componentes afetados / Affected components

<ex: worker2, kube-vip, pihole>

## Como foi testado / How was this tested

<comandos/validações realizadas>

## Checklist

- [ ] clusters/*/config.md atualizado, se aplicável / updated, if applicable
- [ ] Documento de troubleshooting criado, se aplicável / Troubleshooting doc created, if applicable
- [ ] Sem segredos/credenciais commitados / No secrets/credentials committed
EOF
)"
```

## Passo 4: Retornar a URL do PR

Após a criação, devolva ao usuário a URL retornada pelo `gh pr create` — não anuncie "PR criado" sem o link.

## O que NÃO fazer

- Não abrir PR sem antes confirmar com o usuário (seguindo as regras gerais de ações que afetam estado compartilhado/visível a terceiros).
- Não usar `--title`/`--body` genéricos tipo "Update files" — o título e a descrição devem refletir a mudança real.
- Não omitir uma das duas línguas na descrição.
- Não pular a leitura do template atual antes de montar o corpo — ele pode ter mudado desde a última vez que esta skill foi usada.
