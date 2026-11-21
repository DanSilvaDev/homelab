# Idea: Raspberry Pi 3 como utilitário 24/7 fora do cluster (Ansible, WoL, monitoramento, nobreak) / Raspberry Pi 3 as a 24/7 utility outside the cluster (Ansible, WoL, monitoring, UPS)

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: Raspberry Pi 3 (hardware já disponível), ciclo de energia (worker1, worker2, worker3, TrueNAS), nobreaks, observabilidade.
**Origem**: sessão de planejamento de 2026-11-19 — o usuário tem uma Raspberry Pi 3 antiga e cogitou usá-la como control-plane (ver [multi_control_plane_ha.md](multi_control_plane_ha.md)).

### Por que não como control-plane

- **1 GB de RAM** (o k3s pede no mínimo 2 GB para um server), **USB 2.0** (limita qualquer SSD; fsync lento, justamente o que o etcd mais usa) e **rede de 100 Mbps** (~300 Mbps no 3B+, dividindo o mesmo barramento USB).
- Existe o modo "etcd-only" do k3s (server sem API server/scheduler/controller-manager), que caberia em 1 GB — mas **não é recomendado**: de madrugada, com os workers desligados, toda escrita no cluster passaria a depender do ack da Pi, prendendo o cluster inteiro à velocidade dela justo quando não há redundância.

### Papel proposto: utilitário 24/7 fora do cluster (~2-3 W)

1. **Nó de controle do Ansible + emissor de Wake-on-LAN** — fica ligado quando todo o resto dorme, respondendo diretamente a pergunta em aberto "onde esse playbook roda?" de [ansible_power_cycle_automation.md](ansible_power_cycle_automation.md).
2. **Monitor externo** (ex: Uptime Kuma) — hoje a observabilidade roda dentro do cluster e cai junto com ele; um monitor fora do cluster responde "quem vigia o vigia". Notificações precisam de um caminho que não dependa do cluster (o `ntfy` atual roda dentro dele).
3. **Detecção de queda de energia + desligamento ordenado** (detalhado abaixo).

Por ser fora do cluster, a convenção de "nodes limpos/neutros" do [`ia/CLAUDE.md`](../ia/CLAUDE.md) não se aplica a ela da mesma forma — mas o ideal é que a configuração dela também fique versionada (ex: o próprio playbook Ansible).

### Detecção de queda de energia

Contexto: há nobreak para worker1 e worker2 (host TrueNAS), com autonomia de ~10-15 min; o worker3 vai ganhar um em breve. O control-plane é um notebook (Lenovo Z40-70) — tem bateria própria.

**Caminho principal: NUT (Network UPS Tools)**
- O nobreak se conecta à Pi via USB/serial; a Pi roda o servidor NUT (`upsd`), lendo o estado do nobreak: na rede (`OL`), na bateria (`OB`), bateria baixa (`LB`).
- **Depende do modelo do nobreak**: precisa ter porta de dados e driver suportado pelo NUT — o suporte varia entre marcas nacionais. **Modelos ainda não levantados.**
- O TrueNAS SCALE tem serviço de UPS nativo (baseado no NUT) que pode atuar como cliente de um servidor NUT remoto — a validar na versão instalada.

**Plano B, se o nobreak não tiver porta de dados: "canário"**
- Um dispositivo ligado **direto na tomada, fora do nobreak** (ex: tomada inteligente/ESP barato, ou um carregador USB ligado a um GPIO da Pi via optoacoplador). Se ele some, faltou energia. Detecta a queda, mas não lê o nível de bateria — o desligamento passa a ser por tempo.

**Política de desligamento proposta** (autonomia de ~10-15 min):
- Na bateria por mais de ~2-3 min (ou `LB`, o que vier primeiro) → iniciar desligamento ordenado.
- **Ordem**: primeiro os clientes NFS (worker3, worker1), por último o TrueNAS (a VM worker2 desliga junto com ele) — evita mounts NFS travados. O control-plane (bateria própria) e a Pi ficam por último.
- Configurar o **graceful node shutdown** do kubelet (`shutdownGracePeriod`, via `--kubelet-arg` no k3s) para os pods terminarem de forma limpa quando o node desliga.
- **Pré-requisito físico**: a Pi **e o roteador/switch** precisam estar no nobreak — sem rede, a Pi não consegue comandar o desligamento dos nodes.

**Volta da energia:**
- Opção A: BIOS "Restore on AC Power Loss = Power On" em cada máquina (simples, mas sem ordem).
- Opção B: a Pi religa via WoL **na ordem certa** — TrueNAS primeiro, workers depois do NFS responder (mesma lógica de [ansible_power_cycle_automation.md](ansible_power_cycle_automation.md)).
- Respeitar a janela noturna: se a energia voltar de madrugada, não religar o que deveria estar desligado.

### Considerações / perguntas em aberto

- Modelos dos nobreaks (atuais e o futuro do worker3) — têm porta de dados? Suportados pelo NUT?
- Qual nobreak alimenta a Pi, o roteador e o switch?
- Boot pelo cartão SD é aceitável para esses usos (sem etcd), mas convém reduzir escritas (logs em RAM) para poupar o cartão.
- Canal de notificação independente do cluster (ex: `ntfy` público, Telegram, e-mail).

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: Raspberry Pi 3 (hardware already available), power cycle (worker1, worker2, worker3, TrueNAS), UPSes, observability.
**Origin**: planning session on 2026-11-19 — the user has an old Raspberry Pi 3 and considered using it as a control-plane (see [multi_control_plane_ha.md](multi_control_plane_ha.md)).

### Why not as a control-plane

- **1 GB of RAM** (k3s asks for at least 2 GB for a server), **USB 2.0** (limits any SSD; slow fsync, exactly what etcd uses most) and **100 Mbps networking** (~300 Mbps on the 3B+, sharing the same USB bus).
- k3s has an "etcd-only" mode (server without API server/scheduler/controller-manager) that would fit in 1 GB — but it's **not recommended**: overnight, with the workers off, every write to the cluster would depend on the Pi's ack, tying the whole cluster to its speed precisely when there's no redundancy.

### Proposed role: 24/7 utility outside the cluster (~2-3 W)

1. **Ansible control node + Wake-on-LAN sender** — stays on while everything else sleeps, directly answering the open question "where does this playbook run?" in [ansible_power_cycle_automation.md](ansible_power_cycle_automation.md).
2. **External monitor** (e.g. Uptime Kuma) — observability currently runs inside the cluster and goes down with it; a monitor outside the cluster answers "who watches the watchers". Notifications need a path that doesn't depend on the cluster (the current `ntfy` runs inside it).
3. **Power outage detection + orderly shutdown** (detailed below).

Since it's outside the cluster, the "clean/neutral nodes" convention in [`ia/CLAUDE.md`](../ia/CLAUDE.md) doesn't apply to it the same way — but ideally its configuration should also be versioned (e.g. the Ansible playbook itself).

### Power outage detection

Context: worker1 and worker2 (TrueNAS host) are on UPSes with ~10-15 min of runtime; worker3 will get one soon. The control-plane is a laptop (Lenovo Z40-70) — it has its own battery.

**Main path: NUT (Network UPS Tools)**
- The UPS connects to the Pi over USB/serial; the Pi runs the NUT server (`upsd`), reading the UPS state: on line (`OL`), on battery (`OB`), low battery (`LB`).
- **Depends on the UPS model**: it needs a data port and a NUT-supported driver — support varies across Brazilian brands. **Models not yet surveyed.**
- TrueNAS SCALE has a native UPS service (NUT-based) that can act as a client of a remote NUT server — to be validated on the installed version.

**Plan B, if the UPS has no data port: "canary"**
- A device plugged **straight into the wall, outside the UPS** (e.g. a cheap smart plug/ESP, or a USB charger wired to a Pi GPIO through an optocoupler). If it disappears, power is out. Detects the outage, but doesn't read battery level — shutdown becomes time-based.

**Proposed shutdown policy** (~10-15 min of runtime):
- On battery for more than ~2-3 min (or `LB`, whichever comes first) → start an orderly shutdown.
- **Order**: NFS clients first (worker3, worker1), TrueNAS last (the worker2 VM goes down with it) — avoids hung NFS mounts. The control-plane (own battery) and the Pi go last.
- Configure the kubelet's **graceful node shutdown** (`shutdownGracePeriod`, via `--kubelet-arg` in k3s) so pods terminate cleanly when the node powers off.
- **Physical prerequisite**: the Pi **and the router/switch** must be on a UPS — without networking, the Pi can't command the nodes to shut down.

**Power restoration:**
- Option A: BIOS "Restore on AC Power Loss = Power On" on each machine (simple, but no ordering).
- Option B: the Pi wakes machines via WoL **in the right order** — TrueNAS first, workers after NFS responds (same logic as [ansible_power_cycle_automation.md](ansible_power_cycle_automation.md)).
- Respect the overnight window: if power comes back in the middle of the night, don't wake what should stay off.

### Considerations / open questions

- UPS models (current ones and worker3's future one) — do they have a data port? Supported by NUT?
- Which UPS powers the Pi, the router and the switch?
- Booting from the SD card is acceptable for these uses (no etcd), but writes should be reduced (logs in RAM) to spare the card.
- Notification channel independent of the cluster (e.g. public `ntfy`, Telegram, e-mail).
