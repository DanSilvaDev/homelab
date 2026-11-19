# Troubleshooting: Service `nginx` órfão causava IP de LoadBalancer duplicado com o Grafana / Orphaned `nginx` Service caused a duplicate LoadBalancer IP with Grafana

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Um `Service` `nginx` no namespace `default`, sem nenhum Pod/Deployment por trás, disputava o mesmo `loadBalancerIP` (`192.168.3.201`) do `kube-prometheus-stack-grafana`. Causa raiz: resíduo de um teste antigo via Rancher UI, nunca removido, sem uso real há meses.

---

### 1. O Cenário Inicial e Sintomas

Dois problemas registrados separadamente, mas que eram a mesma coisa:
- `known-issues/duplicate_loadbalancer_ip_192_168_3_201.md` (removido, resolvido por este documento): o `Service` `nginx` (namespace `default`) e o `kube-prometheus-stack-grafana` (namespace `monitoring`) estavam ambos anotados com `loadBalancerIP: 192.168.3.201`.
- `known-issues/nginx_service_unidentified_usage.md` (removido, resolvido por este documento): o `Service` `nginx` existia sem confirmação de uso real, suspeita inicial (leftover do Cattle/Rancher) parcialmente descartada por um detalhe de escopo (a limpeza do Rancher ficou restrita aos namespaces `cattle-*`).

#### Sintomas do Erro:
* `kube-vip.io` tinha dois `Service`s pedindo o mesmo `loadBalancerIP`, mesma classe de bug já vista em [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](pihole_dns_wrong_loadbalancer_ip.md) — o Grafana continuava funcionando normalmente (ficou com o IP real), mas o `nginx` representava um risco de confusão/conflito futuro.

---

### 2. Processo de Investigação & Descoberta

Seguido o diagnóstico já sugerido no known-issue `nginx_service_unidentified_usage.md`:

```bash
kubectl get svc nginx -n default -o yaml
kubectl describe svc nginx -n default
kubectl get deployment,pod -n default -o wide
```

O `kubectl get svc nginx -n default -o yaml` revelou dois detalhes decisivos:
- `creationTimestamp: "2026-02-05T00:24:59Z"` — muito mais antigo que qualquer coisa já documentada neste repositório (cluster atual começou a ser construído bem depois disso).
- `annotations: field.cattle.io/publicEndpoints: ...` — essa anotação específica é injetada automaticamente pelo **Rancher** quando um workload é criado via UI dele, mesmo que o recurso em si viva fora dos namespaces `cattle-*`. Isso refina a hipótese anterior: a remoção do Rancher (restrita a `cattle-*`) de fato não foi a origem do resíduo, mas o `Service` **foi criado via Rancher UI** em fevereiro — um teste nunca limpo.

O `kubectl describe svc nginx -n default` confirmou `Endpoints:` vazio, e `kubectl get deployment,pod -n default -o wide` retornou **"No resources found in default namespace"** — não existia nenhum Pod ou Deployment com o label `app: nginx` (o `selector` do Service) em lugar nenhum do cluster. Checado também `kubectl get ingress -A | grep -i nginx` — nenhuma referência.

#### Causa Raiz: `Service` órfão criado via Rancher UI em 2026-02-05, nunca removido
O `Service` `nginx` nunca teve um workload real por trás desde pelo menos o início do histórico documentado deste repositório — um teste manual feito via Rancher (confirmado pela anotação `field.cattle.io`) que ficou pra trás depois que o Rancher foi removido do cluster (2026-10-28). Como ele pedia o mesmo `loadBalancerIP` do Grafana, criava uma disputa silenciosa de IP (mesma classe de bug do Kube-VIP já documentada).

**Ponto-chave**: a anotação `field.cattle.io/publicEndpoints` é uma assinatura confiável de "esse recurso foi criado via Rancher UI", mesmo quando o recurso em si vive fora dos namespaces `cattle-*` — útil pra identificar resíduos do Rancher que escapam de uma varredura só por namespace.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Detalhes completos do Service: selector, annotations, creationTimestamp
kubectl get svc nginx -n default -o yaml

# Confirma ausência de endpoints reais
kubectl describe svc nginx -n default

# Confirma ausência de qualquer workload com o selector do Service
kubectl get deployment,pod -n default -o wide

# Confirma que nenhum Ingress referencia esse Service
kubectl get ingress -A | grep -i nginx

# Confirma o IP atual do Grafana (vítima do conflito, mas já estava funcionando)
kubectl get svc -n monitoring kube-prometheus-stack-grafana -o wide
```

---

### 4. A Correção Definitiva (ou Workaround)

Removido o `Service` órfão diretamente — sem Deployment/dado associado, não havia nada a preservar:

```bash
kubectl delete svc nginx -n default
```

#### Validação final
```bash
# Confirma que o nginx sumiu
kubectl get svc nginx -n default
# Error from server (NotFound)

# Confirma que o Grafana continua saudável, sem precisar de nenhuma mudança
kubectl get svc -n monitoring kube-prometheus-stack-grafana -o wide
curl -sI http://192.168.3.201

# kubectl get svc -n default só mostra o Service "kubernetes" (built-in, esperado em
# todo cluster - aponta pra API server, Endpoints: <IP-do-control-plane>:6443, sem
# selector porque é gerenciado diretamente pelo control-plane, não por um Deployment)
kubectl get svc -n default
```

---

### 5. Lições Aprendidas para o Homelab

1. **A anotação `field.cattle.io/publicEndpoints` identifica recursos criados via Rancher UI mesmo fora dos namespaces `cattle-*`** — útil pra achar resíduos do Rancher que uma busca restrita a `cattle-*` não pegaria.
2. **`Endpoints: <vazio>` + nenhum Deployment/Pod com o `selector` correspondente é confirmação suficiente de que um `Service` está órfão** — não precisa de mais investigação que isso antes de remover com segurança.
3. **Dois `Service`s pedindo o mesmo `loadBalancerIP` nunca geram um erro visível** — o Kube-VIP só atribui ao primeiro que pedir e realoca o outro silenciosamente; vale sempre `grep` por `loadBalancerIP`/`EXTERNAL-IP` duplicado ao auditar o cluster periodicamente (mesma lição já registrada em `troubleshooting/pihole_dns_wrong_loadbalancer_ip.md`).
4. **O `Service` `kubernetes` em `default`, sem `selector`, é normal e built-in em todo cluster Kubernetes** — não é um resíduo a investigar; aponta pro próprio API server (confirmado: `Endpoints` bate com o IP/porta do control-plane).

---

## English

An orphaned `nginx` `Service` in the `default` namespace, with no Pod/Deployment behind it, was competing for the same `loadBalancerIP` (`192.168.3.201`) as `kube-prometheus-stack-grafana`. Root cause: leftover from an old Rancher UI test, never removed, with no real usage for months.

---

### 1. Initial Scenario and Symptoms

Two separately logged problems that turned out to be the same thing:
- `known-issues/duplicate_loadbalancer_ip_192_168_3_201.md` (removed, resolved by this document): the `nginx` `Service` (namespace `default`) and `kube-prometheus-stack-grafana` (namespace `monitoring`) were both annotated with `loadBalancerIP: 192.168.3.201`.
- `known-issues/nginx_service_unidentified_usage.md` (removed, resolved by this document): the `nginx` `Service` existed with no confirmed real usage; the initial suspicion (Cattle/Rancher leftover) was partially ruled out by a scope detail (the Rancher cleanup stayed confined to `cattle-*` namespaces).

#### Error Symptoms:
* `kube-vip.io` had two `Service`s requesting the same `loadBalancerIP`, the same bug class already seen in [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](pihole_dns_wrong_loadbalancer_ip.md) — Grafana kept working normally (it ended up with the real IP), but `nginx` represented a risk of future confusion/conflict.

---

### 2. Investigation & Discovery Process

Followed the diagnostics already suggested in the `nginx_service_unidentified_usage.md` known-issue:

```bash
kubectl get svc nginx -n default -o yaml
kubectl describe svc nginx -n default
kubectl get deployment,pod -n default -o wide
```

`kubectl get svc nginx -n default -o yaml` revealed two decisive details:
- `creationTimestamp: "2026-02-05T00:24:59Z"` — much older than anything already documented in this repository (the current cluster started being built well after that).
- `annotations: field.cattle.io/publicEndpoints: ...` — this specific annotation is automatically injected by **Rancher** when a workload is created through its UI, even if the resource itself lives outside the `cattle-*` namespaces. This refines the earlier hypothesis: the Rancher removal (confined to `cattle-*`) indeed wasn't the origin of the leftover, but the `Service` **was created via the Rancher UI** back in February — a test that was never cleaned up.

`kubectl describe svc nginx -n default` confirmed an empty `Endpoints:`, and `kubectl get deployment,pod -n default -o wide` returned **"No resources found in default namespace"** — there was no Pod or Deployment anywhere in the cluster with the `app: nginx` label (the Service's `selector`). Also checked `kubectl get ingress -A | grep -i nginx` — no references.

#### Root Cause: orphaned `Service` created via the Rancher UI on 2026-02-05, never removed
The `nginx` `Service` never had a real workload behind it for at least as long as this repository's documented history goes back — a manual test done via Rancher (confirmed by the `field.cattle.io` annotation) that was left behind after Rancher was removed from the cluster (2026-10-28). Since it requested the same `loadBalancerIP` as Grafana, it created a silent IP conflict (the same Kube-VIP bug class already documented).

**Key point**: the `field.cattle.io/publicEndpoints` annotation is a reliable signature of "this resource was created through the Rancher UI," even when the resource itself lives outside the `cattle-*` namespaces — useful for spotting Rancher leftovers that a namespace-only scan would miss.

---

### 3. Commands Used for Diagnosis

```bash
# Full Service details: selector, annotations, creationTimestamp
kubectl get svc nginx -n default -o yaml

# Confirms there are no real endpoints
kubectl describe svc nginx -n default

# Confirms there's no workload matching the Service's selector
kubectl get deployment,pod -n default -o wide

# Confirms no Ingress references this Service
kubectl get ingress -A | grep -i nginx

# Confirms Grafana's current IP (the victim of the conflict, but already working)
kubectl get svc -n monitoring kube-prometheus-stack-grafana -o wide
```

---

### 4. The Definitive Fix (or Workaround)

Removed the orphaned `Service` directly — with no Deployment/data attached, there was nothing to preserve:

```bash
kubectl delete svc nginx -n default
```

#### Final validation
```bash
# Confirms nginx is gone
kubectl get svc nginx -n default
# Error from server (NotFound)

# Confirms Grafana stays healthy, no change needed on its side
kubectl get svc -n monitoring kube-prometheus-stack-grafana -o wide
curl -sI http://192.168.3.201

# kubectl get svc -n default now only shows the "kubernetes" Service (built-in,
# expected in every cluster - points at the API server, Endpoints: <control-plane
# IP>:6443, no selector because it's managed directly by the control-plane, not
# by a Deployment)
kubectl get svc -n default
```

---

### 5. Lessons Learned for the Homelab

1. **The `field.cattle.io/publicEndpoints` annotation identifies resources created via the Rancher UI even outside the `cattle-*` namespaces** — useful for finding Rancher leftovers that a `cattle-*`-scoped search would miss.
2. **An empty `Endpoints:` plus no Deployment/Pod matching the `selector` is enough confirmation that a `Service` is orphaned** — no further investigation is needed before safely removing it.
3. **Two `Service`s requesting the same `loadBalancerIP` never produce a visible error** — Kube-VIP just assigns it to whichever one asks first and silently reallocates the other; worth periodically `grep`-ing for duplicate `loadBalancerIP`/`EXTERNAL-IP` values when auditing the cluster (same lesson already logged in `troubleshooting/pihole_dns_wrong_loadbalancer_ip.md`).
4. **The `kubernetes` `Service` in `default`, with no `selector`, is normal and built into every Kubernetes cluster** — not a leftover worth investigating; it points at the API server itself (confirmed: its `Endpoints` match the control-plane's IP/port).
