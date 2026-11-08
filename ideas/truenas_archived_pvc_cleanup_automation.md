# Idea: Limpeza automática de pastas `archived-*` no TrueNAS após 30 dias / Automatic cleanup of `archived-*` folders on TrueNAS after 30 days

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: TrueNAS (datasets `ssd-apps/k8s` e `data/k8s`), `nfs-subdir-external-provisioner` (SSD e HDD).
**Origem**: descoberta durante a migração de namespace do Pi-hole (2026-11-08). O `nfs-subdir-external-provisioner` usa `archiveOnDelete` (default `true` do chart, não sobrescrito em `infrastructure/storage/nfs-provisioner-ssd-values.yaml`/`nfs-provisioner-hdd-values.yaml`) — apagar um PVC nunca libera espaço de verdade, só renomeia a pasta com prefixo `archived-`. Confirmado na prática: **7 pastas `archived-*` já acumuladas** de migrações anteriores (ntfy, jellyseerr x2, qbittorrent, radarr, readarr), a mais antiga de 2026-10-26, além da nova do Pi-hole — nenhuma delas nunca foi limpa manualmente.

Criar uma automação que apaga pastas `archived-*` com mais de 30 dias nos datasets `ssd-apps/k8s` e `data/k8s`, evitando o acúmulo manual que já está acontecendo hoje.

### Considerações / perguntas em aberto

- **Onde rodar**: nativo no TrueNAS (Task Scheduler/cron do próprio host, mais simples já que é um host real com shell) vs. um `CronJob` no k8s montando o export NFS diretamente (mantém a automação versionada neste repo, junto com o resto da infra, mas mais indireto).
- **30 dias** é a janela sugerida — dá tempo de notar um problema numa migração antes da limpeza automática apagar os dados de verdade (hoje eles só ficam renomeados, recuperáveis indefinidamente).
- Precisa cobrir **os dois** datasets/provisioners (`ssd-apps/k8s` e `data/k8s`), não só o SSD.
- Alternativa mais simples seria `archiveOnDelete: false` (apaga de vez na hora, sem pasta renomeada) — mas isso elimina a rede de segurança que a arquitetura atual tem; a ideia aqui é manter essa rede de segurança por 30 dias e só então limpar, não eliminar o archiving.

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: TrueNAS (`ssd-apps/k8s` and `data/k8s` datasets), `nfs-subdir-external-provisioner` (SSD and HDD).
**Origin**: discovered during the Pi-hole namespace migration (2026-11-08). `nfs-subdir-external-provisioner` uses `archiveOnDelete` (chart default `true`, not overridden in `infrastructure/storage/nfs-provisioner-ssd-values.yaml`/`nfs-provisioner-hdd-values.yaml`) — deleting a PVC never actually frees space, it just renames the folder with an `archived-` prefix. Confirmed in practice: **7 `archived-*` folders already accumulated** from earlier migrations (ntfy, jellyseerr x2, qbittorrent, radarr, readarr), the oldest from 2026-10-26, plus the new Pi-hole one — none of them were ever cleaned up manually.

Create an automation that deletes `archived-*` folders older than 30 days in the `ssd-apps/k8s` and `data/k8s` datasets, preventing the manual accumulation that's already happening today.

### Considerations / open questions

- **Where to run it**: natively on TrueNAS (the host's own Task Scheduler/cron, simpler since it's a real host with a shell) vs. a k8s `CronJob` mounting the NFS export directly (keeps the automation versioned in this repo alongside the rest of the infra, but more indirect).
- **30 days** is the suggested window — it gives time to notice a problem with a migration before automatic cleanup deletes the data for real (today it just stays renamed, recoverable indefinitely).
- Needs to cover **both** datasets/provisioners (`ssd-apps/k8s` and `data/k8s`), not just the SSD one.
- A simpler alternative would be `archiveOnDelete: false` (deletes for real immediately, no renamed folder) — but that removes the safety net the current architecture has; the idea here is to keep that safety net for 30 days and only then clean up, not eliminate archiving altogether.
