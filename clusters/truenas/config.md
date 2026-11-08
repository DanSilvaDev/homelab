# TrueNAS SCALE - Resumo de Configuração

Propósito deste host: TrueNAS + hospedagem da VM worker2 do cluster k3s.

Ver também: [Worker2 - config.md](../worker2/config.md)

## Hardware
- **CPU**: AMD FX-8350 (8 Cores, 8 Threads)
- **Placa-mãe**: ASRock 990FX Extreme3
- **Memória RAM**: 28GB DDR3 Non-ECC (+8GB instalados em 2026-11-02, totalizando 28GB; valor anterior de 19GB corrigido em 2026-10-26, documentação antes disso tinha 24GB por engano)
- **GPU boot do host**: NVIDIA GeForce GT 210 (slots inferiores, apenas boot/console do TrueNAS)
- **GPU passthrough VM**: NVIDIA GeForce GTX 970 (slot primário PCIE2, isolada e dedicada à VM worker2)

### Armazenamento físico
- **Boot**: SSD principal (instalação limpa do TrueNAS SCALE)
- **Pool rápido**: SSD 960GB (substituiu o SSD de 120GB em 2026-11-02; isolado para laboratório de VMs/Kubernetes). Segundo SSD de 960GB previsto em breve, para mirror/redundância no mesmo pool.
- **Pool de arquivos**: HD Seagate IronWolf 3TB (substituiu o WD Blue 1TB em 2026-11-02; ainda em Stripe — mirror não confirmado/planejado por enquanto)

## BIOS
- Overclock Mode: Manual
- CPU Frequency Multiplier: x14.0 (2.8 GHz) ou x15.0 (3.0 GHz) para controle de TDP/consumo
- AMD Turbo Core: Disabled
- CPU Voltage Undervolt: iniciado em 1.20V (ajustável para baixo consumo)
- Economia de energia: Cool'n'Quiet (Enabled), C1E (Enabled), Core C6 State (Enabled), APM (Enabled)
- Estabilidade: CPU Spread Spectrum (Disabled), PCIe Spread Spectrum (Disabled)
- Active Core Control: All Cores (8 núcleos ativos)
- IOMMU (AMD-Vi): Enabled (pré-requisito para GPU passthrough)

## Sistema
- **Usuário admin**: `admin`
- **Segurança**:
  - Acesso web: HTTPS via certificado autoassinado
  - SSH: ativado com autenticação por chave Ed25519 (sem senha)
- **GPU isolada**:
  - Dispositivo: NVIDIA GeForce GTX 970
  - Local: System Settings > Advanced > Isolated GPU PCI Device(s)
  - Motivo: isola a GPU do host antes de qualquer VM subir, liberando para passthrough exclusivo

## Rede
- Interface física: `enp6s0` (sem IP próprio, atua só como membro de bridge)
- Bridge `br1`: IP estático `192.168.3.103/24`, membro: `enp6s0`
- Motivo: VMs com NIC anexada diretamente a `enp6s0` (modo macvtap) não conseguem alcançar o próprio host — ver [troubleshooting/worker2_vm_truenas_host_unreachable_macvtap.md](../../troubleshooting/worker2_vm_truenas_host_unreachable_macvtap.md)
- VMs devem anexar a NIC à bridge `br1`, não à interface física diretamente

## Pools ZFS

### `ssd-apps` (Stripe, SSD 960GB — substituiu o SSD de 120GB em 2026-11-02; segundo SSD de 960GB para mirror previsto em breve)
Finalidade: armazenamento veloz para VM/Kubernetes

| Nome | Tipo | Config | Uso |
|---|---|---|---|
| `vm-k3s-disk` | zvol | 40 GiB, volblocksize 16K, LZ4, sync Standard, sparse off, dedup off | Disco virtual exclusivo da VM worker2 |
| `fast-data` | dataset | recordsize 128K, LZ4, sync Standard, atime off | Storage rápido genérico via NFS; contém `os_images/` (ISOs de instalação) |
| `databases` | dataset | recordsize 16K, LZ4, sync Standard | Reservado para bancos (Postgres/Redis) — recordsize ajustado ao page size dessas engines |
| `k8s` | dataset | recordsize 128K, LZ4, sync Standard, quota ~40GiB | Export NFS para PVs dinâmicos do cluster k3s (kube-prometheus-stack, Loki) via `nfs-subdir-external-provisioner`. Hosts autorizados: `192.168.3.101` (control-plane), `192.168.3.102` (worker1), `192.168.3.105` (worker2) |

### `data` (Stripe, HD Seagate IronWolf 3TB — substituiu o WD Blue 1TB em 2026-11-02)
Finalidade: armazenamento geral, mídias (Jellyfin/Plex) e backups

| Nome | Compressão | Observação |
|---|---|---|
| `midias` | off / lz4 | **Biblioteca da media stack (Jellyfin + \*arr)**: `recordsize` 1M (ajustado em 2026-11-02 para arquivos de vídeo grandes/sequenciais), export NFS restrito aos 3 nodes do cluster (`192.168.3.101/102/105`, `rw` — antes aberto para `*`), sem quota. Estrutura de pastas: `movies/`, `tv/`, `music/`, `books/`, `downloads/`. Ver [Media Stack - config.md](../media-stack/config.md). Acesso validado antes de criar o PV/PVC definitivo via um pod descartável (`apps/helpers/test-midia-access-pod.yaml`, template reaproveitável pra validar qualquer export NFS do TrueNAS sem precisar de PV/PVC). |
| `Backups` | ZSTD | Split futuro planejado em sub-datasets por fonte (ex. `smb-pc`, `nfs-k3s`) para retenção/quota independentes — ver [ideias futuras](../../ideas.md) |
| `k8s` | LZ4 | Export NFS (quota 100GiB) para PVs dinâmicos do cluster k3s via `nfs-subdir-external-provisioner` (StorageClass `truenas-nfs-hdd`), para dados que não exigem I/O de SSD. Hosts autorizados: `192.168.3.101`, `192.168.3.102`, `192.168.3.105` |

## Monitoramento
- `node_exporter` (`prom/node-exporter:v1.8.2`) rodando via Docker direto no host (porta 9100), fora do catálogo de Apps da UI (restrição de host path mounts na Community Edition 25.10.7). Coletado pelo Prometheus do cluster k3s como alvo externo. Detalhes: [clusters/observability/config.md](../observability/config.md)

## Virtualização

| VM | Detalhes |
|---|---|
| `vm_k3s_worker_2` | ver [clusters/worker2/config.md](../worker2/config.md) |

Observação: este host hospeda apenas 1 VM — sem disputa de GPU ou de disco zvol com outras VMs.

## Próximos passos

- [x] **Trocar o SSD de 120GB do `ssd-apps` por um de 960GB** — feito em 2026-11-02 via `zpool replace` (resilver concluído com sucesso; disco de 120GB liberado).
- [ ] **Mover o SSD do control-plane para o TrueNAS e formar mirror em `ssd-apps`** (redundância; plano original mantido, só ainda não executado):
  1. Mover o SSD hoje instalado no control-plane para o TrueNAS.
  2. Reaproveitar o SSD de 120GB liberado no passo anterior como disco do control-plane.
  3. No TrueNAS, anexar o disco vindo do control-plane ao `ssd-apps` para formar o mirror:
     ```
     zpool attach ssd-apps <disco-960GB-atual> <disco-vindo-do-control-plane>
     # se o usável ficar preso no tamanho do menor disco, rodar `zpool online -e` (ou autoexpand)
     ```
  - **Risco crítico, cuidado redobrado**: o control-plane é hoje o **único nó com etcd** do cluster (sem HA de etcd — o Kube-VIP só dá IP estável pra API, não replica o etcd). Trocar o disco dele exige snapshot/backup do etcd (`k3s etcd-snapshot save` ou equivalente) **antes** de qualquer coisa, e um clone de disco (não reinstalação do zero) pra preservar a instalação exata do k3s.
- [ ] **Planejar backup em nuvem do TrueNAS** — hoje não existe nenhuma cópia dos dados fora do host físico; numa catástrofe (incêndio, roubo, falha simultânea de disco), os dois pools ZFS (`ssd-apps`, `data`) e tudo que depende deles (PVs dinâmicos do cluster, futura biblioteca de mídia, VM do worker2) seriam perdidos sem recuperação. Avaliar: destino (ex: Backblaze B2, AWS S3 Glacier, outro provedor compatível com `rclone`/replicação nativa do TrueNAS), escopo (que datasets realmente precisam de backup externo — provavelmente não o `k8s` de PVs recriáveis, mas sim `Backups`, `databases`, e dados irrecriáveis), frequência/retenção e custo estimado de armazenamento+egress.
