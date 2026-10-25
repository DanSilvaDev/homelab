# Homelab

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Repositório do meu homelab pessoal: um cluster **Kubernetes (k3s)** multi-servidor, gerenciado via **Rancher**, rodando em hardware físico dedicado — incluindo um nó worker com **aceleração GPU (NVIDIA GTX 970 via PCI passthrough)** hospedado como VM num servidor **TrueNAS SCALE**.

Guarda os manifestos, configurações de infraestrutura e a documentação de estado/diagnóstico de todo o ambiente, servindo tanto de referência pessoal quanto de contexto para agentes de IA que ajudam a operar e evoluir o cluster.

### Topologia do cluster

| Nó | Papel | Onde roda |
|---|---|---|
| control-plane | Control-plane (API, etcd), com HA via Kube-VIP | Servidor físico dedicado |
| worker1 | Worker | Servidor físico dedicado |
| worker2 | Worker com GPU (NVIDIA GTX 970) | VM no servidor TrueNAS SCALE (zvol dedicado) |

### Organização dos diretórios

```
apps/                      Manifestos de aplicações rodando no cluster
  pihole/                    Pi-hole (DNS/ad-blocking) - deployment, service, PVC, API gateway
  gpu-test/                  Pod de teste usado para validar alocação de GPU (benchmark CUDA)

clusters/                  Configuração e estado dos nós do cluster
  control_plane_1/           Reservado para manifestos/config do nó control-plane
  kube-vip/                  Manifestos do Kube-VIP (HA do control-plane + LoadBalancer L2/ARP)
  observability/config.md    Estado da stack de monitoring/logging: Prometheus, Grafana, Alertmanager, storage NFS
  truenas/config.md          Estado completo do host TrueNAS: hardware, BIOS, pools e datasets ZFS
  worker2/config.md          Estado completo da VM worker2: hardware, SO, driver GPU, histórico de comandos

infrastructure/            Configuração de ferramentas de infraestrutura do cluster
  rancher/                   Values do Rancher
  monitoring/                Values do kube-prometheus-stack (Prometheus, Grafana, Alertmanager)
  storage/                   Values dos provisioners NFS dinâmicos (TrueNAS SSD e HDD)

troubleshooting/           Post-mortems de problemas reais já diagnosticados e resolvidos
  *.md                       Um arquivo por problema, seguindo um padrão fixo (ver ia/skills/)

ia/                        Contexto e ferramentas para agentes de IA trabalharem neste repositório
  CLAUDE.md                  Contexto geral do projeto (topologia, convenções, onde olhar primeiro)
  skills/                    Skills reutilizáveis específicas deste projeto
    create-troubleshooting-doc/   Gera documentos de troubleshooting no padrão do repositório
    create-pull-request/          Abre PRs no GitHub com descrição bilíngue (PT-BR/EN)

.github/
  PULL_REQUEST_TEMPLATE.md  Template padrão de PR (descrição bilíngue, objetiva)

.claude/skills/            Cópia das skills de ia/skills/ que o Claude Code carrega automaticamente
```

### Convenções

- **`clusters/*/config.md`**: documentam o **estado atual** de cada nó/host (hardware, configuração, versões, log de comandos executados). Atualizados incrementalmente conforme o ambiente muda — são a fonte da verdade mais atual de cada componente.
- **`troubleshooting/*.md`**: documentam **problemas já resolvidos** (sintomas → causa raiz → diagnóstico → correção → lições aprendidas), para evitar re-investigar o mesmo problema no futuro.
- **`ia/`**: ponto de entrada de contexto para agentes de IA (Claude ou outros) que forem trabalhar neste repo — comece por `ia/CLAUDE.md`.

### Status

Em evolução contínua — próximos passos e detalhes de cada componente estão documentados nos respectivos `config.md` dentro de `clusters/`.

---

## English

Repository for my personal homelab: a multi-server **Kubernetes (k3s)** cluster, managed via **Rancher**, running on dedicated physical hardware — including a worker node with **GPU acceleration (NVIDIA GTX 970 via PCI passthrough)** hosted as a VM on a **TrueNAS SCALE** server.

It holds the manifests, infrastructure configuration, and state/diagnostic documentation for the whole environment, serving both as a personal reference and as context for AI agents that help operate and evolve the cluster.

### Cluster topology

| Node | Role | Where it runs |
|---|---|---|
| control-plane | Control-plane (API, etcd), HA via Kube-VIP | Dedicated physical server |
| worker1 | Worker | Dedicated physical server |
| worker2 | Worker with GPU (NVIDIA GTX 970) | VM on the TrueNAS SCALE server (dedicated zvol) |

### Directory layout

```
apps/                      Application manifests running on the cluster
  pihole/                    Pi-hole (DNS/ad-blocking) - deployment, service, PVC, API gateway
  gpu-test/                  Test pod used to validate GPU allocation (CUDA benchmark)

clusters/                  Configuration and state of the cluster nodes
  control_plane_1/           Reserved for control-plane node manifests/config
  kube-vip/                  Kube-VIP manifests (control-plane HA + L2/ARP LoadBalancer)
  observability/config.md    State of the monitoring/logging stack: Prometheus, Grafana, Alertmanager, NFS storage
  truenas/config.md          Full state of the TrueNAS host: hardware, BIOS, ZFS pools and datasets
  worker2/config.md          Full state of the worker2 VM: hardware, OS, GPU driver, command history

infrastructure/            Cluster infrastructure tooling configuration
  rancher/                   Rancher values
  monitoring/                kube-prometheus-stack values (Prometheus, Grafana, Alertmanager)
  storage/                   Dynamic NFS provisioner values (TrueNAS SSD and HDD)

troubleshooting/           Post-mortems of real issues already diagnosed and resolved
  *.md                       One file per issue, following a fixed template (see ia/skills/)

ia/                        Context and tooling for AI agents working in this repository
  CLAUDE.md                  General project context (topology, conventions, where to look first)
  skills/                    Project-specific reusable skills
    create-troubleshooting-doc/   Generates troubleshooting docs following the repository's pattern
    create-pull-request/          Opens GitHub PRs with a bilingual (PT-BR/EN) description

.github/
  PULL_REQUEST_TEMPLATE.md  Default PR template (bilingual, objective description)

.claude/skills/            Mirror of ia/skills/ that Claude Code loads automatically
```

### Conventions

- **`clusters/*/config.md`**: document the **current state** of each node/host (hardware, configuration, versions, executed command log). Updated incrementally as the environment changes — they are the most up-to-date source of truth for each component.
- **`troubleshooting/*.md`**: document **already-resolved issues** (symptoms → root cause → diagnosis → fix → lessons learned), to avoid re-investigating the same problem in the future.
- **`ia/`**: entry point for AI agent context (Claude or others) working on this repo — start with `ia/CLAUDE.md`.

### Status

Continuously evolving — next steps and component details are documented in the respective `config.md` files under `clusters/`.
