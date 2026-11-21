# Idea: Novos usos para a GTX 970 depois que o Jellyfin migrar para o worker3 / New uses for the GTX 970 after Jellyfin moves to worker3

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: worker2 (GTX 970 via passthrough), host TrueNAS, `nvidia-device-plugin` (time-slicing), Bazarr, Open WebUI/Ollama.
**Origem**: sessão de planejamento de 2026-11-19 — com o Jellyfin migrando para o worker3 (ver [worker3_gpu_node.md](worker3_gpu_node.md)), o slot de time-slicing da GTX 970 fica livre; o usuário perguntou o que mais dá pra fazer com ela.

### Restrições conhecidas

- **Maxwell (GM204), compute capability 5.2, 4 GB de VRAM** (na prática 3,5 GB rápidos + 0,5 GB lentos).
- **Driver travado na branch 580** (última com suporte a Maxwell — ver [`ia/CLAUDE.md`](../ia/CLAUDE.md) e [Worker2 - config.md](../clusters/worker2/config.md)).
- **O CUDA 13 removeu o suporte a Maxwell** — qualquer imagem usada aqui precisa ser construída com **CUDA 12.x**. Validar isso em cada imagem candidata.
- NVENC faz H.264 e HEVC 8-bit; **sem AV1**.
- O host TrueNAS desliga de madrugada — nada aqui pode ser serviço "24/7".

### Opções

1. **Whisper para o Bazarr** — o Bazarr tem integração com um provider de Whisper (ex: `whisper-asr-webservice`), gerando legendas automaticamente para o que não tem legenda disponível nos providers. A GTX 970 dá conta dos modelos `small`/`medium`. Encaixe natural na media stack existente.
2. **IA "companheira" sempre disponível** — Ollama (build CUDA 12) com um modelo de **embeddings** (para RAG no Open WebUI) e/ou um modelo pequeno (1-3B). Fica disponível **enquanto a RX 9070 está ocupada com jogo** (ver a regra de convivência em [worker3_gpu_node.md](worker3_gpu_node.md)).
3. **Immich (backup de fotos) com machine learning na GPU** — segundo a documentação do Immich, a aceleração CUDA exige compute capability 5.2+ (a GTX 970 está exatamente no limite — confirmar). Seria um serviço novo, com storage no TrueNAS.
4. **Remover a GPU** — economiza energia enquanto o TrueNAS está ligado (estimativa de ~10-15 W em idle, a medir) e simplifica o worker2 (sem passthrough, sem driver em hold, sem `nvidia-device-plugin`/`dcgm-exporter`, sem o risco de módulo ausente após upgrade de kernel já documentado em [troubleshooting/worker2_nvidia_module_missing_after_kernel_upgrade.md](../troubleshooting/worker2_nvidia_module_missing_after_kernel_upgrade.md)).

Re-encode em lote da biblioteca (ex: Tdarr) foi descartado para a GTX 970 — faz mais sentido na RX 9070 (AV1).

**Sugestão inicial**: 1 + 2 (aproveitam o time-slicing já existente). Se não forem usados na prática, reavaliar a opção 4.

### Considerações / perguntas em aberto

- Quantos slots de time-slicing manter (hoje `replicas: 2`: `dcgm-exporter` + Jellyfin)?
- Imagens candidatas (Whisper, Ollama, Immich ML) ainda oferecem build CUDA 12.x?
- Medir o consumo real da GTX 970 em idle antes de decidir entre usar e remover (ver "Economia de energia" em [worker3_gpu_node.md](worker3_gpu_node.md)).

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: worker2 (GTX 970 via passthrough), TrueNAS host, `nvidia-device-plugin` (time-slicing), Bazarr, Open WebUI/Ollama.
**Origin**: planning session on 2026-11-19 — with Jellyfin moving to worker3 (see [worker3_gpu_node.md](worker3_gpu_node.md)), the GTX 970's time-slicing slot frees up; the user asked what else it could be used for.

### Known constraints

- **Maxwell (GM204), compute capability 5.2, 4 GB of VRAM** (in practice 3.5 GB fast + 0.5 GB slow).
- **Driver pinned to the 580 branch** (the last one supporting Maxwell — see [`ia/CLAUDE.md`](../ia/CLAUDE.md) and [Worker2 - config.md](../clusters/worker2/config.md)).
- **CUDA 13 dropped Maxwell support** — any image used here must be built with **CUDA 12.x**. Validate this for every candidate image.
- NVENC does H.264 and 8-bit HEVC; **no AV1**.
- The TrueNAS host shuts down overnight — nothing here can be a "24/7" service.

### Options

1. **Whisper for Bazarr** — Bazarr integrates with a Whisper provider (e.g. `whisper-asr-webservice`), automatically generating subtitles for content with no subtitles available from providers. The GTX 970 can handle the `small`/`medium` models. Natural fit with the existing media stack.
2. **Always-available "companion" AI** — Ollama (CUDA 12 build) with an **embeddings** model (for RAG in Open WebUI) and/or a small model (1-3B). Available **while the RX 9070 is busy gaming** (see the coexistence rule in [worker3_gpu_node.md](worker3_gpu_node.md)).
3. **Immich (photo backup) with GPU machine learning** — according to Immich's documentation, CUDA acceleration requires compute capability 5.2+ (the GTX 970 is exactly at the limit — to confirm). It would be a new service, with storage on TrueNAS.
4. **Remove the GPU** — saves power while TrueNAS is on (estimated ~10-15 W at idle, to be measured) and simplifies worker2 (no passthrough, no held driver, no `nvidia-device-plugin`/`dcgm-exporter`, no risk of the missing-module-after-kernel-upgrade issue already documented in [troubleshooting/worker2_nvidia_module_missing_after_kernel_upgrade.md](../troubleshooting/worker2_nvidia_module_missing_after_kernel_upgrade.md)).

Batch re-encoding of the library (e.g. Tdarr) was ruled out for the GTX 970 — it makes more sense on the RX 9070 (AV1).

**Initial suggestion**: 1 + 2 (they reuse the existing time-slicing). If they don't get used in practice, revisit option 4.

### Considerations / open questions

- How many time-slicing slots to keep (currently `replicas: 2`: `dcgm-exporter` + Jellyfin)?
- Do the candidate images (Whisper, Ollama, Immich ML) still offer a CUDA 12.x build?
- Measure the GTX 970's real idle consumption before deciding between using and removing it (see "Power saving" in [worker3_gpu_node.md](worker3_gpu_node.md)).
