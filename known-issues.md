# Problemas Conhecidos / Known Issues

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Problemas já identificados e **ainda não resolvidos** neste homelab. Diferente de [`troubleshooting/`](troubleshooting/) (problemas já resolvidos, com causa raiz e correção documentadas), tudo listado aqui está em aberto — por definição, em investigação.

Quando um problema daqui for resolvido: escreva o post-mortem em `troubleshooting/` (ver skill `create-troubleshooting-doc`), adicione a linha correspondente em [`troubleshooting.md`](troubleshooting.md), e remova a entrada e o arquivo daqui.

| Problema | Componente | Tags | Data | Link |
|---|---|---|---|---|
| qBittorrent trava com frequência em "Downloading metadata" | qBittorrent, Gluetun | Media Stack, VPN, Network | 2026-11-08 | [qbittorrent_stuck_downloading_metadata.md](known-issues/qbittorrent_stuck_downloading_metadata.md) |
| Alertmanager dispara alerta de alto consumo de memória pro TrueNAS (ZFS ARC, comportamento esperado) | Alertmanager, TrueNAS | Alertmanager, Monitoring, TrueNAS | 2026-11-08 | [alertmanager_high_memory_alert_truenas.md](known-issues/alertmanager_high_memory_alert_truenas.md) |
| Loki registra um app não identificado, logando só "ppppppppp" (suspeita: Pi-hole) | Loki, Grafana Alloy, Pi-hole (suspeita) | Logging, Loki, Pi-hole | 2026-11-08 | [loki_unidentified_service_logging_garbage_lines.md](known-issues/loki_unidentified_service_logging_garbage_lines.md) |
| Instabilidade generalizada na stack `*arr` (páginas indisponíveis sem restart de pod) | Prowlarr, Sonarr, Radarr, Lidarr, LazyLibrarian, Bazarr | Media Stack, Performance, Node | 2026-11-07 | [arr_stack_general_instability.md](known-issues/arr_stack_general_instability.md) |
| Pod do Jellyfin reiniciando com frequência anormal (16 restarts/35h) | Jellyfin, GPU | Media Stack, GPU, Resources | 2026-11-04 | [jellyfin_frequent_pod_restarts.md](known-issues/jellyfin_frequent_pod_restarts.md) |
| "Failed to Load Indexers from API" em todos os indexers do Prowlarr | Prowlarr, `*arr` stack | Media Stack, Network | 2026-11-03 | [prowlarr_indexers_failed_to_load_from_api.md](known-issues/prowlarr_indexers_failed_to_load_from_api.md) |
| `192.168.3.201` duplicado entre `nginx` e `kube-prometheus-stack-grafana` | nginx, Grafana, Kube-VIP | LoadBalancer, Kube-VIP, Network | não registrada | [duplicate_loadbalancer_ip_192_168_3_201.md](known-issues/duplicate_loadbalancer_ip_192_168_3_201.md) |
| VM worker2 não reconhece zvol ampliado (40→50GiB) | worker2 (VM), TrueNAS | Storage, VM, Node | não registrada | [worker2_vm_disk_not_resized_after_zvol_expand.md](known-issues/worker2_vm_disk_not_resized_after_zvol_expand.md) |

---

## English

Problems already identified and **not yet resolved** in this homelab. Unlike [`troubleshooting/`](troubleshooting/) (already-resolved problems, with root cause and fix documented), everything listed here is open — by definition, under investigation.

When a problem here gets resolved: write the post-mortem in `troubleshooting/` (see the `create-troubleshooting-doc` skill), add the corresponding row to [`troubleshooting.md`](troubleshooting.md), and remove the entry and the file from here.

| Problem | Component | Tags | Date | Link |
|---|---|---|---|---|
| qBittorrent frequently gets stuck on "Downloading metadata" | qBittorrent, Gluetun | Media Stack, VPN, Network | 2026-11-08 | [qbittorrent_stuck_downloading_metadata.md](known-issues/qbittorrent_stuck_downloading_metadata.md) |
| Alertmanager fires a high memory usage alert for TrueNAS (ZFS ARC, expected behavior) | Alertmanager, TrueNAS | Alertmanager, Monitoring, TrueNAS | 2026-11-08 | [alertmanager_high_memory_alert_truenas.md](known-issues/alertmanager_high_memory_alert_truenas.md) |
| Loki logs an unidentified app, only recording "ppppppppp" (suspected: Pi-hole) | Loki, Grafana Alloy, Pi-hole (suspected) | Logging, Loki, Pi-hole | 2026-11-08 | [loki_unidentified_service_logging_garbage_lines.md](known-issues/loki_unidentified_service_logging_garbage_lines.md) |
| General instability in the `*arr` stack (pages unavailable with no pod restart) | Prowlarr, Sonarr, Radarr, Lidarr, LazyLibrarian, Bazarr | Media Stack, Performance, Node | 2026-11-07 | [arr_stack_general_instability.md](known-issues/arr_stack_general_instability.md) |
| Jellyfin pod restarting with abnormal frequency (16 restarts/35h) | Jellyfin, GPU | Media Stack, GPU, Resources | 2026-11-04 | [jellyfin_frequent_pod_restarts.md](known-issues/jellyfin_frequent_pod_restarts.md) |
| "Failed to Load Indexers from API" on every Prowlarr indexer | Prowlarr, `*arr` stack | Media Stack, Network | 2026-11-03 | [prowlarr_indexers_failed_to_load_from_api.md](known-issues/prowlarr_indexers_failed_to_load_from_api.md) |
| `192.168.3.201` duplicated between `nginx` and `kube-prometheus-stack-grafana` | nginx, Grafana, Kube-VIP | LoadBalancer, Kube-VIP, Network | not recorded | [duplicate_loadbalancer_ip_192_168_3_201.md](known-issues/duplicate_loadbalancer_ip_192_168_3_201.md) |
| worker2 VM doesn't see the expanded zvol (40→50GiB) | worker2 (VM), TrueNAS | Storage, VM, Node | not recorded | [worker2_vm_disk_not_resized_after_zvol_expand.md](known-issues/worker2_vm_disk_not_resized_after_zvol_expand.md) |
