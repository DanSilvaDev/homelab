# Known Issue: CronJob do Recyclarr trava e dispara KubeJobNotCompleted / Recyclarr CronJob hangs and fires KubeJobNotCompleted

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔵 Bloqueado por upstream — causa raiz confirmada: o Recyclarr v8 tenta buscar uma branch `v8` que não existe mais no repositório oficial de templates, e entra num loop infinito de retry em vez de falhar ou cair pro fallback documentado.
**Componente(s)**: Recyclarr, CronJob (`apps/servarr/recyclarr/`), Alertmanager (`KubeJobNotCompleted`).
**Identificado em**: 2026-11-08. Causa raiz confirmada em 2026-11-09.
**Relacionado**: [Media Stack - config.md](../clusters/media-stack/config.md) (seção "Seerr + Recyclarr", "Gotcha de diagnóstico (2026-11-07)" — o mesmo travamento já havia sido visto na instalação original, contornado na época sem investigação profunda).

O `CronJob` `recyclarr-sync` (namespace `media`, agendado diariamente às 9h) disparava o alerta `KubeJobNotCompleted` do Alertmanager — o Job ficava rodando por 12h+ sem terminar. Investigação profunda (2026-11-09) confirmou a causa raiz exata.

### O que foi confirmado

- O pod travado tinha, nos logs, a última linha parada em `[INF] Initializing provider: official (type: config-templates)`.
- Inspeção via `/proc` (sem depender de `ps`/`netstat`, indisponíveis na imagem) mostrou os processos `git fetch --depth 1 https://github.com/recyclarr/config-templates.git v8` e seus helpers (`git-remote-https`) vivos há 15h, em estado `S (sleeping)`, com **`/proc/net/tcp` completamente vazio** — nenhuma conexão TCP aberta, nunca.
- Testes manuais de rede/DNS/HTTPS pro GitHub (via `wget`), feitos no mesmo pod/momento, responderam instantaneamente — **rede, DNS e TLS não são o problema**.
- Reprodução controlada de um `git fetch --depth 1 ... v8` isolado (fora do Recyclarr, direto via shell) **não travou** — falhou rápido e limpo (`fatal: couldn't find remote ref v8`), confirmando via API do GitHub (`api.github.com/repos/recyclarr/config-templates/branches`) que a branch **`v8` genuinamente não existe** no repositório `recyclarr/config-templates` (só `master` + branches de feature/issue, zero tags).
- A documentação oficial do Recyclarr afirma que o provider `config-templates` tenta `v{major}` (`v8` pro Recyclarr 8.x) e **cai pro `master`/`main` se não existir**. Na prática, **esse fallback não acontece**: testado um override explícito (`resource_providers` customizado com `reference: master` e `replace_default: true`, conforme documentado), e o Recyclarr **continuou disparando `git fetch ... v8`**, ignorando completamente a configuração.
- O log mostrava `[WRN] Deleting local git repo and retrying git operation due to error` — ou seja, o Recyclarr detecta o erro, mas **tenta de novo a mesma operação que sempre vai falhar**, indefinidamente (confirmado: múltiplas gerações de processos `git fetch ... v8` acumuladas no mesmo pod, uma por tentativa).

**Conclusão**: bug genuíno no Recyclarr v8 — o provider oficial `config-templates` está hardcoded/quebrado pra tentar uma branch que não existe mais upstream, o fallback documentado pra `master`/`main` não funciona na prática, e a lógica de retry re-tenta a operação falha pra sempre em vez de desistir ou cair pro fallback. Não é configurável/corrigível pelo nosso lado — testamos a correção documentada (`reference` explícito) e ela não teve efeito algum.

### Mitigação aplicada

Adicionados `activeDeadlineSeconds: 600` (10min) ao `Job` e `concurrencyPolicy: Forbid` ao `CronJob`, em `apps/servarr/recyclarr/cronjob.yaml` — não corrige o bug, mas evita o Job ficar preso por 12h+ (interrompido automaticamente após 10min, failing fast) e evita acumular Jobs concorrentes travados uns sobre os outros.

O Job travado original (`recyclarr-sync-29902320`, preso há 15h+ com processos órfãos acumulados dos testes de diagnóstico) foi removido manualmente via `kubectl delete job -n media recyclarr-sync-29902320` (2026-11-09).

### Tentativa de downgrade pra v7 (não concluída)

Avaliado downgrade pra `ghcr.io/recyclarr/recyclarr:7` (`7.5.2`) — a v7 usa a branch `master` pros templates (não a lógica quebrada de `v{major}` introduzida na v8), então teoricamente evitaria o bug acima. Porém:
- O schema de config da v7 é bem diferente (usa `include: - template: <nome>` em vez de `quality_profiles[].trash_id`/`custom_format_groups`) — exigiria reescrever a config do zero, nomes de template não documentados claramente, precisariam ser descobertos via `recyclarr config list templates` dentro da própria imagem v7.
- Ao testar exatamente esse comando (`recyclarr config list templates`) na v7, bateu num problema **diferente e também real**: erro de TLS ao clonar o repositório completo (`error: RPC failed; curl 56 OpenSSL SSL_read: ... decryption failed or bad record mac`, `fatal: early EOF`), reproduzido duas vezes seguidas (mesmo padrão de retry automático visto na v8: `[WRN] Deleting local git repo and retrying`). Hipótese não testada: possível problema de MTU na rede overlay (Flannel/VXLAN) exposto só em transferências maiores (a v7 faz um clone completo; a v8 fazia só um fetch raso de um único ref, por isso nunca expôs esse sintoma).
- **Decisão**: não seguir por esse caminho por agora — reabriria uma investigação de rede separada (MTU/VXLAN) sem garantia de resolver, e o schema da v7 precisaria ser reconstruído do zero mesmo se a rede funcionasse. Mantido como anotação pra não repetir o mesmo caminho sem necessidade numa investigação futura.

### Acompanhamento upstream

- **Projeto**: Recyclarr (`recyclarr/recyclarr` no GitHub — o bug é no app, não no repositório `recyclarr/config-templates` em si, que só está com a branch `v8` ausente/removida).
- **Onde checar**: [issues do recyclarr/recyclarr filtradas por "config-templates branch"](https://github.com/recyclarr/recyclarr/issues?q=is%3Aissue+config-templates+branch) — nenhuma issue específica confirmada ainda (pesquisa não encontrou correspondência exata), mas o bug é fundamental o suficiente (afeta qualquer usuário do Recyclarr 8.x com config padrão) que é provável que já exista ou venha a existir um relato.
- **O que procurar**: uma versão em que o provider `config-templates` resolva corretamente pra `master` quando `v{major}` não existir (sem precisar de override manual), ou que pelo menos falhe rápido em vez de entrar em loop de retry infinito.
- **Versão em uso neste repo**: `ghcr.io/recyclarr/recyclarr:8` (`apps/servarr/recyclarr/cronjob.yaml`).
- **Última checagem**: 2026-11-09.

---

## English

**Status**: 🔵 Blocked on upstream — root cause confirmed: Recyclarr v8 tries to fetch a `v8` branch that no longer exists in the official templates repository, and enters an infinite retry loop instead of failing or falling back as documented.
**Component(s)**: Recyclarr, CronJob (`apps/servarr/recyclarr/`), Alertmanager (`KubeJobNotCompleted`).
**Identified on**: 2026-11-08. Root cause confirmed on 2026-11-09.
**Related**: [Media Stack - config.md](../clusters/media-stack/config.md) ("Seerr + Recyclarr" section, "Gotcha de diagnóstico (2026-11-07)" — the same hang had already been seen during the original installation, worked around at the time without deep investigation).

The `recyclarr-sync` `CronJob` (namespace `media`, scheduled daily at 9am) was firing Alertmanager's `KubeJobNotCompleted` alert — the Job stayed running for 12h+ without finishing. An in-depth investigation (2026-11-09) confirmed the exact root cause.

### What was confirmed

- The stuck pod's logs stopped at `[INF] Initializing provider: official (type: config-templates)`.
- Inspecting via `/proc` (no `ps`/`netstat` available in the image) showed `git fetch --depth 1 https://github.com/recyclarr/config-templates.git v8` and its helpers (`git-remote-https`) alive for 15h, in `S (sleeping)` state, with **`/proc/net/tcp` completely empty** — no TCP connection open, ever.
- Manual network/DNS/HTTPS tests to GitHub (via `wget`), run in the same pod at the same time, responded instantly — **network, DNS, and TLS are not the problem**.
- A controlled, isolated reproduction of `git fetch --depth 1 ... v8` (outside Recyclarr, directly via shell) **didn't hang** — it failed fast and cleanly (`fatal: couldn't find remote ref v8`), confirmed via the GitHub API (`api.github.com/repos/recyclarr/config-templates/branches`) that the **`v8` branch genuinely doesn't exist** in the `recyclarr/config-templates` repository (only `master` + a few feature/issue branches, zero tags).
- Recyclarr's official docs state the `config-templates` provider tries `v{major}` (`v8` for Recyclarr 8.x) and **falls back to `master`/`main` if it doesn't exist**. In practice, **that fallback doesn't happen**: an explicit override was tested (a custom `resource_providers` entry with `reference: master` and `replace_default: true`, per the docs), and Recyclarr **kept firing `git fetch ... v8`**, completely ignoring the configuration.
- The log showed `[WRN] Deleting local git repo and retrying git operation due to error` — i.e. Recyclarr detects the error but **retries the exact same operation that will always fail**, indefinitely (confirmed: multiple generations of `git fetch ... v8` processes accumulated in the same pod, one per attempt).

**Conclusion**: a genuine bug in Recyclarr v8 — the official `config-templates` provider is hardcoded/broken to try a branch that no longer exists upstream, the documented fallback to `master`/`main` doesn't work in practice, and the retry logic retries the failing operation forever instead of giving up or falling back. Not configurable/fixable on our side — the documented fix (explicit `reference`) was tested and had zero effect.

### Mitigation applied

Added `activeDeadlineSeconds: 600` (10min) to the `Job` and `concurrencyPolicy: Forbid` to the `CronJob`, in `apps/servarr/recyclarr/cronjob.yaml` — doesn't fix the bug, but prevents the Job from being stuck for 12h+ (automatically terminated after 10min, failing fast) and prevents stuck Jobs from piling up concurrently.

The original stuck Job (`recyclarr-sync-29902320`, stuck for 15h+ with orphaned processes accumulated from diagnostic testing) was manually removed via `kubectl delete job -n media recyclarr-sync-29902320` (2026-11-09).

### Downgrade attempt to v7 (not completed)

Evaluated downgrading to `ghcr.io/recyclarr/recyclarr:7` (`7.5.2`) — v7 uses the `master` branch for templates (not the broken `v{major}` logic introduced in v8), so it would theoretically avoid the bug above. However:
- v7's config schema is quite different (uses `include: - template: <name>` instead of `quality_profiles[].trash_id`/`custom_format_groups`) — would require rewriting the config from scratch, with template names not clearly documented, needing discovery via `recyclarr config list templates` inside the v7 image itself.
- Testing exactly that command (`recyclarr config list templates`) on v7 hit a **different, also real** problem: a TLS error cloning the full repository (`error: RPC failed; curl 56 OpenSSL SSL_read: ... decryption failed or bad record mac`, `fatal: early EOF`), reproduced twice in a row (same automatic-retry pattern seen on v8: `[WRN] Deleting local git repo and retrying`). Untested hypothesis: a possible MTU issue on the overlay network (Flannel/VXLAN) only exposed by larger transfers (v7 does a full clone; v8 only did a shallow fetch of a single ref, so it never exposed this symptom).
- **Decision**: not pursuing this path for now — it would open a separate networking investigation (MTU/VXLAN) with no guarantee of resolution, and v7's schema would still need to be rebuilt from scratch even if networking worked. Kept as a note to avoid repeating the same path unnecessarily in a future investigation.

### Upstream tracking

- **Project**: Recyclarr (`recyclarr/recyclarr` on GitHub — the bug is in the app, not in the `recyclarr/config-templates` repository itself, which is just missing/has removed the `v8` branch).
- **Where to check**: [recyclarr/recyclarr issues filtered by "config-templates branch"](https://github.com/recyclarr/recyclarr/issues?q=is%3Aissue+config-templates+branch) — no specific matching issue confirmed yet (research found no exact match), but the bug is fundamental enough (affects any Recyclarr 8.x user with default config) that an existing or future report is likely.
- **What to look for**: a version where the `config-templates` provider correctly resolves to `master` when `v{major}` doesn't exist (without needing a manual override), or at least fails fast instead of entering an infinite retry loop.
- **Version in use in this repo**: `ghcr.io/recyclarr/recyclarr:8` (`apps/servarr/recyclarr/cronjob.yaml`).
- **Last checked**: 2026-11-09.
