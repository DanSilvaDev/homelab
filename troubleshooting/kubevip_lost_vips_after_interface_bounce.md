# Troubleshooting: Kube-VIP perdeu todos os IPs virtuais após bounce de rede e travou num impasse de eleição de líder / Kube-VIP lost every virtual IP after a network bounce and got stuck in a leader-election deadlock

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Depois de forçar a renovação do lease de DHCP do control-plane (derrubando e subindo a interface de rede pra aplicar uma nova reserva fixa no roteador), o cluster inteiro ficou instável: `kubectl` parou de funcionar, `worker1`/`worker2` ficaram `NotReady`, e o Kube-VIP não conseguiu se recuperar sozinho — travado num impasse circular que só um `systemctl restart k3s` resolveu.

---

### 1. O Cenário Inicial e Sintomas

Contexto: o export NFS do TrueNAS usado pelo `nfs-subdir-external-provisioner` autorizava hosts por IP, e o IP real da interface do control-plane era dinâmico via DHCP (e mudava — ver [`ideas/nfs_provisioners_control_plane_affinity.md`](../ideas/nfs_provisioners_control_plane_affinity.md)). Pra resolver isso, foi criada uma reserva fixa de DHCP no roteador (`192.168.3.100`) pro MAC do control-plane, e a renovação do lease foi forçada via `nmcli connection down`/`up` na interface `enp1s0`.

#### Sintomas do Erro:
* `kubectl` configurado no PC do usuário, apontando pro IP virtual do Kube-VIP (`192.168.3.101`), passou a dar **timeout** (não um erro de certificado, que seria o esperado só por uma troca de IP de kubeconfig).
* No control-plane, `ip -4 -o addr show` mostrava **só** o novo IP dinâmico (`192.168.3.100/24`) — todos os IPs virtuais que o Kube-VIP gerenciava (`192.168.3.101`, o VIP da API, e o pool inteiro de LoadBalancer `192.168.3.200`–`192.168.3.214`) tinham sumido da interface.
* `journalctl -u k3s` no control-plane, repetidamente: `failed to validate nodeIP: node IP: "192.168.3.101" not found in the host's network interfaces`.
* `kubectl get nodes` (rodado localmente no control-plane via `sudo k3s kubectl`, contornando o problema de rede externa) mostrava `worker1` e `worker2` como `NotReady`.
* Logs do pod `kube-vip-ds` (namespace `kube-system`) cheios de:
  ```
  error retrieving resource lock kube-system/plndr-cp-lock: Get "https://10.43.0.1:443/apis/coordination.k8s.io/v1/namespaces/kube-system/leases/plndr-cp-lock?timeout=10s": dial tcp 10.43.0.1:443: connect: no route to host
  ```

---

### 2. Processo de Investigação & Descoberta

Primeiro passo (que acabou sendo um raciocínio correto, mas insuficiente sozinho): o `kubectl` do usuário estava configurado apontando direto pro IP real do control-plane em vez do VIP do Kube-VIP — corrigido pra apontar pro `.101` (o padrão correto, já usado pelo join do worker2: `K3S_URL=https://192.168.3.101:6443` em `clusters/worker2/config.md`). Mas o erro virou **timeout**, não o erro de certificado esperado só por um IP de origem errado — sinal de que o problema era mais profundo que uma configuração de cliente.

Pra investigar sem depender da rede externa (que era justamente a camada suspeita), a investigação foi direto via SSH no control-plane, usando o kubeconfig local do próprio k3s (`sudo k3s kubectl`, que fala com a API via `127.0.0.1`, imune a qualquer problema de IP/VIP externo):
* `sudo systemctl status k3s` — processo rodando normalmente, sem crash.
* `sudo k3s kubectl get nodes -o wide` — funcionou (confirmando que a API em si estava de pé), revelando `worker1`/`worker2` como `NotReady`.
* `ip -4 -o addr show` — confirmou que todos os `/32` do Kube-VIP tinham sumido, restando só o IP dinâmico novo.
* Logs do `kube-vip-ds` — revelaram que ele estava preso tentando (e falhando) fazer eleição de líder via a API do Kubernetes.

#### Causa Raiz: o bounce de interface removeu os IPs virtuais do Kube-VIP, e ele ficou sem conseguir se recuperar sozinho porque dependia da própria API — que, por sua vez, dependia de rotas internas quebradas pelo mesmo bounce (impasse circular)
O Kube-VIP gerencia seus IPs flutuantes (VIP da API `.101`, pool de LoadBalancer `.200`–`.214`) como endereços secundários `/32` adicionados diretamente na interface física do node via netlink — completamente à parte da configuração de rede do sistema operacional (NetworkManager/DHCP). Esses endereços só existem porque o processo do Kube-VIP os reafirma continuamente enquanto detém o `Lease` de liderança correspondente; não são persistidos em lugar nenhum fora do próprio estado em memória dele.

Quando a interface foi derrubada (pra forçar a renovação do DHCP), o kernel removeu **todos** os endereços daquele dispositivo, inclusive os VIPs do Kube-VIP. Quando ela voltou, o NetworkManager só restaurou o endereço dinâmico de verdade (via DHCP) — nada restaurou os endereços secundários geridos pelo Kube-VIP, porque esses pertencem ao estado do próprio pod, não à pilha de rede do SO.

O container do Kube-VIP também não os readicionou sozinho, porque pra reivindicar o VIP com segurança ele primeiro precisa vencer a eleição de líder (via a API do Kubernetes, pra evitar dois nodes reivindicando o mesmo VIP ao mesmo tempo — mesmo não sendo um risco real aqui, com um único control-plane, o mecanismo não faz essa distinção). Pra fazer essa eleição, ele lê/atualiza um objeto `Lease` através do **ClusterIP interno do cluster** (`10.43.0.1`), não via `localhost` diretamente. Esse ClusterIP só é alcançável graças às regras de DNAT programadas pelo kube-proxy/k3s — e, assim como os VIPs, essas regras não são parte da configuração persistente do SO; são programadas em tempo de execução pelo próprio k3s. O mesmo bounce de interface deixou esse estado de roteamento inconsistente, então até o apiserver (acessível de boa via `localhost`, por isso o `sudo k3s kubectl` funcionava) ficou inalcançável pelo Kube-VIP via o ClusterIP — gerando o loop de `no route to host` nos logs. O Kube-VIP ficou travado esperando por uma infraestrutura que tinha sido quebrada pelo mesmo evento que o afetou, sem nada pra romper o ciclo sozinho.

Isso também explica a reclamação do kubelet (`node IP ... not found`): o control-plane tinha, historicamente, o próprio `.101` (o VIP) registrado como seu IP interno (`node-ip`) — o que funcionava enquanto o Kube-VIP mantinha esse endereço vivo, e virou uma falha de validação dura assim que ele sumiu.

**Ponto-chave**: um `systemctl restart k3s` resolveu tudo de uma vez porque reconstrói, do zero, tanto as regras de rede internas (kube-proxy/flannel) quanto o estado do apiserver — quebrando o ciclo em que o Kube-VIP dependia de uma API que, por sua vez, dependia de uma rede interna quebrada pelo mesmo evento.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# No PC do usuário - sintoma inicial
kubectl get nodes
# Client.Timeout exceeded while awaiting headers

# Via SSH direto no control-plane, contornando o problema de rede externa
sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide
sudo k3s kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip-ds --tail=80

# Estado real dos IPs na interface - revelou que os VIPs tinham sumido
ip -4 -o addr show | grep -v ' lo '

# Log do kubelet confirmando a falha de validação do node-ip
sudo journalctl -u k3s --no-pager | grep -i "nodeIP"
```

---

### 4. A Correção Definitiva (ou Workaround)

A correção foi reiniciar o serviço do k3s no control-plane, o que reconstrói a rede interna (kube-proxy/flannel) e destrava o Kube-VIP pra reconquistar seus VIPs:

```bash
sudo systemctl restart k3s
```

Em paralelo, corrigido também o `kubectl` configurado localmente (que estava apontando pro IP real do node, `.100`, em vez do VIP estável `.101`):
```yaml
# ~/.kube/config
server: https://192.168.3.101:6443
```

#### Validação final
```bash
sudo systemctl status k3s --no-pager
ip -4 -o addr show | grep -v ' lo '
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide
```

Dentro de ~1 minuto após o restart: todos os VIPs do Kube-VIP reapareceram na interface, `worker2` voltou pra `Ready`, o `INTERNAL-IP` do control-plane passou a refletir corretamente seu endereço real (`.100`, não mais o VIP `.101`), e o `kubectl` do usuário voltou a funcionar normalmente apontando pro `.101`. `worker1` continuou `NotReady`, mas por um motivo não relacionado (node fisicamente desligado no momento).

---

### 5. Lições Aprendidas para o Homelab

1. **Nunca derrube (`down`/`up`) a interface de rede principal de um node rodando Kube-VIP (ou qualquer ferramenta que gerencie IPs secundários via netlink) sem esperar que *todos* os IPs virtuais dela desapareçam** — eles não são restaurados automaticamente pelo SO, só pela reconciliação da própria ferramenta, que pode depender justamente da rede que está tentando consertar.
2. **O IP de identidade de um node (`node-ip` do kubelet) deveria ser o endereço real e estável da sua interface — nunca um IP virtual/flutuante gerenciado por outra ferramenta.** Fixar o `node-ip` num VIP por conveniência transforma qualquer instabilidade desse VIP numa falha de validação do próprio node, não só num problema de acesso externo.
3. **O roteamento de Service do kube-proxy (ClusterIP → endpoint real) é programado pelo próprio k3s em tempo de execução** e vive no mesmo tipo de estado frágil e não-persistente que qualquer outro construto dinâmico do Kubernetes — não sobrevive independente do processo do k3s rodando, então um restart limpo costuma ser o jeito mais rápido de recuperar um node cuja rede interna foi perturbada.
4. **Diagnosticar um control-plane com a API externamente inacessível sempre tem um fallback confiável**: entrar via SSH no próprio node e usar o kubectl/kubeconfig local dele (`sudo k3s kubectl ...`), que fala com o apiserver via `localhost` e contorna qualquer problema de IP/VIP/DNS externo.
5. **Mudar a identidade de rede "estável" de um node (mesmo só uma reserva de DHCP) nunca é puramente cosmético num cluster com qualquer camada de IP virtual por cima (Kube-VIP, MetalLB, etc.)** — valide a cadeia inteira (VIP voltado pro cliente, IP próprio registrado do node, roteamento interno de Service) antes e depois da mudança, não só o endereço que você pretendia alterar.

---

## English

After forcing a DHCP lease renewal on the control-plane node (bouncing the network interface to pick up a new fixed reservation on the router), the whole cluster became unstable: `kubectl` stopped working, `worker1`/`worker2` went `NotReady`, and Kube-VIP couldn't recover on its own — stuck in a circular deadlock that only a `systemctl restart k3s` resolved.

---

### 1. Initial Scenario and Symptoms

Context: the TrueNAS NFS export used by `nfs-subdir-external-provisioner` authorized hosts by IP, and the control-plane's real interface IP was dynamic via DHCP (and changed — see [`ideas/nfs_provisioners_control_plane_affinity.md`](../ideas/nfs_provisioners_control_plane_affinity.md)). To fix that, a fixed DHCP reservation (`192.168.3.100`) was set on the router for the control-plane's MAC, and the lease renewal was forced via `nmcli connection down`/`up` on the `enp1s0` interface.

#### Error Symptoms:
* `kubectl`, configured on the user's PC pointing at the Kube-VIP virtual IP (`192.168.3.101`), started **timing out** (not a certificate error, which is what would be expected from just a kubeconfig IP mismatch).
* On the control-plane, `ip -4 -o addr show` showed **only** the new dynamic IP (`192.168.3.100/24`) — every virtual IP Kube-VIP managed (`192.168.3.101`, the API VIP, and the entire LoadBalancer pool `192.168.3.200`–`192.168.3.214`) had vanished from the interface.
* `journalctl -u k3s` on the control-plane, repeatedly: `failed to validate nodeIP: node IP: "192.168.3.101" not found in the host's network interfaces`.
* `kubectl get nodes` (run locally on the control-plane via `sudo k3s kubectl`, bypassing the broken external network path) showed `worker1` and `worker2` as `NotReady`.
* Logs from the `kube-vip-ds` pod (`kube-system` namespace) full of:
  ```
  error retrieving resource lock kube-system/plndr-cp-lock: Get "https://10.43.0.1:443/apis/coordination.k8s.io/v1/namespaces/kube-system/leases/plndr-cp-lock?timeout=10s": dial tcp 10.43.0.1:443: connect: no route to host
  ```

---

### 2. Investigation & Discovery Process

First step (correct reasoning, but not sufficient on its own): the user's `kubectl` was configured pointing directly at the control-plane's real IP instead of Kube-VIP's VIP — fixed to point at `.101` (the correct pattern, already used by worker2's join: `K3S_URL=https://192.168.3.101:6443` in `clusters/worker2/config.md`). But the error turned into a **timeout**, not the certificate error expected from just a wrong source IP — a sign the problem ran deeper than a client-side misconfiguration.

To investigate without depending on the external network (which was itself the suspect layer), the investigation went straight to SSH on the control-plane, using k3s's own local kubeconfig (`sudo k3s kubectl`, which talks to the API over `127.0.0.1`, immune to any external IP/VIP issue):
* `sudo systemctl status k3s` — process running normally, no crash.
* `sudo k3s kubectl get nodes -o wide` — worked (confirming the API itself was up), revealing `worker1`/`worker2` as `NotReady`.
* `ip -4 -o addr show` — confirmed every Kube-VIP `/32` had vanished, leaving only the new dynamic IP.
* `kube-vip-ds` logs — revealed it was stuck trying (and failing) to perform leader election via the Kubernetes API.

#### Root Cause: the interface bounce removed Kube-VIP's virtual IPs, and it couldn't recover on its own because it depended on the API itself — which, in turn, depended on internal routes broken by the same bounce (circular deadlock)
Kube-VIP manages its floating IPs (API VIP `.101`, LoadBalancer pool `.200`–`.214`) as secondary `/32` addresses added directly to the node's physical interface via netlink — entirely separate from the OS's own network configuration (NetworkManager/DHCP). Those addresses only exist because the Kube-VIP process keeps re-asserting them while it holds the corresponding leadership `Lease`; they aren't persisted anywhere outside its own in-memory state.

When the interface was brought down (to force the DHCP renewal), the kernel dropped **every** address on that device, including Kube-VIP's VIPs. When it came back up, NetworkManager only restored the real dynamic address (via DHCP) — nothing restored the Kube-VIP-managed secondary addresses, since those belong to the pod's own state, not to the OS's network stack.

Kube-VIP's own container didn't re-add them on its own either, because to safely reclaim a VIP it first needs to win leader election (via the Kubernetes API, to prevent two nodes from claiming the same VIP at once — not an actual risk here with a single control-plane, but the mechanism doesn't special-case that). To do that election, it reads/updates a `Lease` object through the cluster's **internal ClusterIP** (`10.43.0.1`), not directly via `localhost`. That ClusterIP is only reachable thanks to DNAT rules programmed by kube-proxy/k3s — and, just like the VIPs, those rules aren't part of the OS's persistent config; they're programmed at runtime by k3s itself. The same interface bounce left that routing state inconsistent, so even the apiserver (reachable fine via `localhost`, which is why `sudo k3s kubectl` worked) was unreachable by Kube-VIP through the ClusterIP — producing the `no route to host` loop in its logs. Kube-VIP was stuck waiting on infrastructure broken by the very same event that affected it, with nothing to break the cycle on its own.

This also explains kubelet's own complaint (`node IP ... not found`): the control-plane had historically registered its own `.101` (the VIP) as its internal node IP (`node-ip`) — which worked as long as Kube-VIP kept that address alive, and turned into a hard validation failure the moment it disappeared.

**Key point**: a single `systemctl restart k3s` fixed everything at once because it rebuilds, from scratch, both the internal network rules (kube-proxy/flannel) and the apiserver's own state — breaking the cycle where Kube-VIP depended on an API that, in turn, depended on an internal network broken by the same event.

---

### 3. Commands Used for Diagnosis

```bash
# On the user's PC - initial symptom
kubectl get nodes
# Client.Timeout exceeded while awaiting headers

# Via direct SSH on the control-plane, bypassing the broken external network
sudo systemctl status k3s --no-pager
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide
sudo k3s kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip-ds --tail=80

# Real IP state on the interface - revealed the VIPs had vanished
ip -4 -o addr show | grep -v ' lo '

# kubelet log confirming the node-ip validation failure
sudo journalctl -u k3s --no-pager | grep -i "nodeIP"
```

---

### 4. The Definitive Fix (or Workaround)

The fix was restarting the k3s service on the control-plane, which rebuilds the internal network (kube-proxy/flannel) and unblocks Kube-VIP to reclaim its VIPs:

```bash
sudo systemctl restart k3s
```

In parallel, the locally configured `kubectl` was also fixed (it was pointing at the node's real IP, `.100`, instead of the stable VIP `.101`):
```yaml
# ~/.kube/config
server: https://192.168.3.101:6443
```

#### Final validation
```bash
sudo systemctl status k3s --no-pager
ip -4 -o addr show | grep -v ' lo '
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide
```

Within ~1 minute of the restart: every Kube-VIP virtual IP reappeared on the interface, `worker2` went back to `Ready`, the control-plane's `INTERNAL-IP` correctly reflected its real address (`.100`, no longer the VIP `.101`), and the user's `kubectl` worked normally again pointing at `.101`. `worker1` stayed `NotReady`, but for an unrelated reason (the node was physically powered off at the time).

---

### 5. Lessons Learned for the Homelab

1. **Never bounce (`down`/`up`) the primary network interface of a node running Kube-VIP (or any tool managing secondary IPs via netlink) without expecting *every* virtual IP on it to disappear** — they aren't restored automatically by the OS, only by the tool's own reconciliation, which may itself depend on the very network it's trying to fix.
2. **A node's identity IP (kubelet's `node-ip`) should be its real, stable interface address — never a virtual/floating IP managed by another tool.** Pinning `node-ip` to a VIP for convenience turns any instability in that VIP into a validation failure for the node itself, not just an external-access problem.
3. **kube-proxy's Service routing (ClusterIP → real endpoint) is programmed by k3s itself at runtime** and lives in the same fragile, non-persistent state as any other dynamic Kubernetes construct — it doesn't survive independently of the running k3s process, so a clean restart is often the fastest way to recover a node whose internal network got disturbed.
4. **Diagnosing a control-plane with an externally unreachable API always has a reliable fallback**: SSH into the node itself and use its own local kubectl/kubeconfig (`sudo k3s kubectl ...`), which talks to the apiserver over `localhost` and bypasses any external IP/VIP/DNS issue.
5. **Changing a node's "stable" network identity (even just a DHCP reservation) is never purely cosmetic in a cluster with any virtual IP layer on top (Kube-VIP, MetalLB, etc.)** — validate the whole chain (client-facing VIP, the node's own registered IP, internal Service routing) before and after the change, not just the one address you meant to touch.
