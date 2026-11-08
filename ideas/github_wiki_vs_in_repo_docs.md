# Idea: Avaliar GitHub Wiki vs. manter a documentação no próprio repositório / Evaluate GitHub Wiki vs. keeping docs in the repository

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — avaliação pendente, não decidida.
**Componente(s) relacionado(s)**: toda a documentação do repositório (`README.md`, `ia/`, `clusters/*/config.md`, `troubleshooting/`, `known-issues/`, `ideas/`).
**Origem**: proposta pelo usuário em 2026-11-08, logo após fechar a PR que criou a estrutura atual de `troubleshooting.md`/`known-issues.md`/`ideas.md`.

Validar se vale a pena migrar a documentação pra **GitHub Wiki** em vez de manter a estrutura atual dentro do próprio repositório (arquivos `.md` versionados junto com o código).

### Considerações / perguntas em aberto

- **GitHub Wiki é um repositório Git separado** (`<repo>.wiki.git`), fora do histórico do repositório principal. Isso quebraria o padrão já em uso nesta sessão: a correção do bug de DNS e o troubleshooting doc que a documenta foram commitados e revisados **na mesma PR** — no Wiki, a doc ficaria desacoplada da mudança de código que ela descreve, sem aparecer no diff/review do PR.
- **Toda a arquitetura de contexto pra agentes de IA deste repo pressupõe leitura via filesystem no mesmo clone** — `ia/CLAUDE.md` é o ponto de entrada ("comece por `ia/CLAUDE.md`"), e os três índices (`troubleshooting.md`, `known-issues.md`, `ideas.md`) se cruzam por link relativo (`../known-issues.md`, etc.) e dependem de `git log`/`git blame`/`git show` pra datas e histórico. Um agente rodando numa sessão com o repo clonado **não enxerga o Wiki automaticamente** (é outro repositório Git) — migrar pra lá quebraria esse ponto de entrada, a menos que alguma ferramenta/acesso extra seja configurado.
- **Vantagens reais do Wiki a pesar**: interface de navegação/busca mais amigável, histórico de página próprio — mas o GitHub já renderiza `.md` normalmente pela UI do repo, e o "poluir o histórico de commits com docs" não é um custo real dado o quão baratos são commits de documentação.
- Como os docs aqui são fortemente acoplados a mudanças de código específicas (`clusters/*/config.md` rastreia estado real de infra, troubleshooting docs referenciam manifests/linhas específicas), manter tudo no mesmo repo/PR que o código que descrevem provavelmente vale mais do que os ganhos de navegação do Wiki — mas essa é a decisão que essa ideia existe pra validar, não pra assumir de antemão.

---

## English

**Status**: Idea — evaluation pending, not decided.
**Related component(s)**: all documentation in the repository (`README.md`, `ia/`, `clusters/*/config.md`, `troubleshooting/`, `known-issues/`, `ideas/`).
**Origin**: proposed by the user on 2026-11-08, right after closing the PR that created the current `troubleshooting.md`/`known-issues.md`/`ideas.md` structure.

Validate whether it's worth migrating documentation to **GitHub Wiki** instead of keeping the current structure inside the repository itself (versioned `.md` files alongside the code).

### Considerations / open questions

- **GitHub Wiki is a separate Git repository** (`<repo>.wiki.git`), outside the main repository's history. This would break the pattern already in use this session: the DNS bug fix and the troubleshooting doc documenting it were committed and reviewed **in the same PR** — in the Wiki, the doc would be decoupled from the code change it describes, not showing up in the PR's diff/review.
- **This repo's entire AI-agent context architecture assumes filesystem reads from the same clone** — `ia/CLAUDE.md` is the entry point ("start with `ia/CLAUDE.md`"), and the three indexes (`troubleshooting.md`, `known-issues.md`, `ideas.md`) cross-reference each other via relative links (`../known-issues.md`, etc.) and rely on `git log`/`git blame`/`git show` for dates and history. An agent running in a session with the repo cloned **doesn't automatically see the Wiki** (it's a different Git repository) — migrating there would break this entry point unless some extra tool/access is set up.
- **Real Wiki advantages to weigh**: friendlier browsing/search UI, its own page history — but GitHub already renders `.md` files fine through the repo's UI, and "cluttering commit history with docs" isn't a real cost given how cheap documentation commits are.
- Since the docs here are tightly coupled to specific code changes (`clusters/*/config.md` tracks real infra state, troubleshooting docs reference specific manifests/lines), keeping everything in the same repo/PR as the code it describes is probably worth more than the Wiki's browsing gains — but that's exactly the decision this idea exists to validate, not to assume upfront.
