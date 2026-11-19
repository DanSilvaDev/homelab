# Troubleshooting: Reset do Sonarr invalidou a API key em todos os apps integrados (Bazarr, Prowlarr, Seerr) / Resetting Sonarr invalidated the API key in every integrated app (Bazarr, Prowlarr, Seerr)

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Depois de resetar a configuração do Sonarr (banco corrompido por um import instável), o pod voltou a oscilar com CPU alta e sem reiniciar. A causa não era recurso nem corrupção nova — era a API key antiga, ainda guardada em três outros apps (Bazarr, Prowlarr, Seerr), martelando o Sonarr sem parar.

---

### 1. O Cenário Inicial e Sintomas

O Sonarr oscilou durante o processamento de um import de mídia e entrou num estado de falha interna silenciosa (sem o pod reiniciar), que acabou excluindo um arquivo de episódio já importado. Sem snapshot/backup configurado no TrueNAS, essa mídia específica foi perdida. Decidido resetar só a configuração do Sonarr (`sonarr-pvc`, montada em `/config`) para reconfigurar do zero, mantendo a `media-pvc` intacta (confirmado no manifesto que são PVCs separadas).

#### Sintomas do Erro:
* Sonarr instável durante processamento de import, sem o Pod reiniciar (`RESTARTS: 0`).
* Depois do reset do `/config` e subida do Pod, a instabilidade voltou: CPU sustentada próxima do limite (ex: 727m de 1000m), sem nenhum restart contado pelo Kubernetes.
* Log (`/config/logs/sonarr.txt`) mostrando duas sequências completas de boot ("Bootstrap: Starting Sonarr" → migração do banco → "Now listening") uma atrás da outra, sem o container cair.
* Diretório `/config/logs/` rotacionando arquivos de ~1MB várias vezes em poucos minutos.
* Volume enorme da linha `AuthenticationScheme: SignalR was challenged` / `was not authenticated`, repetindo várias vezes por segundo.

---

### 2. Processo de Investigação & Descoberta

Primeiro, confirmado no `apps/servarr/sonarr/deployment.yaml` que `sonarr-pvc` (`/config`) e `media-pvc` (`/media`) são PVCs completamente separadas — resetar uma não afeta a outra.

Antes de suspeitar de limites de recurso, usado o Prometheus (já disponível no cluster) pra verificar com dado real em vez de chutar:
* `container_cpu_usage_seconds_total{namespace="media", pod=~"sonarr.*"}` confirmou que existe métrica de cAdvisor pro pod.
* `sum(rate(container_cpu_usage_seconds_total{namespace="media", pod=~"sonarr.*", container="sonarr"}[5m])) by (pod)` nunca passou de **0.833** no histórico, abaixo do limite de 1 core configurado.
* `container_memory_working_set_bytes{namespace="media", pod=~"sonarr.*", container="sonarr"} / 1024 / 1024` teve pico de **~200Mi**, bem abaixo do limite de 512Mi.
* `kube_pod_container_status_last_terminated_reason{namespace="media", pod=~"sonarr.*", reason="OOMKilled"}` não retornou nenhuma série.

Conclusão: CPU/memória nunca foram o gargalo real. Um aumento especulativo de limites (`1000m/512Mi` → `2000m/1Gi`) feito antes dessa checagem foi revertido.

Depois do reset e da reinstabilidade reaparecer, `kubectl exec ... -- ps aux` mostrou que o processo do Sonarr (`/app/sonarr/bin/Sonarr`) tinha um horário de início **mais recente** que o PID 1 do container (`s6-svscan`) — ou seja, o processo já tinha reiniciado sozinho, supervisionado pelo `s6-overlay` (comum em imagens linuxserver.io), sem o container nunca cair e sem o `RestartCount` do Kubernetes subir.

#### Causa Raiz: API key antiga em apps integrados, martelando o Sonarr em loop
O reset do `/config` apaga o `config.xml`, e o Sonarr gera uma **API key nova** no primeiro boot. Três outros apps do stack guardam sua própria cópia da API key do Sonarr, em configs separadas (PVCs próprias, não tocadas pelo reset):

* **Bazarr** mantém uma conexão SignalR persistente com o Sonarr (pra receber eventos de arquivo em tempo real) e falha a autenticação em loop apertado (retentando sem backoff), consumindo CPU e gerando a enxurrada de `SignalR was challenged`/`not authenticated` no log do Sonarr.
* **Prowlarr** sincroniza indexers dentro do Sonarr via sua integração "Apps"; com a key antiga, as chamadas falhavam (`Connection refused`/`404 Not Found` em IDs de indexer que não existiam mais no banco novo do Sonarr) e o próprio Prowlarr também logava `API was challenged` do lado dele.
* **Seerr** também guarda a API key do Sonarr pra gerenciar pedidos de mídia.

**Ponto-chave**: resetar a configuração de um app do stack *arr invalida a API key dele pra **todo** app que integra com ele — não só pro app resetado. Instabilidade sem restart de Pod, logo depois de um reset desses, é sinal forte pra checar os outros apps que guardam essa key, antes de suspeitar de recurso ou corrupção nova.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirmar que config e mídia são PVCs separadas antes de resetar
# (ver apps/servarr/sonarr/deployment.yaml)

# Checar se CPU realmente bateu no limite (dado real, não suposição)
# Prometheus UI (http://192.168.3.203/graph):
sum(rate(container_cpu_usage_seconds_total{namespace="media", pod=~"sonarr.*", container="sonarr"}[5m])) by (pod)
container_memory_working_set_bytes{namespace="media", pod=~"sonarr.*", container="sonarr"} / 1024 / 1024
kube_pod_container_status_last_terminated_reason{namespace="media", pod=~"sonarr.*", reason="OOMKilled"}

# Confirmar reinício interno do processo (sem restart do Pod)
kubectl -n media exec <pod-sonarr> -- ps aux

# Ver rotação de log (indício de volume anormal em pouco tempo)
kubectl -n media exec <pod-sonarr> -- ls -la /config/logs/

# Medir a frequência real do loop de autenticação
kubectl -n media exec <pod-sonarr> -- grep -c "SignalR was challenged" /config/logs/sonarr.txt
kubectl -n media exec <pod-sonarr> -- grep "SignalR was challenged" /config/logs/sonarr.txt | tail -5
kubectl -n media exec <pod-sonarr> -- grep -i -B5 "SignalR was challenged" /config/logs/sonarr.debug.txt | tail -60

# Descartar cache/sessão do navegador como causa (testado em aba anônima - não resolveu)

# Procurar quais outros apps do repo referenciam o Sonarr
grep -ril sonarr apps/homepage apps/

# Confirmar nos logs de cada app candidato
kubectl -n media logs <pod-bazarr> --tail=200 | grep -i -B3 -A3 sonarr
kubectl -n media logs <pod-prowlarr> --tail=1000 | grep -i -B3 -A3 sonarr

# Pegar a API key nova do Sonarr pra comparar/corrigir nos outros apps
kubectl -n media exec <pod-sonarr> -- grep -i apikey /config/config.xml
```

---

### 4. A Correção Definitiva (ou Workaround)

1. Reset do `/config` do Sonarr via pod temporário (`apps/helpers/sonarr-config-wipe-pod.yaml`), com o Deployment escalado pra 0 antes e voltando pra 1 depois.
2. Revertido o aumento especulativo de `resources.limits` no `deployment.yaml` (voltou a `1000m`/`512Mi`), já que o Prometheus mostrou que recurso nunca foi o gargalo.
3. Atualizada a API key nova do Sonarr em cada app integrado, um de cada vez, confirmando a melhora a cada passo:
   - **Bazarr** → Settings → Sonarr → API Key → Test → Save.
   - **Prowlarr** → Settings → Apps → (entrada do Sonarr) → API Key → Test → Save, seguido de **Sync App Indexers** manual pra forçar a recriação dos indexers no Sonarr (em vez de esperar o ciclo automático).
   - **Seerr** → configuração equivalente de integração com o Sonarr → atualizar a API key.

#### Validação final
```bash
kubectl -n media exec <pod-sonarr> -- grep -c "SignalR was challenged" /config/logs/sonarr.txt
```
Contador parou de crescer (ficou estável em `2921`) depois da correção nos três apps — confirmando que o loop cessou de vez. `kubectl top pod -n media` também voltou a valores normais de CPU pro Sonarr.

---

### 5. Lições Aprendidas para o Homelab

1. **Resetar a config de um app do stack *arr invalida sua API key em todo app que integra com ele** — não só nele mesmo. Depois de um reset, é preciso atualizar a key manualmente em cada app downstream (visto aqui: Bazarr, Prowlarr, Seerr; potencialmente também Lidarr/Readarr/Overseerr/Jellyseerr em outros stacks) antes de considerar o reset concluído.
2. **Um Pod "no ar, sem restart, mas instável" pode ser o supervisor de processo interno da imagem (s6-overlay, comum em linuxserver.io) reiniciando só o processo da aplicação** — o `RestartCount` do Kubernetes não captura isso. `ps aux` dentro do pod, comparando o horário de início do processo da app com o PID 1 do container, expõe esse tipo de reinício invisível.
3. **Antes de assumir que CPU/memória são a causa de instabilidade, confirme com o Prometheus já existente no cluster** (`container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`, `kube_pod_container_status_last_terminated_reason`) em vez de só aumentar limites por suposição — evita reservar recurso desnecessário num node compartilhado e já com capacidade limitada.
4. **Um loop de autenticação que persiste mesmo numa aba anônima do navegador não é sessão/cache do cliente** — descarta rápido a hipótese de navegador e direciona a investigação pra integrações server-to-server (outros apps do stack), que é exatamente onde estava o problema aqui.

---

## English

After resetting Sonarr's configuration (database corrupted by an unstable import), the pod went back to oscillating with high CPU and no restarts. The cause wasn't resources or new corruption — it was the old API key, still stored in three other apps (Bazarr, Prowlarr, Seerr), hammering Sonarr nonstop.

---

### 1. Initial Scenario and Symptoms

Sonarr became unstable while processing a media import and entered a silent internal failure state (without the pod restarting), which ended up deleting an already-imported episode file. With no snapshot/backup configured on TrueNAS, that specific media was lost. Decided to reset only Sonarr's configuration (`sonarr-pvc`, mounted at `/config`) to reconfigure from scratch, keeping `media-pvc` intact (confirmed from the manifest that they are separate PVCs).

#### Error Symptoms:
* Sonarr unstable while processing an import, without the Pod restarting (`RESTARTS: 0`).
* After resetting `/config` and bringing the Pod back up, the instability returned: sustained CPU close to the limit (e.g. 727m of 1000m), with no restart counted by Kubernetes.
* Log (`/config/logs/sonarr.txt`) showing two complete boot sequences ("Bootstrap: Starting Sonarr" → database migration → "Now listening") back to back, with the container never dying.
* `/config/logs/` directory rotating ~1MB files several times within a few minutes.
* Huge volume of the `AuthenticationScheme: SignalR was challenged` / `was not authenticated` line, repeating several times per second.

---

### 2. Investigation & Discovery Process

First, confirmed in `apps/servarr/sonarr/deployment.yaml` that `sonarr-pvc` (`/config`) and `media-pvc` (`/media`) are completely separate PVCs — resetting one doesn't affect the other.

Before suspecting resource limits, used Prometheus (already available in the cluster) to check with real data instead of guessing:
* `container_cpu_usage_seconds_total{namespace="media", pod=~"sonarr.*"}` confirmed cAdvisor metrics exist for the pod.
* `sum(rate(container_cpu_usage_seconds_total{namespace="media", pod=~"sonarr.*", container="sonarr"}[5m])) by (pod)` never exceeded **0.833** historically, below the configured 1-core limit.
* `container_memory_working_set_bytes{namespace="media", pod=~"sonarr.*", container="sonarr"} / 1024 / 1024` peaked at **~200Mi**, well under the 512Mi limit.
* `kube_pod_container_status_last_terminated_reason{namespace="media", pod=~"sonarr.*", reason="OOMKilled"}` returned no series.

Conclusion: CPU/memory were never the real bottleneck. A speculative limit increase (`1000m/512Mi` → `2000m/1Gi`) made before this check was reverted.

After the reset, once the instability reappeared, `kubectl exec ... -- ps aux` showed the Sonarr process (`/app/sonarr/bin/Sonarr`) had a **more recent** start time than the container's PID 1 (`s6-svscan`) — meaning the process had already restarted on its own, supervised by `s6-overlay` (common in linuxserver.io images), without the container ever dying and without Kubernetes' `RestartCount` incrementing.

#### Root Cause: Stale API key in integrated apps, hammering Sonarr in a loop
Resetting `/config` deletes `config.xml`, and Sonarr generates a **new API key** on first boot. Three other apps in the stack keep their own copy of Sonarr's API key, in separate configs (their own PVCs, untouched by the reset):

* **Bazarr** keeps a persistent SignalR connection to Sonarr (to receive real-time file events) and failed authentication in a tight loop (retrying with no backoff), consuming CPU and flooding Sonarr's log with `SignalR was challenged`/`not authenticated`.
* **Prowlarr** syncs indexers into Sonarr via its "Apps" integration; with the stale key, those calls failed (`Connection refused`/`404 Not Found` on indexer IDs that no longer existed in Sonarr's fresh database), and Prowlarr itself also logged `API was challenged` on its own side.
* **Seerr** also stores Sonarr's API key to manage media requests.

**Key point**: resetting one *arr app's configuration invalidates its API key for **every** app that integrates with it — not just the reset app itself. Instability with no Pod restart, right after a reset like this, is a strong signal to check the other apps storing that key, before suspecting resources or new corruption.

---

### 3. Commands Used for Diagnosis

```bash
# Confirm config and media are separate PVCs before resetting
# (see apps/servarr/sonarr/deployment.yaml)

# Check whether CPU actually hit the limit (real data, not assumption)
# Prometheus UI (http://192.168.3.203/graph):
sum(rate(container_cpu_usage_seconds_total{namespace="media", pod=~"sonarr.*", container="sonarr"}[5m])) by (pod)
container_memory_working_set_bytes{namespace="media", pod=~"sonarr.*", container="sonarr"} / 1024 / 1024
kube_pod_container_status_last_terminated_reason{namespace="media", pod=~"sonarr.*", reason="OOMKilled"}

# Confirm an internal process restart (without a Pod restart)
kubectl -n media exec <pod-sonarr> -- ps aux

# Check log rotation (sign of abnormal volume in a short time)
kubectl -n media exec <pod-sonarr> -- ls -la /config/logs/

# Measure the actual frequency of the authentication loop
kubectl -n media exec <pod-sonarr> -- grep -c "SignalR was challenged" /config/logs/sonarr.txt
kubectl -n media exec <pod-sonarr> -- grep "SignalR was challenged" /config/logs/sonarr.txt | tail -5
kubectl -n media exec <pod-sonarr> -- grep -i -B5 "SignalR was challenged" /config/logs/sonarr.debug.txt | tail -60

# Rule out browser cache/session as the cause (tested in a private tab - didn't fix it)

# Look for which other apps in the repo reference Sonarr
grep -ril sonarr apps/homepage apps/

# Confirm in each candidate app's logs
kubectl -n media logs <pod-bazarr> --tail=200 | grep -i -B3 -A3 sonarr
kubectl -n media logs <pod-prowlarr> --tail=1000 | grep -i -B3 -A3 sonarr

# Get Sonarr's new API key to compare/fix in the other apps
kubectl -n media exec <pod-sonarr> -- grep -i apikey /config/config.xml
```

---

### 4. The Definitive Fix (or Workaround)

1. Reset Sonarr's `/config` via a temporary pod (`apps/helpers/sonarr-config-wipe-pod.yaml`), with the Deployment scaled to 0 beforehand and back to 1 afterward.
2. Reverted the speculative `resources.limits` increase in `deployment.yaml` (back to `1000m`/`512Mi`), since Prometheus showed resources were never the bottleneck.
3. Updated Sonarr's new API key in each integrated app, one at a time, confirming improvement at each step:
   - **Bazarr** → Settings → Sonarr → API Key → Test → Save.
   - **Prowlarr** → Settings → Apps → (Sonarr entry) → API Key → Test → Save, followed by a manual **Sync App Indexers** to force recreating the indexers in Sonarr (instead of waiting for the automatic cycle).
   - **Seerr** → equivalent Sonarr integration settings → update the API key.

#### Final validation
```bash
kubectl -n media exec <pod-sonarr> -- grep -c "SignalR was challenged" /config/logs/sonarr.txt
```
The counter stopped growing (stayed flat at `2921`) after fixing all three apps — confirming the loop stopped for good. `kubectl top pod -n media` also returned to normal CPU values for Sonarr.

---

### 5. Lessons Learned for the Homelab

1. **Resetting one *arr app's config invalidates its API key in every app that integrates with it** — not just in the app itself. After a reset, the key must be manually updated in each downstream app (seen here: Bazarr, Prowlarr, Seerr; potentially also Lidarr/Readarr/Overseerr/Jellyseerr in other stacks) before considering the reset complete.
2. **A Pod that's "up, never restarts, but unstable" can mean the image's internal process supervisor (s6-overlay, common in linuxserver.io) is restarting just the app process** — Kubernetes' `RestartCount` doesn't capture this. Running `ps aux` inside the pod and comparing the app process's start time against the container's PID 1 exposes this invisible kind of restart.
3. **Before assuming CPU/memory are causing instability, confirm with the Prometheus already running in the cluster** (`container_cpu_usage_seconds_total`, `container_memory_working_set_bytes`, `kube_pod_container_status_last_terminated_reason`) instead of just bumping limits on a guess — avoids reserving unnecessary resources on a shared, already capacity-constrained node.
4. **An authentication loop that persists even in a private/incognito browser tab isn't a client session/cache issue** — rule out the browser quickly and point the investigation toward server-to-server integrations (other apps in the stack), which is exactly where the problem was here.
