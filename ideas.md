# Ideias Futuras / Future Ideas

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Backlog de ideias e melhorias futuras que surgiram durante a implementação ou o uso do homelab, mas ainda não foram implementadas. Ideias simples ficam só na tabela abaixo; ideias que já precisam de mais contexto/arquitetura ganham um arquivo próprio em [`ideas/`](ideas/) (mesmo papel que um `plan.md` cumpre antes da implementação de um componente).

Quando uma ideia for implementada: a implementação real já gera seus próprios arquivos/manifests/PR, então a linha (e o arquivo em `ideas/`, se houver) é **removida** daqui — sem precisar manter rastro histórico neste backlog.

| Ideia | Componente | Data | Link |
|---|---|---|---|
| Avaliar GitHub Wiki vs. manter a documentação no repositório | README.md, ia/, troubleshooting.md, known-issues.md, ideas.md | 2026-11-08 | [github_wiki_vs_in_repo_docs.md](ideas/github_wiki_vs_in_repo_docs.md) |
| Limpeza automática de pastas `archived-*` no TrueNAS após 30 dias | TrueNAS, nfs-subdir-external-provisioner | 2026-11-08 | [truenas_archived_pvc_cleanup_automation.md](ideas/truenas_archived_pvc_cleanup_automation.md) |
| HTTPS com certificado válido em todo o cluster, aposentando HTTP | Grafana, Prometheus, Alertmanager, ntfy, Pi-hole, media stack | 2026-11-08 | [cluster_wide_https_valid_certificates.md](ideas/cluster_wide_https_valid_certificates.md) |
| Adicionar mais control-planes ao cluster (HA real de etcd) | control-plane, Kube-VIP, etcd | 2026-11-08 | [multi_control_plane_ha.md](ideas/multi_control_plane_ha.md) |
| Streaming de jogos local via Moonlight/Sunshine | worker2 (GPU), media stack | 2026-11-08 | [game_streaming_moonlight_sunshine.md](ideas/game_streaming_moonlight_sunshine.md) |
| Automação via Ansible do ciclo de desligar/religar workers + TrueNAS | worker1, worker2, TrueNAS, coredns-bypass | 2026-10-31 | [ansible_power_cycle_automation.md](ideas/ansible_power_cycle_automation.md) |
| Acesso remoto via Tailscale (media stack, ntfy, outros serviços LAN-only) | media stack, ntfy | 2026-10-27 | [tailscale_remote_access.md](ideas/tailscale_remote_access.md) |
| Suporte a Usenet (SABnzbd/NZBGet) como fonte de download adicional na media stack | media stack | 2026-11-08 | — |
| Jellystat (estatísticas de uso do Jellyfin, equivalente ao Tautulli) | Jellyfin | 2026-11-08 | — |
| Split do dataset `Backups` do TrueNAS em sub-datasets por fonte (ex: `smb-pc`, `nfs-k3s`) para retenção/quota independentes | TrueNAS | 2026-10-24 | — |

---

## English

Backlog of future ideas and improvements that came up during implementation or use of the homelab, but haven't been implemented yet. Simple ideas just stay in the table below; ideas that already need more context/architecture get their own file in [`ideas/`](ideas/) (the same role a `plan.md` plays before a component is implemented).

When an idea gets implemented: the actual implementation already generates its own files/manifests/PR, so the row (and the file in `ideas/`, if any) is **removed** from here — no need to keep a historical trail in this backlog.

| Idea | Component | Date | Link |
|---|---|---|---|
| Evaluate GitHub Wiki vs. keeping docs in the repository | README.md, ia/, troubleshooting.md, known-issues.md, ideas.md | 2026-11-08 | [github_wiki_vs_in_repo_docs.md](ideas/github_wiki_vs_in_repo_docs.md) |
| Automatic cleanup of `archived-*` folders on TrueNAS after 30 days | TrueNAS, nfs-subdir-external-provisioner | 2026-11-08 | [truenas_archived_pvc_cleanup_automation.md](ideas/truenas_archived_pvc_cleanup_automation.md) |
| Cluster-wide HTTPS with a valid certificate, retiring HTTP | Grafana, Prometheus, Alertmanager, ntfy, Pi-hole, media stack | 2026-11-08 | [cluster_wide_https_valid_certificates.md](ideas/cluster_wide_https_valid_certificates.md) |
| Add more control-plane nodes to the cluster (real etcd HA) | control-plane, Kube-VIP, etcd | 2026-11-08 | [multi_control_plane_ha.md](ideas/multi_control_plane_ha.md) |
| Local game streaming via Moonlight/Sunshine | worker2 (GPU), media stack | 2026-11-08 | [game_streaming_moonlight_sunshine.md](ideas/game_streaming_moonlight_sunshine.md) |
| Ansible automation for the worker + TrueNAS shutdown/startup cycle | worker1, worker2, TrueNAS, coredns-bypass | 2026-10-31 | [ansible_power_cycle_automation.md](ideas/ansible_power_cycle_automation.md) |
| Remote access via Tailscale (media stack, ntfy, other LAN-only services) | media stack, ntfy | 2026-10-27 | [tailscale_remote_access.md](ideas/tailscale_remote_access.md) |
| Usenet support (SABnzbd/NZBGet) as an additional download source for the media stack | media stack | 2026-11-08 | — |
| Jellystat (Jellyfin usage statistics, Tautulli equivalent) | Jellyfin | 2026-11-08 | — |
| Split the TrueNAS `Backups` dataset into per-source sub-datasets (e.g. `smb-pc`, `nfs-k3s`) for independent retention/quota | TrueNAS | 2026-10-24 | — |
