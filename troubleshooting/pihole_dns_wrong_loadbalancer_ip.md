# Troubleshooting: Pi-hole não recebia tráfego de DNS via Kube-VIP / Pi-hole wasn't receiving DNS traffic via Kube-VIP

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

O Pi-hole estava `Running`/`Ready` no cluster, mas consultas DNS contra o IP compartilhado do Kube-VIP (`192.168.3.200`, usado tanto pelo Pi-hole quanto pelo `coredns-bypass` como failover) nunca chegavam a ele de verdade. A investigação revelou **duas causas raiz distintas e sequenciais** — a primeira precisou ser corrigida antes da segunda sequer aparecer.

---

### 1. O Cenário Inicial e Sintomas

Durante a retomada do trabalho no cluster, foi relatado: "o Pi-hole subiu, está respondendo no `192.168.3.200`, mas não está recebendo tráfego, parece que o `coredns` ainda está roteando". O `coredns-bypass` e o Pi-hole compartilham a label `server: dns-provider`, selecionada por um `Service` único (`dns-service-lb`) para permitir failover baseado em `readinessProbe` (ver `ia/CLAUDE.md`, seção "Gerenciamento de energia").

#### Sintomas do Erro:
* Pod do Pi-hole `Running`/`Ready`, sem erros óbvios no `kubectl describe`.
* Consultas DNS contra `192.168.3.200` retornando respostas **sem bloqueio de anúncios** (comportamento do `coredns-bypass`, não do Pi-hole), mesmo com o Pi-hole saudável.
* Log do Pi-hole com um warning chamando atenção: `WARNING: Cannot get exclusive lock for /etc/pihole/pihole.toml: Bad file descriptor` (investigado e descartado como causa — problema conhecido de `flock()` sobre NFS, não-fatal, não relacionado ao roteamento).
* Após a primeira correção (ver Causa Raiz 1), o sintoma mudou para **timeout total**: nenhuma resposta nem do Pi-hole nem do `coredns-bypass` via `192.168.3.200`.

---

### 2. Processo de Investigação & Descoberta

#### Causa Raiz 1: mismatch de porta entre o Pi-hole e o Service compartilhado

O Pi-hole estava configurado para escutar DNS na porta **93**, não 53 — `FTLCONF_dns_port: '93'` em `apps/pihole/config-map.yaml`, refletido em `containerPort: 93` e `readinessProbe.tcpSocket.port: 93` em `apps/pihole/deployment.yaml`. Essa porta 93 era resquício de uma configuração bem mais antiga (commit `69ec532`, "fix: pihole dns resolver issue", de antes do Kube-VIP existir no repositório), quando se usava `hostPort: 93` com `containerPort: 53` para evitar um conflito de porta no host. Ao migrar para Kube-VIP + `Service` `LoadBalancer` (sem mais `hostPort`/`hostNetwork` em lugar nenhum), o número 93 foi carregado para dentro do próprio container via `FTLCONF_dns_port`, mas a razão original (conflito de `hostPort`) não existia mais.

O `Service` compartilhado `dns-service-lb` (`apps/core-dns/service.yaml`), que seleciona tanto o pod do Pi-hole quanto o do `coredns-bypass` pela label `server: dns-provider`, usava `targetPort: 53` fixo e numérico para os dois. Isso funcionava para o `coredns-bypass` (que realmente escuta na 53), mas não para o Pi-hole: mesmo com o pod `Ready` (o probe testava a porta certa, 93) e presente nos endpoints do `Service`, o Kubernetes criava o endpoint como `<ip-do-pod>:53` — e dentro do pod do Pi-hole nada escutava ali, só na 93. Como o `kube-proxy` distribui consultas entre todos os endpoints `Ready` (não é um failover exclusivo, é round-robin), toda consulta que caía no endpoint do Pi-hole falhava silenciosamente, e só as que caíam no `coredns-bypass` respondiam — daí a impressão de "só o coredns está roteando".

**Ponto-chave**: um `Service` com `targetPort` numérico não valida se o pod de destino realmente escuta naquela porta — ele só cria o endpoint cegamente.

#### Causa Raiz 2: dois Services LoadBalancer pedindo o mesmo IP ao Kube-VIP

Depois de corrigida a porta, o sintoma virou timeout total — nem o `coredns-bypass` respondia mais. Isso revelou uma segunda causa, mais profunda. O Pi-hole tinha originalmente **dois `Service` separados**, ambos anotados pedindo o **mesmo** `kube-vip.io/loadbalancerIP: "192.168.3.200"`:
- `pihole-svc` — só porta 80 (HTTP, UI de admin), seletor `app: pihole-app` (só o pod do Pi-hole).
- `dns-service-lb` — portas 53 UDP/TCP (DNS), seletor `server: dns-provider` (Pi-hole + `coredns-bypass`).

O comentário original no `pihole-svc` dizia explicitamente que o IP fixo era "para poder ser compartilhado com o `dns-service-lb`... sem depender da ordem de alocação do pool" — mas essa não é como o Kube-VIP funciona: dois objetos `Service` diferentes pedindo o mesmo `loadbalancerIP` **não** dividem esse IP entre si. Confirmado nos logs do `kube-vip-ds`:
```
2026/10/28 ... (svcs) adding VIP ip=192.168.3.200 interface=enp1s0 namespace=default name=pihole-svc
...
2026/10/31 ... (svcs) adding VIP ip=192.168.3.202 interface=enp1s0 namespace=default name=dns-service-lb
```
O `pihole-svc` reservou o `.200` primeiro. Quando o `dns-service-lb` foi criado depois pedindo o mesmo IP, o Kube-VIP não conseguiu honrar o pedido porque o IP já estava em uso por outro `Service` — e realocou **silenciosamente** (sem erro visível, sem evento óbvio de aviso) para o próximo IP livre do pool (`.202`). Resultado: o DNS real sempre esteve servido em `192.168.3.202`, nunca em `192.168.3.200` — qualquer teste contra `.200` só encontrava o `pihole-svc` (porta 80 HTTP), nada escutando em 53 ali. A UI web respondendo normalmente em `.200` mascarava o problema, reforçando a crença de que esse era o IP "funcional" certo.

**Ponto-chave**: dois objetos `Service` do Kubernetes não podem compartilhar o mesmo `kube-vip.io/loadbalancerIP` — o Kube-VIP atribui o IP ao primeiro `Service` que o solicitar, e realoca o segundo silenciosamente para outro IP do pool.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirma que o pod do Pi-hole está Ready
kubectl get pod -l app=pihole-app -o wide

# Confirma os endpoints reais do Service compartilhado - IPs dos dois pods na porta certa?
kubectl get endpoints dns-service-lb -o wide

# Testa o Pi-hole direto, de dentro do próprio pod (isola se o problema é nele ou no roteamento)
kubectl exec <pod-pihole> -- dig @127.0.0.1 doubleclick.net

# Testa contra o VIP a partir de um cliente da LAN (Windows)
nslookup doubleclick.net 192.168.3.200

# Testa contra o VIP de DENTRO do cluster, eliminando variável de rede/ARP do cliente local
kubectl run tmp-dns-test --rm -it --image=busybox:1.36 --restart=Never -- nslookup doubleclick.net 192.168.3.200

# Confirma que o kube-vip-ds está Running e em qual node (só roda no control-plane)
kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide

# Logs do kube-vip-ds - mostram explicitamente qual IP foi atribuído a qual Service
kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip-ds --tail=80

# Confirma quem detém a lease de líder de anúncio de Services (descarta problema de leader election)
kubectl get lease -n kube-system | grep -i svc
```

---

### 4. A Correção Definitiva (ou Workaround)

**Correção 1** (porta): alinhar o Pi-hole para escutar DNS na porta 53 padrão, igual ao `coredns-bypass`:
- `apps/pihole/config-map.yaml`: `FTLCONF_dns_port` de `'93'` para `'53'`.
- `apps/pihole/deployment.yaml`: `containerPort` (UDP e TCP) e `readinessProbe.tcpSocket.port` de `93` para `53`.

**Correção 2** (IP compartilhado): unificar os dois `Service` em um só objeto, eliminando a disputa pelo IP — só pode haver um `Service` pedindo um dado `loadbalancerIP` por vez.
- `apps/core-dns/service.yaml`: passou a expor as 3 portas (80 HTTP + 53 UDP DNS + 53 TCP DNS), mesmo seletor `server: dns-provider`, uma única annotation `kube-vip.io/loadbalancerIP: "192.168.3.200"`.
- `apps/pihole/service.yaml`: removido do repositório (funcionalidade absorvida pelo `Service` acima).

Como os dois `Service` antigos já existiam no cluster com IPs já atribuídos, só aplicar o manifesto atualizado não forçaria o Kube-VIP a realocar o IP (a atribuição só acontece na criação do objeto / quando `status.loadBalancer` está vazio). Foi necessário deletar os dois e recriar do zero:

```bash
kubectl delete svc pihole-svc
kubectl delete svc dns-service-lb
kubectl apply -f apps/core-dns/service.yaml
```

Isso causou uma interrupção breve (segundos) no acesso via `.200` — tanto DNS quanto a UI do Pi-hole — até o `Service` novo subir e o Kube-VIP reanunciar o IP.

#### Validação final
```bash
kubectl get svc dns-service-lb -o wide   # EXTERNAL-IP esperado: 192.168.3.200
nslookup doubleclick.net 192.168.3.200   # esperado: resposta bloqueada (0.0.0.0 / NXDOMAIN)
```
Usuário confirmou DNS funcionando corretamente após essas duas correções.

---

### 5. Lições Aprendidas para o Homelab

1. **`targetPort` numérico não valida o pod de destino** — quando múltiplos pods com portas diferentes compartilham um `Service` (via label comum, para failover), todos precisam escutar exatamente na mesma porta numérica referenciada pelo `Service`, ou usar portas nomeadas (named ports), que resolvem por nome por-pod.
2. **Dois `Service` não podem compartilhar o mesmo `kube-vip.io/loadbalancerIP`** — o Kube-VIP atribui o IP ao primeiro que pedir e realoca o segundo silenciosamente, sem erro óbvio. Se dois serviços de um mesmo app (ex: DNS e HTTP) precisam de um IP fixo compartilhado, eles precisam ser o **mesmo** objeto `Service` (múltiplas portas), não dois `Service` separados com a mesma annotation.
3. **Um sintoma novo após uma correção geralmente indica uma segunda causa raiz, não que a correção anterior falhou** — aqui, "tráfego só pro coredns" virou "timeout total" depois da correção 1; valeu re-investigar do zero (reconferir o `EXTERNAL-IP` real dos `Service`) em vez de assumir regressão.
4. **Testar de dentro do cluster antes de suspeitar do lado do cliente** — `kubectl run --rm -it --image=busybox -- nslookup ...` descartou rapidamente causas client-side (cache de ARP do Windows, firewall) e confirmou que o problema era do cluster/Kube-VIP.
5. **Os logs do `kube-vip-ds` são a fonte da verdade sobre o IP real atribuído** — a linha `adding VIP ip=... name=...` mostra exatamente qual IP cada `Service` recebeu, mais confiável que assumir que a annotation `kube-vip.io/loadbalancerIP` foi honrada.

---

## English

The Pi-hole pod was `Running`/`Ready` in the cluster, but DNS queries against the shared Kube-VIP IP (`192.168.3.200`, used by both Pi-hole and `coredns-bypass` for failover) never actually reached it. The investigation uncovered **two distinct, sequential root causes** — the first had to be fixed before the second even surfaced.

---

### 1. Initial Scenario and Symptoms

While resuming work on the cluster, it was reported: "Pi-hole is up, responding on `192.168.3.200`, but not receiving traffic, it looks like `coredns` is still routing". `coredns-bypass` and Pi-hole share the `server: dns-provider` label, selected by a single `Service` (`dns-service-lb`) to allow `readinessProbe`-based failover (see `ia/CLAUDE.md`, "Gerenciamento de energia" section).

#### Error Symptoms:
* Pi-hole pod `Running`/`Ready`, no obvious errors in `kubectl describe`.
* DNS queries against `192.168.3.200` returning **unblocked** answers (behavior consistent with `coredns-bypass`, not Pi-hole), even with Pi-hole healthy.
* A Pi-hole log line that caught attention: `WARNING: Cannot get exclusive lock for /etc/pihole/pihole.toml: Bad file descriptor` (investigated and ruled out — a known, non-fatal `flock()`-over-NFS issue, unrelated to routing).
* After the first fix (see Root Cause 1), the symptom changed to **total timeout**: no response at all, from neither Pi-hole nor `coredns-bypass`, via `192.168.3.200`.

---

### 2. Investigation & Discovery Process

#### Root Cause 1: port mismatch between Pi-hole and the shared Service

Pi-hole was configured to listen for DNS on port **93**, not 53 — `FTLCONF_dns_port: '93'` in `apps/pihole/config-map.yaml`, reflected in `containerPort: 93` and `readinessProbe.tcpSocket.port: 93` in `apps/pihole/deployment.yaml`. That port 93 was a leftover from a much older configuration (commit `69ec532`, "fix: pihole dns resolver issue", predating Kube-VIP in this repo), when `hostPort: 93` was used together with `containerPort: 53` to avoid a host port conflict. After migrating to Kube-VIP + a `LoadBalancer` `Service` (with no `hostPort`/`hostNetwork` left anywhere), the number 93 got carried into the container itself via `FTLCONF_dns_port`, but the original reason (`hostPort` conflict) no longer existed.

The shared `dns-service-lb` `Service` (`apps/core-dns/service.yaml`), which selects both the Pi-hole pod and the `coredns-bypass` pod via the `server: dns-provider` label, used a fixed, numeric `targetPort: 53` for both. That worked for `coredns-bypass` (which really listens on 53), but not for Pi-hole: even with the pod `Ready` (the probe tested the right port, 93) and present in the `Service`'s endpoints, Kubernetes built the endpoint as `<pod-ip>:53` — and nothing inside the Pi-hole pod listened there, only on 93. Since `kube-proxy` distributes queries across all `Ready` endpoints (it's round-robin, not an exclusive failover), every query that landed on the Pi-hole endpoint failed silently, and only the ones landing on `coredns-bypass` got answered — hence the impression that "only coredns is routing".

**Key point**: a `Service` with a numeric `targetPort` doesn't validate whether the destination pod actually listens on that port — it just blindly builds the endpoint.

#### Root Cause 2: two LoadBalancer Services requesting the same IP from Kube-VIP

After the port fix, the symptom turned into a total timeout — not even `coredns-bypass` answered anymore. That revealed a second, deeper cause. Pi-hole originally had **two separate `Service` objects**, both annotated requesting the **same** `kube-vip.io/loadbalancerIP: "192.168.3.200"`:
- `pihole-svc` — only port 80 (HTTP, admin UI), selector `app: pihole-app` (Pi-hole pod only).
- `dns-service-lb` — ports 53 UDP/TCP (DNS), selector `server: dns-provider` (Pi-hole + `coredns-bypass`).

The original comment on `pihole-svc` explicitly said the fixed IP was there "so it can be shared with `dns-service-lb`... without depending on the pool's allocation order" — but that's not how Kube-VIP works: two different `Service` objects requesting the same `loadbalancerIP` do **not** split that IP between them. Confirmed in the `kube-vip-ds` logs:
```
2026/10/28 ... (svcs) adding VIP ip=192.168.3.200 interface=enp1s0 namespace=default name=pihole-svc
...
2026/10/31 ... (svcs) adding VIP ip=192.168.3.202 interface=enp1s0 namespace=default name=dns-service-lb
```
`pihole-svc` claimed `.200` first. When `dns-service-lb` was created afterward requesting the same IP, Kube-VIP couldn't honor the request because the IP was already in use by another `Service` — and it **silently** fell back (no visible error, no obvious warning event) to the next free IP in the pool (`.202`). As a result, real DNS was always served on `192.168.3.202`, never on `192.168.3.200` — any test against `.200` only ever found `pihole-svc` (HTTP port 80), with nothing listening on 53 there. The web UI responding normally on `.200` masked the problem, reinforcing the belief that it was the "correct, working" IP.

**Key point**: two Kubernetes `Service` objects cannot share the same `kube-vip.io/loadbalancerIP` — Kube-VIP assigns the IP to whichever `Service` requests it first, and silently reallocates the second one to another IP from the pool.

---

### 3. Commands Used for Diagnosis

```bash
# Confirm the Pi-hole pod is Ready
kubectl get pod -l app=pihole-app -o wide

# Confirm the shared Service's real endpoints - are both pod IPs on the right port?
kubectl get endpoints dns-service-lb -o wide

# Test Pi-hole directly, from inside its own pod (isolates whether the issue is in it or in routing)
kubectl exec <pod-pihole> -- dig @127.0.0.1 doubleclick.net

# Test against the VIP from a LAN client (Windows)
nslookup doubleclick.net 192.168.3.200

# Test against the VIP from INSIDE the cluster, ruling out the local client's network/ARP
kubectl run tmp-dns-test --rm -it --image=busybox:1.36 --restart=Never -- nslookup doubleclick.net 192.168.3.200

# Confirm kube-vip-ds is Running and on which node (only runs on the control-plane)
kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide

# kube-vip-ds logs - explicitly show which IP was assigned to which Service
kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip-ds --tail=80

# Confirm who holds the Service-announcement leader lease (rules out a leader election issue)
kubectl get lease -n kube-system | grep -i svc
```

---

### 4. The Definitive Fix (or Workaround)

**Fix 1** (port): align Pi-hole to listen for DNS on the standard port 53, same as `coredns-bypass`:
- `apps/pihole/config-map.yaml`: `FTLCONF_dns_port` from `'93'` to `'53'`.
- `apps/pihole/deployment.yaml`: `containerPort` (UDP and TCP) and `readinessProbe.tcpSocket.port` from `93` to `53`.

**Fix 2** (shared IP): merge the two `Service` objects into one, eliminating the IP contention — only one `Service` can request a given `loadbalancerIP` at a time.
- `apps/core-dns/service.yaml`: now exposes all 3 ports (80 HTTP + 53 UDP DNS + 53 TCP DNS), same `server: dns-provider` selector, a single `kube-vip.io/loadbalancerIP: "192.168.3.200"` annotation.
- `apps/pihole/service.yaml`: removed from the repository (its functionality absorbed into the `Service` above).

Since both old `Service` objects already existed in the cluster with IPs already assigned, simply applying the updated manifest wouldn't force Kube-VIP to reallocate the IP (allocation only happens on object creation / when `status.loadBalancer` is empty). Both had to be deleted and recreated from scratch:

```bash
kubectl delete svc pihole-svc
kubectl delete svc dns-service-lb
kubectl apply -f apps/core-dns/service.yaml
```

This caused a brief (seconds-long) interruption in access via `.200` — both DNS and the Pi-hole web UI — until the new `Service` came up and Kube-VIP re-announced the IP.

#### Final validation
```bash
kubectl get svc dns-service-lb -o wide   # expected EXTERNAL-IP: 192.168.3.200
nslookup doubleclick.net 192.168.3.200   # expected: blocked answer (0.0.0.0 / NXDOMAIN)
```
User confirmed DNS was working correctly after both fixes.

---

### 5. Lessons Learned for the Homelab

1. **A numeric `targetPort` doesn't validate the destination pod** — when multiple pods with different port specs share a `Service` (via a common label, for failover), all of them must listen on exactly the same numeric port referenced by the `Service`, or use named ports, which resolve by name per-pod.
2. **Two `Service` objects cannot share the same `kube-vip.io/loadbalancerIP`** — Kube-VIP assigns the IP to whichever one requests it first and silently reallocates the second, with no obvious error. If two services of the same app (e.g. DNS and HTTP) need a shared fixed IP, they must be the **same** `Service` object (multiple ports), not two separate `Service` objects with the same annotation.
3. **A new symptom after a fix usually signals a second, distinct root cause, not that the previous fix failed** — here, "traffic only going to coredns" turned into "total timeout" after fix 1; it was worth re-investigating from scratch (re-checking the `Service`s' real `EXTERNAL-IP`) instead of assuming a regression.
4. **Test from inside the cluster before suspecting the client side** — `kubectl run --rm -it --image=busybox -- nslookup ...` quickly ruled out client-side causes (Windows ARP cache, firewall) and confirmed the problem was in the cluster/Kube-VIP.
5. **`kube-vip-ds` logs are the source of truth for the actually assigned IP** — the `adding VIP ip=... name=...` line shows exactly which IP each `Service` received, more reliable than assuming the `kube-vip.io/loadbalancerIP` annotation was honored.
