---
name: create-idea
description: Registra uma nova ideia/melhoria futura no backlog deste repositório (homelab) — ideas.md na raiz (índice bilíngue) e, se a ideia precisar de mais contexto, um arquivo em ideas/. Use quando, durante a implementação ou o uso de algo, surgir uma ideia de melhoria que não será implementada na sessão atual, mas vale registrar para não ser esquecida.
---

# Registrar uma ideia futura (padrão homelab, bilíngue)

Esta skill documenta como registrar uma ideia/melhoria futura no backlog deste repositório, mantendo [`ideas.md`](../../../ideas.md) (índice na raiz) e, quando necessário, um arquivo dedicado em `ideas/` sincronizados.

## Quando usar

Use quando, durante a sessão, surgir uma ideia de melhoria ou funcionalidade que:
- Não será implementada agora, mas vale registrar para não ser esquecida
- Não é um problema/bug a resolver (isso é `known-issues.md`/`known-issues/` — ver skill `create-known-issue`) nem uma decisão de arquitetura já tomada (isso vai no `plan.md`/`config.md` do componente)

## Passo 1: Verifique o contexto já documentado

Antes de registrar, cheque se o(s) componente(s) afetado(s) já têm contexto relevante documentado — não registre a ideia isolada do que o repositório já sabe sobre eles:
- `ia/CLAUDE.md` — convenções ou restrições gerais do projeto (ex: hardware travado em certa versão/branch, decisão de manter nodes neutros) que afetem a ideia?
- `clusters/<nome>/config.md` ou `plan.md` do componente — arquitetura, decisões ou limitações já conhecidas (ex: recursos já compartilhados, capacidade já no limite)?
- [`ideas.md`](../../../ideas.md) — já existe uma ideia relacionada ou sobreposta registrada?

Cite essas restrições/decisões já conhecidas nas "Considerações / perguntas em aberto" do arquivo novo (com link pro arquivo de origem), em vez de propor a ideia sem levar em conta o que já se sabe sobre o componente.

## Passo 2: Decida se a ideia precisa de arquivo próprio

- **Ideia simples** (cabe numa frase, sem hipóteses/arquitetura a detalhar): só adicione a linha na tabela de `ideas.md` — coluna Link fica `—`.
- **Ideia com contexto suficiente** para precisar de mais espaço (motivação, componentes afetados, perguntas em aberto): crie um arquivo em `ideas/<nome_descritivo>.md` e linke na tabela.

## Passo 3: Nome do arquivo (se for criar um)

- Caminho: `ideas/<nome_descritivo>.md`
- `snake_case`, minúsculo, descritivo da ideia (ex: `tailscale_remote_access.md`, `ansible_power_cycle_automation.md`)
- Evite nomes genéricos como `idea1.md`

## Passo 4: Estrutura do arquivo individual (bilíngue)

Mesmo padrão bilíngue usado em `troubleshooting/` e `known-issues/` — conteúdo completo em Português primeiro, depois o mesmo conteúdo completo em Inglês, no mesmo arquivo:

```markdown
# Idea: <título curto em português> / <short title in English>

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: <componentes/apps afetados>
**Origem**: <onde/quando a ideia surgiu — ex: durante a implantação de X, ou referência a outro arquivo>

<Descrição da ideia: o que é, motivação, contexto de por que surgiu>

### Considerações / perguntas em aberto

- <pergunta ou trade-off ainda não decidido>

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: <affected components/apps>
**Origin**: <where/when the idea came up>

<Description of the idea: what it is, motivation, context for why it came up>

### Considerations / open questions

- <question or trade-off not yet decided>
```

## Passo 5: Atualize o índice `ideas.md`

Adicione uma linha na tabela de **ambas** as seções (Português e English) de [`ideas.md`](../../../ideas.md): ideia (resumo curto), componente, data, e link para o arquivo em `ideas/` (ou `—` se for só uma linha sem arquivo próprio).

## Passo 6: Quando a ideia for implementada

A implementação real já gera seus próprios manifests/arquivos/PR — não é necessário manter rastro histórico da ideia no backlog. Remova a linha da tabela de `ideas.md` (nas duas seções) e apague o arquivo em `ideas/`, se houver.

## Regras de conteúdo

- **PT-BR primeiro, depois EN** — mesmo padrão usado no `README.md`, em `troubleshooting/*.md` e em `known-issues/*.md`.
- **Use apenas motivação/contexto real** — não invente justificativas ou detalhes técnicos que não foram de fato mencionados ou observados na sessão.
- **Objetivo**: uma ideia não precisa de uma seção de "lições aprendidas" nem de comandos de diagnóstico (isso é troubleshooting) — foque em descrever a ideia, por que ela surgiu, e o que ainda está em aberto.
- **Termos técnicos mantidos no idioma original** em ambas as versões (nomes de ferramentas, flags, recursos do Kubernetes).
