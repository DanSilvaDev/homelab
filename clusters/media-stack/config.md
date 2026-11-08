# Media Stack (Jellyfin + *arr) - Estado Atual

**Status: Implantada.** Todas as fases propostas foram concluídas (ver "Fases propostas" abaixo) — este arquivo documenta o estado atual da media stack (decisões, arquitetura, inventário de componentes), seguindo o mesmo padrão dos outros `clusters/*/config.md`. Pontos ainda em aberto (problemas não resolvidos e ideias futuras) não ficam aqui — ver [`known-issues.md`](../../known-issues.md) e [`ideas.md`](../../ideas.md) na raiz do projeto.

Ver também: [TrueNAS - config.md](../truenas/config.md) | [Worker2 - config.md](../worker2/config.md) | [Observability - config.md](../observability/config.md)

## Escopo confirmado

- **Jellyfin** — servidor de mídia, transcodificação via GPU (GTX 970/worker2).
- **Prowlarr** — gerenciador de indexers (alimenta Sonarr/Radarr/Lidarr/Readarr).
- **Sonarr** (séries), **Radarr** (filmes), **Lidarr** (música), **LazyLibrarian** (livros/audiobooks — substitui o Readarr, descontinuado; ver "Decisões já tomadas").
- **Bazarr** — legendas (integra com Sonarr/Radarr).
- **Seerr** (antigo Jellyseerr — ver "Seerr + Recyclarr") — UI de pedidos de conteúdo, integra com Jellyfin + Sonarr/Radarr.
- **qBittorrent + Gluetun** — cliente de torrent atrás de VPN (assinatura Surfshark do usuário).
- **Unpackerr** — extração automática de downloads compactados.
- **Recyclarr** — sincroniza perfis de qualidade (guias TRaSH) no Sonarr/Radarr periodicamente (sem suporte oficial a Lidarr/Readarr — ver seção "Seerr + Recyclarr").

Fora de escopo por enquanto: Usenet (SABnzbd/NZBGet) e acesso remoto (Tailscale/reverse proxy, depois de validar o básico só na LAN) — ambos registrados em [`ideas.md`](../../ideas.md) ([tailscale_remote_access.md](../../ideas/tailscale_remote_access.md) para o acesso remoto).

**FlareSolverr**: estava fora de escopo ("só adicionar se algum indexer exigir"), mas confirmado como necessário (2026-11-04) — ver seção "FlareSolverr" abaixo.

## Decisões já tomadas

- **Acesso**: só LAN por enquanto — `Service` tipo `LoadBalancer` via kube-vip + DNS local no Pi-hole, mesmo padrão já usado em Grafana/Prometheus/ntfy/Pi-hole. Acesso remoto fica para uma fase futura.
- **Storage de mídia**: dataset `data/midias` (TrueNAS, pool HDD WD Blue 1TB) como biblioteca central, montado via **PV estático** (não dinâmico).
- **GPU**: habilitar *time-slicing* no `nvidia-device-plugin` para liberar um slot de GPU pro Jellyfin sem mexer no `dcgm-exporter`.
- **Cliente de download**: qBittorrent (torrent), atrás de um container Gluetun fazendo VPN com a assinatura Surfshark do usuário.
  - **Confirmado via pesquisa (2026-10-27): Surfshark não suporta port forwarding.** O torrent funciona normalmente mesmo assim (não é pré-requisito), só com conectividade de entrada reduzida (menos peers conseguem abrir conexão direta com você — mais perceptível em trackers privados ou torrents com poucos seeds). Fontes: [VPNAlert](https://vpnalert.com/guides/surfshark-port-forwarding/), [VPNCentral](https://vpncentral.com/surfshark-vpn-review-2026-excellent-value-with-unlimited-device-connections/), [NatChecker](https://natchecker.com/blog/surfshark-port-forwarding).
- **Imagens**: `linuxserver.io` para os apps `*arr`/qBittorrent (padrão de fato nesse ecossistema, usa o padrão de env vars `TZ`/`PUID`/`PGID`), `jellyfin/jellyfin` oficial para o Jellyfin, `qmcgaw/gluetun` para a VPN.
- **PUID/PGID (Fase 4, 2026-11-02)**: `1000:1000` para todos os apps `linuxserver.io` da media stack (qBittorrent agora, `*arr` depois), consistente entre eles para evitar conflito de permissão ao escrever no mesmo PV compartilhado (`data/midias`). As pastas em `/mnt/data/midias` no TrueNAS foram criadas via `sudo mkdir` (dono `root`) na Fase 1 — precisam de `chown` para `1000:1000` antes do qBittorrent conseguir escrever.
- **Gluetun**: tag `:latest` usada intencionalmente (não pinada em versão fixa como os outros apps) — a Surfshark muda lista de servidores/configs do lado deles com frequência, e o próprio gluetun recomenda manter atualizado para compatibilidade com o provider.
- **Readarr substituído por LazyLibrarian (Fase 5, 2026-11-02)**: o Readarr foi descontinuado pelo projeto upstream em 2025 (metadados pararam de funcionar, falta de desenvolvedores) e a imagem `linuxserver.io/readarr` foi depreciada em seguida — tentar implantar deu `ImagePullBackOff` (`no match for platform in manifest`). LazyLibrarian cobre o mesmo papel (livros/audiobooks), é ativamente mantido, tem imagem `linuxserver.io` atual, e continua na lista oficial de apps suportados pelo Prowlarr — não quebra o padrão de indexer centralizado da stack. Porta padrão `5299` (vs `8787` do Readarr).
- **Diretório `apps/servarr/` (Fase 5, 2026-11-02; ampliado na Fase 6 e com o FlareSolverr, 2026-11-07)**: Prowlarr, Sonarr, Radarr, Lidarr, LazyLibrarian, Bazarr, Unpackerr, Seerr, Recyclarr e FlareSolverr agrupados em `apps/servarr/<app>/` (em vez de `apps/<app>/` solto) — "Servarr" é o nome oficial da comunidade pra essa família de apps (wiki.servarr.com).

## GPU time-slicing

**Feito (2026-11-02)** — `sharing.timeSlicing` habilitado no `nvidia-device-plugin` com `replicas: 2` (dcgm-exporter + Jellyfin), sem mexer na config do `dcgm-exporter`. Detalhes e motivação completa promovidos para [Worker2 - config.md](../worker2/config.md).

## Arquitetura de storage

### TrueNAS — dataset `data/midias`
**Feito (2026-11-02)** — `recordsize` 1M, export NFS restrito aos 3 nodes do cluster, e estrutura de pastas criada. Detalhes promovidos para [TrueNAS - config.md](../truenas/config.md).

### PV estático (não dinâmico) apontando para `data/midias`
Motivo: precisamos de um caminho previsível e compartilhado entre vários apps (não uma subpasta com nome auto-gerado por app, como faz o provisionamento dinâmico via `nfs-subdir-external-provisioner`). Isso também é pré-requisito para o *hardlink* funcionar entre `downloads/` e as pastas finais (`movies/`, `tv/`, etc.) — hardlink só funciona dentro do mesmo filesystem.

| Quem monta | Modo |
|---|---|
| qBittorrent, Unpackerr, Sonarr, Radarr, Lidarr, LazyLibrarian, Bazarr | RW |
| Jellyfin | RO |
| Prowlarr, Seerr, Recyclarr, FlareSolverr | Não monta (só falam com as APIs/entre si) |

### Config PVCs (um por app com estado)

| App | Tamanho sugerido | StorageClass recomendada |
|---|---|---|
| Jellyfin (config) | 5Gi | `truenas-nfs-ssd` |
| Jellyfin (cache) | 10Gi | `truenas-nfs-ssd` — descoberto durante a implantação (Fase 3): Jellyfin exige `/cache` como volume separado de `/config`, não estava no plano original |
| Prowlarr | ~1Gi | `truenas-nfs-ssd` |
| Sonarr | ~1Gi | `truenas-nfs-ssd` |
| Radarr | ~1Gi | `truenas-nfs-ssd` |
| Lidarr | ~1Gi | `truenas-nfs-ssd` |
| LazyLibrarian | ~1Gi | `truenas-nfs-ssd` |
| Bazarr | ~1Gi | `truenas-nfs-ssd` |
| Seerr | ~1Gi | `truenas-nfs-ssd` |
| qBittorrent | ~2Gi | `truenas-nfs-ssd` |

Unpackerr e Recyclarr não precisam de PVC (stateless / config via ConfigMap+Secret).

## SSD vai precisar de upgrade?

**Não mais — resolvido.** O SSD de 120GB foi substituído por um de 960GB em 2026-11-02 (via `zpool replace`, resilver concluído com sucesso). Um segundo SSD vem do control-plane para formar mirror no `ssd-apps` (redundância); o 120GB liberado vai para o control-plane. Ver [TrueNAS - config.md](../truenas/config.md) (seção "Próximos passos") para o plano dessa troca.

Contas atualizadas: pool `ssd-apps` agora com **960GB** no total.
- `vm-k3s-disk` (zvol, disco da VM worker2): **40 GiB fixos**.
- Dataset `k8s`: quota atual 40GiB, ~27Gi declarados (Prometheus 20Gi + Alertmanager 2Gi + Grafana 5Gi).
- `fast-data`/`databases`: sem quota fixa, uso real baixo (ISOs, etc.).

Isso deixa folga ampla (800GB+) mesmo somando os 9 configs novos da media stack (cada um pequeno, 1-5Gi) — **a divisão SSD/HDD por escassez de espaço não é mais necessária.**

**Decisão revisada**: todos os Config PVCs da media stack (Jellyfin, Prowlarr, Sonarr, Radarr, Lidarr, LazyLibrarian, Bazarr, Seerr, qBittorrent) passam a usar `truenas-nfs-ssd`. Justificativa: sobra espaço de sobra, esses configs (SQLite de cada app) se beneficiam de latência menor, e quando o mirror do `ssd-apps` for concluído eles ganham redundância que o pool `data` (HDD, ainda stripe single-disk) não tem.

## Inventário de componentes

| App | Tipo K8s | Monta mídia? | Config PVC? | GPU? | Service/DNS |
|---|---|---|---|---|---|
| Jellyfin | Deployment | RO | Sim (5Gi, SSD) | Sim | `jellyfin.home` |
| Prowlarr | Deployment | Não | Sim (~1Gi, HDD) | Não | `prowlarr.home` |
| Sonarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `sonarr.home` |
| Radarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `radarr.home` |
| Lidarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `lidarr.home` |
| LazyLibrarian | Deployment | RW | Sim (~1Gi, HDD) | Não | `lazylibrarian.home` |
| Bazarr | Deployment | RW | Sim (~1Gi, HDD) | Não | `bazarr.home` |
| Seerr | Deployment | Não | Sim (~1Gi, SSD) | Não | `seerr.home` |
| qBittorrent + Gluetun | Deployment (2 containers, netns compartilhado) | RW | Sim (~2Gi, HDD) | Não | `qbittorrent.home` |
| Unpackerr | Deployment | RW | Não | Não | — (Fase 5, integrado via API ao Sonarr/Radarr) |
| Recyclarr | **CronJob** (sincroniza perfis periodicamente — encaixe nativo de CronJob, não é um daemon) | Não | Não (ConfigMap + Secret p/ API keys) | Não | — |
| FlareSolverr | Deployment | Não | Não | Não | `ClusterIP` interno (`flaresolverr:8191`), sem LoadBalancer/DNS de propósito — ver seção "FlareSolverr" |

## Fases propostas (uma PR por fase)

1. ~~**TrueNAS**: `recordsize` 1M, restringir export, estrutura de pastas~~ — ✅ **feito** (ver [TrueNAS - config.md](../truenas/config.md)).
2. ~~**GPU time-slicing** (libera GPU pro Jellyfin sem mexer no dcgm-exporter)~~ — ✅ **feito** (ver [Worker2 - config.md](../worker2/config.md)).
3. ~~**Jellyfin**~~ — ✅ **feito** (2026-11-02): namespace `media`, PV/PVC estático pro `data/midias` (RO), PVCs de config e cache dedicados, GPU via time-slicing (`nvidia-smi` validado dentro do pod, GTX 970 visível), Service em `192.168.3.202`. Setup wizard concluído, hardware acceleration NVENC/NVDEC habilitado e biblioteca de mídia adicionada pelo usuário. Arquivos em `apps/jellyfin/`.
4. ~~**qBittorrent + Gluetun**~~ — ✅ **feito** (2026-11-02): namespace `media`, PVC de config (2Gi) + PV/PVC compartilhado `media-pvc` montado RW, VPN WireGuard via Gluetun em modo `custom` (não `surfshark`/`SERVER_COUNTRIES`, lista interna desatualizada) + `HEALTH_TARGET_ADDRESSES` apontando pra um IP fixo (não hostname, evita falso-negativo do healthcheck), Service em `192.168.3.206`. Validado: `ping`/DNS funcionando dentro do túnel, IP de saída confirmado como da Surfshark (`146.70.163.212`/`138.199.58.36`, não o IP residencial), 0 restarts internos do túnel. Arquivos em `apps/qbittorrent/`. Detalhe completo do diagnóstico (duas causas raiz distintas): [troubleshooting/gluetun_wireguard_stale_surfshark_server_list.md](../../troubleshooting/gluetun_wireguard_stale_surfshark_server_list.md).
5. ~~**Prowlarr + Sonarr + Radarr + Lidarr + LazyLibrarian + Bazarr + Unpackerr**~~ — ✅ **feito** (2026-11-02): namespace `media`, PVCs de config (~1Gi, `truenas-nfs-ssd`) para cada app, Services em `192.168.3.207` (Prowlarr) a `192.168.3.212` (Bazarr). Configuração via UI concluída: API keys geradas e conectadas entre os apps (Prowlarr alimentando os indexers pros demais, download client qBittorrent conectado). `limits.cpu` em `1` (1000m) em todos os 6 deployments — instabilidade observada com o valor anterior, normalizou após o aumento. Unpackerr adicionado por último (integração via API com Sonarr/Radarr/Lidarr, detalhes na seção "Unpackerr" abaixo). Arquivos em `apps/servarr/`.
6. ~~**Jellyseerr/Seerr + Recyclarr**~~ — ✅ **feito** (2026-11-07): ver seção "Seerr + Recyclarr" abaixo.

## Unpackerr

**Feito (2026-11-02, aplicado e rodando em 2026-11-04)**: `apps/servarr/unpackerr/deployment.yaml` — integração via API com Sonarr, Radarr e Lidarr (env vars `UN_<APP>_0_URL`/`UN_<APP>_0_API_KEY`/`UN_<APP>_0_PATHS_0`, API keys via Secret `unpackerr-api-keys` criado manualmente, nunca commitado — mesmo padrão do Gluetun/Surfshark). Sem PVC (config 100% via env vars, sem arquivo `.conf`), monta `media-pvc` em `/media` (mesmo path usado por Sonarr/Radarr/Lidarr — obrigatório para o Unpackerr casar o path retornado pela API com o path local). Imagem `golift/unpackerr` não segue o padrão PUID/PGID do linuxserver.io — usa `securityContext.runAsUser/runAsGroup: 1000` em vez disso.

```bash
kubectl create secret generic unpackerr-api-keys \
  --namespace media \
  --from-literal=sonarr-api-key='SUA_API_KEY_SONARR' \
  --from-literal=radarr-api-key='SUA_API_KEY_RADARR' \
  --from-literal=lidarr-api-key='SUA_API_KEY_LIDARR'
```

⚠️ **Limitação descoberta**: o Unpackerr só tem integração oficial com Sonarr/Radarr/Lidarr/Readarr (família "Starr"). Como o Readarr foi substituído por LazyLibrarian nesta stack (ver "Decisões já tomadas"), **livros/audiobooks baixados compactados (zip/rar) não são extraídos automaticamente** — fica de fora do escopo do Unpackerr. Se isso vier a ser um problema na prática, verificar se o próprio LazyLibrarian tem extração embutida (não confirmado ainda).

## Seerr + Recyclarr

**Feito (2026-11-07)**: arquivos em `apps/servarr/seerr/` e `apps/servarr/recyclarr/` (agrupados em `apps/servarr/` junto com o resto da família "Servarr", mesmo padrão da Fase 5 — ver "Decisões já tomadas").

- **Jellyseerr descontinuado, virou Seerr**: o projeto Jellyseerr foi fundido com o Overseerr formando o **Seerr** (`ghcr.io/seerr-team/seerr`) — a imagem antiga `fallenbagel/jellyseerr` está arquivada no Docker Hub ("DO NOT USE THIS IMAGE"). Mesma funcionalidade (UI de pedidos, integra com Jellyfin + Sonarr/Radarr). Diferente da primeira versão implantada (que manteve os recursos nomeados `jellyseerr` por continuidade), os manifestos foram renomeados por completo para `seerr` — diretório `apps/servarr/seerr/`, `seerr-deployment`, label `app: seerr`, Service `seerr` (`seerr.home`), PVC `seerr-pvc` — pra refletir o nome real do projeto upstream em vez do nome descontinuado.
  - Não segue PUID/PGID (linuxserver.io) — roda como usuário fixo `node` (UID/GID 1000), via `securityContext.runAsUser/runAsGroup: 1000` (mesmo valor da convenção 1000:1000 já usada no resto da stack, mas aqui só afeta o dono do PVC de config).
  - Path de config é `/app/config` (não `/config`) — confirmado na documentação oficial do Seerr.
  - Service em `192.168.3.213`.
  - Pendente (configuração manual via UI, igual aos outros apps): conectar ao Jellyfin (URL + API key) e ao Sonarr/Radarr (URL + API key) na primeira execução do wizard.
- **Recyclarr**: `apps/servarr/recyclarr/configmap.yaml` (config não-sensível, API keys via `!env_var` lendo de env vars do Secret) + `apps/servarr/recyclarr/cronjob.yaml` (`recyclarr sync` diário às 4h, não o modo daemon embutido na imagem). Imagem `ghcr.io/recyclarr/recyclarr:8` (sem tag `latest`), também rootless (1000:1000, sem PUID/PGID).
  - Secret criado manualmente, nunca commitado (mesmo padrão do Gluetun/Surfshark e Unpackerr):
    ```bash
    kubectl create secret generic recyclarr-api-keys \
      --namespace media \
      --from-literal=sonarr-api-key='SUA_API_KEY_SONARR' \
      --from-literal=radarr-api-key='SUA_API_KEY_RADARR'
    ```
  - ⚠️ **Limitação descoberta**: Recyclarr só tem suporte oficial a **Sonarr e Radarr** (confirmado 2026-11-07) — Lidarr fica fora, mesma classe de gap já documentada pro Unpackerr/LazyLibrarian.
  - **Perfis de qualidade preenchidos (2026-11-07)**: Radarr usa "HD Bluray + WEB" (`trash_id d1d67249d3890e49bc12e275d989a7e9`, exemplo oficial da documentação) e Sonarr usa "WEB-1080p" (`trash_id 72dae194fc92bf828f32cde7744e51a1` + grupo de custom formats, copiado do template oficial `sonarr/templates/web-1080p.yml` em [github.com/recyclarr/config-templates](https://github.com/recyclarr/config-templates), branch `v8`). São perfis "bons o suficiente pra maioria", não uma escolha definitiva — trocar pelo template de outra qualidade (remux, 2160p, etc.) no mesmo repo se preferir outra coisa.
  - **Gotcha de diagnóstico (2026-11-07)**: `recyclarr list quality-profiles` (comando usado pra tentar descobrir os `trash_id`s via CLI) travou indefinidamente no passo `git fetch` dentro de um pod `kubectl run` temporário da imagem `ghcr.io/recyclarr/recyclarr:8`, mesmo com DNS e egress HTTPS/git confirmados funcionando normalmente a partir de outro pod no mesmo namespace (`nicolaka/netshoot`, mesmo repo, clone instantâneo) — ou seja, não é problema de rede do cluster, é algo específico do binário git empacotado nessa imagem/ambiente (não investigado a fundo, baixa prioridade). Os `trash_id`s acima foram obtidos direto do repo de templates via `git clone` no `netshoot`, contornando o CLI do Recyclarr.

## FlareSolverr

**Feito (2026-11-07)**: `apps/servarr/flaresolverr/` — proxy auxiliar do Prowlarr que resolve o desafio anti-bot do Cloudflare (roda um Chromium headless) antes de repassar a resposta real pro indexer. Sem ele, qualquer indexer protegido por Cloudflare falha mesmo com a config correta.

- Imagem oficial `ghcr.io/flaresolverr/flaresolverr:latest`, porta `8191`. Já roda como usuário não-root por padrão (`USER flaresolverr` no próprio Dockerfile do projeto), mas por nome, não UID numérico — tentar `securityContext.runAsNonRoot: true` falhou na admissão do pod (`container has runAsNonRoot and image has non-numeric user ... cannot verify user is non-root`, 2026-11-07): o kubelet não consegue verificar estaticamente um usuário não-numérico. Removido — sem `securityContext`, a imagem usa seu próprio usuário padrão normalmente.
- Sem PVC (stateless — cada request sobe uma instância nova de Chromium, descartada ao final). `/tmp` montado como `emptyDir` em memória (`medium: Memory`), recomendação oficial do projeto (arquivos temporários de 10-30MB por request).
- `resources` mais generosos que o resto da stack de propósito — Chromium headless consome bastante memória por natureza (aviso do próprio projeto: evitar muitos requests simultâneos em máquina com pouca RAM).
- **Service é `ClusterIP`, não `LoadBalancer`** — diferente de todo o resto da stack, de propósito: o projeto avisa explicitamente pra não expor o FlareSolverr na rede/internet (pode ser abusado como proxy anônimo). Só o Prowlarr fala com ele, via DNS interno do cluster (`http://flaresolverr:8191`). Sem IP fixo do kube-vip, sem entrada no Pi-hole.
- Pendente (configuração manual via UI): conectar ao Prowlarr em **Settings → Indexers → Indexer Proxies** e associar aos indexers que exigem Cloudflare.
- ⚠️ **Limitação conhecida (2026-11-07)**: `OOMKilled` observado logo após a configuração — log mostra `PermissionError: [Errno 13] Permission denied` ao tentar terminar o processo do `chromedriver` a cada request (`self.process.terminate()`). É um vazamento de memória conhecido do upstream (processo não é limpo corretamente entre requests), não um erro de config nossa — ver [FlareSolverr#1161](https://github.com/FlareSolverr/FlareSolverr/issues/1161), [#983](https://github.com/FlareSolverr/FlareSolverr/issues/983), [PR #1163](https://github.com/FlareSolverr/FlareSolverr/pull/1163). O Deployment se auto-recupera (restart automático do Kubernetes), e o limite de memória foi aumentado (1Gi → 1.5Gi) só pra espaçar os OOMs, sem resolver a causa.

- **Tentativa de mitigação com `replicas: 2` revertida (2026-11-07)**: rodar 2 réplicas (ideia: requests avulsos são stateless e se distribuem entre pods via Service, então uma OOM não derrubaria o serviço inteiro) causou o efeito oposto — Prowlarr passou a receber `500 InternalServerError` do próprio FlareSolverr (`HTTP request failed: [500:InternalServerError] [POST] at [http://flaresolverr:8191/v1]`, "indexer's server is unavailable"). Hipótese: 2 réplicas dobram a demanda de memória/CPU (cada uma sobe seu próprio Chromium) no mesmo node, e sem folga suficiente o Chromium passou a falhar ao subir durante requests — um dado a mais reforçando a suspeita de contenção de recursos no node, ver [known-issues: instabilidade geral da stack `*arr`](../../known-issues/arr_stack_general_instability.md). Revertido para `replicas: 1`.

## Próximos passos

Todas as fases propostas foram concluídas — não há próximos passos de implantação em aberto. Problemas identificados durante a implantação, ainda não resolvidos, estão em [`known-issues.md`](../../known-issues.md): DNS do `jellyfin.home`, IP duplicado `192.168.3.201`, zvol do worker2 não redimensionado, erro "Failed to Load Indexers from API" no Prowlarr, restarts frequentes do Jellyfin, e instabilidade geral da stack `*arr`. Melhorias futuras (fora do escopo original) estão em [`ideas.md`](../../ideas.md).

## Outros `*arr`/ferramentas do ecossistema (referência)

- **FlareSolverr** — proxy auxiliar do Prowlarr para indexers com proteção Cloudflare. Implantado (2026-11-07) — ver seção "FlareSolverr" acima.
- **Tautulli** equivalente para Jellyfin seria o **Jellystat** (estatísticas de uso) — não pedido, só referência, registrado em [`ideas.md`](../../ideas.md).
