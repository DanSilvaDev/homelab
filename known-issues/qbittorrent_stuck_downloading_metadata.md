# Known Issue: qBittorrent trava com frequência em "Downloading metadata" / qBittorrent frequently gets stuck on "Downloading metadata"

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Em investigação — causa raiz não confirmada.
**Componente(s)**: qBittorrent, Gluetun (media stack).
**Identificado em**: 2026-11-08.
**Relacionado**: [Media Stack - config.md](../clusters/media-stack/config.md) (seção "Decisões já tomadas" — Surfshark não suporta port forwarding, conectividade de entrada já documentada como reduzida), [troubleshooting/gluetun_wireguard_stale_surfshark_server_list.md](../troubleshooting/gluetun_wireguard_stale_surfshark_server_list.md) (histórico de problemas de conectividade do túnel WireGuard do Gluetun nesse mesmo par de containers, já resolvido, mas mesmo componente).

Muitos torrents no qBittorrent ficam travados no estado **"Downloading metadata"** com frequência — comportamento recorrente, ainda não investigado a fundo.

### Hipóteses / o que já foi tentado

Nenhuma hipótese abaixo foi validada ainda — são pontos de partida plausíveis dado o que já se sabe sobre o componente:

- **Falta de port forwarding na Surfshark** já está documentada como uma limitação conhecida (ver `media-stack/config.md`): conectividade de entrada reduzida, mais perceptível em trackers privados ou torrents com poucos seeds — "Downloading metadata" depende de conseguir falar com peers/DHT, então essa limitação é uma suspeita natural, mas não confirmada como causa.
- Não confirmado se **DHT, PeX e Local Peer Discovery** estão de fato habilitados nas configurações do qBittorrent.
- Não confirmado se o túnel WireGuard do Gluetun deixa passar tráfego **UDP** normalmente (DHT depende de UDP) — pode haver alguma restrição não identificada.

### Diagnóstico sugerido (ainda não executado)

- Confirmar se DHT, PeX e LSD estão habilitados em qBittorrent (Settings → BitTorrent).
- Testar um magnet link de um torrent popular/com muitos seeds, pra isolar se o problema é geral ou só afeta torrents com poucos peers (coerente com a limitação já documentada de conectividade de entrada).
- Checar logs do Gluetun e confirmar se tráfego UDP está passando normalmente pelo túnel.
- Confirmar que a porta de listening do qBittorrent está corretamente configurada dentro do netns compartilhado com o Gluetun.
- Como último recurso de isolamento: testar um torrent fora do Gluetun (sem VPN) pra confirmar se o problema é causado pela VPN ou é independente dela — ⚠️ isso expõe o IP residencial, usar só como teste pontual e reverter em seguida.

---

## English

**Status**: 🔴 Under investigation — root cause not confirmed.
**Component(s)**: qBittorrent, Gluetun (media stack).
**Identified on**: 2026-11-08.
**Related**: [Media Stack - config.md](../clusters/media-stack/config.md) ("Decisões já tomadas" section — Surfshark doesn't support port forwarding, reduced inbound connectivity already documented), [troubleshooting/gluetun_wireguard_stale_surfshark_server_list.md](../troubleshooting/gluetun_wireguard_stale_surfshark_server_list.md) (history of WireGuard tunnel connectivity problems on this same container pair, already resolved, but same component).

Many torrents in qBittorrent frequently get stuck on the **"Downloading metadata"** state — a recurring behavior, not yet thoroughly investigated.

### Hypotheses / what's already been tried

None of the hypotheses below have been validated yet — they're plausible starting points given what's already known about the component:

- **Surfshark's lack of port forwarding** is already documented as a known limitation (see `media-stack/config.md`): reduced inbound connectivity, more noticeable on private trackers or low-seed torrents — "Downloading metadata" depends on reaching peers/DHT, so this limitation is a natural suspect, but not confirmed as the cause.
- Not confirmed whether **DHT, PeX, and Local Peer Discovery** are actually enabled in qBittorrent's settings.
- Not confirmed whether Gluetun's WireGuard tunnel passes **UDP** traffic normally (DHT relies on UDP) — there could be some unidentified restriction.

### Suggested diagnostics (not yet run)

- Confirm whether DHT, PeX, and LSD are enabled in qBittorrent (Settings → BitTorrent).
- Test a magnet link for a popular/high-seed torrent, to isolate whether the problem is general or only affects low-peer torrents (consistent with the already-documented inbound connectivity limitation).
- Check Gluetun's logs and confirm UDP traffic passes normally through the tunnel.
- Confirm qBittorrent's listening port is correctly configured within the netns shared with Gluetun.
- As a last-resort isolation step: test a torrent outside Gluetun (no VPN) to confirm whether the problem is caused by the VPN or independent of it — ⚠️ this exposes the home IP, use only as a one-off test and revert right after.
