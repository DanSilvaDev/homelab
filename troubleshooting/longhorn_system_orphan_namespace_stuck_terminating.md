# Troubleshooting: Namespace `longhorn-system` órfão do Rancher, travado em `Terminating` por finalizer / Orphaned Rancher `longhorn-system` namespace stuck in `Terminating` due to a finalizer

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Um namespace `longhorn-system`, resíduo de um app Longhorn nunca de fato implantado via Rancher, estava completamente vazio mas presente no cluster desde janeiro. Ao tentar removê-lo, ficou travado em `Terminating` por um finalizer do próprio Rancher (já removido do cluster), exigindo um `patch` manual pra liberar.

---

### 1. O Cenário Inicial e Sintomas

Durante uma varredura manual do cluster (mesmo espírito da investigação que resolveu [troubleshooting/nginx_orphan_service_duplicate_loadbalancer_ip.md](nginx_orphan_service_duplicate_loadbalancer_ip.md)), o usuário notou um namespace `longhorn-system` que parecia não ter nenhum recurso real dentro.

#### Sintomas do Erro:
* Namespace `longhorn-system` existente no cluster, sem nenhum `Service`/`Pod`/`Deployment` visível.
* Ao tentar `kubectl delete namespace longhorn-system`, o namespace ficou preso indefinidamente em `Terminating`.

---

### 2. Processo de Investigação & Descoberta

Confirmado que o namespace estava genuinamente vazio antes de tentar removê-lo:

```bash
kubectl get all -n longhorn-system                                    # nada
kubectl get statefulset,daemonset,configmap,secret -n longhorn-system # só o kube-root-ca.crt (automático em todo namespace)
kubectl get pvc -n longhorn-system                                    # nada
kubectl get crd | grep -i longhorn                                    # nenhuma CRD do Longhorn instalada
kubectl get storageclass | grep -i longhorn                           # nenhuma
kubectl get pv -o wide | grep -i longhorn                             # nenhum PV usando o driver
```

`kubectl get namespace longhorn-system -o yaml` revelou a origem: `creationTimestamp: "2026-01-14T22:38:22Z"` (mais antigo até que o `Service` `nginx`, ver documento irmão), e anotações inequívocas de Rancher — `cattle.io/status`, `field.cattle.io/projectId`, `lifecycle.cattle.io/create.namespace-auth` — a mesma classe de evidência (criação via Rancher UI/catálogo de Apps) já usada pra confirmar a origem do `nginx`.

Com isso confirmado como 100% órfão e seguro de remover, `kubectl delete namespace longhorn-system` foi executado — mas o namespace ficou preso em `Terminating` por 298 dias de idade aparente (nunca tinha sido tocado antes).

#### Causa Raiz: finalizer do Rancher (`controller.cattle.io/namespace-auth`) travado após a remoção do Rancher
O `metadata.finalizers` do namespace tinha `controller.cattle.io/namespace-auth` — um finalizer adicionado pelo próprio Rancher quando criou o namespace, que seria normalmente removido pelo controller do Rancher assim que todas as dependências daquele namespace fossem limpas. Como o Rancher foi removido do cluster (2026-10-28, ver [troubleshooting/rancher_removal_traefik_crd_conflict.md](rancher_removal_traefik_crd_conflict.md)), não existe mais nenhum controller rodando capaz de processar e remover esse finalizer — o namespace fica permanentemente preso em `Terminating`, esperando por um controller que nunca mais vai responder.

**Tentativa inicial incorreta**: o endpoint `/api/v1/namespaces/<nome>/finalize` (via `kubectl replace --raw`) é a forma padrão de liberar um namespace preso em `Terminating`, mas ele edita especificamente `spec.finalizers` (onde normalmente só mora o finalizer `kubernetes` embutido) — **não** `metadata.finalizers` (onde estava o finalizer do Rancher). Usar esse endpoint pro problema errado resultou em `Error from server (NotFound)`.

**Ponto-chave**: `metadata.finalizers` (mecanismo genérico de qualquer objeto Kubernetes) e `spec.finalizers` de um `Namespace` (mecanismo específico do controller de namespace, exposto só via o subresource `/finalize`) são dois mecanismos diferentes — o sintoma (`Terminating` travado) é o mesmo pros dois, mas a correção é diferente dependendo de qual campo tem o finalizer problemático.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirma que o namespace está genuinamente vazio antes de qualquer remoção
kubectl get all -n longhorn-system
kubectl get statefulset,daemonset,configmap,secret -n longhorn-system
kubectl get pvc -n longhorn-system
kubectl get crd | grep -i longhorn
kubectl get storageclass | grep -i longhorn
kubectl get pv -o wide | grep -i longhorn

# Revela a origem (data de criação + anotações do Rancher)
kubectl get namespace longhorn-system -o yaml
```

---

### 4. A Correção Definitiva (ou Workaround)

```bash
# Tentativa inicial (errada pro campo certo - serve pra spec.finalizers, não metadata.finalizers)
kubectl delete namespace longhorn-system
# fica preso em Terminating

kubectl get namespace longhorn-system -o json > /tmp/ns-longhorn.json
kubectl replace --raw "/api/v1/namespaces/longhorn-system/finalize" -f /tmp/ns-longhorn.json
# Error from server (NotFound) - endpoint errado pro finalizer em metadata.finalizers

# Correção: patch direto em metadata.finalizers (mecanismo genérico de objeto, não
# precisa do subresource /finalize)
kubectl patch namespace longhorn-system -p '{"metadata":{"finalizers":[]}}' --type=merge
```

#### Validação final
```bash
kubectl get namespace longhorn-system
# Error from server (NotFound) - namespace removido com sucesso
```

---

### 5. Lições Aprendidas para o Homelab

1. **Namespaces criados via Rancher UI (mesmo fora dos namespaces `cattle-*`) carregam anotações e finalizers do Rancher** — a mesma assinatura (`cattle.io/status`, `field.cattle.io/projectId`, `lifecycle.cattle.io/create.namespace-auth`) já vista no `Service` `nginx` apareceu aqui também; útil pra identificar outros resíduos do Rancher que a limpeza original (escopada só a `cattle-*`) não pegou.
2. **Um namespace vazio pode ainda assim ficar preso em `Terminating` por um finalizer órfão de um controller que não existe mais** — especialmente relevante depois de remover uma ferramenta de gerenciamento como o Rancher, que injeta seus próprios finalizers em recursos fora do seu próprio namespace.
3. **`metadata.finalizers` e `spec.finalizers` (de um `Namespace`) são mecanismos diferentes com correções diferentes** — o subresource `/finalize` só edita `spec.finalizers`; pra limpar `metadata.finalizers` travado, um `kubectl patch --type=merge` direto no objeto resolve. Sempre inspecionar `kubectl get namespace <nome> -o yaml` pra ver **qual campo** tem o finalizer problemático antes de escolher a abordagem.
4. **A anotação `lifecycle.cattle.io/create.namespace-auth` sozinha NÃO é sinal confiável de "namespace órfão"** — testado filtrar todos os namespaces por ela (`kubectl get namespace -o json | ... | select ... lifecycle.cattle.io/create.namespace-auth`) e o resultado incluiu namespaces legítimos e ativos (`kube-system`, `kube-public`, `kube-node-lease`, `monitoring`, `nfs-provisioner`, `nvidia-device-plugin`, `cert-manager`, `default`) — o controller `namespace-auth` do Rancher aparentemente marca **todo** namespace do cluster (pra propagar RBAC do conceito de "Projects"), não só os que ele cria via UI. O sinal real de "órfão" é a combinação de **vazio de verdade** (sem Deployment/Pod/CRD/PV reais, confirmado individualmente) **+ idade antiga** — nunca a anotação isolada.

### O finalizer do Rancher nos outros 8 namespaces precisa ser removido agora?

**Não.** Esses 8 namespaces (`kube-system`, `kube-public`, `kube-node-lease`, `monitoring`, `nfs-provisioner`, `nvidia-device-plugin`, `cert-manager`, `default`) provavelmente carregam o mesmo finalizer `controller.cattle.io/namespace-auth` dormente em `metadata.finalizers` — mas o finalizer é **inerte**: só trava alguma coisa no momento em que o namespace é efetivamente **deletado**, não antes. Enquanto esses namespaces continuarem existindo normalmente (3 são embutidos do Kubernetes, nunca deletados em operação normal; os outros 5 estão ativos e em uso neste cluster), o finalizer não tem efeito nenhum — não afeta leitura, escrita, nem performance.

Se um dia algum desses componentes for descomissionado (ex: trocar o `cert-manager`, remover o `nfs-provisioner`), a mesma trava de `Terminating` pode reaparecer — a correção já está documentada acima (`kubectl patch namespace <nome> -p '{"metadata":{"finalizers":[]}}' --type=merge`) pra esse momento, sem necessidade de ação preventiva agora.

---

## English

A `longhorn-system` namespace, leftover from a Longhorn app never actually deployed via Rancher, was completely empty but present in the cluster since January. Attempting to remove it, it got stuck in `Terminating` due to a Rancher-owned finalizer (Rancher itself already removed from the cluster), requiring a manual `patch` to release it.

---

### 1. Initial Scenario and Symptoms

During a manual sweep of the cluster (same spirit as the investigation that resolved [troubleshooting/nginx_orphan_service_duplicate_loadbalancer_ip.md](nginx_orphan_service_duplicate_loadbalancer_ip.md)), the user noticed a `longhorn-system` namespace that appeared to have no real resources inside it.

#### Error Symptoms:
* A `longhorn-system` namespace existed in the cluster, with no visible `Service`/`Pod`/`Deployment`.
* Attempting `kubectl delete namespace longhorn-system`, the namespace got stuck indefinitely in `Terminating`.

---

### 2. Investigation & Discovery Process

Confirmed the namespace was genuinely empty before attempting removal:

```bash
kubectl get all -n longhorn-system                                    # nothing
kubectl get statefulset,daemonset,configmap,secret -n longhorn-system # only kube-root-ca.crt (automatic in every namespace)
kubectl get pvc -n longhorn-system                                    # nothing
kubectl get crd | grep -i longhorn                                    # no Longhorn CRDs installed
kubectl get storageclass | grep -i longhorn                           # none
kubectl get pv -o wide | grep -i longhorn                             # no PV using its driver
```

`kubectl get namespace longhorn-system -o yaml` revealed the origin: `creationTimestamp: "2026-01-14T22:38:22Z"` (even older than the `nginx` `Service`, see the sibling document), and unmistakable Rancher annotations — `cattle.io/status`, `field.cattle.io/projectId`, `lifecycle.cattle.io/create.namespace-auth` — the same class of evidence (creation via the Rancher UI/App catalog) already used to confirm `nginx`'s origin.

With it confirmed as 100% orphaned and safe to remove, `kubectl delete namespace longhorn-system` was run — but the namespace got stuck in `Terminating`, at an apparent age of 298 days (it had never been touched before).

#### Root Cause: Rancher finalizer (`controller.cattle.io/namespace-auth`) stuck after Rancher's removal
The namespace's `metadata.finalizers` had `controller.cattle.io/namespace-auth` — a finalizer added by Rancher itself when it created the namespace, which would normally be removed by Rancher's own controller once all of that namespace's dependencies were cleaned up. Since Rancher was removed from the cluster (2026-10-28, see [troubleshooting/rancher_removal_traefik_crd_conflict.md](rancher_removal_traefik_crd_conflict.md)), there's no longer any controller running that's capable of processing and removing that finalizer — the namespace stays permanently stuck in `Terminating`, waiting on a controller that will never respond again.

**Initial, incorrect attempt**: the `/api/v1/namespaces/<name>/finalize` endpoint (via `kubectl replace --raw`) is the standard way to release a namespace stuck in `Terminating`, but it specifically edits `spec.finalizers` (where normally only the built-in `kubernetes` finalizer lives) — **not** `metadata.finalizers` (where Rancher's finalizer was). Using that endpoint for the wrong problem resulted in `Error from server (NotFound)`.

**Key point**: `metadata.finalizers` (the generic mechanism available on any Kubernetes object) and a `Namespace`'s `spec.finalizers` (a namespace-controller-specific mechanism, only exposed via the `/finalize` subresource) are two different mechanisms — the symptom (stuck `Terminating`) is the same for both, but the fix differs depending on which field holds the problematic finalizer.

---

### 3. Commands Used for Diagnosis

```bash
# Confirms the namespace is genuinely empty before any removal
kubectl get all -n longhorn-system
kubectl get statefulset,daemonset,configmap,secret -n longhorn-system
kubectl get pvc -n longhorn-system
kubectl get crd | grep -i longhorn
kubectl get storageclass | grep -i longhorn
kubectl get pv -o wide | grep -i longhorn

# Reveals the origin (creation date + Rancher annotations)
kubectl get namespace longhorn-system -o yaml
```

---

### 4. The Definitive Fix (or Workaround)

```bash
# Initial attempt (wrong field for this problem - targets spec.finalizers, not metadata.finalizers)
kubectl delete namespace longhorn-system
# stuck in Terminating

kubectl get namespace longhorn-system -o json > /tmp/ns-longhorn.json
kubectl replace --raw "/api/v1/namespaces/longhorn-system/finalize" -f /tmp/ns-longhorn.json
# Error from server (NotFound) - wrong endpoint for a metadata.finalizers finalizer

# Fix: patch metadata.finalizers directly (generic object mechanism, doesn't need
# the /finalize subresource)
kubectl patch namespace longhorn-system -p '{"metadata":{"finalizers":[]}}' --type=merge
```

#### Final validation
```bash
kubectl get namespace longhorn-system
# Error from server (NotFound) - namespace successfully removed
```

---

### 5. Lessons Learned for the Homelab

1. **Namespaces created via the Rancher UI (even outside `cattle-*` namespaces) carry Rancher annotations and finalizers** — the same signature (`cattle.io/status`, `field.cattle.io/projectId`, `lifecycle.cattle.io/create.namespace-auth`) already seen on the `nginx` `Service` showed up here too; useful for spotting other Rancher leftovers that the original cleanup (scoped only to `cattle-*`) missed.
2. **An empty namespace can still get stuck in `Terminating` due to an orphaned finalizer from a controller that no longer exists** — especially relevant after removing a management tool like Rancher, which injects its own finalizers into resources outside its own namespace.
3. **`metadata.finalizers` and a `Namespace`'s `spec.finalizers` are different mechanisms with different fixes** — the `/finalize` subresource only edits `spec.finalizers`; to clear a stuck `metadata.finalizers`, a direct `kubectl patch --type=merge` on the object does the job. Always inspect `kubectl get namespace <name> -o yaml` to see **which field** holds the problematic finalizer before picking an approach.
4. **The `lifecycle.cattle.io/create.namespace-auth` annotation alone is NOT a reliable "orphaned namespace" signal** — tested filtering all namespaces by it (`kubectl get namespace -o json | ... | select ... lifecycle.cattle.io/create.namespace-auth`) and the result included legitimate, actively-used namespaces (`kube-system`, `kube-public`, `kube-node-lease`, `monitoring`, `nfs-provisioner`, `nvidia-device-plugin`, `cert-manager`, `default`) — Rancher's `namespace-auth` controller apparently marks **every** namespace in the cluster (to propagate RBAC for its "Projects" concept), not just the ones it creates via the UI. The real "orphaned" signal is the combination of **genuinely empty** (no real Deployment/Pod/CRD/PV, confirmed individually) **+ old age** — never the annotation alone.

### Does the Rancher finalizer on the other 8 namespaces need to be removed now?

**No.** Those 8 namespaces (`kube-system`, `kube-public`, `kube-node-lease`, `monitoring`, `nfs-provisioner`, `nvidia-device-plugin`, `cert-manager`, `default`) likely carry the same dormant `controller.cattle.io/namespace-auth` finalizer in `metadata.finalizers` — but the finalizer is **inert**: it only blocks anything at the moment the namespace is actually **deleted**, not before. As long as those namespaces keep existing normally (3 are built into Kubernetes, never deleted in normal operation; the other 5 are active and in use in this cluster), the finalizer has zero effect — it doesn't affect reads, writes, or performance.

If one of those components ever gets decommissioned someday (e.g. replacing `cert-manager`, removing `nfs-provisioner`), the same `Terminating` lock may show up again — the fix is already documented above (`kubectl patch namespace <name> -p '{"metadata":{"finalizers":[]}}' --type=merge`) for that moment, with no need for preventive action now.
