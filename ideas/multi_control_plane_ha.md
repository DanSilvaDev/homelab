# Idea: Adicionar mais control-planes ao cluster (HA real de etcd) / Add more control-plane nodes to the cluster (real etcd HA)

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: control-plane, Kube-VIP, etcd, k3s.
**Origem**: proposta pelo usuário em 2026-11-08. Risco relacionado já documentado anteriormente em [TrueNAS - config.md](../clusters/truenas/config.md) (seção sobre troca de disco do control-plane): "o control-plane é hoje o único nó com etcd do cluster (sem HA de etcd — o Kube-VIP só dá IP estável pra API, não replica o etcd)".

Hoje o cluster tem **um único control-plane** (servidor físico dedicado). O Kube-VIP dá um IP virtual estável pra API do Kubernetes, mas isso é HA de **endereço**, não HA de **etcd** — se esse único node falhar, o etcd (e o cluster inteiro) fica inoperante, mesmo com o Kube-VIP ativo. A ideia é adicionar mais nós control-plane pra formar um cluster etcd de verdade, distribuído entre múltiplos nodes.

### Considerações / perguntas em aberto

- **Quórum de etcd exige número ímpar de control-planes** (3, 5...) — hoje há só 1. O salto mínimo pra HA real seria pra 3.
- **Hardware disponível**: o cluster hoje tem só 3 nodes no total (control-plane físico, worker1 físico, worker2 VM no TrueNAS). Adicionar control-planes dedicados provavelmente exige hardware novo (físico ou VM) — ainda não decidido se viria de uma VM adicional no TrueNAS, um servidor físico novo, ou outra opção.
- **Alternativa "stacked"**: o k3s permite promover os workers atuais a control-plane também (server + agent no mesmo node), mas isso reduz isolamento e soma carga de etcd a nodes que já rodam workloads — especialmente delicado no worker2, que já compartilha a GPU via time-slicing (ver [Media Stack - config.md](../clusters/media-stack/config.md)).
- **Papel do Kube-VIP muda**: hoje ele só dá IP estável pra um único backend real; com múltiplos control-planes, passa a balancear de fato entre vários servers.
- **Resolve diretamente** o risco já apontado em `truenas/config.md`: hoje qualquer manutenção que envolva trocar o disco do control-plane é de alto risco (requer snapshot de etcd e clone de disco, não reinstalação do zero) justamente por ele ser o único nó com etcd.
- Snapshot/backup de etcd continuaria recomendado mesmo com quórum de 3 (reduz risco, mas não elimina a necessidade de backup).

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: control-plane, Kube-VIP, etcd, k3s.
**Origin**: proposed by the user on 2026-11-08. Related risk already documented earlier in [TrueNAS - config.md](../clusters/truenas/config.md) (section about replacing the control-plane's disk): "the control-plane is today the cluster's only node with etcd (no etcd HA — Kube-VIP only gives a stable IP for the API, it doesn't replicate etcd)".

Today the cluster has a **single control-plane** (dedicated physical server). Kube-VIP gives a stable virtual IP for the Kubernetes API, but that's HA for the **address**, not HA for **etcd** — if that single node fails, etcd (and the whole cluster) becomes inoperative, even with Kube-VIP active. The idea is to add more control-plane nodes to form a real distributed etcd cluster across multiple nodes.

### Considerations / open questions

- **etcd quorum requires an odd number of control-planes** (3, 5...) — there's only 1 today. The minimum jump to real HA would be to 3.
- **Available hardware**: the cluster today only has 3 nodes total (physical control-plane, physical worker1, worker2 VM on TrueNAS). Adding dedicated control-planes likely requires new hardware (physical or VM) — not yet decided whether it would come from an additional VM on TrueNAS, a new physical server, or another option.
- **"Stacked" alternative**: k3s allows promoting the current workers to control-plane as well (server + agent on the same node), but this reduces isolation and adds etcd load to nodes that already run workloads — especially delicate on worker2, which already shares its GPU via time-slicing (see [Media Stack - config.md](../clusters/media-stack/config.md)).
- **Kube-VIP's role changes**: today it just gives a stable IP for a single real backend; with multiple control-planes, it starts actually load-balancing across several servers.
- **Directly resolves** the risk already flagged in `truenas/config.md`: today any maintenance involving replacing the control-plane's disk is high-risk (requires an etcd snapshot and a disk clone, not a fresh reinstall) precisely because it's the only node with etcd.
- etcd snapshot/backup would still be recommended even with a 3-node quorum (reduces risk, but doesn't eliminate the need for backups).
