# Troubleshooting: Instabilidade na stack `*arr` por "connection refused" ao acessar o Prowlarr via IP do LoadBalancer / `*arr` stack instability from "connection refused" reaching Prowlarr via its LoadBalancer IP

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

A stack `*arr` (Sonarr/Radarr/Lidarr) ficava intermitentemente indisponível/instável sem nenhum pod reiniciar. Causa raiz: Sonarr, Radarr e Lidarr estavam configurados para falar com o Prowlarr através do **IP externo do LoadBalancer** (`192.168.3.207`, kube-vip) em vez do `ClusterIP` interno do próprio Service — um caminho de rede mais frágil que ocasionalmente recusava a conexão.

---

### 1. O Cenário Inicial e Sintomas

Registrado em [known-issues/arr_stack_general_instability.md](../known-issues/arr_stack_general_instability.md) (2026-11-07): páginas da stack `*arr` (Prowlarr/Sonarr/Radarr/Lidarr/LazyLibrarian/Bazarr) ficando indisponíveis com frequência, **sem nenhum pod mostrar restart** (`RESTARTS` inalterado). Isso descartava `OOMKilled`/`CrashLoopBackOff` como causa direta.

#### Sintomas do Erro:
* Indisponibilidade intermitente de apps da stack `*arr`, recorrente mesmo com todos os nodes e o TrueNAS ligados (confirmado que não é o ciclo de economia de energia noturno — ver "Processo de Investigação").
* Jellyfin tinha problema parecido mas **com restart contabilizado** (40 restarts observados) — confirmado como [known-issue separado](../known-issues/jellyfin_frequent_pod_restarts.md), não coberto por este documento.
* Nos logs do Sonarr/Radarr/Lidarr, exceções repetidas: `System.Net.Http.HttpRequestException: Connection refused (192.168.3.207:80)`.

---

### 2. Processo de Investigação & Descoberta

Primeiro, descartadas as hipóteses mais óbvias:
- **Ciclo de economia de energia** (worker1/worker2/TrueNAS desligados de madrugada, ver `README.md`): descartado — confirmado que a instabilidade acontece em qualquer horário, mesmo com tudo ligado.
- **Contenção de recursos**: `kubectl top pods -n media` e `kubectl top nodes` mostraram uso baixo em todos os 3 nodes (7-14% CPU, 14-32% memória) — sem sinal de throttling/pressão.
- **Nodes instáveis**: `kubectl get nodes -o wide` e `kubectl describe nodes` — todos `Ready`, sem transições recentes de condição, sem `MemoryPressure`/`DiskPressure`/`PIDPressure`.
- **`kube-vip` com leader election instável**: só existe 1 pod `kube-vip-ds` (fixado só no control-plane via `nodeAffinity`, responsável por anunciar via ARP **todos** os IPs de `LoadBalancer` do cluster — não só a API). Os logs mostraram o mesmo líder desde 2026-10-28, sem nenhuma re-eleição — descarta flakiness geral do ARP/kube-vip como causa.

O que resolveu a investigação foi `kubectl logs -n media -l app=sonarr --tail=100`, mostrando repetidamente:
```
System.Net.Http.HttpRequestException: Connection refused (192.168.3.207:80)
```
`192.168.3.207` é o IP do `LoadBalancer` do **Prowlarr**. Repetindo a busca em Radarr e Lidarr (`kubectl logs ... | grep -i refused`), **todas** as ocorrências de "connection refused" apontavam especificamente para `192.168.3.207:80` — nenhum outro IP. Bazarr e LazyLibrarian não apresentaram o erro (não dependem do proxy Torznab do Prowlarr da mesma forma). Isso isolou o problema especificamente ao caminho de rede até o Prowlarr, não a uma instabilidade geral do cluster.

#### Causa Raiz: tráfego interno Sonarr/Radarr/Lidarr → Prowlarr saindo pelo IP externo do LoadBalancer
Em `apps/servarr/prowlarr/service.yaml`, o Service `prowlarr` é do tipo `LoadBalancer` (`loadBalancerIP: 192.168.3.207`, `port: 80` → `targetPort: 9696`). Toda `Service` `LoadBalancer` no Kubernetes **também** recebe um `ClusterIP` automaticamente — um caminho que nunca sai da rede overlay do cluster. Porém, em **Prowlarr → Settings → Apps → (Sonarr/Radarr/Lidarr) → campo "Prowlarr Server"** (a URL que o Prowlarr embute nas respostas Torznab que ele entrega a cada app, usada por eles pra falar de volta com o Prowlarr), o valor configurado era o **IP do LoadBalancer** (`192.168.3.207`), não o `ClusterIP`/DNS interno.

Isso fazia tráfego 100% interno ao cluster (pod Sonarr em `worker2` → pod Prowlarr, também em `worker2`) sair da rede overlay (flannel), atravessar a rede física/LAN até o node que anuncia o IP via ARP (o control-plane, único pod `kube-vip-ds`), e voltar — um caminho adicional e mais frágil (depende de resolução ARP, switch físico, iptables DNAT no control-plane) do que simplesmente resolver `prowlarr.media.svc.cluster.local` via `CoreDNS` e falar direto com o `ClusterIP`. Ocasionalmente essa volta pela rede física resultava em "connection refused" — sem nenhum container reiniciar, já que o problema nunca foi o pod do Prowlarr em si (confirmado também que `apps/servarr/prowlarr/deployment.yaml` não define nenhum `readinessProbe`/`livenessProbe`, descartando flapping de prontidão como mecanismo).

**Ponto-chave**: apps dentro do mesmo cluster que se comunicam entre si **não devem** usar o IP externo de um `LoadBalancer`/kube-vip — esse IP existe pra acesso de fora (LAN), e usá-lo pra tráfego interno adiciona uma dependência desnecessária em ARP/rede física que o `ClusterIP` (sempre disponível, sempre dentro do overlay) não tem.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Snapshot geral: nodes de cada pod e contagem de restarts
kubectl get pods -n media -o wide

# Descarta contenção de recursos
kubectl top pods -n media
kubectl top nodes

# Descarta nodes instáveis/NotReady recentemente
kubectl get nodes -o wide
kubectl describe nodes | grep -A5 "Conditions:"

# Descarta falha de leader election do kube-vip (ponto único de ARP pros LoadBalancer IPs)
kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide
kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip-ds --tail=500 | grep -i -E "leader|lease|error|lost|elected|retry"

# Evidência decisiva: erro real nos logs do app, apontando pro IP do Prowlarr
kubectl logs -n media -l app=sonarr --tail=100
kubectl logs -n media -l app=radarr --tail=1000 | grep -i "refused"
kubectl logs -n media -l app=lidarr --tail=1000 | grep -i "refused"

# Confirma que Bazarr/LazyLibrarian não apresentam o mesmo erro (isola o problema ao Prowlarr)
kubectl logs -n media -l app=bazarr --tail=1000 | grep -i "refused"
kubectl logs -n media -l app=lazylibrarian --tail=1000 | grep -i "refused"
```

---

### 4. A Correção Definitiva (ou Workaround)

Correção aplicada via UI do Prowlarr (não há manifesto Kubernetes pra esse dado — fica no SQLite do próprio Prowlarr): em **Settings → Apps**, para cada app (Sonarr, Radarr, Lidarr), o campo **"Prowlarr Server"** foi trocado do IP do LoadBalancer para o `ClusterIP`/DNS interno do Service, usando o nome curto já que todos os apps estão no mesmo namespace `media`:

```
Antes: http://192.168.3.207:80
Depois: http://prowlarr:80
```

Confirmado via navegador que o valor ficou correto em Sonarr e Radarr após a mudança.

#### Validação final
Após a mudança, observação do comportamento confirmou maior estabilidade (sem recorrência do "connection refused" nos logs). Validação formal recomendada para o futuro, caso o problema reapareça:
```bash
# Confirma ausência do erro depois de um período de observação
kubectl logs -n media -l app=sonarr --tail=200 | grep -i "refused"
kubectl logs -n media -l app=radarr --tail=200 | grep -i "refused"
kubectl logs -n media -l app=lidarr --tail=200 | grep -i "refused"
```

---

### 5. Lições Aprendidas para o Homelab

1. **Tráfego interno ao cluster nunca deve depender do IP externo de um `LoadBalancer`** — toda `Service` `LoadBalancer` no Kubernetes também ganha um `ClusterIP` automaticamente; apps no mesmo cluster (e, melhor ainda, no mesmo namespace) devem usar o DNS interno (`<service>.<namespace>.svc.cluster.local` ou só `<service>` no mesmo namespace), nunca o IP pensado pra acesso externo/LAN.
2. **"Connection refused" (não timeout) sem nenhum restart de pod aponta pra um problema de caminho de rede, não do processo em si** — se o processo estivesse crashando/lento, o sintoma mais comum seria timeout (handshake TCP completa, resposta HTTP não vem) ou o pod reiniciando; "recusado" de forma consistente sugere que o pacote nem chegou a um listener ativo no caminho percorrido.
3. **Isolar o IP específico no erro, não só o sintoma geral, é o que destrava o diagnóstico** — "instabilidade geral da stack" parecia um problema difuso; o log bruto do Sonarr apontando `192.168.3.207` (e confirmando o mesmo padrão em Radarr/Lidarr, ausente em Bazarr/LazyLibrarian) reduziu o escopo de "algo errado em algum lugar do cluster" para "um app específico, configurado de um jeito específico".
4. **Um pod único de `kube-vip-ds` pinado ao control-plane é um ponto único pra ARP de todos os `LoadBalancer` IPs** — vale lembrar disso ao investigar qualquer instabilidade de rede futura envolvendo múltiplos `LoadBalancer` Services ao mesmo tempo, mesmo que não tenha sido a causa desta vez (o líder estava estável há 12 dias).

**Nota**: o known-issue [`prowlarr_indexers_failed_to_load_from_api.md`](../known-issues/prowlarr_indexers_failed_to_load_from_api.md) (Prowlarr falhando ao carregar indexers **externos**) pode estar relacionado — mesma classe de app, mesmo node — mas trata de tráfego de **saída** do Prowlarr para a internet, uma direção diferente da corrigida aqui (entrada no Prowlarr vinda do Sonarr/Radarr/Lidarr). Não foi validado que esta correção o resolve; continua em aberto em `known-issues.md`.

---

## English

The `*arr` stack (Sonarr/Radarr/Lidarr) was intermittently unavailable/unstable with no pod ever restarting. Root cause: Sonarr, Radarr, and Lidarr were configured to talk to Prowlarr through its **external LoadBalancer IP** (`192.168.3.207`, kube-vip) instead of the Service's internal `ClusterIP` — a more fragile network path that occasionally refused the connection.

---

### 1. Initial Scenario and Symptoms

Logged in [known-issues/arr_stack_general_instability.md](../known-issues/arr_stack_general_instability.md) (2026-11-07): `*arr` stack pages (Prowlarr/Sonarr/Radarr/Lidarr/LazyLibrarian/Bazarr) becoming unavailable frequently, **with no pod ever showing a restart** (`RESTARTS` unchanged). This ruled out `OOMKilled`/`CrashLoopBackOff` as the direct cause.

#### Error Symptoms:
* Intermittent unavailability of `*arr` stack apps, recurring even with every node and TrueNAS powered on (confirmed this wasn't the overnight power-saving cycle — see "Investigation Process").
* Jellyfin had a similar-looking problem but **with an actual restart count** (40 restarts observed) — confirmed as a [separate known-issue](../known-issues/jellyfin_frequent_pod_restarts.md), not covered by this document.
* In Sonarr/Radarr/Lidarr logs, repeated exceptions: `System.Net.Http.HttpRequestException: Connection refused (192.168.3.207:80)`.

---

### 2. Investigation & Discovery Process

First, the more obvious hypotheses were ruled out:
- **Power-saving cycle** (worker1/worker2/TrueNAS shut down overnight, see `README.md`): ruled out — confirmed the instability happens at any time of day, even with everything powered on.
- **Resource contention**: `kubectl top pods -n media` and `kubectl top nodes` showed low usage across all 3 nodes (7-14% CPU, 14-32% memory) — no sign of throttling/pressure.
- **Unstable nodes**: `kubectl get nodes -o wide` and `kubectl describe nodes` — all `Ready`, no recent condition transitions, no `MemoryPressure`/`DiskPressure`/`PIDPressure`.
- **Unstable `kube-vip` leader election**: only 1 `kube-vip-ds` pod exists (pinned to the control-plane only via `nodeAffinity`, responsible for ARP-announcing **every** `LoadBalancer` IP in the cluster — not just the API's). Logs showed the same leader since 2026-10-28, with no re-elections — ruling out general ARP/kube-vip flakiness as the cause.

What cracked the investigation was `kubectl logs -n media -l app=sonarr --tail=100`, repeatedly showing:
```
System.Net.Http.HttpRequestException: Connection refused (192.168.3.207:80)
```
`192.168.3.207` is **Prowlarr**'s `LoadBalancer` IP. Repeating the search in Radarr and Lidarr (`kubectl logs ... | grep -i refused`), **every** "connection refused" occurrence pointed specifically to `192.168.3.207:80` — no other IP. Bazarr and LazyLibrarian showed no such error (they don't depend on Prowlarr's Torznab proxy the same way). This isolated the problem specifically to the network path to Prowlarr, not a general cluster instability.

#### Root Cause: internal Sonarr/Radarr/Lidarr → Prowlarr traffic going out through the external LoadBalancer IP
In `apps/servarr/prowlarr/service.yaml`, the `prowlarr` Service is type `LoadBalancer` (`loadBalancerIP: 192.168.3.207`, `port: 80` → `targetPort: 9696`). Every `LoadBalancer` Service in Kubernetes **also** automatically gets a `ClusterIP` — a path that never leaves the cluster's overlay network. However, in **Prowlarr → Settings → Apps → (Sonarr/Radarr/Lidarr) → "Prowlarr Server" field** (the URL Prowlarr embeds in the Torznab responses it hands each app, used by them to talk back to Prowlarr), the configured value was the **LoadBalancer IP** (`192.168.3.207`), not the internal `ClusterIP`/DNS.

This sent 100%-internal cluster traffic (the Sonarr pod on `worker2` → the Prowlarr pod, also on `worker2`) out of the overlay network (flannel), across the physical/LAN network to the node announcing the IP via ARP (the control-plane, sole `kube-vip-ds` pod), and back — an extra, more fragile path (dependent on ARP resolution, the physical switch, control-plane iptables DNAT) compared to simply resolving `prowlarr.media.svc.cluster.local` via `CoreDNS` and talking directly to the `ClusterIP`. That round trip through the physical network occasionally resulted in "connection refused" — with no container ever restarting, since the problem was never Prowlarr's pod itself (also confirmed that `apps/servarr/prowlarr/deployment.yaml` defines no `readinessProbe`/`livenessProbe`, ruling out readiness flapping as the mechanism).

**Key point**: apps within the same cluster talking to each other **should never** use a `LoadBalancer`/kube-vip's external IP — that IP exists for access from outside (the LAN), and using it for internal traffic adds an unnecessary dependency on ARP/physical networking that the `ClusterIP` (always available, always inside the overlay) doesn't have.

---

### 3. Commands Used for Diagnosis

```bash
# General snapshot: which node each pod is on and restart counts
kubectl get pods -n media -o wide

# Rules out resource contention
kubectl top pods -n media
kubectl top nodes

# Rules out recently unstable/NotReady nodes
kubectl get nodes -o wide
kubectl describe nodes | grep -A5 "Conditions:"

# Rules out kube-vip leader election failure (single point of ARP for all LoadBalancer IPs)
kubectl get pods -n kube-system -l app.kubernetes.io/name=kube-vip-ds -o wide
kubectl logs -n kube-system -l app.kubernetes.io/name=kube-vip-ds --tail=500 | grep -i -E "leader|lease|error|lost|elected|retry"

# Decisive evidence: the real error in the app's own logs, pointing at Prowlarr's IP
kubectl logs -n media -l app=sonarr --tail=100
kubectl logs -n media -l app=radarr --tail=1000 | grep -i "refused"
kubectl logs -n media -l app=lidarr --tail=1000 | grep -i "refused"

# Confirms Bazarr/LazyLibrarian don't show the same error (isolates the problem to Prowlarr)
kubectl logs -n media -l app=bazarr --tail=1000 | grep -i "refused"
kubectl logs -n media -l app=lazylibrarian --tail=1000 | grep -i "refused"
```

---

### 4. The Definitive Fix (or Workaround)

Fix applied via Prowlarr's UI (there's no Kubernetes manifest for this data — it lives in Prowlarr's own SQLite): under **Settings → Apps**, for each app (Sonarr, Radarr, Lidarr), the **"Prowlarr Server"** field was changed from the LoadBalancer IP to the Service's internal `ClusterIP`/DNS, using the short name since every app lives in the same `media` namespace:

```
Before: http://192.168.3.207:80
After:  http://prowlarr:80
```

Confirmed via browser that the value is correctly set on both Sonarr and Radarr after the change.

#### Final validation
After the change, observed behavior confirmed improved stability (no recurrence of "connection refused" in the logs). Recommended formal validation if the problem were to reappear:
```bash
# Confirms the error is absent after an observation period
kubectl logs -n media -l app=sonarr --tail=200 | grep -i "refused"
kubectl logs -n media -l app=radarr --tail=200 | grep -i "refused"
kubectl logs -n media -l app=lidarr --tail=200 | grep -i "refused"
```

---

### 5. Lessons Learned for the Homelab

1. **Traffic internal to the cluster should never depend on a `LoadBalancer`'s external IP** — every Kubernetes `LoadBalancer` Service also automatically gets a `ClusterIP`; apps in the same cluster (and, better yet, the same namespace) should use internal DNS (`<service>.<namespace>.svc.cluster.local`, or just `<service>` within the same namespace), never the IP meant for external/LAN access.
2. **"Connection refused" (not a timeout) with no pod restart points to a network-path problem, not the process itself** — if the process were crashing/slow, the more common symptom would be a timeout (TCP handshake completes, HTTP response never arrives) or the pod restarting; a consistent "refused" suggests the packet never even reached an active listener along the path it took.
3. **Isolating the specific IP in the error, not just the general symptom, is what unlocks the diagnosis** — "general stack instability" looked like a diffuse problem; Sonarr's raw log pointing at `192.168.3.207` (and confirming the same pattern in Radarr/Lidarr, absent in Bazarr/LazyLibrarian) narrowed "something's wrong somewhere in the cluster" down to "one specific app, configured one specific way."
4. **A single `kube-vip-ds` pod pinned to the control-plane is a single point of ARP for every `LoadBalancer` IP** — worth remembering when investigating any future network instability involving multiple `LoadBalancer` Services at once, even though it wasn't the cause this time (the leader had been stable for 12 days).

**Note**: the known-issue [`prowlarr_indexers_failed_to_load_from_api.md`](../known-issues/prowlarr_indexers_failed_to_load_from_api.md) (Prowlarr failing to load **external** indexers) may be related — same app, same node — but it's about Prowlarr's **outbound** traffic to the internet, a different direction from what was fixed here (inbound to Prowlarr from Sonarr/Radarr/Lidarr). It hasn't been validated that this fix resolves it; it remains open in `known-issues.md`.
