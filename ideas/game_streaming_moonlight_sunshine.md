# Idea: Streaming de jogos local via Moonlight/Sunshine / Local game streaming via Moonlight/Sunshine

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: worker2 (GPU GTX 970), media stack.
**Origem**: proposta pelo usuário em 2026-11-08, ideia inicial de stack: **Sunshine** (host/servidor) + **Moonlight** (cliente).

Implementar streaming de jogos local — rodar os jogos num servidor com GPU (worker2) e jogar a partir de outro dispositivo na rede local via **Moonlight** (cliente) conectando num **Sunshine** (servidor de streaming) rodando no mesmo servidor de mídia.

### Considerações / perguntas em aberto

- **GPU compartilhada**: worker2 já usa `time-slicing` no `nvidia-device-plugin` pra dividir a GTX 970 entre `dcgm-exporter` e Jellyfin (ver [Media Stack - config.md](../clusters/media-stack/config.md)) — Sunshine precisaria de mais um slot, e não está claro ainda se time-slicing é adequado pra uma carga de trabalho de streaming em tempo real (latência é crítica aqui, diferente de transcodificação assíncrona do Jellyfin).
- **Limitação de driver já conhecida**: a GTX 970 (Maxwell) só é suportada até a branch `580` do driver NVIDIA (ver `ia/CLAUDE.md`) — precisa confirmar se o Sunshine funciona bem com essa branch, incluindo o encoder NVENC necessário pro streaming.
- **Servidor headless**: worker2 roda sem ambiente desktop/display físico — Sunshine normalmente espera capturar a tela de uma sessão gráfica ativa; precisa investigar se dá pra rodar com um display virtual (ex: `X11` com driver dummy, ou configuração headless específica do Sunshine) dentro do ambiente atual (k3s/container ou direto na VM).
- **Rodar em container (k3s) ou direto na VM worker2?** — diferente dos outros apps da media stack, Sunshine pode exigir acesso mais direto ao hardware gráfico (não só o device `nvidia.com/gpu` usado hoje por Jellyfin/dcgm-exporter), o que pode favorecer rodar fora do k3s.
- **Escopo de rede**: por enquanto só LAN, mesmo padrão dos outros serviços (ver [ideias futuras: acesso remoto via Tailscale](tailscale_remote_access.md) pra uma eventual extensão de acesso remoto).

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: worker2 (GTX 970 GPU), media stack.
**Origin**: proposed by the user on 2026-11-08, initial stack idea: **Sunshine** (host/server) + **Moonlight** (client).

Implement local game streaming — run games on a GPU server (worker2) and play from another device on the local network via **Moonlight** (client) connecting to a **Sunshine** (streaming server) running on the same media server.

### Considerations / open questions

- **Shared GPU**: worker2 already uses `time-slicing` in the `nvidia-device-plugin` to split the GTX 970 between `dcgm-exporter` and Jellyfin (see [Media Stack - config.md](../clusters/media-stack/config.md)) — Sunshine would need yet another slot, and it's not yet clear whether time-slicing is suitable for a real-time streaming workload (latency is critical here, unlike Jellyfin's asynchronous transcoding).
- **Already-known driver limitation**: the GTX 970 (Maxwell) is only supported up to NVIDIA driver branch `580` (see `ia/CLAUDE.md`) — need to confirm Sunshine works well with that branch, including the NVENC encoder required for streaming.
- **Headless server**: worker2 runs with no physical desktop environment/display — Sunshine normally expects to capture the screen of an active graphical session; needs investigation into whether a virtual display (e.g. `X11` with a dummy driver, or Sunshine's specific headless setup) can run within the current environment (k3s/container or directly on the VM).
- **Run in a container (k3s) or directly on the worker2 VM?** — unlike the other media stack apps, Sunshine may require more direct access to graphics hardware (not just the `nvidia.com/gpu` device used today by Jellyfin/dcgm-exporter), which may favor running it outside k3s.
- **Network scope**: LAN-only for now, same pattern as the other services (see [future ideas: remote access via Tailscale](tailscale_remote_access.md) for a possible future remote-access extension).
