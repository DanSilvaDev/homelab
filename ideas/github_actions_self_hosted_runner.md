# Idea: Runner self-hosted do GitHub Actions no cluster / Self-hosted GitHub Actions runner in the cluster

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: novo componente no cluster k3s (namespace a definir); repositórios pessoais do usuário no GitHub, incluindo este próprio homelab.

Implantar um runner self-hosted do GitHub Actions (ex: via `actions-runner-controller`, operador oficial da GitHub que gerencia runners como pods efêmeros) rodando no cluster k3s, pra usar em fluxos de CI/CD dos repositórios pessoais do usuário no GitHub — este próprio repositório do homelab incluso.

### Considerações / perguntas em aberto

- **Abordagem**: `actions-runner-controller` (ARC, runners efêmeros como pods, escala sob demanda) vs. um runner estático simples (Deployment fixo, sempre registrado). ARC é o padrão mais atual recomendado pela própria GitHub, mas adiciona complexidade operacional (CRDs, webhook, etc.) — vale avaliar se compensa pro volume de uso esperado.
- **Segurança/isolamento**: workflows de Actions executam código arbitrário definido nos repositórios — rodar isso dentro do cluster doméstico (mesma rede que tem acesso a storage NFS, outros serviços internos) exige pensar em isolamento (namespace dedicado, `NetworkPolicy` restringindo acesso a outros serviços do cluster, resources limits) antes de apontar qualquer repositório real pra esse runner.
- **Escopo**: só este repositório (homelab) ou múltiplos repositórios pessoais? Runner por repositório vs. runner compartilhado a nível de organização/conta.
- **Ciclo de energia**: workers desligam de madrugada (ver `README.md`, seção "Gerenciamento de energia") — se o runner rodar num worker, CI/CD fica indisponível nesse período; avaliar se isso é aceitável ou se o runner precisa ficar no control-plane (ligado 24/7).
- **Autenticação**: runners self-hosted precisam de um token/App do GitHub pra se registrar — definir se via Personal Access Token (mais simples) ou GitHub App (mais seguro, escopo mais granular, recomendado pelo ARC pra uso continuado).

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: a new component in the k3s cluster (namespace TBD); the user's personal GitHub repositories, including this homelab repo itself.

Deploy a self-hosted GitHub Actions runner (e.g. via `actions-runner-controller`, GitHub's official operator that manages runners as ephemeral pods) running in the k3s cluster, to use in CI/CD workflows for the user's personal GitHub repositories — this homelab repository included.

### Considerations / open questions

- **Approach**: `actions-runner-controller` (ARC, ephemeral runners as pods, scales on demand) vs. a simple static runner (fixed Deployment, always registered). ARC is GitHub's current recommended standard, but adds operational complexity (CRDs, webhook, etc.) — worth evaluating whether it's justified for the expected usage volume.
- **Security/isolation**: Actions workflows run arbitrary code defined in the repositories — running this inside the home cluster (same network with access to NFS storage, other internal services) requires thinking about isolation (dedicated namespace, `NetworkPolicy` restricting access to other cluster services, resource limits) before pointing any real repository at this runner.
- **Scope**: just this repository (homelab) or multiple personal repositories? Per-repository runner vs. a runner shared at the organization/account level.
- **Power cycle**: workers shut down overnight (see `README.md`, "Power management" section) — if the runner runs on a worker, CI/CD becomes unavailable during that window; evaluate whether that's acceptable or whether the runner needs to live on the control-plane (on 24/7).
- **Authentication**: self-hosted runners need a GitHub token/App to register — decide between a Personal Access Token (simpler) or a GitHub App (more secure, more granular scope, recommended by ARC for ongoing use).
