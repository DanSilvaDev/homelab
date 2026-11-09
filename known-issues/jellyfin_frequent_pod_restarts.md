# Known Issue: Pod do Jellyfin reiniciando com frequência anormal / Jellyfin pod restarting with abnormal frequency

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔵 Bloqueado por upstream — causa raiz provável identificada (crash em torno de reconexões de WebSocket), mas aparenta ser um bug no próprio Jellyfin, sem correção disponível do nosso lado.
**Componente(s)**: Jellyfin (media stack).
**Identificado em**: 2026-11-04 (16 restarts em 35 horas). Investigado a fundo em 2026-11-08/09.
**Relacionado**: [troubleshooting/jellyfin_subtitle_permission_denied_nfs_root.md](../troubleshooting/jellyfin_subtitle_permission_denied_nfs_root.md) — um problema real e distinto encontrado durante esta investigação (Jellyfin não conseguia ler legendas `.srt` via NFS rodando como root) **já foi corrigido** e não é mais a causa dos restarts atuais.

Pod do Jellyfin reiniciando com frequência anormal (exit code `139` = SIGSEGV). Diagnóstico aprofundado em 2026-11-08/09 descartou GPU, contenção de recursos e o bug de permissão NFS (resolvido à parte), e isolou o padrão a uma causa bem mais específica: **o crash acontece em torno de reconexões de WebSocket** (ex: ao atualizar a aba do navegador com o dashboard do Jellyfin aberto), sem nenhum stack trace ou log de erro — consistente com um crash nativo/fail-fast do runtime .NET, não uma exception tratada normalmente.

### Hipóteses / o que já foi tentado

- **Contenção de recursos**: descartada — `kubectl top pod`/`kubectl top nodes` mostraram uso baixo (CPU/memória) no momento dos crashes.
- **Falha de GPU/hwaccel**: descartada — um crash reproduzido teve a sondagem completa de hwaccel do ffmpeg (`Available hwaccel types: [cuda, vaapi, ...]`) terminando com sucesso, com o crash acontecendo só depois, durante uma conexão WebSocket.
- **Bug de permissão NFS** (`.srt` externos ilegíveis rodando como root): era real, mas **já corrigido** (ver [troubleshooting/jellyfin_subtitle_permission_denied_nfs_root.md](../troubleshooting/jellyfin_subtitle_permission_denied_nfs_root.md)) e confirmado que não é a causa dos restarts que continuaram depois do fix.
- **Causa mais provável, não corrigível por aqui**: pesquisa encontrou múltiplos relatos na comunidade do Jellyfin (fórum oficial) de um crash de processo inteiro causado por `System.ObjectDisposedException` na rotina de keepalive do WebSocket (`SessionWebSocketListener.SendForceKeepAlive`/`KeepAliveSockets`), tentando enviar um keepalive pra uma conexão que o cliente já fechou — um padrão clássico de race condition entre o fechamento da conexão e a thread de keepalive. Relatado em várias versões ao longo do tempo, sem fix confirmado encontrado na pesquisa. Reproduzido aqui: log mostra `WS ... closed` seguido ~375ms depois por `WS ... request` (reconexão rápida, ex: refresh de aba), e então silêncio total — sem exception logada, consistente com um crash que mata o processo inteiro antes do .NET conseguir registrar algo.

### Mitigação aplicada (não resolve a causa raiz)

Adicionado `readinessProbe` (`tcpSocket` na porta `8096`) ao `apps/jellyfin/deployment.yaml` — não evita o crash, mas garante que o `Service` pare de rotear tráfego pro pod durante os poucos segundos de restart, em vez de deixar requisições caírem num pod morto/subindo.

### Acompanhamento upstream

- **Projeto**: Jellyfin (`jellyfin/jellyfin` no GitHub).
- **Onde checar**: [issues do jellyfin/jellyfin filtradas por "websocket crash keepalive"](https://github.com/jellyfin/jellyfin/issues?q=is%3Aissue+websocket+crash+keepalive) — não existe um número de issue único confirmado como correspondência exata (nosso crash nunca gerou uma stack trace capturada, o processo morre em silêncio total; a hipótese de `ObjectDisposedException`/`SendForceKeepAlive` vem de padrão recorrente em relatos no fórum oficial, não de um log nosso). A issue mais próxima estruturalmente encontrada até agora é [#12090](https://github.com/jellyfin/jellyfin/issues/12090) ("crash if client 'disappears' without completing the close handshake"), fechada em 2025-04-29 e corrigida pelos PRs #13457/#13459 — mas o tipo de exception lá (`NullReferenceException` em `WebSocketController.GetActiveSockets`) **não bate** com a hipótese inicial, então não é tratada como confirmada.
- **O que procurar**: uma issue fechada cuja stack trace combine com um crash de processo inteiro (sem exception .NET logada) disparado por reconexão rápida de WebSocket (ex: refresh de aba do dashboard) — e confirmar se o PR que a fecha já está incluído na tag de imagem em uso neste repo.
- **Versão em uso neste repo**: `jellyfin/jellyfin:12.4` (`apps/jellyfin/deployment.yaml`).
- **Última checagem**: 2026-11-09.

### Diagnóstico sugerido (próximos passos, se o problema persistir)

- Testar outra tag de imagem (mais nova ou mais antiga) pra ver se o bug é específico da versão atual (`12.4`).
- Se for crítico, capturar um core dump do crash pra identificar a stack trace nativa/exception exata (não tentado ainda, exige setup adicional) — resolveria a incerteza sobre qual issue upstream corresponde.

---

## English

**Status**: 🔵 Blocked on upstream — likely root cause identified (crash around WebSocket reconnections), but appears to be a bug in Jellyfin itself, no fix available on our side.
**Component(s)**: Jellyfin (media stack).
**Identified on**: 2026-11-04 (16 restarts in 35 hours). Investigated in depth on 2026-11-08/09.
**Related**: [troubleshooting/jellyfin_subtitle_permission_denied_nfs_root.md](../troubleshooting/jellyfin_subtitle_permission_denied_nfs_root.md) — a real, distinct problem found during this investigation (Jellyfin couldn't read `.srt` subtitles over NFS while running as root) has **already been fixed** and is no longer the cause of the current restarts.

The Jellyfin pod restarts with abnormal frequency (exit code `139` = SIGSEGV). An in-depth diagnosis on 2026-11-08/09 ruled out GPU, resource contention, and the NFS permission bug (fixed separately), and narrowed the pattern down to something much more specific: **the crash happens around WebSocket reconnections** (e.g. refreshing the browser tab with the Jellyfin dashboard open), with no stack trace or error log at all — consistent with a native/fail-fast .NET runtime crash, not a normally-handled exception.

### Hypotheses / what's already been tried

- **Resource contention**: ruled out — `kubectl top pod`/`kubectl top nodes` showed low CPU/memory usage at crash time.
- **GPU/hwaccel failure**: ruled out — one reproduced crash had the full ffmpeg hwaccel probe (`Available hwaccel types: [cuda, vaapi, ...]`) finish successfully, with the crash happening afterward, during a WebSocket connection.
- **NFS permission bug** (external `.srt` unreadable while running as root): was real, but **already fixed** (see [troubleshooting/jellyfin_subtitle_permission_denied_nfs_root.md](../troubleshooting/jellyfin_subtitle_permission_denied_nfs_root.md)), and confirmed not to be the cause of the restarts that continued after the fix.
- **Most likely cause, not fixable from here**: research found multiple community reports (official Jellyfin forum) of a whole-process crash caused by a `System.ObjectDisposedException` in the WebSocket keepalive routine (`SessionWebSocketListener.SendForceKeepAlive`/`KeepAliveSockets`), trying to send a keepalive to a connection the client already closed — a classic race condition between the connection closing and the keepalive thread. Reported across multiple versions over time, no confirmed fix found in research. Reproduced here: the log shows `WS ... closed` followed ~375ms later by `WS ... request` (a fast reconnect, e.g. a tab refresh), then total silence — no exception logged, consistent with a crash that kills the whole process before .NET gets a chance to log anything.

### Mitigation applied (doesn't fix the root cause)

Added a `readinessProbe` (`tcpSocket` on port `8096`) to `apps/jellyfin/deployment.yaml` — doesn't prevent the crash, but ensures the `Service` stops routing traffic to the pod during the few seconds of restart, instead of letting requests hit a dead/starting pod.

### Upstream tracking

- **Project**: Jellyfin (`jellyfin/jellyfin` on GitHub).
- **Where to check**: [jellyfin/jellyfin issues filtered by "websocket crash keepalive"](https://github.com/jellyfin/jellyfin/issues?q=is%3Aissue+websocket+crash+keepalive) — there's no single confirmed matching issue number (our crash never produced a captured stack trace, the process dies in total silence; the `ObjectDisposedException`/`SendForceKeepAlive` hypothesis comes from a recurring pattern in official forum reports, not from one of our own logs). The structurally closest issue found so far is [#12090](https://github.com/jellyfin/jellyfin/issues/12090) ("crash if client 'disappears' without completing the close handshake"), closed 2025-04-29 and fixed by PRs #13457/#13459 — but the exception type there (`NullReferenceException` in `WebSocketController.GetActiveSockets`) **doesn't match** the initial hypothesis, so it isn't treated as confirmed.
- **What to look for**: a closed issue whose stack trace matches a whole-process crash (no .NET exception logged) triggered by a fast WebSocket reconnect (e.g. refreshing the dashboard tab) — and confirm whether the PR that closes it is already included in the image tag used in this repo.
- **Version in use in this repo**: `jellyfin/jellyfin:12.4` (`apps/jellyfin/deployment.yaml`).
- **Last checked**: 2026-11-09.

### Suggested diagnostics (next steps, if the problem persists)

- Try a different image tag (newer or older) to see if the bug is specific to the current version (`12.4`).
- If it becomes critical, capture a core dump of the crash to identify the exact native stack trace/exception (not attempted yet, needs additional setup) — would resolve the uncertainty about which upstream issue matches.
