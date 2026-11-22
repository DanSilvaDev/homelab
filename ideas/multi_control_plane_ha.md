# Idea: Adicionar mais control-planes ao cluster (HA real de etcd) / Add more control-plane nodes to the cluster (real etcd HA)

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada. **Adiada** (2026-11-19): depende de um 2º dispositivo ligado 24/7 e de a rede estar estável.
**Componente(s) relacionado(s)**: control-plane, Kube-VIP, etcd, k3s, `coredns-bypass`.
**Origem**: proposta pelo usuário em 2026-11-08. Risco relacionado já documentado anteriormente em [TrueNAS - config.md](../clusters/truenas/config.md) (seção sobre troca de disco do control-plane): "o control-plane é hoje o único nó com etcd do cluster (sem HA de etcd — o Kube-VIP só dá IP estável pra API, não replica o etcd)". Revisitada na sessão de planejamento do worker3 em 2026-11-19 (ver [worker3_gpu_node.md](worker3_gpu_node.md)).

Hoje o cluster tem **um único control-plane** (notebook Lenovo Z40-70, ligado 24/7). O Kube-VIP dá um IP virtual estável pra API do Kubernetes, mas isso é HA de **endereço**, não HA de **etcd** — se esse único node falhar, o cluster inteiro fica inoperante, mesmo com o Kube-VIP ativo. A ideia é adicionar mais nós control-plane pra formar um cluster etcd de verdade.

### Restrição central: o ciclo de desligamento noturno

Workers e TrueNAS desligam de madrugada (ver "Gerenciamento de energia" no [`ia/CLAUDE.md`](../ia/CLAUDE.md)). O etcd precisa de maioria (2 de 3) viva:

- Com **só 1 membro ligado 24/7** (o control-plane atual) e os outros 2 desligando à noite → de madrugada fica 1/3, **o etcd perde quórum e a API para**. A eleição de líder do Kube-VIP depende da API, então os IPs de `LoadBalancer` — incluindo o DNS `192.168.3.200` do `coredns-bypass` — podem deixar de ser anunciados. **Seria pior do que hoje.**
- Com **2 membros 24/7 + 1 que desliga à noite** → de dia 3/3 (tolera 1 falha); de madrugada 2/3 (quórum mantido, mas sem tolerância a falha).
- **Conclusão: HA exige um 2º dispositivo ligado 24/7.** Candidatos: mini PC de baixo consumo (ex: N100, ~6 W em idle) ou Raspberry Pi 4/5 com 4 GB+ bootando por SSD.

### Candidatos avaliados

- **Raspberry Pi 3 — descartado** (2026-11-19): 1 GB de RAM, USB 2.0 (fsync lento para o etcd), rede de 100 Mbps. O modo "etcd-only" do k3s caberia, mas de madrugada toda escrita no cluster dependeria do ack da Pi. Ela ganha outro papel: [raspberry_pi3_utility_node.md](raspberry_pi3_utility_node.md).
- **worker3 — não recomendado**: reinicia com mais frequência (drivers, kernel) e downloads da Steam saturam o NVMe (latência de fsync → eleições de líder no etcd).
- **VM no TrueNAS — descartada**: o TrueNAS desliga de madrugada e é também o storage do cluster (mesmo domínio de falha).
- **worker1 (stacked: server + agent) — candidato a 3º membro**, decisão em aberto. Histórico de instabilidade de rede/hardware: [worker1_eno1_nic_link_down.md](../troubleshooting/worker1_eno1_nic_link_down.md), [worker1_unexpected_poweroff_investigation.md](../troubleshooting/worker1_unexpected_poweroff_investigation.md).

### Pré-requisitos antes de implementar

1. **Rede estável** — o usuário relatou oscilações de rede; etcd com rede instável gera eleições de líder em sequência e pode deixar o cluster **menos** estável do que com um único node. Incidentes já registrados: `eno1` do worker1, [Kube-VIP após bounce de interface](../troubleshooting/kubevip_lost_vips_after_interface_bounce.md), incidente de rede/NFS de 2026-11-17 (ver [troubleshooting.md](../troubleshooting.md)).
2. ~~**Confirmar o tipo de datastore**~~ — ✅ **confirmado em 2026-11-21: SQLite** (via kine), não etcd. Em `/var/lib/rancher/k3s/server/db/` existem `etcd/` e `state.db`, mas só o `state.db`/`state.db-wal` estão sendo modificados (timestamps atuais), nada escuta em `:2379`/`:2380` e `etcd/member/` não existe — o diretório `etcd/` é só um resíduo vazio criado na instalação (2025-10-19). ✅ **Migrado para etcd embutido em 2026-11-22** (`cluster-init: true`, junto com `tls-san: [192.168.3.101]` para futuros servers entrarem pelo VIP) — este pré-requisito está cumprido; ver [control_plane_1/config.md](../clusters/control_plane_1/config.md).
3. **2º dispositivo 24/7** (ver acima).

### Considerações / perguntas em aberto

- **Curto prazo, independente do HA**: snapshots do etcd já são automáticos desde a migração (12 h, 5 últimos), mas ficam no disco do próprio control-plane — falta copiá-los para fora do node (cabe na ideia [truenas_cloud_backup_snapshot_routine.md](truenas_cloud_backup_snapshot_routine.md)). Continua recomendado mesmo com quórum de 3.
- **Papel do Kube-VIP muda**: hoje o DaemonSet `kube-vip-ds` está fixado só no control-plane via `nodeAffinity`; com múltiplos control-planes, passa a rodar em todos com eleição de líder. O node líder precisa estar em rede cabeada (anúncio ARP).
- O `coredns-bypass` hoje só pode rodar no control-plane; com um 2º node 24/7, pode ser distribuído e o DNS deixa de depender de um único node.
- Novos servers devem ingressar pelo IP virtual do Kube-VIP, com `--tls-san` coerente em todos.
- **Resolve diretamente** o risco já apontado em `truenas/config.md`: trocar o disco do control-plane deixa de ser uma manutenção de alto risco.

---

## English

**Status**: Idea — not implemented. **Deferred** (2026-11-19): depends on a second always-on device and on the network being stable.
**Related component(s)**: control-plane, Kube-VIP, etcd, k3s, `coredns-bypass`.
**Origin**: proposed by the user on 2026-11-08. Related risk already documented earlier in [TrueNAS - config.md](../clusters/truenas/config.md) (section about replacing the control-plane's disk): "the control-plane is today the cluster's only node with etcd (no etcd HA — Kube-VIP only gives a stable IP for the API, it doesn't replicate etcd)". Revisited in the worker3 planning session on 2026-11-19 (see [worker3_gpu_node.md](worker3_gpu_node.md)).

Today the cluster has a **single control-plane** (Lenovo Z40-70 laptop, on 24/7). Kube-VIP gives a stable virtual IP for the Kubernetes API, but that's HA for the **address**, not HA for **etcd** — if that single node fails, the whole cluster becomes inoperative, even with Kube-VIP active. The idea is to add more control-plane nodes to form a real etcd cluster.

### Core constraint: the overnight shutdown cycle

Workers and TrueNAS shut down overnight (see "Power management" in [`ia/CLAUDE.md`](../ia/CLAUDE.md)). etcd needs a majority (2 of 3) alive:

- With **only 1 member on 24/7** (the current control-plane) and the other 2 shutting down at night → overnight it's 1/3, **etcd loses quorum and the API stops**. Kube-VIP's leader election depends on the API, so `LoadBalancer` IPs — including `coredns-bypass`'s DNS `192.168.3.200` — may stop being announced. **It would be worse than today.**
- With **2 members on 24/7 + 1 that shuts down at night** → daytime 3/3 (tolerates 1 failure); overnight 2/3 (quorum held, but no fault tolerance).
- **Conclusion: HA requires a second always-on device.** Candidates: a low-power mini PC (e.g. N100, ~6 W idle) or a Raspberry Pi 4/5 with 4 GB+ booting from an SSD.

### Candidates evaluated

- **Raspberry Pi 3 — ruled out** (2026-11-19): 1 GB of RAM, USB 2.0 (slow fsync for etcd), 100 Mbps networking. k3s's "etcd-only" mode would fit, but overnight every write to the cluster would depend on the Pi's ack. It gets another role instead: [raspberry_pi3_utility_node.md](raspberry_pi3_utility_node.md).
- **worker3 — not recommended**: reboots more often (drivers, kernel) and Steam downloads saturate the NVMe (fsync latency → etcd leader elections).
- **VM on TrueNAS — ruled out**: TrueNAS shuts down overnight and is also the cluster's storage (same failure domain).
- **worker1 (stacked: server + agent) — candidate third member**, decision open. History of network/hardware instability: [worker1_eno1_nic_link_down.md](../troubleshooting/worker1_eno1_nic_link_down.md), [worker1_unexpected_poweroff_investigation.md](../troubleshooting/worker1_unexpected_poweroff_investigation.md).

### Prerequisites before implementing

1. **Stable network** — the user reported network fluctuations; etcd on an unstable network causes repeated leader elections and can make the cluster **less** stable than with a single node. Incidents already recorded: worker1's `eno1`, [Kube-VIP after an interface bounce](../troubleshooting/kubevip_lost_vips_after_interface_bounce.md), the network/NFS incident of 2026-11-17 (see [troubleshooting.md](../troubleshooting.md)).
2. ~~**Confirm the datastore type**~~ — ✅ **confirmed on 2026-11-21: SQLite** (via kine), not etcd. `/var/lib/rancher/k3s/server/db/` holds both `etcd/` and `state.db`, but only `state.db`/`state.db-wal` are being modified (current timestamps), nothing listens on `:2379`/`:2380` and `etcd/member/` doesn't exist — the `etcd/` directory is just an empty leftover created at install time (2025-10-19). ✅ **Migrated to embedded etcd on 2026-11-22** (`cluster-init: true`, together with `tls-san: [192.168.3.101]` so future servers can join through the VIP) — this prerequisite is met; see [control_plane_1/config.md](../clusters/control_plane_1/config.md).
3. **Second always-on device** (see above).

### Considerations / open questions

- **Short term, independent of HA**: etcd snapshots are automatic since the migration (every 12 h, last 5 kept), but they live on the control-plane's own disk — they still need to be copied off the node (fits under the [truenas_cloud_backup_snapshot_routine.md](truenas_cloud_backup_snapshot_routine.md) idea). Still recommended even with a 3-node quorum.
- **Kube-VIP's role changes**: today the `kube-vip-ds` DaemonSet is pinned to the control-plane only via `nodeAffinity`; with multiple control-planes, it runs on all of them with leader election. The leader node must be on wired networking (ARP announcements).
- `coredns-bypass` can currently only run on the control-plane; with a second always-on node, it can be spread out and DNS stops depending on a single node.
- New servers should join through the Kube-VIP virtual IP, with a consistent `--tls-san` on all of them.
- **Directly resolves** the risk already flagged in `truenas/config.md`: replacing the control-plane's disk stops being a high-risk maintenance.
