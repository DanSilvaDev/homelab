# Problemas Conhecidos / Known Issues

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Problemas já identificados e **ainda não resolvidos** neste homelab. Diferente de [`troubleshooting/`](troubleshooting/) (problemas já resolvidos, com causa raiz e correção documentadas), tudo listado aqui está em aberto. Dois status possíveis: 🔴 **em investigação** (causa raiz ainda não confirmada, ou já confirmada mas corrigível por nós) ou 🔵 **bloqueado por upstream** (causa raiz confirmada, mas a correção depende de um projeto de terceiros — ver skill `track-upstream-blocker`).

Quando um problema daqui for resolvido: escreva o post-mortem em `troubleshooting/` (ver skill `create-troubleshooting-doc`), adicione a linha correspondente em [`troubleshooting.md`](troubleshooting.md), e remova a entrada e o arquivo daqui.

| Problema | Componente | Tags | Data | Link |
|---|---|---|---|---|
| 🔵 CronJob do Recyclarr trava e dispara `KubeJobNotCompleted` (branch `v8` inexistente, loop de retry infinito) | Recyclarr, CronJob, Alertmanager | Media Stack, CronJob, Alertmanager | 2026-11-08 | [recyclarr_cronjob_hangs_kubejobnotcompleted.md](known-issues/recyclarr_cronjob_hangs_kubejobnotcompleted.md) |
| qBittorrent trava com frequência em "Downloading metadata" | qBittorrent, Gluetun | Media Stack, VPN, Network | 2026-11-08 | [qbittorrent_stuck_downloading_metadata.md](known-issues/qbittorrent_stuck_downloading_metadata.md) |
| VM worker2 não reconhece zvol ampliado (40→50GiB) | worker2 (VM), TrueNAS | Storage, VM, Node | não registrada | [worker2_vm_disk_not_resized_after_zvol_expand.md](known-issues/worker2_vm_disk_not_resized_after_zvol_expand.md) |

---

## English

Problems already identified and **not yet resolved** in this homelab. Unlike [`troubleshooting/`](troubleshooting/) (already-resolved problems, with root cause and fix documented), everything listed here is open. Two possible statuses: 🔴 **under investigation** (root cause not yet confirmed, or confirmed but fixable by us) or 🔵 **blocked on upstream** (root cause confirmed, but the fix depends on a third-party project — see the `track-upstream-blocker` skill).

When a problem here gets resolved: write the post-mortem in `troubleshooting/` (see the `create-troubleshooting-doc` skill), add the corresponding row to [`troubleshooting.md`](troubleshooting.md), and remove the entry and the file from here.

| Problem | Component | Tags | Date | Link |
|---|---|---|---|---|
| 🔵 Recyclarr CronJob hangs and fires `KubeJobNotCompleted` (nonexistent `v8` branch, infinite retry loop) | Recyclarr, CronJob, Alertmanager | Media Stack, CronJob, Alertmanager | 2026-11-08 | [recyclarr_cronjob_hangs_kubejobnotcompleted.md](known-issues/recyclarr_cronjob_hangs_kubejobnotcompleted.md) |
| qBittorrent frequently gets stuck on "Downloading metadata" | qBittorrent, Gluetun | Media Stack, VPN, Network | 2026-11-08 | [qbittorrent_stuck_downloading_metadata.md](known-issues/qbittorrent_stuck_downloading_metadata.md) |
| worker2 VM doesn't see the expanded zvol (40→50GiB) | worker2 (VM), TrueNAS | Storage, VM, Node | not recorded | [worker2_vm_disk_not_resized_after_zvol_expand.md](known-issues/worker2_vm_disk_not_resized_after_zvol_expand.md) |
