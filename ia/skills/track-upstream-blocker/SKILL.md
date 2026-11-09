# Rastrear um known-issue bloqueado por correção upstream (padrão homelab)

Esta skill documenta como marcar um [known-issue](../../../known-issues.md) já diagnosticado, mas cuja correção depende de um projeto/ferramenta de terceiros (upstream) ainda não ter lançado um fix — e como deixar um ponto de acompanhamento fácil de achar, pra não depender só de alguém lembrar de checar manualmente.

## Quando usar

Use quando, durante a investigação de um [known-issue](../../../known-issues.md), você (o agente) chegar a uma causa raiz **confirmada ou fortemente evidenciada**, mas a correção **não está ao alcance deste repositório** — depende de uma mudança no código/binário de um projeto de terceiros (ex: um bug num app como Jellyfin, Pi-hole, um chart Helm, uma imagem Docker).

Não use para:
- Problemas ainda em investigação, sem causa raiz confirmada — isso continua sendo o status 🔴 normal da skill `create-known-issue`.
- Problemas que são nossos (configuração, manifests, infra deste repo) só ainda não corrigidos por falta de tempo/prioridade — isso também continua 🔴, não é "bloqueado por upstream".
- Problemas já resolvidos — isso é `troubleshooting/`, skill `create-troubleshooting-doc`.

## Passo 1: Verifique o contexto já documentado

Antes de aplicar, confirme que o known-issue já existe em `known-issues/<nome>.md` (se ainda não existir, crie primeiro via skill `create-known-issue`) e releia o que já foi escrito lá — essa skill **atualiza** o arquivo existente, não cria um novo.

## Passo 2: Atualize o `Status` do known-issue

Troque o `**Status**` de 🔴 para **🔵 Bloqueado por upstream** (nas duas seções, Português e English). O texto do status deve deixar claro, em uma frase: o que já foi confirmado como causa raiz, e por que a correção não é algo que este repositório pode aplicar.

Exemplo:
```markdown
**Status**: 🔵 Bloqueado por upstream — causa raiz confirmada, mas é um bug no próprio <app>, sem correção disponível na versão atual.
```

## Passo 3: Substitua "Hipóteses"/"Diagnóstico sugerido" pela seção "Acompanhamento upstream"

Nesse status, as seções padrão de `create-known-issue` ("Hipóteses / o que já foi tentado", "Diagnóstico sugerido") deixam de fazer sentido — a investigação terminou, o que resta é vigiar uma correção externa. Substitua por esta seção, na mesma posição (depois da descrição do problema):

```markdown
### Acompanhamento upstream

- **Projeto**: <nome do projeto/ferramenta>
- **Onde checar**: <link direto e específico — issue do GitHub, changelog, release notes. Nunca a página genérica do projeto>
- **O que procurar**: <critério objetivo pra saber se foi resolvido — ex: "changelog menciona fix pro crash de X", "issue fechada como resolved/completed". Nunca uma expectativa vaga tipo "ver se melhorou">
- **Versão em uso neste repo**: <imagem/tag atual, ex: `jellyfin/jellyfin:12.4`>
- **Última checagem**: <data>
```

Mesma estrutura, mesmo conteúdo (sem tradução palavra-por-palavra, mas mesma informação) na seção **English**, sob "### Upstream tracking".

## Passo 4: Ofereça (não assuma) uma checagem periódica automatizada

Pergunte ao usuário se ele quer uma rotina automatizada de verificação, em vez de depender de revisão manual. Se ele aceitar, use a skill `/schedule` deste Claude Code pra criar uma rotina (ex: mensal) que:
1. Abre o link de "Onde checar".
2. Compara contra o critério de "O que procurar".
3. Se nada mudou: atualiza o campo "Última checagem" no known-issue com a data (sem precisar de aprovação extra — é só atualizar uma data, nenhuma mudança de estado real).
4. Se algo relevante mudou: avisa o usuário diretamente, não decide sozinho se o problema está resolvido.

Se o usuário não quiser automação, deixe só o known-issue como está — ele já é o ponto de acompanhamento manual (linkado em [`known-issues.md`](../../../known-issues.md), fácil de achar).

## Passo 5: Quando o upstream corrigir

Se uma checagem (manual ou automatizada) confirmar que a correção saiu:
1. **Valide no próprio homelab antes de comemorar** — atualize a imagem/versão pra que inclui o fix, reproduza o cenário que antes quebrava, confirme que realmente parou.
2. Se confirmado: use a skill `create-troubleshooting-doc` pra promover o known-issue a `troubleshooting/` — deixe explícito no documento que a causa raiz era um bug upstream e linke a correção real de lá (PR/release/changelog), não invente comandos de "correção" que não existem neste repo (a "correção" aqui é só atualizar a versão/imagem).
3. Se o fix não resolver completamente (parcial, ou introduziu outro problema): atualize a seção "Acompanhamento upstream" com o novo estado e continue rastreando — não promova a known-issue a resolvido prematuramente.

## Regras de conteúdo

- **PT-BR primeiro, depois EN** — mesmo padrão usado em `troubleshooting/*.md`, `known-issues/*.md` e `ideas/*.md`.
- **"Onde checar" e "O que procurar" precisam ser específicos e verificáveis** — um link genérico pro repositório do projeto ou um critério vago tornam a checagem inútil (ninguém sabe quando parar de olhar).
- **Esse status é pra bugs genuinamente fora do nosso controle** — não use como desculpa pra adiar um problema que na real é nosso.
- **Nunca marque como resolvido sem validar no homelab** — "upstream lançou uma correção" não é o mesmo que "está corrigido aqui"; sempre teste antes de promover pra `troubleshooting/`.
