# Idea: HTTPS com certificado válido em todo o cluster, aposentando HTTP / Cluster-wide HTTPS with a valid certificate, retiring HTTP

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: praticamente todos os serviços expostos na LAN hoje só em HTTP — Grafana, Prometheus, Alertmanager, ntfy, Pi-hole (UI), Jellyfin, todos os apps de `apps/servarr/`, qBittorrent.
**Origem**: proposta pelo usuário em 2026-11-08.

Fazer o cluster usar HTTPS com certificado válido (gerado via OpenSSL) em vez de HTTP puro, aposentando as conexões só-HTTP usadas hoje por padrão em todos os serviços expostos na LAN.

### Considerações / perguntas em aberto

- **Contexto já documentado**: várias notas de segurança espalhadas pelo repo (`clusters/observability/config.md`) já dizem explicitamente que Prometheus/Alertmanager/ntfy "não têm autenticação própria" e que isso é "aceitável dentro da LAN confiável de casa". Essa aceitação pressupõe tráfego HTTP simples; migrar pra HTTPS resolve a parte de confidencialidade/integridade do tráfego, mas **não substitui** a falta de autenticação — vale decidir se autenticação entra no mesmo esforço ou fica pra depois.
- **O que conta como "certificado válido" aqui**: os domínios são todos internos (`.home`, resolvidos só pelo Pi-hole), não públicos — não dá pra emitir via Let's Encrypt padrão (exige validação de domínio público). "Válido" provavelmente significa uma **CA própria** (criada com OpenSSL), cujo certificado raiz precisaria ser instalado/confiado manualmente em cada dispositivo da rede que for acessar os serviços sem aviso de certificado inválido no navegador.
- **TrueNAS já usa HTTPS, mas autoassinado** (`clusters/truenas/config.md`) — mesma categoria de problema que essa ideia tentaria resolver pros serviços do cluster; vale considerar se o TrueNAS entra no mesmo esforço.
- **Traefik foi desabilitado de propósito** (dependência do Rancher, removido - ver `ia/CLAUDE.md`), e seria o jeito mais natural de centralizar terminação TLS num único Ingress (um certificado, ou um wildcard `*.home`, pra todos os serviços). Sem ele, a alternativa é configurar TLS individualmente em cada `Service`/`Deployment` - mais trabalho, sem ponto único de gerência.
- **Alternativa ao gerenciamento manual via OpenSSL**: o `cert-manager` é o padrão de fato do Kubernetes pra emissão/rotação automática de certificados (inclusive com uma CA interna própria, via `ClusterIssuer` do tipo `ca`) — a ideia menciona OpenSSL especificamente, mas vale avaliar se compensa automatizar em vez de gerar/renovar certificados manualmente um a um conforme o número de serviços cresce.
- **Escopo grande**: é basicamente todo serviço LAN do cluster hoje. Decidir se a migração é de uma vez só ou por fases (ex: começar pelos mais sensíveis - Pi-hole admin, qBittorrent - antes do resto).

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: essentially every service currently exposed on the LAN over plain HTTP — Grafana, Prometheus, Alertmanager, ntfy, Pi-hole (UI), Jellyfin, every app under `apps/servarr/`, qBittorrent.
**Origin**: proposed by the user on 2026-11-08.

Make the cluster use HTTPS with a valid certificate (generated via OpenSSL) instead of plain HTTP, retiring the HTTP-only connections currently used by default across every LAN-exposed service.

### Considerations / open questions

- **Already-documented context**: several security notes scattered through the repo (`clusters/observability/config.md`) already explicitly say Prometheus/Alertmanager/ntfy "have no authentication of their own" and that this is "acceptable within the trusted home LAN". That acceptance assumes plain HTTP traffic; moving to HTTPS addresses traffic confidentiality/integrity, but **doesn't replace** the lack of authentication — worth deciding whether authentication is part of the same effort or deferred.
- **What counts as a "valid certificate" here**: the domains are all internal (`.home`, resolved only by Pi-hole), not public — standard Let's Encrypt issuance isn't possible (it requires public domain validation). "Valid" likely means a **private CA** (created with OpenSSL), whose root certificate would need to be manually installed/trusted on every device on the network that accesses these services without a browser certificate warning.
- **TrueNAS already uses HTTPS, but self-signed** (`clusters/truenas/config.md`) — same category of problem this idea would try to solve for the cluster's services; worth considering whether TrueNAS is included in the same effort.
- **Traefik was intentionally disabled** (Rancher dependency, removed - see `ia/CLAUDE.md`), and it would be the most natural way to centralize TLS termination at a single Ingress (one certificate, or a `*.home` wildcard, for every service). Without it, the alternative is configuring TLS individually on each `Service`/`Deployment` - more work, no single point of management.
- **Alternative to manual OpenSSL management**: `cert-manager` is the de facto Kubernetes standard for automatic certificate issuance/rotation (including with your own internal CA, via a `ca`-type `ClusterIssuer`) — the idea specifically mentions OpenSSL, but it's worth evaluating whether automating this pays off instead of manually generating/renewing certificates one by one as the number of services grows.
- **Large scope**: this is essentially every LAN service in the cluster today. Decide whether the migration happens all at once or in phases (e.g. starting with the most sensitive ones - Pi-hole admin, qBittorrent - before the rest).
