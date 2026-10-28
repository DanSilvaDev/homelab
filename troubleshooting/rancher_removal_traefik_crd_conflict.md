# Troubleshooting: Conflito de CRD entre `traefik-crd` e `gateway-api-crd` levou à remoção completa do Rancher / CRD conflict between `traefik-crd` and `gateway-api-crd` led to a full Rancher removal

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

O Job nativo do k3s `helm-install-traefik-crd` ficava em `CrashLoopBackOff` por um conflito de ownership de CRD entre dois HelmCharts nativos do k3s. A investigação mostrou que a causa raiz só podia ser resolvida desabilitando o Traefik — e isso exigia remover o Rancher primeiro, já que ele dependia do Traefik para sua própria UI.

---

### 1. O Cenário Inicial e Sintomas

Durante uma checagem de rotina do cluster, o Job `helm-install-traefik-crd` (criado automaticamente pelo k3s, namespace `kube-system`) apareceu em `CrashLoopBackOff`.

#### Sintomas do Erro:
* `kubectl get pods -n kube-system` mostrava `helm-install-traefik-crd-xxxxx` reiniciando repetidamente.
* `kubectl logs` do pod mostrava:
  ```
  Error: UPGRADE FAILED: ... CustomResourceDefinition "backendtlspolicies.gateway.networking.k8s.io" ...
  annotation validation error: key "meta.helm.sh/release-name" must equal "traefik-crd": current value is "gateway-api-crd"
  ```
* `kubectl get helmchart -A` confirmava dois HelmCharts nativos do k3s (`traefik-crd` e `gateway-api-crd`) tentando gerenciar a mesma CRD do Gateway API.

---

### 2. Processo de Investigação & Descoberta

Primeira hipótese: desabilitar o Traefik direto, já que o cluster usa `kube-vip` (não Traefik) para IPs de `LoadBalancer`. Mas `kubectl get ingress -A`, `kubectl get svc -A | grep traefik` e `kubectl get svc -n cattle-system` mostraram que a **UI do Rancher** (`rancher.home`, IP `192.168.3.202`) dependia inteiramente do Traefik — o Service do próprio Rancher era só `ClusterIP`, exposto via `Ingress` classe `traefik`.

Com a stack de observability (Prometheus/Grafana/Alertmanager/Loki) já cobrindo o que o Rancher oferecia de forma independente, a decisão foi remover o Rancher por completo — o que resolveria o conflito de CRD na raiz, liberando o Traefik para ser desabilitado.

#### Causa Raiz: controller do Rancher reconcilia suas próprias dependências
Ao tentar `helm uninstall rancher-turtles -n cattle-turtles-system` (removendo as dependências do Rancher antes do release principal, ordem "folha primeiro" planejada inicialmente), o release voltou sozinho segundos depois — `helm list` mostrou revisão `2` (um `helm upgrade --install` novo, não uma reinstalação do zero) e o deployment do controller tinha `creationTimestamp` mais recente que a tentativa de uninstall.

O motivo: o pod principal do `rancher` (namespace `cattle-system`) roda seu próprio controller de gerenciamento de apps, que trata `fleet`, `fleet-crd`, `fleet-agent-local`, `rancher-turtles`, `rancher-webhook` e `system-upgrade-controller` como dependências internas e as reinstala sempre que detecta que sumiram — confirmado via `kubectl get apps.catalog.cattle.io -A`, que mostra um App CR espelhando **todo** release Helm do cluster.

**Ponto-chave**: não é possível remover essas dependências do Rancher individualmente enquanto o release `rancher` principal ainda está rodando — é preciso removê-lo **primeiro**, invertendo a ordem "dependências antes do principal" que seria natural supor.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Sintoma original
kubectl get pods -n kube-system
kubectl logs -n kube-system <pod-helm-install-traefik-crd>
kubectl get helmchart -A

# Confirma que nada além do Rancher usa o Traefik
kubectl get ingress -A
kubectl get svc -A | grep -i traefik
kubectl get svc -n cattle-system

# Tentativa inicial (ordem errada): remover dependência antes do principal
helm uninstall rancher-turtles -n cattle-turtles-system
kubectl get all -n cattle-turtles-system   # controller já está de volta, Running, poucos segundos de idade

# Confirma a causa: release voltou sozinho (revisão 2, não 1)
helm list -n cattle-turtles-system
helm status rancher-turtles -n cattle-turtles-system
kubectl get deployment -n cattle-turtles-system rancher-turtles-controller-manager -o jsonpath='{.metadata.creationTimestamp}'

# Confirma que o Rancher espelha/gerencia todo release do cluster via App CR
kubectl get apps.catalog.cattle.io -A
```

---

### 4. A Correção Definitiva (ou Workaround)

**Fase 1 — remover o `rancher` principal primeiro** (inverte a ordem originalmente planejada):
```bash
helm uninstall rancher -n cattle-system
# Resultado: "Error: uninstallation completed with 1 error(s): job rancher-post-delete failed: BackoffLimitExceeded"
# O release saiu mesmo assim - o hook de pós-delete só falhou numa etapa posterior de limpeza.
# Efeito colateral (cascata do próprio hook, antes de falhar): rancher-webhook e
# system-upgrade-controller também saíram sozinhos, sem precisar de uninstall manual.
kubectl delete job rancher-post-delete -n cattle-system   # limpa o Job que sobrou falho
```

**Fase 2 — remover as dependências que sobraram** (agora estáveis, sem reconciliação):
```bash
helm uninstall rancher-turtles -n cattle-turtles-system      # limpo desta vez, roda hooks de limpeza do CAPI
helm uninstall fleet-agent-local -n cattle-fleet-local-system
helm uninstall fleet-crd -n cattle-fleet-system
# "fleet" (sem sufixo) já tinha saído sozinho, efeito cascata do hook do rancher
```

**Fase 3 — limpar o que o Helm nunca remove sozinho** (CRDs de um diretório `crds/` de chart **nunca** são deletadas por `helm uninstall`, por design do Helm):
```bash
# Namespaces cattle-* presos em Terminating por finalizer controller.cattle.io/namespace-auth
# (em .metadata.finalizers, não em .spec.finalizers - precisa de patch direto, não do truque /finalize)
kubectl delete ns cattle-fleet-clusters-system cattle-fleet-system cattle-global-data \
  cattle-impersonation-system cattle-local-user-passwords cattle-system \
  cattle-turtles-system cattle-ui-plugin-system cattle-fleet-local-system

kubectl patch namespace cattle-system --type=merge -p '{"metadata":{"finalizers":[]}}'
# (repetido para cada namespace preso)

# Descoberta: namespaces órfãos do Fleet/Rancher sem prefixo "cattle-" também precisavam sair
kubectl delete ns fleet-default fleet-local cluster-fleet-local-local-1a3d67d0a899 local p-29qr8 p-xffxz user-phdcf

# CRDs do Rancher/Fleet (todas *.cattle.io, exceto as 4 nativas do k3s)
kubectl get crd -o name | grep -E '\.cattle\.io$' | grep -v -E 'k3s\.cattle\.io$|helm\.cattle\.io$' | xargs kubectl delete --wait=false

# Instâncias de CR presas em Terminating por finalizers de controllers que não existem mais
# (ex: controller.cattle.io/mgmt-auth-gr-controller, wrangler.cattle.io/*, fleet.cattle.io/*)
# - mesmo padrão dos namespaces: strip do finalizer via patch direto na instância
for crd in clusters.management.cattle.io globalrolebindings.management.cattle.io globalroles.management.cattle.io \
  kontainerdrivers.management.cattle.io nodedrivers.management.cattle.io proxyendpoints.management.cattle.io \
  roletemplates.management.cattle.io users.management.cattle.io; do
  kubectl get $crd -o name | xargs -I{} kubectl patch {} --type=merge -p '{"metadata":{"finalizers":[]}}'
done
```

**Fase 4 — desabilitar o Traefik no k3s** (resolve o conflito de CRD na raiz):
```bash
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<'EOF'
write-kubeconfig-mode: "0644"
disable:
  - traefik
EOF
sudo systemctl restart k3s
```
Desabilitar `traefik` também desabilita `traefik-crd` automaticamente — o k3s trata os dois como um par. O `gateway-api-crd`, porém, é um manifest estático **separado** (`/var/lib/rancher/k3s/server/manifests/gateway-api-crd.yaml`), não ligado à flag `disable: traefik`:
```bash
sudo mv /var/lib/rancher/k3s/server/manifests/gateway-api-crd.yaml \
        /var/lib/rancher/k3s/server/manifests/gateway-api-crd.yaml.skip
kubectl delete helmchart gateway-api-crd -n kube-system
kubectl delete crd backendtlspolicies.gateway.networking.k8s.io gatewayclasses.gateway.networking.k8s.io \
  gateways.gateway.networking.k8s.io grpcroutes.gateway.networking.k8s.io httproutes.gateway.networking.k8s.io \
  listenersets.gateway.networking.k8s.io referencegrants.gateway.networking.k8s.io \
  tcproutes.gateway.networking.k8s.io tlsroutes.gateway.networking.k8s.io udproutes.gateway.networking.k8s.io
```

Antes de remover as CRDs do Gateway API, uma checagem (`kubectl get gateways.gateway.networking.k8s.io -A`) encontrou um `Gateway`/`HTTPRoute` reais em uso (`apps/pihole/api-gateway.yaml`). Investigação mostrou que esse arquivo nunca funcionou de verdade: `GatewayClass.spec.controllerName: pihole.local/gateway-controller` não existe como controller real, e o `HTTPRoute` apontava pro backend `pihole-service`, enquanto o Service real se chama `pihole-svc` (`apps/pihole/service.yaml`). O acesso real do Pi-hole sempre foi via `Service` `LoadBalancer` (`kube-vip.io/ipam: "true"`), independente do Gateway API. O arquivo morto foi removido do repositório e os recursos órfãos, do cluster:
```bash
kubectl delete httproute pihole-http-route pihole-gateway -n default
kubectl delete gateway pihole-gateway -n default
kubectl delete gatewayclass pihole-gateway-class
```

#### Validação final
```bash
kubectl get pods -A | grep -v Running   # só jobs Completed normais
kubectl get nodes                        # todos Ready
kubectl get helmchart -A                 # vazio
kubectl get crd | grep -i -E "cattle|fleet|gateway"   # só as 4 nativas do k3s (addons/etcdsnapshotfiles.k3s.cattle.io, helmcharts/helmchartconfigs.helm.cattle.io)
curl -I http://192.168.3.200             # Pi-hole: 404 normal na raiz (UI real fica em /admin)
```
Grafana, Prometheus, Alertmanager, ntfy e Pi-hole mantiveram seus IPs de `LoadBalancer` (kube-vip) intactos durante toda a operação.

---

### 5. Lições Aprendidas para o Homelab

1. **O controller do Rancher reconcilia suas próprias dependências internas** — sempre remova o release `rancher` principal **antes** de `fleet`, `fleet-crd`, `fleet-agent-local`, `rancher-turtles`, `rancher-webhook` e `system-upgrade-controller`, nunca depois. Tentar na ordem "dependência primeiro" resulta em reinstalação automática.
2. **Helm nunca deleta CRDs do diretório `crds/` de um chart no `helm uninstall`** — por design, para evitar perda de dados. Qualquer remoção de um chart que define CRDs (Rancher, Fleet, gateway-api-crd) precisa de um passo manual de limpeza de CRD depois.
3. **Finalizers de controllers que não existem mais bloqueiam exclusão de namespaces e CRs** — verifique `.metadata.finalizers` (não só `.spec.finalizers`) no objeto preso, e confirme que o controller dono realmente não existe mais antes de limpar com `kubectl patch --type=merge -p '{"metadata":{"finalizers":[]}}'`.
4. **`status.conditions` de um namespace preso em `Terminating` já diz exatamente o que falta** (`NamespaceContentRemaining`/`NamespaceFinalizersRemaining` nomeiam o tipo de recurso e o finalizer) — consulte isso antes de adivinhar.
5. **O `disable:` do k3s pode cobrir mais de um addon de uma vez só** (`traefik` também desliga `traefik-crd`), **mas não necessariamente todo manifest relacionado** — `gateway-api-crd` era um arquivo separado em `/var/lib/rancher/k3s/server/manifests/`, não coberto pela mesma flag. Sempre cheque essa pasta em vez de assumir o escopo da flag.
6. **Confirme uso real antes de apagar algo que parece órfão** — o `Gateway`/`HTTPRoute` do Pi-hole pareciam uma dependência real do Gateway API, mas eram código morto de um experimento antigo (controller inexistente, nome de Service errado). A checagem evitou tanto apagar algo em uso de verdade quanto deixar lixo achando que era funcional.

---

## English

The k3s-native `helm-install-traefik-crd` Job kept `CrashLoopBackOff`-ing due to a CRD ownership conflict between two native k3s HelmCharts. The investigation showed the root cause could only be fixed by disabling Traefik — which in turn required removing Rancher first, since it depended on Traefik for its own UI.

---

### 1. Initial Scenario and Symptoms

During a routine cluster check, the `helm-install-traefik-crd` Job (auto-created by k3s, `kube-system` namespace) showed up in `CrashLoopBackOff`.

#### Error Symptoms:
* `kubectl get pods -n kube-system` showed `helm-install-traefik-crd-xxxxx` restarting repeatedly.
* `kubectl logs` on the pod showed:
  ```
  Error: UPGRADE FAILED: ... CustomResourceDefinition "backendtlspolicies.gateway.networking.k8s.io" ...
  annotation validation error: key "meta.helm.sh/release-name" must equal "traefik-crd": current value is "gateway-api-crd"
  ```
* `kubectl get helmchart -A` confirmed two native k3s HelmCharts (`traefik-crd` and `gateway-api-crd`) both trying to manage the same Gateway API CRD.

---

### 2. Investigation & Discovery Process

First hypothesis: disable Traefik directly, since the cluster uses `kube-vip` (not Traefik) for `LoadBalancer` IPs. But `kubectl get ingress -A`, `kubectl get svc -A | grep traefik` and `kubectl get svc -n cattle-system` showed that the **Rancher UI** (`rancher.home`, IP `192.168.3.202`) depended entirely on Traefik — Rancher's own Service was `ClusterIP`-only, exposed via a `traefik`-class `Ingress`.

With the observability stack (Prometheus/Grafana/Alertmanager/Loki) already covering what Rancher offered independently, the decision was to remove Rancher entirely — which would fix the CRD conflict at the root by freeing up Traefik to be disabled.

#### Root Cause: Rancher's own controller reconciles its dependencies
Trying `helm uninstall rancher-turtles -n cattle-turtles-system` (removing Rancher's dependencies before the main release, the "leaf-first" order initially planned), the release came back on its own seconds later — `helm list` showed revision `2` (a fresh `helm upgrade --install`, not a from-scratch reinstall) and the controller deployment's `creationTimestamp` was newer than the uninstall attempt.

The reason: the main `rancher` pod (`cattle-system` namespace) runs its own app-management controller, which treats `fleet`, `fleet-crd`, `fleet-agent-local`, `rancher-turtles`, `rancher-webhook` and `system-upgrade-controller` as internal dependencies and reinstalls them whenever it detects they're gone — confirmed via `kubectl get apps.catalog.cattle.io -A`, which showed an App CR mirroring **every** Helm release in the cluster.

**Key point**: these Rancher dependencies cannot be removed individually while the main `rancher` release is still running — it must be removed **first**, reversing the "dependencies before the main release" order one would naturally assume.

---

### 3. Commands Used for Diagnosis

```bash
# Original symptom
kubectl get pods -n kube-system
kubectl logs -n kube-system <helm-install-traefik-crd-pod>
kubectl get helmchart -A

# Confirms nothing besides Rancher uses Traefik
kubectl get ingress -A
kubectl get svc -A | grep -i traefik
kubectl get svc -n cattle-system

# Initial attempt (wrong order): remove dependency before the main release
helm uninstall rancher-turtles -n cattle-turtles-system
kubectl get all -n cattle-turtles-system   # controller is already back, Running, seconds old

# Confirms the cause: the release came back on its own (revision 2, not 1)
helm list -n cattle-turtles-system
helm status rancher-turtles -n cattle-turtles-system
kubectl get deployment -n cattle-turtles-system rancher-turtles-controller-manager -o jsonpath='{.metadata.creationTimestamp}'

# Confirms Rancher mirrors/manages every release in the cluster via an App CR
kubectl get apps.catalog.cattle.io -A
```

---

### 4. The Definitive Fix (or Workaround)

**Phase 1 — remove the main `rancher` release first** (reverses the originally planned order):
```bash
helm uninstall rancher -n cattle-system
# Result: "Error: uninstallation completed with 1 error(s): job rancher-post-delete failed: BackoffLimitExceeded"
# The release still came out - the post-delete hook only failed on a later cleanup step.
# Side effect (cascade from the hook itself, before it failed): rancher-webhook and
# system-upgrade-controller also came out on their own, no manual uninstall needed.
kubectl delete job rancher-post-delete -n cattle-system   # cleans up the leftover failed Job
```

**Phase 2 — remove the remaining dependencies** (now stable, no more reconciliation):
```bash
helm uninstall rancher-turtles -n cattle-turtles-system      # clean this time, runs CAPI cleanup hooks
helm uninstall fleet-agent-local -n cattle-fleet-local-system
helm uninstall fleet-crd -n cattle-fleet-system
# "fleet" (no suffix) had already come out on its own, cascade effect of the rancher hook
```

**Phase 3 — clean up what Helm never removes by itself** (CRDs from a chart's `crds/` folder are **never** deleted by `helm uninstall`, by Helm's own design):
```bash
# cattle-* namespaces stuck Terminating on the controller.cattle.io/namespace-auth finalizer
# (in .metadata.finalizers, not .spec.finalizers - needs a direct patch, not the /finalize trick)
kubectl delete ns cattle-fleet-clusters-system cattle-fleet-system cattle-global-data \
  cattle-impersonation-system cattle-local-user-passwords cattle-system \
  cattle-turtles-system cattle-ui-plugin-system cattle-fleet-local-system

kubectl patch namespace cattle-system --type=merge -p '{"metadata":{"finalizers":[]}}'
# (repeated for every stuck namespace)

# Discovery: orphaned Fleet/Rancher namespaces without the "cattle-" prefix also needed removal
kubectl delete ns fleet-default fleet-local cluster-fleet-local-local-1a3d67d0a899 local p-29qr8 p-xffxz user-phdcf

# Rancher/Fleet CRDs (all *.cattle.io, except the 4 native k3s ones)
kubectl get crd -o name | grep -E '\.cattle\.io$' | grep -v -E 'k3s\.cattle\.io$|helm\.cattle\.io$' | xargs kubectl delete --wait=false

# CR instances stuck Terminating on finalizers from controllers that no longer exist
# (e.g. controller.cattle.io/mgmt-auth-gr-controller, wrangler.cattle.io/*, fleet.cattle.io/*)
# - same pattern as the namespaces: strip the finalizer with a direct patch on the instance
for crd in clusters.management.cattle.io globalrolebindings.management.cattle.io globalroles.management.cattle.io \
  kontainerdrivers.management.cattle.io nodedrivers.management.cattle.io proxyendpoints.management.cattle.io \
  roletemplates.management.cattle.io users.management.cattle.io; do
  kubectl get $crd -o name | xargs -I{} kubectl patch {} --type=merge -p '{"metadata":{"finalizers":[]}}'
done
```

**Phase 4 — disable Traefik in k3s** (fixes the CRD conflict at the root):
```bash
sudo tee /etc/rancher/k3s/config.yaml > /dev/null <<'EOF'
write-kubeconfig-mode: "0644"
disable:
  - traefik
EOF
sudo systemctl restart k3s
```
Disabling `traefik` also automatically disables `traefik-crd` — k3s treats the two as a pair. `gateway-api-crd`, however, is a **separate** static manifest (`/var/lib/rancher/k3s/server/manifests/gateway-api-crd.yaml`), not tied to the `disable: traefik` flag:
```bash
sudo mv /var/lib/rancher/k3s/server/manifests/gateway-api-crd.yaml \
        /var/lib/rancher/k3s/server/manifests/gateway-api-crd.yaml.skip
kubectl delete helmchart gateway-api-crd -n kube-system
kubectl delete crd backendtlspolicies.gateway.networking.k8s.io gatewayclasses.gateway.networking.k8s.io \
  gateways.gateway.networking.k8s.io grpcroutes.gateway.networking.k8s.io httproutes.gateway.networking.k8s.io \
  listenersets.gateway.networking.k8s.io referencegrants.gateway.networking.k8s.io \
  tcproutes.gateway.networking.k8s.io tlsroutes.gateway.networking.k8s.io udproutes.gateway.networking.k8s.io
```

Before removing the Gateway API CRDs, a check (`kubectl get gateways.gateway.networking.k8s.io -A`) found a real-looking `Gateway`/`HTTPRoute` in use (`apps/pihole/api-gateway.yaml`). Investigation showed this file never actually worked: `GatewayClass.spec.controllerName: pihole.local/gateway-controller` isn't a real controller, and the `HTTPRoute` pointed to backend `pihole-service`, while the real Service is named `pihole-svc` (`apps/pihole/service.yaml`). Pi-hole's real access has always been through its `LoadBalancer` `Service` (`kube-vip.io/ipam: "true"`), independent of Gateway API. The dead file was removed from the repository, and the orphaned cluster resources too:
```bash
kubectl delete httproute pihole-http-route pihole-gateway -n default
kubectl delete gateway pihole-gateway -n default
kubectl delete gatewayclass pihole-gateway-class
```

#### Final validation
```bash
kubectl get pods -A | grep -v Running   # only normal Completed jobs
kubectl get nodes                        # all Ready
kubectl get helmchart -A                 # empty
kubectl get crd | grep -i -E "cattle|fleet|gateway"   # only the 4 native k3s ones (addons/etcdsnapshotfiles.k3s.cattle.io, helmcharts/helmchartconfigs.helm.cattle.io)
curl -I http://192.168.3.200             # Pi-hole: normal 404 at root (real UI is at /admin)
```
Grafana, Prometheus, Alertmanager, ntfy and Pi-hole all kept their `LoadBalancer` (kube-vip) IPs intact throughout the whole operation.

---

### 5. Lessons Learned for the Homelab

1. **Rancher's own controller reconciles its internal dependencies** — always remove the main `rancher` release **before** `fleet`, `fleet-crd`, `fleet-agent-local`, `rancher-turtles`, `rancher-webhook` and `system-upgrade-controller`, never after. Trying the "dependency first" order results in automatic reinstallation.
2. **Helm never deletes CRDs from a chart's `crds/` folder on `helm uninstall`** — by design, to avoid data loss. Removing any chart that ships CRDs (Rancher, Fleet, gateway-api-crd) needs a manual CRD cleanup step afterward.
3. **Finalizers from controllers that no longer exist block deletion of both namespaces and CRs** — check `.metadata.finalizers` (not just `.spec.finalizers`) on the stuck object, and confirm the owning controller is truly gone before clearing it with `kubectl patch --type=merge -p '{"metadata":{"finalizers":[]}}'`.
4. **A namespace's `status.conditions` while stuck `Terminating` already names exactly what's missing** (`NamespaceContentRemaining`/`NamespaceFinalizersRemaining` name the resource type and finalizer) — check that before guessing.
5. **k3s's `disable:` list can cover more than one addon at once** (`traefik` also turns off `traefik-crd`), **but not necessarily every related manifest** — `gateway-api-crd` was a separate file under `/var/lib/rancher/k3s/server/manifests/`, not covered by the same flag. Always check that directory instead of assuming the flag's scope.
6. **Confirm real usage before deleting something that looks orphaned** — Pi-hole's `Gateway`/`HTTPRoute` looked like a real Gateway API dependency, but were dead code from an old experiment (non-existent controller, wrong Service name). The check avoided both deleting something actually in use and leaving dead code mistaken for something functional.
