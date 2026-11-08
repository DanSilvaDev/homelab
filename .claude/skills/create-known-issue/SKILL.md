---
name: create-known-issue
description: Registra um problema real identificado mas ainda não resolvido neste repositório (homelab) — known-issues.md na raiz (índice bilíngue) e, se precisar de mais contexto, um arquivo em known-issues/. Use quando, durante a sessão, você identificar um bug/comportamento incorreto/configuração quebrada que não será resolvido agora, mas precisa ficar registrado para não ser esquecido nem re-investigado do zero depois.
---

# Registrar um problema conhecido, ainda não resolvido (padrão homelab, bilíngue)

Esta skill documenta como registrar um problema ativo no backlog deste repositório, mantendo [`known-issues.md`](../../../known-issues.md) (índice na raiz) e, quando necessário, um arquivo dedicado em `known-issues/` sincronizados.

## Quando usar

Use quando, durante a sessão, você identificar:
- Um sintoma real e observável (erro, comportamento incorreto, serviço instável) que não é esperado
- Que não vai ser resolvido nesta sessão (falta tempo, precisa de mais investigação, não é prioridade agora)

Não use para:
- **Ideias/melhorias** (nada está quebrado, é só algo que poderia ser feito) — isso é `ideas.md`/`ideas/`, skill `create-idea`.
- **Problemas já resolvidos** — isso é `troubleshooting/`, skill `create-troubleshooting-doc`. Se o problema for resolvido ainda nesta sessão, pule direto pra essa skill em vez de passar por `known-issues/`.
- **Decisões de configuração já tomadas** — isso vai no `plan.md`/`config.md` do componente.

## Passo 1: Verifique o contexto já documentado

Antes de registrar, cheque se o(s) componente(s) afetado(s) já têm contexto relevante documentado — não registre o problema isolado do que o repositório já sabe sobre ele:
- [`troubleshooting.md`](../../../troubleshooting.md) — um problema igual ou parecido já foi resolvido antes nesse componente?
- `clusters/<nome>/config.md` ou `plan.md` do componente — restrições, decisões de arquitetura ou limitações já conhecidas que ajudem a explicar o sintoma?
- [`known-issues.md`](../../../known-issues.md) — já existe um problema relacionado registrado (mesmo componente, sintoma parecido)?

Linke o que for relevante no campo "Relacionado" do arquivo novo, e cite restrições/decisões já conhecidas nas hipóteses, em vez de especular do zero.

## Passo 2: Decida se o problema precisa de arquivo próprio

- **Sintoma simples e autoexplicativo** (cabe numa frase, sem hipóteses a registrar ainda): só adicione a linha na tabela de `known-issues.md` — coluna Link fica `—`.
- **Problema com contexto** (hipóteses já descartadas, comandos sugeridos/tentados, diagnóstico parcial): crie um arquivo em `known-issues/<nome_descritivo>.md` e linke na tabela.

## Passo 3: Nome do arquivo (se for criar um)

- Caminho: `known-issues/<nome_descritivo>.md`
- `snake_case`, minúsculo, descritivo do **sintoma observado** — ex: `jellyfin_frequent_pod_restarts.md`, `prowlarr_indexers_failed_to_load_from_api.md`
- **Nomeie sempre pelo sintoma, nunca por um componente suspeito ou causa raiz ainda não confirmada — mesmo que a suspeita seja forte.** Ex: um log não identificado no Loki, suspeito de ser do Pi-hole, vira `loki_unidentified_service_logging_garbage_lines.md` (sintoma), nunca `pihole_logging_issue.md` (componente suspeito ainda não validado) — a suspeita vai no corpo do arquivo ("Hipóteses"), não no nome.
- Evite nomes genéricos como `bug1.md`

## Passo 4: Estrutura do arquivo individual (bilíngue)

Mesmo padrão bilíngue usado em `troubleshooting/` e `ideas/` — conteúdo completo em Português primeiro, depois o mesmo conteúdo completo em Inglês, no mesmo arquivo:

```markdown
# Known Issue: <título curto em português> / <short title in English>

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 <status real e atual — ex: "Em investigação — não resolvido", "Diagnóstico pendente", "Causa raiz não confirmada", "Comandos sugeridos, ainda não executados">
**Componente(s)**: <componentes/apps afetados>
**Identificado em**: <data, ou "não registrado explicitamente" se não houver data confiável>
**Relacionado**: <links para outros known-issues ou troubleshooting docs relacionados, se houver — omita a linha se não houver nenhum>

<Descrição do problema: sintomas observados, contexto de quando apareceu>

### Hipóteses / o que já foi tentado (se houver)

- <hipótese já descartada ou tentativa que não resolveu>

### Diagnóstico sugerido (ainda não executado)

- <próximo passo de investigação sugerido>

---

## English

**Status**: 🔴 <current real status, mirrored>
**Component(s)**: <affected components/apps>
**Identified on**: <date, or "not explicitly recorded" if there's no reliable date>
**Related**: <links to other related known issues or troubleshooting docs, if any — omit the line if none>

<Description of the problem: observed symptoms, context of when it appeared>

### Hypotheses / what's already been tried (if any)

- <already-ruled-out hypothesis or attempt that didn't fix it>

### Suggested diagnostics (not yet run)

- <suggested next investigation step>
```

O template acima é o modo padrão, pra quando a causa raiz **ainda não é conhecida**. Existe um segundo modo, pra quando a causa **já é conhecida/confirmada** (ex: pelo próprio operador) e só falta aplicar a correção — nesse caso, troque as duas últimas seções:

- `### Hipóteses / o que já foi tentado` → `### Causa identificada` (PT) / `### Identified cause` (EN): descreva a causa já confirmada, direto, sem framing de incerteza.
- `### Diagnóstico sugerido (ainda não executado)` → `### Ação sugerida (ainda não aplicada)` (PT) / `### Suggested action (not yet applied)` (EN): descreva o que falta fazer pra resolver (não passos de investigação, já que não há mais o que investigar).
- O `Status` também deve refletir isso — ex: "Causa identificada — ação pendente" em vez de "Causa raiz não confirmada".

Exemplo real desse segundo modo: um alerta do Alertmanager disparando incorretamente pro TrueNAS por causa do ZFS ARC (comportamento esperado, confirmado pelo operador) — a causa já era conhecida, só faltava ajustar o alerta (ver `known-issues/alertmanager_high_memory_alert_truenas.md`).

## Passo 5: Atualize o índice `known-issues.md`

Adicione uma linha na tabela de **ambas** as seções (Português e English) de [`known-issues.md`](../../../known-issues.md): problema (resumo em linguagem de sintoma, não de categoria), componente, tags, data, e link para o arquivo em `known-issues/` (ou `—` se for só uma linha sem arquivo próprio).

## Passo 6: Quando o problema for resolvido

Use a skill `create-troubleshooting-doc` para escrever o post-mortem em `troubleshooting/` — ela já cobre, no próprio "Passo 4", como remover a entrada e o arquivo correspondente de `known-issues.md`/`known-issues/` nesse momento. Não remova o registro daqui antes de o problema estar de fato resolvido.

## Regras de conteúdo

- **PT-BR primeiro, depois EN** — mesmo padrão usado no `README.md`, em `troubleshooting/*.md` e em `ideas/*.md`.
- **Use apenas sintomas e hipóteses reais** — nunca apresente uma causa raiz como confirmada se ela ainda não foi validada. Quando ela já tiver sido validada (pelo operador ou por diagnóstico já feito), use o modo "Causa identificada" do Passo 4 em vez de forçar a linguagem de incerteza do modo padrão.
- **Resumo orientado a sintoma** — tanto no título quanto na linha da tabela, descreva o que se observa (ex: "pod reiniciando com frequência anormal"), não uma categoria abstrata (ex: "problema de recursos").
- **Termos técnicos mantidos no idioma original** em ambas as versões (nomes de comandos, flags, ferramentas, recursos do Kubernetes).
