# Control-plane (k3s server) - Resumo de Configuração

Host físico: notebook Lenovo Z40-70 (`kubecluster-lenovo-z40-70`), ligado 24/7.
Status: único server do cluster — API, scheduler, controller-manager e datastore **etcd embutido** (migrado de SQLite em 2026-11-22). Sem HA de datastore — ver [ideas/multi_control_plane_ha.md](../../ideas/multi_control_plane_ha.md).

Este arquivo foi criado em 2026-11-22, durante a migração para etcd — o histórico anterior do node está espalhado em `troubleshooting/` e nos `config.md` dos outros componentes. Os subdiretórios deste diretório (`cert-manager/`, `helm/`, etc.) são manifests/config antigos, alguns da época do Rancher.

## Hardware / sistema

- Notebook Lenovo Z40-70, 4 CPUs, ~15 GiB RAM, SSD (troca por SSD de 120 GB planejada — ver [TrueNAS - config.md](../truenas/config.md), "Próximos passos")
- Ubuntu Server 26.04.1, kernel `7.0.0-44-generic` (atualizado de `-43` em 2026-11-22)
- Interface `enp1s0`, IP `192.168.3.100/24` (DHCP + reserva por MAC no roteador). Nessa mesma interface o Kube-VIP adiciona os VIPs `/32`: API `192.168.3.101` e pool de LoadBalancer `192.168.3.200`–`.214`
- Usuário: `kubecluster`

## k3s server

- Versão: `v1.36.5+k3s1`. `ExecStart`: `/usr/local/bin/k3s server --disable servicelb`
- `/etc/rancher/k3s/config.yaml` (as chaves estão todas indentadas em 2 espaços desde a instalação — YAML válido, mas chaves novas precisam seguir a mesma indentação):
  ```yaml
    write-kubeconfig-mode: "0644"
    disable:
      - traefik
    cluster-init: true                  # 2026-11-22: migração SQLite -> etcd embutido
    tls-san:
      - 192.168.3.101                   # VIP do Kube-VIP; antes só estava no cert via dynamiclistener
    node-taint:
      - node-role.kubernetes.io/control-plane:NoSchedule   # só vale no registro do node; o taint atual foi aplicado via kubectl
  ```
- ROLES: `control-plane,etcd,master`
- Taint `node-role.kubernetes.io/control-plane:NoSchedule` desde 2026-11-22 (o k3s **não** aplica por padrão) — quem roda aqui e por quê: ver "Gerenciamento de energia" em [`ia/CLAUDE.md`](../../ia/CLAUDE.md)

## Datastore: etcd embutido

Até 2026-11-21 a documentação assumia etcd, mas o cluster usava **SQLite** (kine) — o server foi instalado sem `--cluster-init`. Migrado em 2026-11-22 reiniciando o server com `cluster-init: true`: o k3s detectou o `state.db`, copiou **1591 chaves** para o etcd em ~3 s e renomeou o SQLite para `state.db.migrated`. Agents não precisaram de re-join.

- etcd `3.6.14`, single-member, escutando em `127.0.0.1` e `192.168.3.100` nas portas `2379` (client) e `2380` (peer); métricas em `127.0.0.1:2381`
- Dados: `/var/lib/rancher/k3s/server/db/etcd/`
- Impacto do restart da migração: os VIPs do Kube-VIP voltaram sozinhos; durante ~10 s a API respondeu `503`/`502` para agents e pods (esperado)

### Snapshots / backup

- Automáticos (padrão do k3s): a cada 12 h, mantém os 5 últimos, em `/var/lib/rancher/k3s/server/db/snapshots/` — **no mesmo disco do node**
- Manual: `sudo k3s etcd-snapshot save --name <nome>`; listar: `sudo k3s etcd-snapshot ls`
- Restore (se precisar): `k3s server --cluster-reset --cluster-reset-restore-path=<snapshot>` com o serviço parado
- Primeiro snapshot: `post-migration-kubecluster-lenovo-z40-70-1795350477` (16,5 MB, 2026-11-22 09:27)
- Os `WARN Unknown flag --... found in config.yaml, skipping` ao rodar `k3s etcd-snapshot` são inofensivos: o subcomando lê o mesmo `config.yaml` do server e ignora as chaves que não se aplicam a ele
- Backup pré-migração (SQLite + `server/tls`, `token`, `cred` + `config.yaml`): `/root/k3s-backup-pre-etcd/` e cópia em `/mnt/data/backups/k3s-control-plane/` no TrueNAS. Contém as chaves da CA e o token do cluster
- Snapshot restaurado precisa do **token** do server (`/var/lib/rancher/k3s/server/token`) — guardar junto de qualquer cópia externa

## Próximos passos

- [x] Migração SQLite → etcd embutido (2026-11-22)
- [x] Taint `NoSchedule` no control-plane + tolerations de Alloy/loki-canary (2026-11-22)
- [ ] Limpeza (por volta de 2026-11-29, se estável): apagar `db/state.db.migrated`, `db/state.db-wal`, `db/state.db-shm` (~56 MB) e `sudo apt remove sqlite3` (instalado só para os backups pré-worker3/pré-etcd — convenção 5 do [`ia/CLAUDE.md`](../../ia/CLAUDE.md)); depois `/root/k3s-backup-pre-worker3/` e `/root/k3s-backup-pre-etcd/` também podem sair (a cópia no TrueNAS fica)
- [ ] Copiar os snapshots automáticos para fora do node (S3/TrueNAS) — ver [ideas/truenas_cloud_backup_snapshot_routine.md](../../ideas/truenas_cloud_backup_snapshot_routine.md)
- [ ] CRDs órfãs do Cluster API (`*.cluster.x-k8s.io`, sobra do Rancher) apareceram na migração — avaliar remoção
- [ ] Atualizar worker1 e worker2 para o kernel `7.0.0-44` (um node por vez), para ficarem iguais ao control-plane e ao worker3

## Log de comandos executados

```bash
# 2026-11-21 — confirmação do datastore (SQLite) e backup pré-worker3
sudo ls -la /var/lib/rancher/k3s/server/db/
sudo ss -ltnp | grep -E ':2379|:2380'                       # vazio = sem etcd
sudo apt install -y sqlite3                                  # temporário
sudo install -d -m 700 /root/k3s-backup-pre-worker3
sudo sqlite3 /var/lib/rancher/k3s/server/db/state.db ".backup '/root/k3s-backup-pre-worker3/state.db'"
sudo sqlite3 /root/k3s-backup-pre-worker3/state.db "PRAGMA integrity_check;"
sudo tar -C /var/lib/rancher/k3s/server -czf /root/k3s-backup-pre-worker3/server-tls-token-cred.tgz tls token cred

# 2026-11-22 — reboot para kernel 7.0.0-44
cat /var/run/reboot-required.pkgs                            # linux-image-7.0.0-44-generic, linux-base
sudo reboot

# 2026-11-22 — taint (kubectl a partir do PC)
kubectl taint node kubecluster-lenovo-z40-70 node-role.kubernetes.io/control-plane=:NoSchedule

# 2026-11-22 — migração SQLite -> etcd
sudo install -d -m 700 /root/k3s-backup-pre-etcd
sudo sqlite3 /var/lib/rancher/k3s/server/db/state.db ".backup '/root/k3s-backup-pre-etcd/state.db'"
sudo sqlite3 /root/k3s-backup-pre-etcd/state.db "PRAGMA integrity_check;"
sudo tar -C /var/lib/rancher/k3s/server -czf /root/k3s-backup-pre-etcd/server-tls-token-cred.tgz tls token cred
sudo cp /etc/rancher/k3s/config.yaml /root/k3s-backup-pre-etcd/config.yaml
sudo tar -C /root -czf ~/k3s-backup-pre-etcd.tgz k3s-backup-pre-etcd && sudo chown "$USER": ~/k3s-backup-pre-etcd.tgz
#   [PC, Git Bash] scp -3 ... admin@192.168.3.103:/mnt/data/backups/k3s-control-plane/  + sha256sum nas duas pontas
rm ~/k3s-backup-pre-etcd.tgz
printf '  cluster-init: true\n  tls-san:\n    - 192.168.3.101\n  node-taint:\n    - node-role.kubernetes.io/control-plane:NoSchedule\n' | sudo tee -a /etc/rancher/k3s/config.yaml
sudo python3 -c 'import yaml; print(yaml.safe_load(open("/etc/rancher/k3s/config.yaml")))'
sudo systemctl restart k3s
sudo journalctl -u k3s --since "09:19" --no-pager | grep -c 'Migrating etcd key'      # 1591
sudo ss -ltnp | grep -E ':2379|:2380'
kubectl get nodes                                            # control-plane,etcd,master
ip -br a show enp1s0                                         # VIPs de volta
sudo k3s etcd-snapshot save --name post-migration
sudo k3s etcd-snapshot ls
```
