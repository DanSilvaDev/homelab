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
troubleshooting.md        -> índice bilíngue de troubleshooting/ (tabela: arquivo, data, componente, tags, resumo)
known-issues.md           -> índice bilíngue de problemas ativos, ainda não resolvidos
known-issues/             -> um arquivo por problema ativo que precisa de mais contexto que uma linha de tabela
ideas.md                  -> índice bilíngue do backlog de ideias/melhorias futuras ainda não implementadas
ideas/                    -> um arquivo por ideia que já precisa de mais contexto/arquitetura (equivalente a um plan.md antes da implementação)
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
2. **Documentação de troubleshooting** (`troubleshooting/*.md`): post-mortems de problemas não-óbvios já resolvidos, seguindo uma estrutura fixa (cenário/sintomas → investigação/causa raiz → comandos de diagnóstico → correção → lições aprendidas). Ver a skill `create-troubleshooting-doc` em `ia/skills/` para o template exato e quando usá-la. Indexados em [`troubleshooting.md`](../troubleshooting.md).
3. **Versões de k3s**: sem o Rancher, não há mais trava externa de minor version — mas ainda confirme compatibilidade (chart versions, drivers) antes de sugerir upgrades.
4. **Branch de driver NVIDIA**: a GTX 970 (Maxwell) só é suportada até a branch `580` do driver NVIDIA — não sugerir upgrade para branches `590+` nesse hardware.
5. **Manter os nodes limpos/neutros**: o objetivo é manter os nodes do cluster o mais parecidos possível entre si, com o mínimo de diferenças — só o que é realmente necessário pro funcionamento correto fica instalado permanentemente. Quando um pacote for instalado num node (via `apt install` ou similar) só para ajudar a diagnosticar um problema (ex: `conntrack`, `tcpdump`), ao final da investigação — quando a causa raiz for encontrada e resolvida — liste explicitamente quais pacotes foram instalados e em quais nodes, e sugira a desinstalação deles. A decisão de manter ou remover é sempre do operador (nunca desinstalar automaticamente sem confirmação), mas a sugestão deve ser feita proativamente, não só se perguntado.
6. **Problemas ativos** (`known-issues.md` / `known-issues/`): problemas já identificados mas **ainda não resolvidos** — diferente de `troubleshooting/`, que só documenta o que já foi resolvido. Ao identificar um problema não-trivial que não será resolvido na mesma sessão, use a skill `create-known-issue` para registrá-lo. Se a causa raiz for confirmada mas a correção depender de um projeto de terceiros (bug upstream), use a skill `track-upstream-blocker` para marcar o status como 🔵 e deixar um ponto de acompanhamento claro (em vez de ficar 🔴 "em investigação" indefinidamente). Quando o problema for resolvido, use a skill `create-troubleshooting-doc` para escrever o post-mortem em `troubleshooting/`, adicione a linha em `troubleshooting.md`, e **remova** a entrada/arquivo de `known-issues.md`/`known-issues/`.
7. **Ideias futuras** (`ideas.md` / `ideas/`): backlog de ideias/melhorias que surgem durante implementação ou uso, mas que não serão implementadas na sessão atual. Use a skill `create-idea` para registrar. Quando implementada, a ideia **sai** do backlog (a implementação real já gera seus próprios arquivos/PR) — não mantenha histórico de ideias implementadas aqui.

## Gerenciamento de energia (em andamento)

Os workers (worker1, worker2) **e o host TrueNAS** serão desligados durante a madrugada para economizar energia. O control-plane permanece ligado 24/7 (hospeda Kube-VIP e a API do cluster) e passou a ser o único node onde o `coredns-bypass` (`apps/core-dns/`) pode rodar.

O Pi-hole, por sua vez, tem **anti-afinidade explícita** contra o control-plane (`apps/pihole/deployment.yaml`) — e essa restrição não é arbitrária: o PVC do Pi-hole é servido via NFS pelo TrueNAS (`truenas-nfs-ssd`), então com o TrueNAS desligado o storage dele fica inacessível de qualquer forma. Não faz sentido permitir que o pod suba no control-plane só pra ficar preso em `ContainerCreating`/`CrashLoopBackOff` esperando um storage que não vai responder; o esperado é que o Pi-hole simplesmente fique indisponível durante a madrugada, junto com os workers e o TrueNAS.

- Ambos rodam no namespace `dns-services`. O `coredns-bypass` é o **único listener real de DNS**, fixo no IP do Kube-VIP `192.168.3.200` (`Service` `dns-service-lb`, `apps/core-dns/service.yaml`) — a UI de admin do Pi-hole tem seu próprio `Service`/IP (`pihole-ui`, `192.168.3.214`, `apps/pihole/service.yaml`). Dois `Service` nunca podem pedir o mesmo `loadbalancerIP` (ver `troubleshooting/pihole_dns_wrong_loadbalancer_ip.md`: o segundo é realocado silenciosamente para outro IP do pool).
- O failover é feito pelo próprio CoreDNS: o Corefile do `coredns-bypass` (`apps/core-dns/config-map.yaml`) usa `forward . <IP-fixo-do-pihole-dns> 1.1.1.1 1.1.1.2 { policy sequential; health_check 5s }` — o Pi-hole é sempre tentado primeiro, com health-check ativo decidindo o failover de verdade pra Cloudflare quando ele está indisponível. **Não depende mais de `readinessProbe`/`Service` compartilhado** — essa era a abordagem original e tinha um bug sério: os dois pods ficavam `Ready` o tempo todo, e o `kube-proxy` fazia round-robin cego entre eles mesmo com o Pi-hole saudável, derrubando consultas `.home` aleatoriamente. Ver `troubleshooting/pihole_coredns_bypass_dns_round_robin_failover.md`.
- **Não é só um bypass de DNS** — é parte de um plano maior de economia de energia, onde workers e TrueNAS saem do ar junto. Automação do ciclo desligar/religar via **Ansible** ainda não existe neste repositório — ver [ideias futuras: automação Ansible do ciclo de energia](../ideas/ansible_power_cycle_automation.md) para o raciocínio de coordenação com esse mecanismo de failover.

## Diretório `ia/skills/`

Contém skills reutilizáveis específicas deste projeto. Para que uma skill seja carregável automaticamente pelo Claude Code via `Skill` tool, ela também precisa estar referenciada em `.claude/skills/` (ou um link/cópia lá) — os arquivos aqui em `ia/skills/` servem como fonte versionada junto ao projeto, mas podem precisar ser copiados/linkados para o diretório de configuração do Claude Code para serem auto-descobertos.

**Skills atualmente espelhadas em `.claude/skills/`** (mantenha as duas cópias em sincronia ao editar):
- `create-troubleshooting-doc` → espelhada em `.claude/skills/create-troubleshooting-doc/SKILL.md` (também cobre a promoção de `known-issues/` para `troubleshooting/`)
- `create-pull-request` → espelhada em `.claude/skills/create-pull-request/SKILL.md` (usa o template `.github/PULL_REQUEST_TEMPLATE.md`)
- `create-idea` → espelhada em `.claude/skills/create-idea/SKILL.md` (registra uma ideia nova em `ideas.md`/`ideas/`)
- `create-known-issue` → espelhada em `.claude/skills/create-known-issue/SKILL.md` (registra um problema ativo, ainda não resolvido, em `known-issues.md`/`known-issues/`)
- `track-upstream-blocker` → espelhada em `.claude/skills/track-upstream-blocker/SKILL.md` (marca um known-issue como bloqueado por correção de terceiros, status 🔵, com ponto de acompanhamento)
