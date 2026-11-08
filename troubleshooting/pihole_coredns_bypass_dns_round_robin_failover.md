# Troubleshooting: DNS para domínios `.home` falhava de forma intermitente (round-robin indevido entre Pi-hole e coredns-bypass) / `.home` domains failed intermittently (improper round-robin between Pi-hole and coredns-bypass)

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Consultas DNS contra domínios `.home` (ex: `jellyfin.home`, `bottoflix.home`) falhavam de forma intermitente mesmo com o Pi-hole saudável. A causa raiz não tinha nada a ver com o app específico sendo acessado: o `coredns-bypass`, pensado como fallback exclusivo pro Pi-hole, nunca foi implementado como failover de verdade — ele ficava `Ready` e recebendo tráfego o tempo todo, inclusive com o Pi-hole saudável, e o `kube-proxy` dividia as consultas entre os dois aleatoriamente.

---

### 1. O Cenário Inicial e Sintomas

Foi registrado um [known-issue](../known-issues.md) de que `jellyfin.home` não resolvia via DNS, embora o acesso direto por IP funcionasse normalmente. A investigação inicial suspeitou de algo específico do Jellyfin ou um erro de digitação no registro. O operador então observou que o problema parecia estar relacionado ao `coredns-bypass`/Pi-hole, especialmente perceptível depois de reiniciar os workers (ciclo de economia de energia) — nesse momento o domínio real em uso era `bottoflix.home`, não `jellyfin.home` (correção feita durante a investigação).

#### Sintomas do Erro:
* `nslookup bottoflix.home 192.168.3.200` às vezes resolvia corretamente (`192.168.3.202`), às vezes retornava `Non-existent domain` — em um teste de 10 consultas seguidas, 3 falharam.
* O problema não era específico de um domínio — qualquer registro `.home` estava sujeito à mesma falha.
* Ficava mais perceptível depois de reiniciar os workers (cache de DNS dos clientes fica frio, expondo um problema que já existia o tempo todo).

---

### 2. Processo de Investigação & Descoberta

A investigação começou revisando [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](pihole_dns_wrong_loadbalancer_ip.md) (mesmo `Service` `dns-service-lb`), que já havia estabelecido que o `kube-proxy` faz round-robin entre **todos** os endpoints `Ready` de um `Service`, sem preferência.

Lendo os manifests reais (`apps/core-dns/deployment.yaml` e `apps/pihole/deployment.yaml`), ficou claro que:
- O `readinessProbe` do `coredns-bypass` testa só o próprio endpoint `/ready` do CoreDNS — sem nenhuma relação com o estado do Pi-hole.
- O `priorityClassName` de cada um (`pihole-principal-priority`/`dns-bypass-priority`) é um `PriorityClass` do Kubernetes, que só afeta ordem de agendamento/preempção sob escassez de recursos — **não tem nenhum efeito em roteamento de tráfego**. O comentário original no manifest ("dá preferência ao Pi-hole quando ele ligar") descrevia um comportamento que esse recurso nunca implementou.

Confirmado via `kubectl`: os dois pods (`coredns-bypass` e `pihole-deployment`) apareciam `1/1 Running`/`Ready` simultaneamente, ambos listados como endpoints do `dns-service-lb` ao mesmo tempo — esse é o estado **normal**, não um estado transitório de inicialização. Um teste de 10 consultas seguidas contra `bottoflix.home` confirmou a divisão: 7 corretas (via Pi-hole), 3 com `Non-existent domain` (via `coredns-bypass`, que encaminha pra Cloudflare e `.home` não existe lá).

#### Causa Raiz: `coredns-bypass` nunca foi um failover exclusivo
O `Service` `dns-service-lb` selecionava os dois pods pela label compartilhada `server: dns-provider`, sempre que cada um estava `Ready` individualmente. Como a prontidão do `coredns-bypass` independe do Pi-hole, ele ficava `Ready` o tempo todo — não era um fallback exclusivo, era um segundo endpoint permanente. O `kube-proxy`, fazendo round-robin cego entre os dois, derrubava uma fração real de toda consulta `.home`.

**Ponto-chave**: nem `readinessProbe` nem `PriorityClass` implementam failover exclusivo entre dois `Deployment`s por trás de um `Service` compartilhado — eles respondem só pela prontidão/prioridade do próprio pod, nunca pela saúde de outro recurso. Failover de verdade precisa de um mecanismo que dependa ativamente da saúde do primário.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirma que os dois pods aparecem como endpoint do Service ao mesmo tempo
kubectl get endpoints dns-service-lb -o wide
kubectl get pods -o wide -l server=dns-provider

# Dispara a mesma consulta várias vezes seguidas - confirma o padrão de divisão
for i in 1 2 3 4 5 6 7 8 9 10; do nslookup bottoflix.home 192.168.3.200; done
```

---

### 4. A Correção Definitiva

A arquitetura foi reestruturada pra usar o próprio CoreDNS como único front-end de DNS, com failover nativo via health-check (plugin `forward`), em vez de depender de `readinessProbe`/`Service` compartilhado:

- **`apps/pihole/dns-service.yaml`** (novo): `Service` `ClusterIP` interno, só pro Pi-hole, alvo do `forward`.
- **`apps/core-dns/config-map.yaml`**: Corefile trocado pra `forward . <IP> 1.1.1.1 1.1.1.2 { policy sequential; max_fails 2; health_check 5s }` — o Pi-hole passa a ser sempre tentado primeiro, com health-check ativo decidindo o failover de verdade. `policy sequential` é essencial: o padrão do plugin é `random`, que reintroduziria o mesmo problema.
- **`apps/core-dns/service.yaml`** (`dns-service-lb`): perdeu a porta 80 e o selector virou só `app: coredns-bypass` — ele é o único listener real de DNS agora.
- **`apps/pihole/service.yaml`** (novo): `Service` `LoadBalancer` só pra UI do Pi-hole, IP próprio (`192.168.3.214`) — a porta 80 não faz mais parte do `dns-service-lb`.
- Removida a label compartilhada `server: dns-provider` e os `PriorityClass` (`pihole-principal-priority`/`dns-bypass-priority`), que não implementavam o que o comentário dizia.

**Gotcha encontrado na aplicação**: o plugin `forward` do CoreDNS **não resolve hostname/FQDN de Service** como upstream — só aceita IP literal (ou `/etc/resolv.conf`). Usar `pihole-dns.default.svc.cluster.local:53` quebrou com o erro `plugin/forward: not an IP address or file`. Corrigido fixando um `clusterIP` estático no `Service` `pihole-dns` (mesma técnica usada pelo `Service` do CoreDNS interno do próprio cluster, que também precisa de um IP estável conhecido antecipadamente) e referenciando esse IP literal no Corefile.

Posteriormente, Pi-hole e `coredns-bypass` também foram movidos do namespace `default` pro namespace dedicado `dns-services`, mantendo os mesmos IPs fixados (`ClusterIP` interno e os dois `LoadBalancer`). Como o PVC do Pi-hole usa provisionamento dinâmico (`truenas-nfs-ssd`), mudar o namespace não move os dados automaticamente — só editar `namespace:` e reaplicar criaria um volume novo vazio. Os dados foram migrados com um pod temporário montando os dois caminhos NFS diretamente (contornando a limitação de um Pod só poder montar PVC do seu próprio namespace), validados com `diff -rq` antes do corte final.

```bash
kubectl apply -f apps/pihole/deployment.yaml
kubectl apply -f apps/pihole/dns-service.yaml
kubectl apply -f apps/pihole/service.yaml
kubectl apply -f apps/core-dns/config-map.yaml
kubectl apply -f apps/core-dns/deployment.yaml
kubectl apply -f apps/core-dns/service.yaml
kubectl rollout restart deployment/coredns-bypass
kubectl delete priorityclass pihole-principal-priority dns-bypass-priority
```

#### Validação final

Validação do estado estável (10x cada, sem falhas):
```bash
for i in 1 2 3 4 5 6 7 8 9 10; do nslookup bottoflix.home 192.168.3.200; done   # 10/10 -> 192.168.3.202
for i in 1 2 3 4 5 6 7 8 9 10; do nslookup doubleclick.net 192.168.3.200; done  # 10/10 -> bloqueado (0.0.0.0)
```

Validação do failover de verdade (Pi-hole derrubado deliberadamente):
```bash
kubectl scale deployment pihole-deployment -n dns-services --replicas=0
# bottoflix.home passou a falhar (NXDOMAIN) e doubleclick.net passou a resolver
# pro IP PÚBLICO real (172.217.29.238) - confirma que o failover pro Cloudflare
# ativou de verdade (ad-blocking temporariamente desligado, como documentado)
for i in 1 2 3 4 5; do nslookup bottoflix.home 192.168.3.200; done
for i in 1 2 3 4 5; do nslookup doubleclick.net 192.168.3.200; done

kubectl scale deployment pihole-deployment -n dns-services --replicas=1
# Após o pod ficar Ready + o health_check do forward confirmar o Pi-hole de
# volta, os dois testes voltaram a 10/10 corretos (mesmo bloco do estado estável acima).
```

---

### 5. Lições Aprendidas para o Homelab

1. **`Service` com label compartilhada entre dois `Deployment`s não implementa failover exclusivo** — o `kube-proxy` faz round-robin entre todos os endpoints `Ready`, sem distinguir "primário" de "backup". Failover de verdade exige um mecanismo que dependa ativamente da saúde do primário (ex: health-check de um plugin de forwarding), não dois backends independentemente saudáveis atrás do mesmo `Service`.
2. **`PriorityClass` não afeta roteamento de tráfego** — só ordem de agendamento/preempção sob escassez de recursos. Um comentário no manifest alegando "dá preferência" não torna isso verdade; sempre validar o que o recurso realmente faz antes de confiar no comentário de quem o escreveu.
3. **O plugin `forward` do CoreDNS usa `policy random` por padrão** — listar o upstream preferido primeiro na lista não cria preferência nenhuma sozinho; é obrigatório setar `policy sequential` explicitamente, senão o comportamento observado é idêntico ao bug original.
4. **O plugin `forward` só aceita IP literal como upstream**, nunca hostname/FQDN de `Service` — resolver o próprio servidor de nomes via DNS seria um problema de ovo-e-galinha. Fixar um `clusterIP` estático no `Service` de destino é a mesma técnica usada pelo DNS interno do próprio cluster.
5. **PVC com provisionamento dinâmico não "muda de namespace"** só editando o campo e reaplicando — isso cria um volume novo vazio. Migrar dados entre namespaces exige um pod temporário, e como PVCs são presos ao namespace do Pod que os monta, montar os caminhos NFS diretamente (bypass do PVC) é o jeito de copiar entre dois namespaces diferentes.
6. **Validar só o estado estável não é suficiente pra um mecanismo de failover** — só o teste de "está tudo saudável" não teria revelado se o `health_check` do `forward` de fato detecta e reage à queda do primário. Derrubar o componente de propósito e confirmar o comportamento de fallback (e a recuperação depois) é o que realmente valida a correção.

---

## English

DNS queries against `.home` domains (e.g. `jellyfin.home`, `bottoflix.home`) failed intermittently even with Pi-hole healthy. The root cause had nothing to do with the specific app being accessed: `coredns-bypass`, meant as an exclusive fallback for Pi-hole, was never actually implemented as real failover — it stayed `Ready` and receiving traffic all the time, even with Pi-hole healthy, and `kube-proxy` randomly split queries between the two.

---

### 1. Initial Scenario and Symptoms

A [known-issue](../known-issues.md) had been logged stating that `jellyfin.home` wasn't resolving via DNS, even though direct IP access worked fine. The initial investigation suspected something Jellyfin-specific or a typo in the record. The operator then observed the problem seemed related to `coredns-bypass`/Pi-hole, especially noticeable after restarting the workers (power-saving cycle) — at that point the real domain in use was `bottoflix.home`, not `jellyfin.home` (corrected during the investigation).

#### Error Symptoms:
* `nslookup bottoflix.home 192.168.3.200` sometimes resolved correctly (`192.168.3.202`), sometimes returned `Non-existent domain` — in a test of 10 consecutive queries, 3 failed.
* The problem wasn't specific to one domain — any `.home` record was subject to the same failure.
* More noticeable after restarting the workers (clients' DNS caches go cold, exposing a problem that had existed the whole time).

---

### 2. Investigation & Discovery Process

The investigation started by reviewing [troubleshooting/pihole_dns_wrong_loadbalancer_ip.md](pihole_dns_wrong_loadbalancer_ip.md) (same `dns-service-lb` `Service`), which had already established that `kube-proxy` round-robins across **all** `Ready` endpoints of a `Service`, with no preference.

Reading the actual manifests (`apps/core-dns/deployment.yaml` and `apps/pihole/deployment.yaml`) made it clear that:
- `coredns-bypass`'s `readinessProbe` only tests CoreDNS's own `/ready` endpoint — with no relation to Pi-hole's state at all.
- Each one's `priorityClassName` (`pihole-principal-priority`/`dns-bypass-priority`) is a Kubernetes `PriorityClass`, which only affects scheduling/preemption order under resource scarcity — **it has no effect on traffic routing**. The original manifest comment ("gives preference to Pi-hole when it's up") described behavior that resource never implemented.

Confirmed via `kubectl`: both pods (`coredns-bypass` and `pihole-deployment`) showed up `1/1 Running`/`Ready` simultaneously, both listed as `dns-service-lb` endpoints at the same time — this is the **normal** state, not a transient startup race. A test of 10 consecutive queries against `bottoflix.home` confirmed the split: 7 correct (via Pi-hole), 3 with `Non-existent domain` (via `coredns-bypass`, which forwards to Cloudflare, and `.home` doesn't exist there).

#### Root Cause: `coredns-bypass` was never a real exclusive failover
The `dns-service-lb` `Service` selected both pods via the shared `server: dns-provider` label, whenever each was individually `Ready`. Since `coredns-bypass`'s readiness is independent of Pi-hole's, it stayed `Ready` all the time — it wasn't an exclusive fallback, it was a permanent second endpoint. `kube-proxy`, blindly round-robining between the two, dropped a real fraction of every `.home` query.

**Key point**: neither `readinessProbe` nor `PriorityClass` implement exclusive failover between two `Deployment`s behind a shared `Service` — they only answer for their own pod's readiness/priority, never another resource's health. Real failover needs a mechanism that actively depends on the primary's health.

---

### 3. Commands Used for Diagnosis

```bash
# Confirm both pods show up as endpoints of the Service at the same time
kubectl get endpoints dns-service-lb -o wide
kubectl get pods -o wide -l server=dns-provider

# Fire the same query repeatedly - confirms the split pattern
for i in 1 2 3 4 5 6 7 8 9 10; do nslookup bottoflix.home 192.168.3.200; done
```

---

### 4. The Definitive Fix

The architecture was restructured to use CoreDNS itself as the single DNS front-end, with native health-check-based failover (`forward` plugin), instead of relying on `readinessProbe`/a shared `Service`:

- **`apps/pihole/dns-service.yaml`** (new): internal `ClusterIP` `Service`, Pi-hole only, the `forward` target.
- **`apps/core-dns/config-map.yaml`**: Corefile changed to `forward . <IP> 1.1.1.1 1.1.1.2 { policy sequential; max_fails 2; health_check 5s }` — Pi-hole is now always tried first, with active health-checking deciding real failover. `policy sequential` is essential: the plugin's default is `random`, which would reintroduce the same problem.
- **`apps/core-dns/service.yaml`** (`dns-service-lb`): lost port 80, and the selector became just `app: coredns-bypass` — it's the single real DNS listener now.
- **`apps/pihole/service.yaml`** (new): `LoadBalancer` `Service` just for Pi-hole's UI, its own IP (`192.168.3.214`) — port 80 is no longer part of `dns-service-lb`.
- Removed the shared `server: dns-provider` label and the `PriorityClass`es (`pihole-principal-priority`/`dns-bypass-priority`), which didn't implement what the comment claimed.

**Gotcha hit while applying it**: CoreDNS's `forward` plugin **doesn't resolve a Service hostname/FQDN** as an upstream — it only accepts a literal IP (or `/etc/resolv.conf`). Using `pihole-dns.default.svc.cluster.local:53` broke with the error `plugin/forward: not an IP address or file`. Fixed by pinning a static `clusterIP` on the `pihole-dns` `Service` (the same technique the cluster's own internal CoreDNS `Service` uses, since it also needs a stable, known-in-advance IP) and referencing that literal IP in the Corefile.

Afterward, Pi-hole and `coredns-bypass` were also moved from the `default` namespace into a dedicated `dns-services` namespace, keeping the same pinned IPs (the internal `ClusterIP` and both `LoadBalancer`s). Since Pi-hole's PVC uses dynamic provisioning (`truenas-nfs-ssd`), changing the namespace doesn't move the data automatically — just editing `namespace:` and reapplying would create a brand-new, empty volume. The data was migrated with a temporary pod mounting both NFS paths directly (working around the fact that a Pod can only mount a PVC from its own namespace), validated with `diff -rq` before the final cutover.

```bash
kubectl apply -f apps/pihole/deployment.yaml
kubectl apply -f apps/pihole/dns-service.yaml
kubectl apply -f apps/pihole/service.yaml
kubectl apply -f apps/core-dns/config-map.yaml
kubectl apply -f apps/core-dns/deployment.yaml
kubectl apply -f apps/core-dns/service.yaml
kubectl rollout restart deployment/coredns-bypass
kubectl delete priorityclass pihole-principal-priority dns-bypass-priority
```

#### Final validation

Steady-state validation (10x each, no failures):
```bash
for i in 1 2 3 4 5 6 7 8 9 10; do nslookup bottoflix.home 192.168.3.200; done   # 10/10 -> 192.168.3.202
for i in 1 2 3 4 5 6 7 8 9 10; do nslookup doubleclick.net 192.168.3.200; done  # 10/10 -> blocked (0.0.0.0)
```

Real failover validation (Pi-hole deliberately taken down):
```bash
kubectl scale deployment pihole-deployment -n dns-services --replicas=0
# bottoflix.home started failing (NXDOMAIN) and doubleclick.net started
# resolving to the real PUBLIC IP (172.217.29.238) - confirms failover to
# Cloudflare actually kicked in (ad-blocking temporarily off, as documented)
for i in 1 2 3 4 5; do nslookup bottoflix.home 192.168.3.200; done
for i in 1 2 3 4 5; do nslookup doubleclick.net 192.168.3.200; done

kubectl scale deployment pihole-deployment -n dns-services --replicas=1
# After the pod became Ready + the forward plugin's health_check confirmed
# Pi-hole was back, both tests returned to 10/10 correct (same block as the
# steady-state validation above).
```

---

### 5. Lessons Learned for the Homelab

1. **A `Service` with a shared label across two `Deployment`s doesn't implement exclusive failover** — `kube-proxy` round-robins across every `Ready` endpoint, with no notion of "primary" vs "backup". Real failover needs a mechanism that actively depends on the primary's health (e.g. a forwarding plugin's health-check), not two independently-healthy backends behind the same `Service`.
2. **`PriorityClass` doesn't affect traffic routing** — only scheduling/preemption order under resource scarcity. A manifest comment claiming it "gives preference" doesn't make it true; always verify what a resource actually does instead of trusting whoever wrote the comment.
3. **CoreDNS's `forward` plugin uses `policy random` by default** — listing the preferred upstream first in the list creates no preference on its own; `policy sequential` must be set explicitly, or the observed behavior is identical to the original bug.
4. **The `forward` plugin only accepts a literal IP as an upstream**, never a `Service` hostname/FQDN — resolving the name server itself via DNS would be a chicken-and-egg problem. Pinning a static `clusterIP` on the target `Service` is the same technique the cluster's own internal DNS uses.
5. **A dynamically-provisioned PVC doesn't "change namespace"** just by editing the field and reapplying — that creates a brand-new, empty volume. Migrating data between namespaces requires a temporary pod, and since PVCs are tied to the namespace of the Pod mounting them, mounting the NFS paths directly (bypassing the PVC) is how you copy between two different namespaces.
6. **Validating only the steady state isn't enough for a failover mechanism** — the "everything's healthy" test alone wouldn't have revealed whether the `forward` plugin's `health_check` actually detects and reacts to the primary going down. Deliberately taking the component down and confirming the fallback behavior (and the recovery afterward) is what actually validates the fix.
