# Known Issue: Instabilidade generalizada na stack `*arr` / General instability in the `*arr` stack

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Não isolado — causa ainda não encontrada.
**Componente(s)**: Prowlarr, Sonarr, Radarr, Lidarr, LazyLibrarian, Bazarr (media stack), node compartilhado.
**Identificado em**: 2026-11-07.
**Relacionado**: [prowlarr_indexers_failed_to_load_from_api.md](prowlarr_indexers_failed_to_load_from_api.md), [jellyfin_frequent_pod_restarts.md](jellyfin_frequent_pod_restarts.md).

Instabilidade generalizada na stack `*arr` (Prowlarr/Sonarr/Radarr/Lidarr/LazyLibrarian/Bazarr) — páginas web ficando indisponíveis com frequência, mas **sem os pods mostrarem restart** (`kubectl get pods -n media` com `RESTARTS` inalterado durante o problema). Isso descarta `OOMKilled`/`CrashLoopBackOff` como causa direta, e aponta mais pra lentidão/timeout (CPU throttling, contenção de recursos no node, ou o `Service`/LoadBalancer ficando momentaneamente inacessível) do que pra crash de fato.

**Confirmado que o problema já acontecia antes da implantação do FlareSolverr** — não é causado por ele (descarta a hipótese de contenção de recursos introduzida pelo Chromium headless como origem), embora ele possa estar agravando algo que já existia, já que roda no mesmo node compartilhado.

Pode estar relacionado a:
- CPU dos 6 deployments já precisou de ajuste uma vez na Fase 5 por instabilidade.
- O erro "Failed to Load Indexers from API" ([prowlarr_indexers_failed_to_load_from_api.md](prowlarr_indexers_failed_to_load_from_api.md)) sugere problema de rede/egress no pod do Prowlarr.
- Ou pode ser algo novo, ainda não isolado.

### Diagnóstico sugerido (ainda não executado)

- `kubectl top pods -n media` e `kubectl top nodes` (contenção de recursos?).
- `kubectl get events -n media --sort-by='.lastTimestamp'` (throttling, `FailedScheduling`, eventos de rede).
- `kubectl describe pod` nos apps afetados durante um episódio de indisponibilidade (liveness/readiness falhando sem reiniciar o container?).
- Checar se os pods afetados compartilham o mesmo node (sintoma de node sobrecarregado) ou estão espalhados (sintoma mais de rede/storage compartilhado, ex: latência do `truenas-nfs-ssd`).

---

## English

**Status**: 🔴 Not isolated — cause not yet found.
**Component(s)**: Prowlarr, Sonarr, Radarr, Lidarr, LazyLibrarian, Bazarr (media stack), shared node.
**Identified on**: 2026-11-07.
**Related**: [prowlarr_indexers_failed_to_load_from_api.md](prowlarr_indexers_failed_to_load_from_api.md), [jellyfin_frequent_pod_restarts.md](jellyfin_frequent_pod_restarts.md).

General instability across the `*arr` stack (Prowlarr/Sonarr/Radarr/Lidarr/LazyLibrarian/Bazarr) — web pages becoming unavailable frequently, but **with no pod showing a restart** (`kubectl get pods -n media` shows `RESTARTS` unchanged during the issue). This rules out `OOMKilled`/`CrashLoopBackOff` as the direct cause, pointing more toward slowness/timeouts (CPU throttling, node resource contention, or the `Service`/LoadBalancer becoming momentarily unreachable) than an actual crash.

**Confirmed that the problem already happened before FlareSolverr was deployed** — it's not caused by it (rules out headless-Chromium-induced resource contention as the origin), though FlareSolverr may be aggravating something that already existed, since it runs on the same shared node.

May be related to:
- The 6 deployments' CPU already needed one adjustment during Phase 5 due to instability.
- The "Failed to Load Indexers from API" error ([prowlarr_indexers_failed_to_load_from_api.md](prowlarr_indexers_failed_to_load_from_api.md)) suggests a network/egress problem on the Prowlarr pod.
- Or it could be something new, not yet isolated.

### Suggested diagnostics (not yet run)

- `kubectl top pods -n media` and `kubectl top nodes` (resource contention?).
- `kubectl get events -n media --sort-by='.lastTimestamp'` (throttling, `FailedScheduling`, network events).
- `kubectl describe pod` on the affected apps during an unavailability episode (liveness/readiness failing without restarting the container?).
- Check whether the affected pods share the same node (symptom of an overloaded node) or are spread out (more a symptom of shared network/storage, e.g. `truenas-nfs-ssd` latency).
