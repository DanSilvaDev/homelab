# Idea: Stack de dados pra apps locais (PostgreSQL, RabbitMQ, Redis, Kafka) / Data stack for local apps (PostgreSQL, RabbitMQ, Redis, Kafka)

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: novo(s) componente(s) no cluster k3s (namespace a definir), dataset `databases` do TrueNAS (já reservado pra Postgres/Redis, ver [TrueNAS - config.md](../clusters/truenas/config.md)).
**Origem**: dúvida de dimensionamento de hardware levantada pelo usuário (2026-11-09) — ele pretende usar essa stack pra apps locais que vai desenvolver.

Implantar PostgreSQL, RabbitMQ, Redis e Kafka no cluster, em escala "homelab/dev" (não produção), pra servir de infraestrutura de dados pra aplicações locais em desenvolvimento.

### Dimensionamento estimado (escala dev, não produção)

Com o hardware atual (control-plane/worker1: 4 vCPU/15GiB cada; worker2: 5 vCPU/12GiB, compartilhado com GPU), os quatro juntos cabem confortavelmente num único worker:

| Serviço | Request (CPU/Mem) | Limit (CPU/Mem) | Observação |
|---|---|---|---|
| Redis | 100m / 64Mi | 500m / 256Mi | O mais leve — ajustar `maxmemory` baixo (128-256MB) se for só cache |
| PostgreSQL | 250m / 256Mi | 1 / 1Gi | PVC pequeno; dataset `databases` do TrueNAS já reservado pra isso |
| RabbitMQ | 250m / 512Mi | 1 / 1Gi | Overhead da Erlang VM mesmo ocioso — ~256-512MB é o piso realista |
| Kafka (broker único, modo KRaft) | 500m / 1Gi | 1.5 / 2Gi | De longe o mais pesado — evitar Zookeeper (KRaft, Kafka 3.3+) |

Soma dos limits: ~4 vCPU / ~4.25Gi — cabe folgado no worker1 (mais folga disponível hoje, sem disputa com GPU), sem precisar mexer em recursos de mais nada no cluster.

### Considerações / perguntas em aberto

- **Storage do Kafka**: `truenas-nfs-ssd` é SSD de verdade (pool `ssd-apps`), mas é servido via **NFS** — a documentação oficial do Kafka recomenda evitar NFS pros diretórios de log, já que o padrão de escrita (log sequencial + `fsync` frequente) sofre mais com latência de round-trip de rede e semântica de lock do NFS do que com disco local, independente do meio físico ser SSD. Pra volume de apps locais/dev, `truenas-nfs-ssd` provavelmente é suficiente mesmo assim — a recomendação de evitar NFS é mais relevante em alta carga/produção. Alternativa seria `local-path` (StorageClass padrão do cluster, disco local do node), mas aí o PV fica preso a um node específico, sem portabilidade.
- **Namespace**: dedicado (ex: `data-infra` ou similar) ou dentro de algum namespace já existente? Provavelmente dedicado, já que não tem relação direta com a media stack nem observability.
- **Credenciais**: mesmo padrão já usado no resto do repo — Secrets criados manualmente via `kubectl create secret`, nunca commitados.
- **Exposição**: provavelmente só `ClusterIP` (uso interno pelas apps locais que vão rodar no próprio cluster), sem necessidade de LoadBalancer/DNS externo, a menos que alguma ferramenta de administração (ex: pgAdmin, RabbitMQ Management UI) precise de acesso via LAN.
- **Qual worker usar**: recomendado worker1 (mais folga, sem disputa com GPU) em vez do worker2 ou do control-plane (único nó com etcd, sem HA — melhor manter leve).

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: new component(s) in the k3s cluster (namespace TBD), TrueNAS's `databases` dataset (already reserved for Postgres/Redis, see [TrueNAS - config.md](../clusters/truenas/config.md)).
**Origin**: a hardware-sizing question raised by the user (2026-11-09) — they intend to use this stack for local apps they're developing.

Deploy PostgreSQL, RabbitMQ, Redis, and Kafka in the cluster, at a "homelab/dev" scale (not production), to serve as data infrastructure for local applications under development.

### Estimated sizing (dev scale, not production)

With the current hardware (control-plane/worker1: 4 vCPU/15GiB each; worker2: 5 vCPU/12GiB, shared with GPU), all four fit comfortably on a single worker:

| Service | Request (CPU/Mem) | Limit (CPU/Mem) | Note |
|---|---|---|---|
| Redis | 100m / 64Mi | 500m / 256Mi | The lightest — set `maxmemory` low (128-256MB) if just caching |
| PostgreSQL | 250m / 256Mi | 1 / 1Gi | Small PVC; the TrueNAS `databases` dataset is already reserved for this |
| RabbitMQ | 250m / 512Mi | 1 / 1Gi | Erlang VM overhead even idle — ~256-512MB is the realistic floor |
| Kafka (single broker, KRaft mode) | 500m / 1Gi | 1.5 / 2Gi | By far the heaviest — avoid Zookeeper (KRaft, Kafka 3.3+) |

Sum of limits: ~4 vCPU / ~4.25Gi — fits comfortably on worker1 (most headroom available today, no GPU contention), without needing to touch resources for anything else in the cluster.

### Considerations / open questions

- **Kafka's storage**: `truenas-nfs-ssd` is genuinely SSD (the `ssd-apps` pool), but it's served over **NFS** — Kafka's official documentation recommends avoiding NFS for log directories, since its write pattern (sequential log + frequent `fsync`) suffers more from network round-trip latency and NFS's lock semantics than from local disk, regardless of whether the underlying medium is SSD. For local/dev-app volume, `truenas-nfs-ssd` is probably good enough anyway — the recommendation to avoid NFS matters more at high load/production scale. The alternative would be `local-path` (the cluster's default StorageClass, backed by the node's local disk), but then the PV is pinned to that specific node, with no portability.
- **Namespace**: dedicated (e.g. `data-infra` or similar) or inside an existing namespace? Probably dedicated, since it has no direct relation to the media stack or observability.
- **Credentials**: same pattern already used across the repo — Secrets created manually via `kubectl create secret`, never committed.
- **Exposure**: probably `ClusterIP` only (internal use by local apps running in the same cluster), no need for a LoadBalancer/external DNS, unless some admin tool (e.g. pgAdmin, RabbitMQ Management UI) needs LAN access.
- **Which worker to use**: worker1 recommended (more headroom, no GPU contention) over worker2 or the control-plane (the sole etcd node, no HA — better kept light).
