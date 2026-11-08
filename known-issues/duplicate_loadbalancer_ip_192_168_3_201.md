# Known Issue: `192.168.3.201` duplicado entre dois Services / `192.168.3.201` duplicated across two Services

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Não investigado/corrigido ainda.
**Componente(s)**: `nginx` (namespace `default`, pré-existente), `kube-prometheus-stack-grafana` (namespace `monitoring`), Kube-VIP.
**Identificado em**: não registrado explicitamente.
**Relacionado**: [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](../troubleshooting/pihole_dns_wrong_loadbalancer_ip.md) (mesma classe de bug), [Observability - config.md](../clusters/observability/config.md).

O `Service` `nginx` (namespace `default`) e o `kube-prometheus-stack-grafana` (namespace `monitoring`) estão ambos anotados com o mesmo `loadBalancerIP` (`192.168.3.201`).

Essa é a mesma classe de bug já documentada em [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](../troubleshooting/pihole_dns_wrong_loadbalancer_ip.md): dois `Service` não podem compartilhar um `loadBalancerIP` — o Kube-VIP atribui ao primeiro que pedir, e o segundo é realocado silenciosamente para outro IP do pool, sem erro visível.

Detalhe adicional do ambiente em [Observability - config.md](../clusters/observability/config.md).

---

## English

**Status**: 🔴 Not yet investigated/fixed.
**Component(s)**: `nginx` (namespace `default`, pre-existing), `kube-prometheus-stack-grafana` (namespace `monitoring`), Kube-VIP.
**Identified on**: not explicitly recorded.
**Related**: [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](../troubleshooting/pihole_dns_wrong_loadbalancer_ip.md) (same bug class), [Observability - config.md](../clusters/observability/config.md).

The `nginx` `Service` (namespace `default`) and `kube-prometheus-stack-grafana` (namespace `monitoring`) are both annotated with the same `loadBalancerIP` (`192.168.3.201`).

This is the same bug class already documented in [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](../troubleshooting/pihole_dns_wrong_loadbalancer_ip.md): two `Service`s can't share a `loadBalancerIP` — Kube-VIP assigns it to whichever one asks first, and the second one is silently reallocated to another IP from the pool, with no visible error.

Further environment detail in [Observability - config.md](../clusters/observability/config.md).
