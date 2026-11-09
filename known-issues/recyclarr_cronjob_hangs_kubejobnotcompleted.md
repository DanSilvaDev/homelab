# Known Issue: CronJob do Recyclarr trava e dispara KubeJobNotCompleted / Recyclarr CronJob hangs and fires KubeJobNotCompleted

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Causa raiz não confirmada (hipótese: mesmo travamento no `git fetch` já observado durante a instalação, agora acontecendo também na execução periódica via CronJob).
**Componente(s)**: Recyclarr, CronJob (`apps/servarr/recyclarr/`), Alertmanager (`KubeJobNotCompleted`).
**Identificado em**: 2026-11-08.
**Relacionado**: [Media Stack - config.md](../clusters/media-stack/config.md) (seção "Seerr + Recyclarr", "Gotcha de diagnóstico (2026-11-07)" — já documenta o `recyclarr list quality-profiles` travando indefinidamente no passo `git fetch` durante a instalação, contornado na época com `git clone` manual via um pod `netshoot`, mesma rede/DNS confirmados funcionando normalmente).

O `CronJob` `recyclarr-sync` (namespace `media`, agendado diariamente às 9h) está disparando o alerta `KubeJobNotCompleted` do Alertmanager — o Job fica rodando por mais de 12 horas sem terminar, sendo necessário interrompê-lo manualmente. Esse comportamento já havia aparecido durante a instalação original do Recyclarr (2026-11-07): o comando `recyclarr list quality-profiles`, rodado manualmente para descobrir os `trash_id`s, travou indefinidamente no passo de `git fetch` dentro de um pod temporário da imagem `ghcr.io/recyclarr/recyclarr:8`, mesmo com conectividade de rede/DNS/egress HTTPS confirmada funcionando normalmente a partir de outro pod no mesmo namespace. Na época, não foi investigado a fundo (baixa prioridade, contornado buscando os `trash_id`s via `git clone` direto em um pod `netshoot`). Agora o mesmo tipo de travamento parece estar afetando a execução periódica real (`recyclarr sync`), não só o comando de diagnóstico manual.

### Hipóteses / o que já foi tentado

- Hipótese principal: o binário git empacotado na imagem `ghcr.io/recyclarr/recyclarr:8` trava ao buscar o repositório de templates/guias TRaSH (mesmo mecanismo já visto no `git fetch` travado durante a instalação) — ainda não confirmado que é exatamente a mesma causa, só uma suspeita forte baseada no precedente.
- Confirmado que o `CronJob` (`apps/servarr/recyclarr/cronjob.yaml`) **não define `activeDeadlineSeconds`** nem no `CronJob` nem no `Job`/`Pod` — não há timeout configurado, então um travamento no `git fetch` (ou em qualquer outro passo) pode deixar o Job rodando indefinidamente até ser interrompido manualmente, o que por si só já explica por que o Job consegue passar de 12h.

### Diagnóstico sugerido (ainda não executado)

- Antes do pod do Job travado ser limpo/reciclado, coletar `kubectl logs -n media -l job-name=<nome-do-job-travado>` para confirmar se o travamento acontece no mesmo ponto (`git fetch`) ou em outro passo do `recyclarr sync`.
- Testar reproduzir o travamento manualmente (`kubectl run` com a mesma imagem `ghcr.io/recyclarr/recyclarr:8`, mesmo comando `sync`) para confirmar se é determinístico/sempre reproduzível ou intermitente.
- Adicionar `activeDeadlineSeconds` ao `CronJob`/`Job` como mitigação independente da causa raiz — mesmo que não resolva o travamento em si, evita o Job ficar preso por 12h+ e reduz o ruído do alerta `KubeJobNotCompleted` (o Job falharia rápido e seria re-tentado no próximo agendamento, em vez de ficar pendurado).
- Avaliar se existe uma forma de evitar depender do `git fetch` em tempo de execução (ex: vendorizar/cachear os templates do TRaSH Guides usados, já que os perfis não mudam com frequência) — só faz sentido se a causa raiz for de fato confirmada como relacionada ao git.

---

## English

**Status**: 🔴 Root cause not confirmed (hypothesis: the same `git fetch` hang already observed during installation, now also happening during the periodic CronJob run).
**Component(s)**: Recyclarr, CronJob (`apps/servarr/recyclarr/`), Alertmanager (`KubeJobNotCompleted`).
**Identified on**: 2026-11-08.
**Related**: [Media Stack - config.md](../clusters/media-stack/config.md) ("Seerr + Recyclarr" section, "Gotcha de diagnóstico (2026-11-07)" — already documents `recyclarr list quality-profiles` hanging indefinitely at the `git fetch` step during installation, worked around at the time with a manual `git clone` inside a `netshoot` pod, with network/DNS confirmed working normally).

The `recyclarr-sync` `CronJob` (namespace `media`, scheduled daily at 9am) is firing Alertmanager's `KubeJobNotCompleted` alert — the Job keeps running for more than 12 hours without finishing, requiring a manual interruption. This same behavior already showed up during the original Recyclarr installation (2026-11-07): the `recyclarr list quality-profiles` command, run manually to discover the `trash_id`s, hung indefinitely at the `git fetch` step inside a temporary pod of the `ghcr.io/recyclarr/recyclarr:8` image, even with network/DNS/HTTPS egress confirmed working normally from another pod in the same namespace. At the time it wasn't investigated further (low priority, worked around by fetching the `trash_id`s via a direct `git clone` in a `netshoot` pod). Now the same kind of hang appears to be affecting the actual periodic run (`recyclarr sync`), not just the manual diagnostic command.

### Hypotheses / what's already been tried

- Main hypothesis: the git binary bundled in the `ghcr.io/recyclarr/recyclarr:8` image hangs while fetching the TRaSH Guides templates repo (same mechanism already seen in the hung `git fetch` during installation) — not yet confirmed this is exactly the same cause, just a strong suspicion based on the precedent.
- Confirmed the `CronJob` (`apps/servarr/recyclarr/cronjob.yaml`) **does not set `activeDeadlineSeconds`** on either the `CronJob` or the `Job`/`Pod` — there's no configured timeout, so a hang in `git fetch` (or any other step) can leave the Job running indefinitely until manually interrupted, which by itself already explains why the Job can run past 12h.

### Suggested diagnostics (not yet run)

- Before the stuck Job's pod gets cleaned up/recycled, collect `kubectl logs -n media -l job-name=<stuck-job-name>` to confirm whether the hang happens at the same point (`git fetch`) or somewhere else in `recyclarr sync`.
- Try reproducing the hang manually (`kubectl run` with the same `ghcr.io/recyclarr/recyclarr:8` image, same `sync` command) to confirm whether it's deterministic/always reproducible or intermittent.
- Add `activeDeadlineSeconds` to the `CronJob`/`Job` as a mitigation independent of the root cause — even if it doesn't fix the hang itself, it prevents the Job from being stuck for 12h+ and reduces `KubeJobNotCompleted` alert noise (the Job would fail fast and retry on the next schedule instead of hanging).
- Evaluate whether there's a way to avoid depending on `git fetch` at runtime (e.g. vendoring/caching the TRaSH Guides templates used, since the profiles don't change often) — only makes sense if the root cause is actually confirmed as git-related.
