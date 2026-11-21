# Idea: Worker3 como nó GPU dedicado (mídia, jogos e IA local) / Worker3 as a dedicated GPU node (media, gaming and local AI)

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada (hardware já montado, plano em definição).
**Componente(s) relacionado(s)**: worker3 (node novo), Jellyfin, Wolf/Moonlight (streaming de jogos), Ollama/Open WebUI (IA local), worker2 (GTX 970), observabilidade, ciclo de energia.
**Origem**: sessão de planejamento de 2026-11-19 — o usuário montou uma máquina nova para entrar no cluster como `worker3`.

O `worker3` passa a ser o nó "GPU de verdade" do cluster, com três papéis: **servidor de mídia** (Jellyfin migra do worker2), **streaming de jogos da Steam** via Moonlight (ver [game_streaming_moonlight_sunshine.md](game_streaming_moonlight_sunshine.md)) e **IA local com modelos pequenos**.

### Hardware

| Item | Detalhe |
|---|---|
| CPU | AMD Ryzen 7 5700X (Zen 3, 8C/16T, **sem vídeo integrado**, PPT padrão 76 W) |
| Placa-mãe | ASUS TUF B550-Plus |
| RAM | 32 GB DDR4 |
| GPU | AMD Radeon RX 9070 16 GB (RDNA4) |
| Storage | NVMe Lexar NM790 512 GB (upgrade para um disco maior planejado) |
| Nobreak | compra prevista (os de worker1/worker2 duram ~10-15 min) |

### Cluster após o worker3

| Nó | Recursos | Ligado 24/7? |
|---|---|---|
| control-plane (Lenovo Z40-70) | 4 vCPU / 15 GiB | sim |
| worker1 | 4 vCPU / 15 GiB | não (desliga de madrugada) |
| worker2 (VM no TrueNAS) | 5 vCPU / 12 GiB + GTX 970 4 GB | não |
| **worker3** | **8C/16T / 32 GB + RX 9070 16 GB** | não (por enquanto entra no ciclo noturno) |

### Decisões já tomadas

- **Streaming de jogos via Wolf (Games on Whales) no k3s**, não Sunshine instalado no host — mantém o node "limpo" (convenção 5 do [`ia/CLAUDE.md`](../ia/CLAUDE.md)). Sunshine no host fica como plano de fallback.
- **worker3 entra como `agent`, não como control-plane** — vai reiniciar com mais frequência (drivers, kernel) e downloads da Steam saturam o NVMe, o que geraria latência de fsync e eleições de líder no etcd. Ver [multi_control_plane_ha.md](multi_control_plane_ha.md).
- **worker3 entra no ciclo de desligamento noturno**, por enquanto.
- **Raspberry Pi 3 descartado como control-plane** — ganha papel de utilitário 24/7 fora do cluster, ver [raspberry_pi3_utility_node.md](raspberry_pi3_utility_node.md).

### GPU compartilhada: regra de convivência

| Carga | Usa principalmente | Convive bem? |
|---|---|---|
| Jellyfin | bloco VCN (encoder/decoder), pouca VRAM | com tudo |
| Streaming (Wolf) | VCN + **muita VRAM** (jogo: 8-12 GB) | conflita com LLM |
| LLM (Ollama) | **muita VRAM** (modelo carregado) | conflita com jogo |

- **Regra**: jogo e LLM não rodam ao mesmo tempo. Ollama com `keep_alive` curto (descarrega o modelo da VRAM quando ocioso) e, para garantir, `kubectl scale --replicas=0` no Ollama antes de jogar — automatizável depois.
- **Exposição da GPU no Kubernetes**: proposta de usar um **generic-device-plugin** expondo `/dev/dri` com N "slots" alocáveis — mesmo modelo mental do time-slicing já usado no worker2 (ver [Worker2 - config.md](../clusters/worker2/config.md)). Pelo que se sabe hoje, o device plugin oficial da AMD não compartilha a GPU entre pods como o time-slicing da NVIDIA — **a validar na implementação**. O Wolf provavelmente vai precisar de `privileged` + `/dev/uinput` de qualquer forma.
- A GPU AMD não exige mudança de runtime no containerd (diferente do runtime NVIDIA configurado como default no worker2) — o acesso é via device files (`/dev/dri`, e `/dev/kfd` para ROCm).

### Storage local (NVMe 512 GB)

Orçamento aproximado:

| Uso | Espaço |
|---|---|
| Ubuntu + logs | ~30 GB |
| Imagens de container (Wolf/Steam são grandes) | ~40-60 GB |
| Cache de transcode do Jellyfin | ~20 GB |
| Modelos de IA (5-8 modelos de 7B a 24B em Q4, ~5-15 GB cada) | ~60-80 GB |
| **Sobra para a Steam** | **~300 GB** (2 a 4 jogos AAA) |

- **A IA cabe tranquilamente — o gargalo são os jogos.**
- **Separar em LVM**: um volume para sistema + k3s + imagens, outro para "Steam + modelos". Motivo: se a biblioteca da Steam encher o disco do sistema, o kubelet entra em **`DiskPressure`** e começa a **expulsar pods** (Jellyfin incluso).
- Biblioteca da Steam e modelos **no NVMe local**, nunca via NFS. PVCs locais via `local-path-provisioner` (já embutido no k3s), naturalmente fixados no worker3.
- **NVMe futuro**: na TUF B550-Plus, o slot M.2_1 é PCIe 4.0 direto da CPU (onde fica o NM790) e o M.2_2 é PCIe 3.0 via chipset — suficiente para jogos/modelos. Confirmar no manual se o M.2_2 compartilha lanes com alguma porta SATA. Com o disco novo, a Steam muda para ele.

### Jellyfin

- Transcodificação via **VA-API (Mesa)**. A RX 9070 faz encode/decode de H.264, HEVC (incluindo 10-bit) e **AV1** por hardware — salto grande em relação à GTX 970 (Maxwell).
- **A validar**: RDNA4 exige **Mesa ≥ 25 e kernel ≥ 6.14**. O Ubuntu 26.04 atende no host, mas a **imagem do Jellyfin** precisa trazer um Mesa novo o suficiente.
- A mídia continua no TrueNAS via NFS (`data/midias`, PV estático — ver [Media Stack - config.md](../clusters/media-stack/config.md)); o config PVC continua em `truenas-nfs-ssd`. Muda só onde o pod roda.
- **Cache de transcode no NVMe local** em vez de NFS (menos tráfego de rede).
- Libera o slot de time-slicing da GTX 970 — ver [gtx970_reuse_after_jellyfin_migration.md](gtx970_reuse_after_jellyfin_migration.md).

### IA local

- **Ollama + Open WebUI**. A RX 9070 (gfx1201) tem suporte oficial no ROCm 6.4+; se o ROCm der trabalho, o backend Vulkan do llama.cpp é alternativa.
- O que cabe em 16 GB de VRAM: 7-14B em Q4 com folga (ex: Qwen3 14B, Gemma 3 12B, Llama 3.1 8B); ~20-24B apertado (ex: gpt-oss-20b, Mistral Small 24B Q4). Acima disso, offload para os 32 GB de RAM funciona, mas fica lento.

### Economia de energia

Importante: **limitar potência economiza sob carga, não em idle** — em idle a CPU já consome bem abaixo de qualquer limite. Em idle, quem manda são C-states, ASPM e a política de frequência. Várias destas medidas valem também para os outros nodes.

**Sob carga:**
- **CPU**: limitar a ~80% do PPT de 76 W ≈ 60 W, que é exatamente o **"Eco Mode 45 W"** já disponível na BIOS (AMD CBS → PBO). Perda estimada de ~5-8% em multi-thread e ~0% em single-thread. Somar **undervolt via Curve Optimizer** (-10 a -20) economiza mais sem perder desempenho (requer teste de estabilidade).
- **GPU**: reduzir o power limit da RX 9070 (220 W de fábrica) via sysfs do `amdgpu` (`power1_cap`) — a faixa permitida depende da placa. Undervolt também é possível em RDNA4.

**Em idle:**
- **BIOS**: Global C-states ligado; "Power Supply Idle Control" em **Low Current Idle**; **PCIe ASPM** ligado; **CPPC** ligado (habilita o driver `amd-pstate`); ErP ligado; desligar o que não usa (RGB, áudio onboard).
- **Linux**: driver `amd-pstate-epp` com EPP em `balance_power`, aplicado via unit systemd simples — sem instalar `tuned`/`powertop` permanentemente (convenção de nodes limpos/neutros).
- **GPU headless**: sem monitor conectado a RX 9070 tende a ter idle baixo; `keep_alive` curto no Ollama libera a VRAM e deixa a GPU baixar de estado.

**Liga/desliga inteligente:**
- **Wake-on-LAN** habilitado na BIOS e persistido no Linux.
- O cliente **Moonlight tem "Wake PC"** nativo — permite o worker3 ficar desligado mais tempo e acordar quando for jogar (tensão: o Jellyfin também roda nele).
- O Raspberry Pi 3 dispara o WoL no horário de religar (ver [raspberry_pi3_utility_node.md](raspberry_pi3_utility_node.md)).

**Medir antes de otimizar:**
- Sem hardware novo: potência do pacote da CPU via **RAPL** e da GPU via **hwmon do `amdgpu`**, coletadas pelo node_exporter já existente, num painel do Grafana.
- Opcional: tomada inteligente com medição (Shelly/Tasmota) + exporter — mede o consumo real na tomada.
- O maior consumidor do homelab provavelmente é o host TrueNAS (FX-8350 + GT 210 + GTX 970) — ver [gtx970_reuse_after_jellyfin_migration.md](gtx970_reuse_after_jellyfin_migration.md) para a decisão sobre a GTX 970.

### Fases propostas

0. **Pré-checagens**: tipo de datastore do k3s no control-plane, snapshot antes de mexer no cluster.
1. **Worker3 base**: BIOS (energia, WoL, Resizable BAR), Ubuntu Server 26.04 com LVM separado, join como `agent` (binário via `scp`, mesmo bloqueio de ISP já documentado), validar GPU (`vainfo`, `rocminfo`).
2. **GPU no k8s + migração do Jellyfin** (VA-API, cache de transcode local).
3. **Ollama + Open WebUI** (modelos no volume local, `keep_alive` curto).
4. **Wolf** (streaming de jogos) + Moonlight "Wake PC".
5. **Observabilidade de energia** (RAPL/hwmon no Grafana, tomada com medição opcional).
6. **Raspberry Pi 3 como utilitário 24/7** (Ansible + WoL + monitoramento externo + NUT).

HA do control-plane fica fora deste plano — ver [multi_control_plane_ha.md](multi_control_plane_ha.md).

### Considerações / perguntas em aberto

- A imagem oficial do Jellyfin traz Mesa ≥ 25 (suporte a RDNA4)?
- generic-device-plugin vs. device plugin oficial da AMD — qual atende melhor o compartilhamento entre Jellyfin, Ollama e Wolf?
- worker3 no ciclo noturno fixo vs. ligar sob demanda via WoL — o Jellyfin morando nele limita o "sob demanda".
- Tamanho/momento do upgrade de disco.
- Nobreak do worker3 e integração com NUT (ver [raspberry_pi3_utility_node.md](raspberry_pi3_utility_node.md)).

---

## English

**Status**: Idea — not implemented (hardware already assembled, plan being defined).
**Related component(s)**: worker3 (new node), Jellyfin, Wolf/Moonlight (game streaming), Ollama/Open WebUI (local AI), worker2 (GTX 970), observability, power cycle.
**Origin**: planning session on 2026-11-19 — the user built a new machine to join the cluster as `worker3`.

`worker3` becomes the cluster's "real GPU" node, with three roles: **media server** (Jellyfin moves from worker2), **Steam game streaming** via Moonlight (see [game_streaming_moonlight_sunshine.md](game_streaming_moonlight_sunshine.md)) and **local AI with small models**.

### Hardware

| Item | Detail |
|---|---|
| CPU | AMD Ryzen 7 5700X (Zen 3, 8C/16T, **no integrated graphics**, default PPT 76 W) |
| Motherboard | ASUS TUF B550-Plus |
| RAM | 32 GB DDR4 |
| GPU | AMD Radeon RX 9070 16 GB (RDNA4) |
| Storage | Lexar NM790 512 GB NVMe (upgrade to a larger drive planned) |
| UPS | purchase planned (worker1/worker2's UPSes last ~10-15 min) |

### Cluster after worker3

| Node | Resources | On 24/7? |
|---|---|---|
| control-plane (Lenovo Z40-70) | 4 vCPU / 15 GiB | yes |
| worker1 | 4 vCPU / 15 GiB | no (shuts down overnight) |
| worker2 (VM on TrueNAS) | 5 vCPU / 12 GiB + GTX 970 4 GB | no |
| **worker3** | **8C/16T / 32 GB + RX 9070 16 GB** | no (joins the overnight cycle for now) |

### Decisions already made

- **Game streaming via Wolf (Games on Whales) in k3s**, not Sunshine installed on the host — keeps the node "clean" (convention 5 in [`ia/CLAUDE.md`](../ia/CLAUDE.md)). Sunshine on the host remains the fallback plan.
- **worker3 joins as an `agent`, not a control-plane** — it will reboot more often (drivers, kernel) and Steam downloads saturate the NVMe, which would cause fsync latency and leader elections in etcd. See [multi_control_plane_ha.md](multi_control_plane_ha.md).
- **worker3 joins the overnight shutdown cycle**, for now.
- **Raspberry Pi 3 ruled out as a control-plane** — it gets a 24/7 utility role outside the cluster instead, see [raspberry_pi3_utility_node.md](raspberry_pi3_utility_node.md).

### Shared GPU: coexistence rule

| Workload | Mainly uses | Coexists well? |
|---|---|---|
| Jellyfin | VCN block (encoder/decoder), little VRAM | with everything |
| Streaming (Wolf) | VCN + **lots of VRAM** (game: 8-12 GB) | conflicts with LLM |
| LLM (Ollama) | **lots of VRAM** (loaded model) | conflicts with gaming |

- **Rule**: gaming and LLM don't run at the same time. Ollama with a short `keep_alive` (unloads the model from VRAM when idle) and, to be sure, `kubectl scale --replicas=0` on Ollama before gaming — can be automated later.
- **Exposing the GPU in Kubernetes**: proposal to use a **generic-device-plugin** exposing `/dev/dri` with N allocatable "slots" — same mental model as the time-slicing already used on worker2 (see [Worker2 - config.md](../clusters/worker2/config.md)). As far as known today, AMD's official device plugin doesn't share the GPU across pods like NVIDIA's time-slicing does — **to be validated during implementation**. Wolf will likely need `privileged` + `/dev/uinput` anyway.
- The AMD GPU requires no containerd runtime change (unlike the NVIDIA runtime set as default on worker2) — access is via device files (`/dev/dri`, plus `/dev/kfd` for ROCm).

### Local storage (512 GB NVMe)

Rough budget:

| Use | Space |
|---|---|
| Ubuntu + logs | ~30 GB |
| Container images (Wolf/Steam are large) | ~40-60 GB |
| Jellyfin transcode cache | ~20 GB |
| AI models (5-8 models from 7B to 24B in Q4, ~5-15 GB each) | ~60-80 GB |
| **Left for Steam** | **~300 GB** (2 to 4 AAA games) |

- **AI fits easily — games are the bottleneck.**
- **Split with LVM**: one volume for system + k3s + images, another for "Steam + models". Reason: if the Steam library fills the system disk, the kubelet enters **`DiskPressure`** and starts **evicting pods** (Jellyfin included).
- Steam library and models **on the local NVMe**, never over NFS. Local PVCs via `local-path-provisioner` (already built into k3s), naturally pinned to worker3.
- **Future NVMe**: on the TUF B550-Plus, slot M.2_1 is PCIe 4.0 straight from the CPU (where the NM790 sits) and M.2_2 is PCIe 3.0 via the chipset — enough for games/models. Check the manual for whether M.2_2 shares lanes with any SATA port. With the new drive, Steam moves to it.

### Jellyfin

- Transcoding via **VA-API (Mesa)**. The RX 9070 does hardware encode/decode of H.264, HEVC (including 10-bit) and **AV1** — a big leap over the GTX 970 (Maxwell).
- **To validate**: RDNA4 requires **Mesa ≥ 25 and kernel ≥ 6.14**. Ubuntu 26.04 covers the host, but the **Jellyfin image** needs to ship a recent enough Mesa.
- Media stays on TrueNAS over NFS (`data/midias`, static PV — see [Media Stack - config.md](../clusters/media-stack/config.md)); the config PVC stays on `truenas-nfs-ssd`. Only where the pod runs changes.
- **Transcode cache on the local NVMe** instead of NFS (less network traffic).
- Frees the GTX 970's time-slicing slot — see [gtx970_reuse_after_jellyfin_migration.md](gtx970_reuse_after_jellyfin_migration.md).

### Local AI

- **Ollama + Open WebUI**. The RX 9070 (gfx1201) is officially supported in ROCm 6.4+; if ROCm is troublesome, llama.cpp's Vulkan backend is an alternative.
- What fits in 16 GB of VRAM: 7-14B in Q4 comfortably (e.g. Qwen3 14B, Gemma 3 12B, Llama 3.1 8B); ~20-24B tightly (e.g. gpt-oss-20b, Mistral Small 24B Q4). Beyond that, offloading to the 32 GB of RAM works, but is slow.

### Power saving

Important: **power limits save energy under load, not at idle** — at idle the CPU already draws far below any limit. At idle, C-states, ASPM and the frequency policy are what matter. Several of these measures also apply to the other nodes.

**Under load:**
- **CPU**: capping at ~80% of the 76 W PPT ≈ 60 W, which is exactly the **"Eco Mode 45 W"** already available in the BIOS (AMD CBS → PBO). Estimated loss of ~5-8% multi-thread and ~0% single-thread. Adding a **Curve Optimizer undervolt** (-10 to -20) saves more without losing performance (requires stability testing).
- **GPU**: lower the RX 9070's power limit (220 W stock) via `amdgpu` sysfs (`power1_cap`) — the allowed range depends on the card. Undervolting is also possible on RDNA4.

**At idle:**
- **BIOS**: Global C-states on; "Power Supply Idle Control" set to **Low Current Idle**; **PCIe ASPM** on; **CPPC** on (enables the `amd-pstate` driver); ErP on; disable unused features (RGB, onboard audio).
- **Linux**: `amd-pstate-epp` driver with EPP at `balance_power`, applied via a simple systemd unit — without permanently installing `tuned`/`powertop` (clean/neutral nodes convention).
- **Headless GPU**: with no monitor attached the RX 9070 tends to idle low; a short `keep_alive` in Ollama frees VRAM and lets the GPU drop into a lower power state.

**Smart on/off:**
- **Wake-on-LAN** enabled in the BIOS and persisted in Linux.
- The **Moonlight client has a native "Wake PC"** — lets worker3 stay off longer and wake up when it's time to game (tension: Jellyfin also runs on it).
- The Raspberry Pi 3 sends WoL at power-on time (see [raspberry_pi3_utility_node.md](raspberry_pi3_utility_node.md)).

**Measure before optimizing:**
- No new hardware needed: CPU package power via **RAPL** and GPU power via **`amdgpu` hwmon**, collected by the existing node_exporter, in a Grafana panel.
- Optional: smart plug with power metering (Shelly/Tasmota) + exporter — measures real wall consumption.
- The homelab's biggest consumer is probably the TrueNAS host (FX-8350 + GT 210 + GTX 970) — see [gtx970_reuse_after_jellyfin_migration.md](gtx970_reuse_after_jellyfin_migration.md) for the GTX 970 decision.

### Proposed phases

0. **Pre-checks**: k3s datastore type on the control-plane, snapshot before touching the cluster.
1. **Worker3 base**: BIOS (power, WoL, Resizable BAR), Ubuntu Server 26.04 with split LVM, join as `agent` (binary via `scp`, same ISP block already documented), validate GPU (`vainfo`, `rocminfo`).
2. **GPU in k8s + Jellyfin migration** (VA-API, local transcode cache).
3. **Ollama + Open WebUI** (models on the local volume, short `keep_alive`).
4. **Wolf** (game streaming) + Moonlight "Wake PC".
5. **Power observability** (RAPL/hwmon in Grafana, optional metering plug).
6. **Raspberry Pi 3 as a 24/7 utility** (Ansible + WoL + external monitoring + NUT).

Control-plane HA is out of scope for this plan — see [multi_control_plane_ha.md](multi_control_plane_ha.md).

### Considerations / open questions

- Does the official Jellyfin image ship Mesa ≥ 25 (RDNA4 support)?
- generic-device-plugin vs. AMD's official device plugin — which better handles sharing between Jellyfin, Ollama and Wolf?
- worker3 on a fixed overnight cycle vs. on-demand power-on via WoL — Jellyfin living on it limits "on-demand".
- Size/timing of the disk upgrade.
- worker3's UPS and NUT integration (see [raspberry_pi3_utility_node.md](raspberry_pi3_utility_node.md)).
