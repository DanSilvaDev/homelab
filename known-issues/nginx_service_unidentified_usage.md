# Known Issue: Service `nginx` em `default` sem uso/dono confirmado / `nginx` Service in `default` with unconfirmed usage/owner

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Não investigado — uso e necessidade ainda não confirmados.
**Componente(s)**: `Service` `nginx` (namespace `default`).
**Identificado em**: não registrado explicitamente — já existia em 2026-10-22 (mencionado em `troubleshooting/kubevip_lost_ip_attribution_capability.md`), antes até da remoção do Rancher (2026-10-28).
**Relacionado**: [duplicate_loadbalancer_ip_192_168_3_201.md](duplicate_loadbalancer_ip_192_168_3_201.md) (mesmo `Service`, sintoma de IP duplicado com o Grafana), [troubleshooting/kubevip_lost_ip_attribution_capability.md](../troubleshooting/kubevip_lost_ip_attribution_capability.md), [troubleshooting/rancher_removal_traefik_crd_conflict.md](../troubleshooting/rancher_removal_traefik_crd_conflict.md).

Existe um `Service` chamado `nginx` no namespace `default`, descrito como "pré-existente" em pelo menos dois documentos anteriores, mas nunca foi confirmado **o que ele serve de fato**: se há um Deployment/Pod real por trás, se ainda está em uso por algo, ou se é resíduo de uma configuração antiga que ninguém mais usa.

### Hipóteses / o que já foi tentado

Nenhuma hipótese abaixo foi validada ainda:

- **Suspeita original**: leftover da limpeza do Rancher/Cattle. Ressalva encontrada ao checar o contexto já documentado: a remoção do Rancher foi inteiramente dentro dos namespaces `cattle-*` (`cattle-system`, `cattle-turtles-system`, `cattle-fleet-*`, etc.) — o `nginx` vive em `default`, fora desse escopo, então não é literalmente um release Helm do Rancher que sobrou da limpeza.
- Já existia em 2026-10-22, **antes** da remoção do Rancher (2026-10-28) — não foi criado por aquela limpeza.
- Pode ser um teste manual anterior ao Kube-VIP/Rancher (ex: um `nginx-ingress-controller` de teste, ou um pod simples de validação) que nunca foi removido.

### Diagnóstico sugerido (ainda não executado)

- `kubectl get svc nginx -n default -o yaml` — ver `selector`, annotations, `creationTimestamp`.
- `kubectl describe svc nginx -n default` — ver se tem **endpoints reais** (se não tiver nenhum, é forte indício de que não há pod rodando atrás, ou seja, já está órfão).
- `kubectl get deployment,pod -n default -o wide` e cruzar com o `selector` do Service, pra achar o workload (se houver) por trás dele.
- Comparar o `creationTimestamp` encontrado com as datas já documentadas (instalação/remoção do Rancher) pra confirmar ou descartar de vez a hipótese do cattle.
- Se confirmado órfão (sem pods, sem tráfego): avaliar remoção — isso também resolveria [duplicate_loadbalancer_ip_192_168_3_201.md](duplicate_loadbalancer_ip_192_168_3_201.md) automaticamente, já que o conflito de IP lá é exatamente esse `Service` disputando `192.168.3.201` com o Grafana.

---

## English

**Status**: 🔴 Not investigated — usage and necessity not yet confirmed.
**Component(s)**: `nginx` `Service` (namespace `default`).
**Identified on**: not explicitly recorded — already existed on 2026-10-22 (mentioned in `troubleshooting/kubevip_lost_ip_attribution_capability.md`), even before Rancher was removed (2026-10-28).
**Related**: [duplicate_loadbalancer_ip_192_168_3_201.md](duplicate_loadbalancer_ip_192_168_3_201.md) (same `Service`, duplicate-IP-with-Grafana symptom), [troubleshooting/kubevip_lost_ip_attribution_capability.md](../troubleshooting/kubevip_lost_ip_attribution_capability.md), [troubleshooting/rancher_removal_traefik_crd_conflict.md](../troubleshooting/rancher_removal_traefik_crd_conflict.md).

There's a `Service` called `nginx` in the `default` namespace, described as "pre-existing" in at least two earlier documents, but it was never confirmed **what it actually serves**: whether there's a real Deployment/Pod behind it, whether it's still used by anything, or whether it's leftover from an old setup nobody uses anymore.

### Hypotheses / what's already been tried

None of the hypotheses below have been validated yet:

- **Original suspicion**: leftover from the Rancher/Cattle cleanup. Caveat found while checking already-documented context: the Rancher removal happened entirely within `cattle-*` namespaces (`cattle-system`, `cattle-turtles-system`, `cattle-fleet-*`, etc.) — `nginx` lives in `default`, outside that scope, so it's not literally a leftover Rancher Helm release from that cleanup.
- It already existed on 2026-10-22, **before** Rancher was removed (2026-10-28) — it wasn't created by that cleanup.
- It could be a manual test predating Kube-VIP/Rancher (e.g. a test `nginx-ingress-controller`, or a simple validation pod) that was never removed.

### Suggested diagnostics (not yet run)

- `kubectl get svc nginx -n default -o yaml` — check `selector`, annotations, `creationTimestamp`.
- `kubectl describe svc nginx -n default` — check for **real endpoints** (if none, it's a strong sign there's no pod behind it at all, i.e. it's already orphaned).
- `kubectl get deployment,pod -n default -o wide` and cross-reference with the Service's `selector` to find the workload (if any) behind it.
- Compare the found `creationTimestamp` against already-documented dates (Rancher install/removal) to confirm or rule out the cattle hypothesis for good.
- If confirmed orphaned (no pods, no traffic): consider removing it — this would also resolve [duplicate_loadbalancer_ip_192_168_3_201.md](duplicate_loadbalancer_ip_192_168_3_201.md) automatically, since the IP conflict there is exactly this `Service` competing for `192.168.3.201` with Grafana.
