# Known Issue: `jellyfin.home` não resolve via DNS / `jellyfin.home` doesn't resolve via DNS

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Em investigação — não resolvido.
**Componente(s)**: Pi-hole, Jellyfin (media stack).
**Identificado em**: não registrado explicitamente — provavelmente desde a implantação do Jellyfin (Fase 3, 2026-11-02).
**Relacionado**: [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](../troubleshooting/pihole_dns_wrong_loadbalancer_ip.md) (mesma classe de componente, bug distinto).

O registro `jellyfin.home` → `192.168.3.202` foi criado no Pi-hole (Local DNS Records), mas a resolução não funciona na prática. Acesso direto por IP (`http://192.168.3.202`) funciona normalmente, mas qualquer serviço/cliente que tenta acessar via o nome DNS falha.

### Hipóteses a checar (diagnóstico ainda não executado)

- Propagação/cache do Pi-hole após criar o registro.
- Se o domínio `.home` está realmente caindo no Pi-hole e não sendo resolvido por outro DNS upstream primeiro.
- TTL de cache do lado do cliente.
- Erro de digitação no registro criado.

Vale revisar [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](../troubleshooting/pihole_dns_wrong_loadbalancer_ip.md) antes de investigar do zero — mesmo componente (Pi-hole/DNS) já teve um incidente de duas causas raiz sequenciais.

---

## English

**Status**: 🔴 Under investigation — unresolved.
**Component(s)**: Pi-hole, Jellyfin (media stack).
**Identified on**: not explicitly recorded — likely since Jellyfin was deployed (Phase 3, 2026-11-02).
**Related**: [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](../troubleshooting/pihole_dns_wrong_loadbalancer_ip.md) (same component class, different bug).

The `jellyfin.home` → `192.168.3.202` record was created in Pi-hole (Local DNS Records), but resolution doesn't actually work. Direct IP access (`http://192.168.3.202`) works fine, but any service/client trying to reach it by DNS name fails.

### Hypotheses to check (diagnosis not yet run)

- Pi-hole propagation/cache after creating the record.
- Whether the `.home` domain is actually landing in Pi-hole and not being resolved by another upstream DNS first.
- Client-side cache TTL.
- A typo in the created record.

Worth reviewing [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](../troubleshooting/pihole_dns_wrong_loadbalancer_ip.md) before investigating from scratch — the same component (Pi-hole/DNS) already had an incident with two sequential root causes.
