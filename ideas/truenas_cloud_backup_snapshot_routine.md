# Idea: Rotina de snapshot + backup em nuvem do TrueNAS / TrueNAS snapshot + cloud backup routine

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: TrueNAS (pools `ssd-apps` e `data`).
**Origem**: já estava registrada como item em aberto no "Próximos passos" de [TrueNAS - config.md](../clusters/truenas/config.md) ("Planejar backup em nuvem do TrueNAS") — promovida pra cá com o complemento de incluir também uma rotina de snapshot local, não só o backup externo.

Hoje não existe nenhuma cópia dos dados do TrueNAS fora do host físico, nem uma rotina programada de snapshots ZFS. Numa catástrofe (incêndio, roubo, falha simultânea de disco) ou mesmo um erro humano (deleção acidental), os dois pools (`ssd-apps`, `data`) e tudo que depende deles (PVs dinâmicos do cluster, biblioteca de mídia, VM do worker2) seriam perdidos sem recuperação.

A ideia tem duas partes complementares:
1. **Snapshots locais programados** (ZFS, nativo do TrueNAS — `Data Protection` → `Periodic Snapshot Tasks`): proteção rápida contra erro humano/corrupção lógica, sem depender de rede/nuvem, com rollback quase instantâneo.
2. **Backup/replicação em nuvem**: cópia fora do host físico, protegendo contra perda total do hardware/local (incêndio, roubo, falha simultânea de disco).

### Considerações / perguntas em aberto

- **Destino da nuvem**: Backblaze B2, AWS S3 Glacier, ou outro provedor compatível com `rclone`/a task nativa de Cloud Sync do TrueNAS — avaliar custo de armazenamento + egress pro volume real de dados.
- **Escopo**: quais datasets realmente precisam de backup externo — provavelmente não o `k8s` (PVs dinâmicos recriáveis via manifests + reinstalação de apps), mas sim `Backups`, `databases`, e qualquer dado irrecriável (ex: biblioteca de mídia adquirida, se o custo de re-obter for alto).
- **Frequência/retenção**: tanto pros snapshots locais (ex: horário/diário com retenção curta, semanal/mensal com retenção mais longa) quanto pro backup externo (provavelmente menos frequente que os snapshots locais, dado o custo de egress).
- **Relação com o split do dataset `Backups`**: a [ideia já registrada de dividir `Backups` em sub-datasets por fonte](../ideas.md) (`smb-pc`, `nfs-k3s`) facilitaria aplicar políticas de snapshot/retenção diferentes por origem dos dados — vale considerar as duas ideias juntas.
- **etcd do control-plane**: já documentado separadamente em `clusters/truenas/config.md` (item sobre mover o SSD do control-plane) que o cluster não tem HA de etcd — um `k3s etcd-snapshot save` periódico também merece entrar nesse guarda-chuva de "rotina de backup", mesmo não sendo um dataset do TrueNAS.

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: TrueNAS (`ssd-apps` and `data` pools).
**Origin**: was already logged as an open item in [TrueNAS - config.md](../clusters/truenas/config.md)'s "Next steps" ("Plan cloud backup for TrueNAS") — promoted here with the addition of also including a local snapshot routine, not just the offsite backup.

There's currently no copy of TrueNAS's data outside the physical host, nor a scheduled ZFS snapshot routine. In a disaster (fire, theft, simultaneous disk failure) or even human error (accidental deletion), both pools (`ssd-apps`, `data`) and everything depending on them (the cluster's dynamic PVs, the media library, the worker2 VM) would be lost with no recovery.

The idea has two complementary parts:
1. **Scheduled local snapshots** (ZFS, native to TrueNAS — `Data Protection` → `Periodic Snapshot Tasks`): quick protection against human error/logical corruption, with no dependency on network/cloud, with near-instant rollback.
2. **Cloud backup/replication**: a copy outside the physical host, protecting against total hardware/site loss (fire, theft, simultaneous disk failure).

### Considerations / open questions

- **Cloud destination**: Backblaze B2, AWS S3 Glacier, or another provider compatible with `rclone`/TrueNAS's native Cloud Sync task — evaluate storage + egress cost for the actual data volume.
- **Scope**: which datasets actually need offsite backup — probably not `k8s` (dynamic PVs, recreatable via manifests + app reinstallation), but `Backups`, `databases`, and any non-recreatable data (e.g. an acquired media library, if the cost of re-obtaining it is high).
- **Frequency/retention**: both for local snapshots (e.g. hourly/daily with short retention, weekly/monthly with longer retention) and for the offsite backup (probably less frequent than local snapshots, given egress cost).
- **Relationship with the `Backups` dataset split**: the [already-logged idea to split `Backups` into per-source sub-datasets](../ideas.md) (`smb-pc`, `nfs-k3s`) would make it easier to apply different snapshot/retention policies per data source — worth considering the two ideas together.
- **control-plane etcd**: already separately documented in `clusters/truenas/config.md` (the item about moving the control-plane's SSD) that the cluster has no etcd HA — a periodic `k3s etcd-snapshot save` also deserves to fall under this "backup routine" umbrella, even though it's not a TrueNAS dataset.
