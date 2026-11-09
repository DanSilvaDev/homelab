# Known Issue: Pod do Jellyfin reiniciando com frequência anormal / Jellyfin pod restarting with abnormal frequency

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Causa raiz provável identificada, mas aparenta ser um bug upstream do próprio Jellyfin — sem correção disponível do nosso lado.
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

### Diagnóstico sugerido (próximos passos, se o problema persistir)

- Monitorar os [GitHub issues do jellyfin/jellyfin](https://github.com/jellyfin/jellyfin/issues) por `SessionWebSocketListener`/`ObjectDisposedException`/`WebSocket` pra ver se alguma versão futura corrige.
- Testar outra tag de imagem (mais nova ou mais antiga) pra ver se o bug é específico da versão atual (`12.4`).
- Se for crítico, capturar um core dump do crash pra identificar a stack trace nativa exata (não tentado ainda, exige setup adicional).

---

## English

**Status**: 🔴 Likely root cause identified, but appears to be an upstream Jellyfin bug — no fix available on our side.
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

### Suggested diagnostics (next steps, if the problem persists)

- Monitor [jellyfin/jellyfin's GitHub issues](https://github.com/jellyfin/jellyfin/issues) for `SessionWebSocketListener`/`ObjectDisposedException`/`WebSocket` to see if a future release fixes it.
- Try a different image tag (newer or older) to see if the bug is specific to the current version (`12.4`).
- If it becomes critical, capture a core dump of the crash to identify the exact native stack trace (not attempted yet, needs additional setup).
