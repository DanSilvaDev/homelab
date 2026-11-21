# Idea: Streaming de jogos local via Moonlight + Wolf no worker3 / Local game streaming via Moonlight + Wolf on worker3

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: worker3 (GPU RX 9070), Wolf (Games on Whales), Steam, cliente Moonlight.
**Origem**: proposta pelo usuário em 2026-11-08, ideia inicial de stack Sunshine + Moonlight no worker2. **Revisada em 2026-11-19**: o alvo passou a ser o novo **worker3** (ver [worker3_gpu_node.md](worker3_gpu_node.md)) e o servidor escolhido passou a ser o **Wolf**.

Rodar jogos da Steam num servidor com GPU no cluster e jogar a partir de outro dispositivo na rede local via **Moonlight** (cliente).

### Por que saiu do worker2

A versão original (worker2/GTX 970) tinha três bloqueios que o worker3 elimina: GPU já dividida por time-slicing entre `dcgm-exporter` e Jellyfin (latência crítica do streaming não combina bem com isso), driver travado na branch 580 (Maxwell), e uma GPU de 4 GB/Maxwell fraca para jogos atuais.

### Decisão: Wolf (Games on Whales) no k3s

- O **Wolf** é um servidor compatível com o protocolo do Moonlight, feito para rodar em containers: cria um compositor Wayland virtual por sessão, **sem precisar de display físico nem dummy plug** — relevante porque o Ryzen 7 5700X do worker3 **não tem vídeo integrado** e o servidor é headless.
- Mantém o node "limpo" (convenção 5 do [`ia/CLAUDE.md`](../ia/CLAUDE.md)) — tudo em manifesto, sem ambiente gráfico instalado no host.
- **Fallback**: Sunshine + Steam instalados direto no host (setup mais testado pela comunidade, mas quebra a convenção de node limpo e exige display virtual/dummy plug HDMI).

### Considerações / perguntas em aberto

- **Requisitos do pod** (a confirmar na documentação do Wolf): provavelmente `privileged`, acesso a `/dev/dri`, `/dev/uinput`/`/dev/uhid` (input virtual), `hostNetwork` (portas do protocolo GameStream) e regras `udev` no host.
- **GPU compartilhada com Jellyfin e Ollama**: jogo e LLM não rodam ao mesmo tempo (disputam VRAM) — ver a regra de convivência em [worker3_gpu_node.md](worker3_gpu_node.md).
- **Biblioteca da Steam no NVMe local** do worker3 (não NFS), num volume LVM separado do sistema para não causar `DiskPressure`.
- **Wake-on-LAN**: o Moonlight tem "Wake PC" nativo — permite acordar o worker3 para jogar fora do horário em que ele estaria ligado.
- Jogos com anti-cheat podem não funcionar via Proton (limitação geral de jogos no Linux, não específica do Wolf).
- **Escopo de rede**: só LAN por enquanto, mesmo padrão dos outros serviços (ver [acesso remoto via Tailscale](tailscale_remote_access.md) para uma extensão futura).

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: worker3 (RX 9070 GPU), Wolf (Games on Whales), Steam, Moonlight client.
**Origin**: proposed by the user on 2026-11-08, initial stack idea Sunshine + Moonlight on worker2. **Revised on 2026-11-19**: the target became the new **worker3** (see [worker3_gpu_node.md](worker3_gpu_node.md)) and the chosen server became **Wolf**.

Run Steam games on a GPU server in the cluster and play from another device on the local network via **Moonlight** (client).

### Why it moved off worker2

The original version (worker2/GTX 970) had three blockers that worker3 removes: a GPU already split via time-slicing between `dcgm-exporter` and Jellyfin (streaming's critical latency doesn't fit well with that), a driver pinned to the 580 branch (Maxwell), and a 4 GB Maxwell GPU too weak for current games.

### Decision: Wolf (Games on Whales) in k3s

- **Wolf** is a Moonlight-protocol-compatible server built to run in containers: it creates a virtual Wayland compositor per session, **with no physical display or dummy plug needed** — relevant because worker3's Ryzen 7 5700X **has no integrated graphics** and the server is headless.
- Keeps the node "clean" (convention 5 in [`ia/CLAUDE.md`](../ia/CLAUDE.md)) — everything in manifests, no desktop environment installed on the host.
- **Fallback**: Sunshine + Steam installed directly on the host (the setup most tested by the community, but it breaks the clean-node convention and requires a virtual display/HDMI dummy plug).

### Considerations / open questions

- **Pod requirements** (to confirm in Wolf's documentation): likely `privileged`, access to `/dev/dri`, `/dev/uinput`/`/dev/uhid` (virtual input), `hostNetwork` (GameStream protocol ports) and `udev` rules on the host.
- **GPU shared with Jellyfin and Ollama**: gaming and LLM don't run at the same time (they compete for VRAM) — see the coexistence rule in [worker3_gpu_node.md](worker3_gpu_node.md).
- **Steam library on worker3's local NVMe** (not NFS), on an LVM volume separate from the system so it doesn't cause `DiskPressure`.
- **Wake-on-LAN**: Moonlight has a native "Wake PC" — lets you wake worker3 to game outside the hours it would normally be on.
- Games with anti-cheat may not work through Proton (a general Linux gaming limitation, not Wolf-specific).
- **Network scope**: LAN-only for now, same pattern as the other services (see [remote access via Tailscale](tailscale_remote_access.md) for a future extension).
