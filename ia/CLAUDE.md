# Contexto do Projeto - Homelab (TrueNAS + Kubernetes)

Este arquivo dá contexto rápido para qualquer agente de IA (Claude ou outro) que for trabalhar neste repositório.

## Visão geral

Homelab pessoal com um cluster **k3s** multi-servidor, rodando em hardware físico separado. Um dos servidores (TrueNAS SCALE) hospeda uma VM que atua como worker node com aceleração GPU via PCI passthrough.

O cluster já foi gerenciado via Rancher; removido em 2026-10-28 (observability stack própria já cobria o que o Rancher oferecia, e sua remoção também resolveu um conflito de ownership de CRD entre `traefik-crd` e `gateway-api-crd` — ver [troubleshooting/rancher_removal_traefik_crd_conflict.md](../troubleshooting/rancher_removal_traefik_crd_conflict.md)). O Traefik (dependência do Rancher) também foi desabilitado no k3s.

## Estrutura do repositório

```
clusters/
  control_plane_1/      -> nó control-plane (servidor físico separado)
  worker2/config.md      -> estado atual completo da VM worker2 (hospedada no TrueNAS)
  truenas/config.md      -> estado atual completo do host TrueNAS (hardware, BIOS, pools ZFS)
  observability/config.md -> estado da stack de monitoring/logging (Prometheus, Grafana, Alertmanager, storage NFS)
  kube-vip/               -> manifestos do Kube-VIP (HA do control-plane + LoadBalancer L2/ARP)
apps/
  pihole/                 -> manifestos do Pi-hole rodando no cluster
  core-dns/               -> CoreDNS fallback que assume o IP do Pi-hole via Kube-VIP (ver seção "Gerenciamento de energia" abaixo)
troubleshooting/
  *.md                    -> post-mortems de problemas reais já resolvidos (ver convenção abaixo)
ia/
  CLAUDE.md               -> este arquivo
  skills/                 -> skills reutilizáveis para agentes de IA trabalharem neste repo
```

## Topologia do cluster k3s

| Nó | Papel | Localização |
|---|---|---|
| control-plane | Control-plane (API, etcd) | Servidor físico dedicado |
| worker1 | Worker | Servidor físico dedicado |
| worker2 | Worker (com GPU GTX 970 via passthrough) | VM no servidor TrueNAS SCALE |

Antes de propor mudanças de infraestrutura, **sempre leia primeiro** `clusters/worker2/config.md`, `clusters/truenas/config.md` e, se a mudança envolver monitoring/logging/storage NFS, `clusters/observability/config.md` — eles contêm o estado real e atualizado de hardware, storage ZFS, configuração da VM e do cluster, incluindo o log de comandos já executados.

## Convenções importantes

1. **Documentação de estado** (`clusters/*/config.md`): formato Markdown, atualizado incrementalmente conforme mudanças são feitas — nunca reescrito do zero. Sempre mantenha o checklist de "Próximos passos" e o "Log de comandos executados" sincronizados com a realidade.
2. **Documentação de troubleshooting** (`troubleshooting/*.md`): post-mortems de problemas não-óbvios já resolvidos, seguindo uma estrutura fixa (cenário/sintomas → investigação/causa raiz → comandos de diagnóstico → correção → lições aprendidas). Ver a skill `create-troubleshooting-doc` em `ia/skills/` para o template exato e quando usá-la.
3. **Versões de k3s**: sem o Rancher, não há mais trava externa de minor version — mas ainda confirme compatibilidade (chart versions, drivers) antes de sugerir upgrades.
4. **Branch de driver NVIDIA**: a GTX 970 (Maxwell) só é suportada até a branch `580` do driver NVIDIA — não sugerir upgrade para branches `590+` nesse hardware.

## Gerenciamento de energia (em andamento)

Os workers (worker1, worker2) **e o host TrueNAS** serão desligados durante a madrugada para economizar energia. O control-plane permanece ligado 24/7 (hospeda Kube-VIP e a API do cluster) e passou a ser o único node onde o `coredns-bypass` (`apps/core-dns/`) pode rodar.

O Pi-hole, por sua vez, tem **anti-afinidade explícita** contra o control-plane (`apps/pihole/deployment.yaml`) — e essa restrição não é arbitrária: o PVC do Pi-hole é servido via NFS pelo TrueNAS (`truenas-nfs-ssd`), então com o TrueNAS desligado o storage dele fica inacessível de qualquer forma. Não faz sentido permitir que o pod suba no control-plane só pra ficar preso em `ContainerCreating`/`CrashLoopBackOff` esperando um storage que não vai responder; o esperado é que o Pi-hole simplesmente fique indisponível durante a madrugada, junto com os workers e o TrueNAS.

- `coredns-bypass` e `pihole-deployment` compartilham a label `server: dns-provider`, selecionada pelo Service `dns-service-lb` (`apps/core-dns/service.yaml`), fixo no IP do Kube-VIP `192.168.3.200`.
- O failover depende do `readinessProbe` do Pi-hole (porta 93/TCP): quando ele fica `NotReady` (workers/TrueNAS desligados ou Pi-hole fora do ar), o Service passa a rotear só pro `coredns-bypass`, que faz apenas forward puro pra Cloudflare (sem ad-blocking, sem registros locais).
- **Não é só um bypass de DNS** — é parte de um plano maior de economia de energia, onde workers e TrueNAS saem do ar junto. Automação do ciclo desligar/religar via **Ansible** ainda não existe neste repositório; se for implementada, provavelmente precisará coordenar com esse mecanismo de failover (ex: desligar TrueNAS/workers só depois de confirmar o failover do DNS, religar o Pi-hole só depois do TrueNAS/NFS já estarem de volta).

## Diretório `ia/skills/`

Contém skills reutilizáveis específicas deste projeto. Para que uma skill seja carregável automaticamente pelo Claude Code via `Skill` tool, ela também precisa estar referenciada em `.claude/skills/` (ou um link/cópia lá) — os arquivos aqui em `ia/skills/` servem como fonte versionada junto ao projeto, mas podem precisar ser copiados/linkados para o diretório de configuração do Claude Code para serem auto-descobertos.

**Skills atualmente espelhadas em `.claude/skills/`** (mantenha as duas cópias em sincronia ao editar):
- `create-troubleshooting-doc` → espelhada em `.claude/skills/create-troubleshooting-doc/SKILL.md`
- `create-pull-request` → espelhada em `.claude/skills/create-pull-request/SKILL.md` (usa o template `.github/PULL_REQUEST_TEMPLATE.md`)
