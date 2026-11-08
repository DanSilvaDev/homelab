# Idea: Automação via Ansible do ciclo de desligar/religar workers + TrueNAS / Ansible automation for the worker + TrueNAS power cycle

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: worker1, worker2, TrueNAS, coredns-bypass (Pi-hole failover).
**Origem**: mencionada desde o plano de economia de energia (README.md, `ia/CLAUDE.md`), 2026-10-31.

Os servidores worker (worker1, worker2) e o host TrueNAS são desligados durante a madrugada para economizar energia — o control-plane permanece ligado 24/7. Hoje esse ciclo de desligar/religar é **manual**; a ideia é automatizá-lo via **Ansible**.

### Por que isso não é trivial

Não é só desligar/ligar máquinas — precisa coordenar com o mecanismo de failover de DNS já existente (`coredns-bypass`, ver `apps/core-dns/` e a seção "Gerenciamento de energia" do `README.md`):

- **Ao desligar**: só desligar TrueNAS/workers depois de confirmar que o `coredns-bypass` já assumiu o IP do Kube-VIP (failover de DNS completo), para não haver uma janela sem resolução de DNS na rede doméstica.
- **Ao religar**: só religar o Pi-hole (ou deixar que ele volte a receber tráfego) depois que o TrueNAS e o storage NFS já estiverem de volta — caso contrário o Pi-hole sobe e fica preso esperando um storage que ainda não respondeu.

### Perguntas em aberto

- O playbook dispara por horário fixo (cron) ou por algum outro gatilho?
- Como verificar programaticamente que o failover de DNS completou antes de prosseguir com o desligamento?
- Onde esse playbook roda (precisa estar em uma máquina que fica ligada 24/7, ex: o próprio control-plane)?

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: worker1, worker2, TrueNAS, coredns-bypass (Pi-hole failover).
**Origin**: mentioned since the power-saving plan (README.md, `ia/CLAUDE.md`), 2026-10-31.

The worker servers (worker1, worker2) and the TrueNAS host are shut down overnight to save power — the control-plane stays on 24/7. Today this shutdown/startup cycle is **manual**; the idea is to automate it with **Ansible**.

### Why this isn't trivial

It's not just turning machines off/on — it needs to coordinate with the existing DNS failover mechanism (`coredns-bypass`, see `apps/core-dns/` and the "Power management" section of `README.md`):

- **On shutdown**: only power off TrueNAS/workers after confirming `coredns-bypass` has already taken over the Kube-VIP IP (DNS failover complete), so there's no window without DNS resolution on the home network.
- **On startup**: only let Pi-hole take back traffic after TrueNAS and NFS storage are already back up — otherwise Pi-hole comes up and gets stuck waiting on storage that isn't responding yet.

### Open questions

- Does the playbook trigger on a fixed schedule (cron) or some other trigger?
- How to programmatically verify DNS failover completed before proceeding with shutdown?
- Where does this playbook run (needs to be on a machine that's on 24/7, e.g. the control-plane itself)?
