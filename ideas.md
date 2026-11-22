# Ideias Futuras / Future Ideas

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Backlog de ideias e melhorias futuras que surgiram durante a implementação ou o uso do homelab, mas ainda não foram implementadas. Ideias simples ficam só na tabela abaixo; ideias que já precisam de mais contexto/arquitetura ganham um arquivo próprio em [`ideas/`](ideas/) (mesmo papel que um `plan.md` cumpre antes da implementação de um componente).

Quando uma ideia for implementada: a implementação real já gera seus próprios arquivos/manifests/PR, então a linha (e o arquivo em `ideas/`, se houver) é **removida** daqui — sem precisar manter rastro histórico neste backlog.

| Ideia | Componente | Data | Link |
|---|---|---|---|
| Migrar o datastore do control-plane de SQLite para etcd embutido (`--cluster-init`) — snapshots automáticos, pré-requisito do HA; previsto logo após a entrada do worker3 | control-plane, Kube-VIP, backup do datastore | 2026-11-21 | [control_plane_sqlite_to_etcd_migration.md](ideas/control_plane_sqlite_to_etcd_migration.md) |
| Worker3 como nó GPU dedicado (Jellyfin, streaming de jogos, IA local) + estratégias de economia de energia | worker3 (novo node), Jellyfin, Wolf, Ollama/Open WebUI | 2026-11-19 | [worker3_gpu_node.md](ideas/worker3_gpu_node.md) |
| Raspberry Pi 3 como utilitário 24/7 fora do cluster (Ansible, WoL, monitoramento externo, detecção de queda de energia via NUT) | Raspberry Pi 3, ciclo de energia, nobreaks | 2026-11-19 | [raspberry_pi3_utility_node.md](ideas/raspberry_pi3_utility_node.md) |
| Novos usos para a GTX 970 depois que o Jellyfin migrar (Whisper, IA leve, Immich — ou remover) | worker2 (GTX 970), TrueNAS | 2026-11-19 | [gtx970_reuse_after_jellyfin_migration.md](ideas/gtx970_reuse_after_jellyfin_migration.md) |
| Runner self-hosted do GitHub Actions no cluster, pra CI/CD dos repositórios pessoais | novo componente no cluster, repositórios pessoais no GitHub | 2026-11-09 | [github_actions_self_hosted_runner.md](ideas/github_actions_self_hosted_runner.md) |
| Rotina de snapshot local + backup em nuvem do TrueNAS | TrueNAS | 2026-11-09 | [truenas_cloud_backup_snapshot_routine.md](ideas/truenas_cloud_backup_snapshot_routine.md) |
| Stack de dados (PostgreSQL, RabbitMQ, Redis, Kafka) pra apps locais em desenvolvimento | novo componente no cluster | 2026-11-09 | [local_dev_data_stack_postgres_rabbitmq_redis_kafka.md](ideas/local_dev_data_stack_postgres_rabbitmq_redis_kafka.md) |
| Avaliar GitHub Wiki vs. manter a documentação no repositório | README.md, ia/, troubleshooting.md, known-issues.md, ideas.md | 2026-11-08 | [github_wiki_vs_in_repo_docs.md](ideas/github_wiki_vs_in_repo_docs.md) |
| Limpeza automática de pastas `archived-*` no TrueNAS após 30 dias | TrueNAS, nfs-subdir-external-provisioner | 2026-11-08 | [truenas_archived_pvc_cleanup_automation.md](ideas/truenas_archived_pvc_cleanup_automation.md) |
| HTTPS com certificado válido em todo o cluster, aposentando HTTP | Grafana, Prometheus, Alertmanager, ntfy, Pi-hole, media stack | 2026-11-08 | [cluster_wide_https_valid_certificates.md](ideas/cluster_wide_https_valid_certificates.md) |
| Adicionar mais control-planes ao cluster (HA real de etcd) — adiada: exige 2º dispositivo 24/7 e rede estável | control-plane, Kube-VIP, etcd | 2026-11-08 | [multi_control_plane_ha.md](ideas/multi_control_plane_ha.md) |
| Streaming de jogos local via Moonlight + Wolf | worker3 (GPU RX 9070) | 2026-11-08 | [game_streaming_moonlight_sunshine.md](ideas/game_streaming_moonlight_sunshine.md) |
| Automação via Ansible do ciclo de desligar/religar workers + TrueNAS | worker1, worker2, TrueNAS, coredns-bypass | 2026-10-31 | [ansible_power_cycle_automation.md](ideas/ansible_power_cycle_automation.md) |
| Acesso remoto via Tailscale (media stack, ntfy, outros serviços LAN-only) | media stack, ntfy | 2026-10-27 | [tailscale_remote_access.md](ideas/tailscale_remote_access.md) |
| Suporte a Usenet (SABnzbd/NZBGet) como fonte de download adicional na media stack | media stack | 2026-11-08 | — |
| Jellystat (estatísticas de uso do Jellyfin, equivalente ao Tautulli) | Jellyfin | 2026-11-08 | — |
| Split do dataset `backups` do TrueNAS em sub-datasets por fonte (ex: `smb-pc`, `nfs-k3s`) para retenção/quota independentes | TrueNAS | 2026-10-24 | — |

---

## English

Backlog of future ideas and improvements that came up during implementation or use of the homelab, but haven't been implemented yet. Simple ideas just stay in the table below; ideas that already need more context/architecture get their own file in [`ideas/`](ideas/) (the same role a `plan.md` plays before a component is implemented).

When an idea gets implemented: the actual implementation already generates its own files/manifests/PR, so the row (and the file in `ideas/`, if any) is **removed** from here — no need to keep a historical trail in this backlog.

| Idea | Component | Date | Link |
|---|---|---|---|
| Migrate the control-plane datastore from SQLite to embedded etcd (`--cluster-init`) — automatic snapshots, HA prerequisite; planned right after worker3 joins | control-plane, Kube-VIP, datastore backup | 2026-11-21 | [control_plane_sqlite_to_etcd_migration.md](ideas/control_plane_sqlite_to_etcd_migration.md) |
| Worker3 as a dedicated GPU node (Jellyfin, game streaming, local AI) + power-saving strategies | worker3 (new node), Jellyfin, Wolf, Ollama/Open WebUI | 2026-11-19 | [worker3_gpu_node.md](ideas/worker3_gpu_node.md) |
| Raspberry Pi 3 as a 24/7 utility outside the cluster (Ansible, WoL, external monitoring, power outage detection via NUT) | Raspberry Pi 3, power cycle, UPSes | 2026-11-19 | [raspberry_pi3_utility_node.md](ideas/raspberry_pi3_utility_node.md) |
| New uses for the GTX 970 after Jellyfin moves (Whisper, light AI, Immich — or remove it) | worker2 (GTX 970), TrueNAS | 2026-11-19 | [gtx970_reuse_after_jellyfin_migration.md](ideas/gtx970_reuse_after_jellyfin_migration.md) |
| Self-hosted GitHub Actions runner in the cluster, for personal repositories' CI/CD | new cluster component, personal GitHub repositories | 2026-11-09 | [github_actions_self_hosted_runner.md](ideas/github_actions_self_hosted_runner.md) |
| TrueNAS local snapshot + cloud backup routine | TrueNAS | 2026-11-09 | [truenas_cloud_backup_snapshot_routine.md](ideas/truenas_cloud_backup_snapshot_routine.md) |
| Data stack (PostgreSQL, RabbitMQ, Redis, Kafka) for local apps under development | new cluster component | 2026-11-09 | [local_dev_data_stack_postgres_rabbitmq_redis_kafka.md](ideas/local_dev_data_stack_postgres_rabbitmq_redis_kafka.md) |
| Evaluate GitHub Wiki vs. keeping docs in the repository | README.md, ia/, troubleshooting.md, known-issues.md, ideas.md | 2026-11-08 | [github_wiki_vs_in_repo_docs.md](ideas/github_wiki_vs_in_repo_docs.md) |
| Automatic cleanup of `archived-*` folders on TrueNAS after 30 days | TrueNAS, nfs-subdir-external-provisioner | 2026-11-08 | [truenas_archived_pvc_cleanup_automation.md](ideas/truenas_archived_pvc_cleanup_automation.md) |
| Cluster-wide HTTPS with a valid certificate, retiring HTTP | Grafana, Prometheus, Alertmanager, ntfy, Pi-hole, media stack | 2026-11-08 | [cluster_wide_https_valid_certificates.md](ideas/cluster_wide_https_valid_certificates.md) |
| Add more control-plane nodes to the cluster (real etcd HA) — deferred: needs a second always-on device and a stable network | control-plane, Kube-VIP, etcd | 2026-11-08 | [multi_control_plane_ha.md](ideas/multi_control_plane_ha.md) |
| Local game streaming via Moonlight + Wolf | worker3 (RX 9070 GPU) | 2026-11-08 | [game_streaming_moonlight_sunshine.md](ideas/game_streaming_moonlight_sunshine.md) |
| Ansible automation for the worker + TrueNAS shutdown/startup cycle | worker1, worker2, TrueNAS, coredns-bypass | 2026-10-31 | [ansible_power_cycle_automation.md](ideas/ansible_power_cycle_automation.md) |
| Remote access via Tailscale (media stack, ntfy, other LAN-only services) | media stack, ntfy | 2026-10-27 | [tailscale_remote_access.md](ideas/tailscale_remote_access.md) |
| Usenet support (SABnzbd/NZBGet) as an additional download source for the media stack | media stack | 2026-11-08 | — |
| Jellystat (Jellyfin usage statistics, Tautulli equivalent) | Jellyfin | 2026-11-08 | — |
| Split the TrueNAS `backups` dataset into per-source sub-datasets (e.g. `smb-pc`, `nfs-k3s`) for independent retention/quota | TrueNAS | 2026-10-24 | — |
