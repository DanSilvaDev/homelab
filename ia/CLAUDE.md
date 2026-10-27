# Contexto do Projeto - Homelab (TrueNAS + Kubernetes)

Este arquivo dá contexto rápido para qualquer agente de IA (Claude ou outro) que for trabalhar neste repositório.

## Visão geral

Homelab pessoal com um cluster **k3s** multi-servidor, gerenciado via **Rancher**, rodando em hardware físico separado. Um dos servidores (TrueNAS SCALE) hospeda uma VM que atua como worker node com aceleração GPU via PCI passthrough.

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
infrastructure/
  rancher/                -> values do Rancher
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
3. **Versões de k3s**: o cluster é gerenciado via Rancher, que impõe restrição de compatibilidade de minor version do Kubernetes. Sempre confirmar a versão suportada antes de sugerir upgrades.
4. **Branch de driver NVIDIA**: a GTX 970 (Maxwell) só é suportada até a branch `580` do driver NVIDIA — não sugerir upgrade para branches `590+` nesse hardware.

## Diretório `ia/skills/`

Contém skills reutilizáveis específicas deste projeto. Para que uma skill seja carregável automaticamente pelo Claude Code via `Skill` tool, ela também precisa estar referenciada em `.claude/skills/` (ou um link/cópia lá) — os arquivos aqui em `ia/skills/` servem como fonte versionada junto ao projeto, mas podem precisar ser copiados/linkados para o diretório de configuração do Claude Code para serem auto-descobertos.

**Skills atualmente espelhadas em `.claude/skills/`** (mantenha as duas cópias em sincronia ao editar):
- `create-troubleshooting-doc` → espelhada em `.claude/skills/create-troubleshooting-doc/SKILL.md`
- `create-pull-request` → espelhada em `.claude/skills/create-pull-request/SKILL.md` (usa o template `.github/PULL_REQUEST_TEMPLATE.md`)
