# Media Stack (Jellyfin + *arr) - Plano de Implantação

**Status: Planejamento — nada implantado ainda.** Este arquivo documenta decisões e arquitetura antes da execução. Conforme as fases forem implantadas, promover o conteúdo relevante para um `config.md` normal (padrão dos outros `clusters/*/`), mantendo este `plan.md` só com o que ainda não foi feito.

Ver também: [TrueNAS - config.md](../truenas/config.md) | [Worker2 - config.md](../worker2/config.md) | [Observability - config.md](../observability/config.md)

## Escopo confirmado

- **Jellyfin** — servidor de mídia, transcodificação via GPU (GTX 970/worker2).
- **Prowlarr** — gerenciador de indexers (alimenta Sonarr/Radarr/Lidarr/Readarr).
- **Sonarr** (séries), **Radarr** (filmes), **Lidarr** (música), **Readarr** (livros/audiobooks).
- **Bazarr** — legendas (integra com Sonarr/Radarr).
- **Jellyseerr** — UI de pedidos de conteúdo, integra com Jellyfin + Sonarr/Radarr.
- **qBittorrent + Gluetun** — cliente de torrent atrás de VPN (assinatura Surfshark do usuário).
- **Unpackerr** — extração automática de downloads compactados.
- **Recyclarr** — sincroniza perfis de qualidade (guias TRaSH) no Sonarr/Radarr/Lidarr/Readarr periodicamente.

Fora de escopo por enquanto: FlareSolverr (só adicionar se algum indexer do Prowlarr exigir), Usenet (SABnzbd/NZBGet), acesso remoto (Tailscale/reverse proxy — próxima etapa futura, depois de validar o básico só na LAN).

## Decisões já tomadas

- **Acesso**: só LAN por enquanto — `Service` tipo `LoadBalancer` via kube-vip + DNS local no Pi-hole, mesmo padrão já usado em Grafana/Prometheus/ntfy/Pi-hole. Acesso remoto fica para uma fase futura.
- **Storage de mídia**: dataset `data/midias` (TrueNAS, pool HDD WD Blue 1TB) como biblioteca central, montado via **PV estático** (não dinâmico).
- **GPU**: habilitar *time-slicing* no `nvidia-device-plugin` para liberar um slot de GPU pro Jellyfin sem mexer no `dcgm-exporter`.
- **Cliente de download**: qBittorrent (torrent), atrás de um container Gluetun fazendo VPN com a assinatura Surfshark do usuário.
  - **Confirmado via pesquisa (2026-10-27): Surfshark não suporta port forwarding.** O torrent funciona normalmente mesmo assim (não é pré-requisito), só com conectividade de entrada reduzida (menos peers conseguem abrir conexão direta com você — mais perceptível em trackers privados ou torrents com poucos seeds). Fontes: [VPNAlert](https://vpnalert.com/guides/surfshark-port-forwarding/), [VPNCentral](https://vpncentral.com/surfshark-vpn-review-2026-excellent-value-with-unlimited-device-connections/), [NatChecker](https://natchecker.com/blog/surfshark-port-forwarding).
- **Imagens**: `linuxserver.io` para os apps `*arr`/qBittorrent (padrão de fato nesse ecossistema, já usamos o padrão de env vars `TZ`/`PUID`/`PGID` deles no Pi-hole), `jellyfin/jellyfin` oficial para o Jellyfin, `qmcgaw/gluetun` para a VPN.

## Por que GPU precisa de ajuste (time-slicing)

Hoje `nvidia.com/gpu: 1/1` já está 100% alocado pelo `dcgm-exporter`, mesmo ele usando só uma fração mínima da GPU de verdade (só leitura de métricas via NVML/DCGM, nenhum trabalho pesado). O motivo: o modelo de *device plugin* do Kubernetes é tudo-ou-nada — não existe "acesso de leitura parcial" à GPU. Qualquer container que precise falar com o driver (seja só pra ler métricas, seja pra transcodificar) precisa declarar `resources.limits["nvidia.com/gpu"]`, e sem *time-slicing* o node só anuncia 1 unidade alocável (1 GPU física = 1 slot). Fix: habilitar `sharing.timeSlicing` no `nvidia-device-plugin` com `replicas: 2` (dcgm-exporter + Jellyfin), sem precisar tocar na config do dcgm-exporter.

## Arquitetura de storage

### TrueNAS — dataset `data/midias`
- Mudar `recordsize` de 128K (padrão atual) para **1M** (melhor para arquivos de vídeo grandes/sequenciais) — fazer **antes** de qualquer dado ser escrito.
- Restringir o export NFS aos 3 nodes do cluster (`192.168.3.101/102/105`, `rw`) — hoje está aberto para `*` (qualquer host).
- Sem quota por enquanto (899G disponíveis no pool; dataset vazio hoje).
- Estrutura de pastas:
  ```
  data/midias/
    movies/
    tv/
    music/
    books/
    downloads/
  ```

### PV estático (não dinâmico) apontando para `data/midias`
Motivo: precisamos de um caminho previsível e compartilhado entre vários apps (não uma subpasta com nome auto-gerado por app, como faz o provisionamento dinâmico via `nfs-subdir-external-provisioner`). Isso também é pré-requisito para o *hardlink* funcionar entre `downloads/` e as pastas finais (`movies/`, `tv/`, etc.) — hardlink só funciona dentro do mesmo filesystem.

| Quem monta | Modo |
|---|---|
| qBittorrent, Unpackerr, Sonarr, Radarr, Lidarr, Readarr, Bazarr | RW |
| Jellyfin | RO |
| Prowlarr, Jellyseerr, Recyclarr | Não monta (só falam com as APIs dos outros apps) |

### Config PVCs (um por app com estado)

| App | Tamanho sugerido | StorageClass recomendada |
|---|---|---|
| Jellyfin | 5Gi | `truenas-nfs-ssd` (único com motivo técnico real para SSD — ver seção abaixo) |
| Prowlarr | ~1Gi | `truenas-nfs-hdd` |
| Sonarr | ~1Gi | `truenas-nfs-hdd` |
| Radarr | ~1Gi | `truenas-nfs-hdd` |
| Lidarr | ~1Gi | `truenas-nfs-hdd` |
| Readarr | ~1Gi | `truenas-nfs-hdd` |
| Bazarr | ~1Gi | `truenas-nfs-hdd` |
| Jellyseerr | ~1Gi | `truenas-nfs-hdd` |
| qBittorrent | ~2Gi | `truenas-nfs-hdd` |

Unpackerr e Recyclarr não precisam de PVC (stateless / config via ConfigMap+Secret).

## SSD vai precisar de upgrade?

**Sim, em algum momento — mas não é bloqueante pra começar, se distribuirmos os configs como na tabela acima.**

Contas: o pool SSD físico tem **120GB** no total.
- `vm-k3s-disk` (zvol, disco da VM worker2): **40 GiB fixos** (`sparse off` = totalmente pré-alocado, não é sob demanda).
- Dataset `k8s`: quota atual 40GiB, já com ~27Gi declarados (Prometheus 20Gi + Alertmanager 2Gi + Grafana 5Gi).
- `fast-data`/`databases`: sem quota fixa, mas já com uso real (ISOs, etc.).

Isso já deixa pouca margem **antes** de qualquer coisa nova. Se colocássemos todos os 9 configs novos no SSD (plano original), precisaríamos subir a quota do `k8s` pra ~70GiB — e `70 + 40 (VM) = 110GB` de 120GB físicos deixaria só ~10GB de folga pro resto, perigosamente apertado (ZFS degrada e fica arriscado perto de 100% cheio).

**Mitigação adotada no plano acima**: só o Jellyfin (SQLite + cache de thumbnails, acessado constantemente pela UI, concorrendo com streaming se estivesse no mesmo HD do acervo) fica no SSD. Os outros 8 apps têm bancos de configuração minúsculos e I/O bem mais leve — vão no HDD sem risco real. Isso adia a necessidade de comprar hardware novo.

**Quando comprar**: conforme a biblioteca de mídia crescer, mais apps forem adicionados, ou normal uso cotidiano. Vale considerar um segundo SSD dedicado (hoje o `vm-k3s-disk` sozinho consome 1/3 do pool atual) em vez de só trocar por um maior, já que isolaria a VM do restante.

## Inventário de componentes

| App | Tipo K8s | Monta mídia? | Config PVC? | GPU? | Service/DNS |
|---|---|---|---|---|---|
| Jellyfin | Deployment | RO | Sim (5Gi, SSD) | Sim | `jellyfin.home` |
| Prowlarr | Deployment | Não | Sim (~1Gi, HDD) | Não | `prowlarr.home` |
| Sonarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `sonarr.home` |
| Radarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `radarr.home` |
| Lidarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `lidarr.home` |
| Readarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `readarr.home` |
| Bazarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `bazarr.home` |
| Jellyseerr | Deployment | Não | Sim (~1Gi, HDD) | Não | `jellyseerr.home` |
| qBittorrent + Gluetun | Deployment (2 containers, netns compartilhado) | RW | Sim (~2Gi, HDD) | Não | `qbittorrent.home` |
| Unpackerr | Deployment | RW | Não | Não | — |
| Recyclarr | **CronJob** (sincroniza perfis periodicamente — encaixe nativo de CronJob, não é um daemon) | Não | Não (ConfigMap + Secret p/ API keys) | Não | — |

## Fases propostas (uma PR por fase)

1. **TrueNAS**: `recordsize` 1M, restringir export, estrutura de pastas, (sem mudança de quota SSD agora — mitigado pela distribuição SSD/HDD acima).
2. **GPU time-slicing** (libera GPU pro Jellyfin sem mexer no dcgm-exporter).
3. **Jellyfin**.
4. **qBittorrent + Gluetun + Unpackerr** (pipeline de download).
5. **Prowlarr + Sonarr + Radarr + Lidarr + Readarr + Bazarr** (núcleo *arr).
6. **Jellyseerr + Recyclarr** (camada de conveniência por cima).

## Pendências / decisões ainda em aberto

- Credenciais WireGuard da Surfshark para o Gluetun (Secret, criado manualmente via `kubectl create secret`, nunca commitado).
- Conectar API keys entre os apps `*arr` (cada um gera a própria no primeiro boot — configuração manual via UI, não dá pra automatizar antecipadamente).
- Config do Recyclarr (ConfigMap para perfis de qualidade + Secret para as API keys que ele consome).
- Confirmar se algum indexer configurado no Prowlarr vai exigir FlareSolverr (só se descobre na prática, ao adicionar indexers reais).
- Decidir IPs do `kube-vip` para os 9 novos `LoadBalancer` (tem sobra no range `192.168.3.200-254`, inclusive o `.202` que ficou livre com a remoção do Rancher).

## Outros `*arr`/ferramentas do ecossistema (referência, fora de escopo por enquanto)

- **FlareSolverr** — proxy auxiliar do Prowlarr para indexers com proteção Cloudflare.
- **Tautulli** equivalente para Jellyfin seria o **Jellystat** (estatísticas de uso) — não pedido, só referência.
