# Idea: Migrar o datastore do control-plane de SQLite para etcd embutido / Migrate the control-plane datastore from SQLite to embedded etcd

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada. **Prevista para logo após o worker3 entrar no cluster** (decisão do operador em 2026-11-21), como passo separado, depois que o node novo estiver estável.
**Componente(s) relacionado(s)**: control-plane (k3s server, `v1.36.5+k3s1`), Kube-VIP, `coredns-bypass` (DNS `192.168.3.200`), backup do datastore.
**Origem**: pré-checagens da entrada do worker3 (2026-11-21, ver [worker3_gpu_node.md](worker3_gpu_node.md)). A checagem do datastore — pré-requisito em aberto de [multi_control_plane_ha.md](multi_control_plane_ha.md) — mostrou que o cluster usa **SQLite** (via kine), não etcd como a documentação assumia: só `state.db`/`state.db-wal` estão sendo modificados, nada escuta em `:2379`/`:2380` e `db/etcd/member/` não existe (o diretório `db/etcd/` é um resíduo vazio da instalação, 2025-10-19).

O k3s suporta converter um server SQLite para etcd embutido reiniciando-o com `--cluster-init` (suportado desde a v1.22): na primeira subida ele detecta o `state.db` e migra os dados. Não exige reinstalar o server nem refazer o join dos agents.

### Por que migrar

- **Snapshots automáticos embutidos**: com etcd, o k3s tira snapshot por padrão a cada 12h e mantém os 5 últimos (`--etcd-snapshot-schedule-cron`, `--etcd-snapshot-retention`), com opção de envio para S3. Com SQLite, `k3s etcd-snapshot` não funciona — o backup hoje é manual (`sqlite3 ... ".backup"` + `server/tls`, `server/token`, `server/cred`).
- **Restore padronizado** (`k3s server --cluster-reset --cluster-reset-restore-path=<snapshot>`) — torna mais segura a **troca de disco do control-plane** já planejada em [TrueNAS - config.md](../clusters/truenas/config.md) (seção "Mover o SSD do control-plane para o TrueNAS"). **Fazer a migração antes dessa troca.**
- **Pré-requisito do HA**: adicionar qualquer server exige etcd — tira esse passo do caminho de [multi_control_plane_ha.md](multi_control_plane_ha.md).

### Esboço da execução

1. Backup do SQLite antes: `sqlite3 state.db ".backup '<destino>'"` + `PRAGMA integrity_check` + tar de `server/tls`, `server/token`, `server/cred` (mesmo procedimento usado nas pré-checagens do worker3).
2. Configuração atual do server (levantada em 2026-11-21): `/etc/rancher/k3s/config.yaml` tem `write-kubeconfig-mode: "0644"` e `disable: [traefik]` — **com todas as chaves indentadas em 2 espaços**, então `cluster-init: true` precisa entrar com a mesma indentação; o `ExecStart` tem só `--disable servicelb`. **Não há `tls-san`** em nenhum dos dois — o certificado da API inclui `192.168.3.101` hoje só porque o dynamiclistener do k3s o adicionou automaticamente (confirmado via `openssl s_client` em 2026-11-21, junto com IPs antigos como `192.168.3.78` e vários IPv6 SLAAC do próprio node). Adicionar `tls-san: [192.168.3.101]` (VIP do Kube-VIP) junto com o `cluster-init`, já que é pré-requisito para futuros servers ingressarem pelo VIP.
3. Adicionar `cluster-init: true` e `sudo systemctl restart k3s`.
4. Validar: `db/etcd/member/` existe, `:2379`/`:2380` escutando, `kubectl get nodes` com ROLES incluindo `etcd`, VIPs `.101`/`.200` respondendo.
5. `sudo k3s etcd-snapshot save` para confirmar que snapshots funcionam.

### Considerações / perguntas em aberto

- **Sozinho, não dá HA**: um etcd de um membro continua sendo ponto único de falha. HA real continua dependendo do 2º dispositivo 24/7 ([multi_control_plane_ha.md](multi_control_plane_ha.md)).
- **Restart derruba os VIPs do Kube-VIP por alguns segundos**: `vip_renewdeadline: 10` em `clusters/kube-vip/arp.yaml` — o restart do k3s passa disso, então o Kube-VIP perde o lease e para de anunciar os VIPs, incluindo o DNS `192.168.3.200` da rede toda. Já houve incidente de VIPs que não voltaram sozinhos ([kubevip_lost_vips_after_interface_bounce.md](../troubleshooting/kubevip_lost_vips_after_interface_bounce.md)). Fazer em horário tranquilo e conferir os VIPs logo depois.
- **Mão única na prática**: não há caminho oficial etcd → SQLite; rollback = restaurar o backup do SQLite, perdendo o que mudou depois.
- **Mais RAM e I/O com fsync** no control-plane (Lenovo Z40-70, 15 GiB, SSD) — tranquilo hoje; reavaliar quando o control-plane passar para o SSD de 120 GB.
- **Destino dos snapshots**: ficam no disco local do control-plane por padrão — copiar para fora do node se encaixa em [truenas_cloud_backup_snapshot_routine.md](truenas_cloud_backup_snapshot_routine.md). S3 via TrueNAS?
- **`sqlite3`** foi instalado temporariamente no control-plane para o backup pré-worker3 (convenção 5 do [`ia/CLAUDE.md`](../ia/CLAUDE.md)) — depois da migração deixa de ser necessário e pode ser removido.

---

## English

**Status**: Idea — not implemented. **Planned for right after worker3 joins the cluster** (operator's decision on 2026-11-21), as a separate step, once the new node is stable.
**Related component(s)**: control-plane (k3s server, `v1.36.5+k3s1`), Kube-VIP, `coredns-bypass` (DNS `192.168.3.200`), datastore backup.
**Origin**: worker3 join pre-checks (2026-11-21, see [worker3_gpu_node.md](worker3_gpu_node.md)). The datastore check — an open prerequisite in [multi_control_plane_ha.md](multi_control_plane_ha.md) — showed the cluster uses **SQLite** (via kine), not etcd as the documentation assumed: only `state.db`/`state.db-wal` are being modified, nothing listens on `:2379`/`:2380` and `db/etcd/member/` doesn't exist (the `db/etcd/` directory is an empty leftover from the install, 2025-10-19).

k3s supports converting a SQLite server to embedded etcd by restarting it with `--cluster-init` (supported since v1.22): on first start it detects `state.db` and migrates the data. No server reinstall or agent re-join needed.

### Why migrate

- **Built-in automatic snapshots**: with etcd, k3s takes a snapshot every 12h by default and keeps the last 5 (`--etcd-snapshot-schedule-cron`, `--etcd-snapshot-retention`), with an option to ship them to S3. With SQLite, `k3s etcd-snapshot` doesn't work — backups are manual today (`sqlite3 ... ".backup"` + `server/tls`, `server/token`, `server/cred`).
- **Standardized restore** (`k3s server --cluster-reset --cluster-reset-restore-path=<snapshot>`) — makes the **control-plane disk swap** already planned in [TrueNAS - config.md](../clusters/truenas/config.md) (section "Move the control-plane SSD to TrueNAS") safer. **Do the migration before that swap.**
- **HA prerequisite**: adding any server requires etcd — takes this step out of the way of [multi_control_plane_ha.md](multi_control_plane_ha.md).

### Execution outline

1. Back up SQLite first: `sqlite3 state.db ".backup '<destination>'"` + `PRAGMA integrity_check` + tar of `server/tls`, `server/token`, `server/cred` (same procedure used in the worker3 pre-checks).
2. Current server configuration (surveyed on 2026-11-21): `/etc/rancher/k3s/config.yaml` has `write-kubeconfig-mode: "0644"` and `disable: [traefik]` — **with every key indented by 2 spaces**, so `cluster-init: true` must use the same indentation; `ExecStart` has only `--disable servicelb`. **There's no `tls-san`** in either — the API certificate includes `192.168.3.101` today only because k3s's dynamiclistener added it automatically (confirmed via `openssl s_client` on 2026-11-21, along with old IPs like `192.168.3.78` and several of the node's own SLAAC IPv6 addresses). Add `tls-san: [192.168.3.101]` (the Kube-VIP VIP) together with `cluster-init`, since it's a prerequisite for future servers to join through the VIP.
3. Add `cluster-init: true` and `sudo systemctl restart k3s`.
4. Validate: `db/etcd/member/` exists, `:2379`/`:2380` listening, `kubectl get nodes` ROLES includes `etcd`, VIPs `.101`/`.200` responding.
5. `sudo k3s etcd-snapshot save` to confirm snapshots work.

### Considerations / open questions

- **No HA on its own**: a single-member etcd is still a single point of failure. Real HA still depends on the second always-on device ([multi_control_plane_ha.md](multi_control_plane_ha.md)).
- **The restart drops Kube-VIP's VIPs for a few seconds**: `vip_renewdeadline: 10` in `clusters/kube-vip/arp.yaml` — the k3s restart takes longer than that, so Kube-VIP loses its lease and stops announcing the VIPs, including the whole network's DNS `192.168.3.200`. VIPs have failed to come back on their own before ([kubevip_lost_vips_after_interface_bounce.md](../troubleshooting/kubevip_lost_vips_after_interface_bounce.md)). Do it at a quiet time and check the VIPs right after.
- **One-way in practice**: there's no official etcd → SQLite path; rollback = restore the SQLite backup, losing whatever changed afterwards.
- **More RAM and fsync I/O** on the control-plane (Lenovo Z40-70, 15 GiB, SSD) — fine today; re-evaluate when the control-plane moves to the 120 GB SSD.
- **Snapshot destination**: stored on the control-plane's local disk by default — copying them off the node fits [truenas_cloud_backup_snapshot_routine.md](truenas_cloud_backup_snapshot_routine.md). S3 via TrueNAS?
- **`sqlite3`** was installed temporarily on the control-plane for the pre-worker3 backup (convention 5 in [`ia/CLAUDE.md`](../ia/CLAUDE.md)) — after the migration it's no longer needed and can be removed.
