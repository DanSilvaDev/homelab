# Troubleshooting: Dashboard importado via `gnetId` no values.yaml do Grafana nunca aparecia (sem nenhum erro) / Dashboard imported via `gnetId` in Grafana's values.yaml never showed up (with no error at all)

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Documenta por que o dashboard oficial do `dcgm-exporter` (grafana.com ID 12239), configurado via `grafana.dashboards.<provider>.<nome>.gnetId` no `values.yaml` do kube-prometheus-stack, nunca apareceu no Grafana — mesmo com o Helm aplicando o valor corretamente e nenhum log de erro em nenhuma camada da stack.

---

### 1. O Cenário Inicial e Sintomas

Depois de instalar o `dcgm-exporter` (métricas de GPU do worker2) e confirmar as métricas chegando no Prometheus, faltava importar o dashboard oficial da NVIDIA no Grafana. Usei o mecanismo nativo do chart `grafana/grafana` (embutido no kube-prometheus-stack) pra isso:

```yaml
grafana:
  dashboards:
    default:
      dcgm-exporter:
        gnetId: 12239
        revision: 2
        datasource: Prometheus
```

Depois de `helm upgrade`, o pod do Grafana reiniciou normalmente (`3/3 Running`, 0 restarts), mas o dashboard não aparecia em nenhum lugar.

#### Sintomas do Erro:
* Dashboard "NVIDIA DCGM Exporter Dashboard" não aparecia na lista/busca do Grafana (UI nem API).
* `curl .../api/search?query=NVIDIA` retornava vazio, enquanto buscar por outros dashboards conhecidos funcionava normalmente (descartando problema de autenticação/API).
* `curl .../api/search?query=*` (todos os dashboards) retornava exatamente os mesmos **24 dashboards padrão** de antes — nenhum a mais.
* **Nenhum erro em lugar nenhum**: logs do container principal do Grafana mostravam só `level=info msg="starting to provision dashboards"` / `"finished to provision dashboards"`, sem menção a falha. Logs do initContainer responsável pelo download (`download-dashboards`, imagem `curlimages/curl`) vinham **completamente vazios** — nem sucesso, nem erro.
* `helm get values` confirmava que o Helm tinha registrado nosso `dashboards.default.dcgm-exporter` corretamente — não era um erro de digitação no `values.yaml`.

---

### 2. Processo de Investigação & Descoberta

Primeiro suspeitei de um problema de merge do Helm (como já tinha acontecido antes com o Alertmanager — ver [troubleshooting/alertmanager_helm_receiver_list_replace.md](alertmanager_helm_receiver_list_replace.md)), mas `helm get values` descartou essa hipótese: o valor estava lá, certinho.

Em seguida, `helm get manifest` mostrou algo revelador: o ConfigMap que o chart gera a partir da chave `dashboards:` (`kube-prometheus-stack-grafana-dashboards-default`) tinha `data: {}` — **vazio**. E, no mesmo manifesto, apareceu um trecho de shell script (parte de um initContainer) fazendo `curl` direto pra `https://grafana.com/api/dashboards/12239/revisions/2/download`, com um `sed` pra substituir o campo `datasource` pelo nome literal do datasource, salvando o resultado em `/var/lib/grafana/dashboards/default/dcgm-exporter.json`.

Ou seja: **dashboards definidos via `gnetId` não passam pelo ConfigMap + sidecar** (mecanismo que já sabíamos funcionar, confirmado pelos 24 dashboards padrão) — eles usam um **initContainer separado** (`download-dashboards`) que baixa o JSON direto da internet no startup do pod e escreve num diretório local do container, sem gerar nenhum recurso Kubernetes intermediário.

Tentei inspecionar esse diretório diretamente dentro do pod pra confirmar se o arquivo foi baixado, mas a imagem do Grafana usada aqui não tem `ls` nem `cat` (imagem bem enxuta, sem coreutils) — `kubectl exec` falhava com `executable file not found in $PATH` pros dois.

Voltando aos logs do container principal do Grafana, filtrados por "dashboard", apareceu uma pista que parecia positiva: a construção do índice de busca interno (`bleve-backend`) mostrava `doc_count=25` — um a mais que os 24 de antes. Essa contagem acabou sendo **enganosa**: a busca real via API (`/api/search?query=*`) confirmou que só existiam 24 dashboards de fato pesquisáveis; o `doc_count=25` provavelmente contava algum outro objeto interno do novo backend de "unified storage" do Grafana, não o dashboard novo.

#### Causa Raiz: dashboard antigo (schema de 2020) + mecanismo de download sem log de erro por item
O dashboard 12239 revisão 2 é de 2020 (`schemaVersion: 22`, `"datasource": "${DS_PROMETHEUS}"` como string literal — formato pensado pro fluxo antigo de "importar com inputs" da UI do Grafana, que não se aplica quando o JSON é jogado direto numa pasta de provisionamento). O cluster roda uma versão bem recente do Grafana (confirmado pelos logs mostrando o novo backend `dashboard.grafana.app`/"unified storage"). O `curl`+`sed` do initContainer provavelmente baixou e "consertou" o campo `datasource`, mas o Grafana recusou ou ignorou silenciosamente esse dashboard específico ao processá-lo — sem nenhum log de erro individual, só o genérico `"finished to provision dashboards"` (que só confirma que o *processo* terminou, não que *cada* dashboard foi carregado com sucesso).

**Ponto-chave**: logs de nível `info` como "finished to provision dashboards" não são prova de sucesso item a item — eles confirmam que o loop/processo rodou até o fim, mesmo que um item específico tenha sido descartado no meio do caminho sem gerar um log de nível `warn`/`error` visível.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirma que o Helm tem o valor customizado registrado (descarta erro de digitação/merge)
helm get values kube-prometheus-stack -n monitoring -a | grep -A6 "dashboards:"

# Procura o ConfigMap do dashboard por qualquer nome (não só o label padrão)
kubectl get configmap -n monitoring | grep -i dcgm

# Mostra o que o Helm efetivamente renderizou para esse dashboard - revelou o
# initContainer de download via curl, mecanismo diferente do ConfigMap+sidecar
helm get manifest kube-prometheus-stack -n monitoring | grep -B5 -A30 "dcgm-exporter"

# Confirma quais initContainers o pod do Grafana tem e se já rodaram
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].metadata.name}{"\n"}{.items[0].status.initContainerStatuses[*].name}{"\n"}'

# Log do initContainer de download (veio vazio - nem sucesso nem erro)
kubectl logs -n monitoring <pod-grafana> -c download-dashboards

# Exit code/estado do initContainer
kubectl describe pod -n monitoring <pod-grafana> | grep -A15 "^  download-dashboards:"

# Tentativa de inspecionar o filesystem do pod diretamente - falhou (sem ls/cat na imagem)
kubectl exec -n monitoring <pod-grafana> -c grafana -- ls -la /var/lib/grafana/dashboards/default/
kubectl exec -n monitoring <pod-grafana> -c grafana -- cat /var/lib/grafana/dashboards/default/dcgm-exporter.json

# Log do container principal filtrado por "dashboard" - só mensagens genéricas de info
kubectl logs -n monitoring <pod-grafana> -c grafana | grep -i dashboard

# Confirmação definitiva via API de busca do próprio Grafana
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
curl -s -u admin:'<senha>' "http://dashboard.home/api/search?query=NVIDIA"          # vazio
curl -s -u admin:'<senha>' "http://dashboard.home/api/search?query=*" \
  | grep -o '"title":"[^"]*"'                                                        # só os 24 padrão
```

---

### 4. A Correção Definitiva

Abandonado o mecanismo `gnetId` do chart. Em vez disso, o dashboard foi baixado manualmente, corrigido, e aplicado como um **ConfigMap próprio** usando o mesmo label (`grafana_dashboard: "1"`) que o sidecar do chart já usa pros 24 dashboards padrão — mecanismo comprovadamente funcional nesse cluster (confirmado nos logs do container `grafana-sc-dashboard`).

```bash
# Baixa o JSON do dashboard diretamente da API do grafana.com
curl -fsSL "https://grafana.com/api/dashboards/12239/revisions/2/download" -o dcgm_dashboard.json

# Substitui o placeholder de datasource (formato antigo) pelo nome literal do datasource
# já configurado no Grafana - mesma lógica que o initContainer quebrado tentava fazer
sed '/-- .* --/! s/"datasource":[[:space:]]*"\${DS_PROMETHEUS}"/"datasource": "Prometheus"/g' \
  dcgm_dashboard.json > dcgm_dashboard_fixed.json

# JSON embutido como ConfigMap (infrastructure/gpu-monitoring/dcgm-exporter-dashboard-configmap.yaml),
# labels: { grafana_dashboard: "1" }, aplicado direto (fora do Helm release, mesmo padrão já usado em
# infrastructure/monitoring/truenas-external-target.yaml e apps/ntfy/)
kubectl apply -f infrastructure/gpu-monitoring/dcgm-exporter-dashboard-configmap.yaml
```

A chave `grafana.dashboards.default.dcgm-exporter.gnetId` foi removida do `values.yaml` do kube-prometheus-stack — não fazia nada além de acionar o initContainer quebrado.

#### Validação final
```bash
# Confirma que o sidecar (mesmo mecanismo dos dashboards padrão) pegou o ConfigMap novo
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard --tail=10
# -> "Writing /tmp/dashboards/dcgm-exporter.json"

# Confirma na API de busca: 25 dashboards agora, incluindo o novo
curl -s -u admin:'<senha>' "http://dashboard.home/api/search?query=*" | grep -o '"title":"[^"]*"'
```
Dashboard "NVIDIA DCGM Exporter Dashboard" confirmado visível e funcional na UI do Grafana.

---

### 5. Lições Aprendidas para o Homelab

1. **Log de nível `info` tipo "finished to provision dashboards" não garante sucesso item a item** — só confirma que o processo/loop terminou. Pra confirmar que um dashboard específico carregou, a fonte de verdade é a API de busca do próprio Grafana (`/api/search`), não os logs de provisionamento.
2. **O mecanismo `dashboards.<provider>.<nome>.gnetId` do chart `grafana/grafana` usa um caminho totalmente diferente do ConfigMap+sidecar** (initContainer com `curl`, direto no filesystem do pod) — mesmo que a chave pareça irmã de um JSON inline, o comportamento de debug é diferente (sem ConfigMap pra inspecionar, sem log do sidecar). Preferir o caminho manual (ConfigMap com `grafana_dashboard: "1"`) por ser mais previsível e observável.
3. **Dashboards antigos baixados por ID do grafana.com podem ter schema desatualizado** (`"datasource": "${DS_PROMETHEUS}"` como string, pensado pro fluxo de import manual da UI) e falhar silenciosamente em versões recentes do Grafana — vale checar a data/versão do dashboard antes de confiar no import automático, e sempre validar o resultado via busca, não só "sem erro no log".
4. **Imagens de container minimalistas (sem `ls`/`cat`/shell) dificultam debug via `kubectl exec`** — quando isso acontece, usar os logs e a API HTTP da própria aplicação (se houver) como fonte de verdade, em vez de insistir em inspecionar o filesystem do pod diretamente.

---

## English

Documents why the official `dcgm-exporter` dashboard (grafana.com ID 12239), configured via `grafana.dashboards.<provider>.<name>.gnetId` in the kube-prometheus-stack `values.yaml`, never showed up in Grafana — even though Helm applied the value correctly and there was no error log at any layer of the stack.

---

### 1. Initial Scenario and Symptoms

After installing `dcgm-exporter` (GPU metrics from worker2) and confirming the metrics were reaching Prometheus, the last step was importing NVIDIA's official dashboard into Grafana. I used the native mechanism of the `grafana/grafana` chart (bundled inside kube-prometheus-stack) for this:

```yaml
grafana:
  dashboards:
    default:
      dcgm-exporter:
        gnetId: 12239
        revision: 2
        datasource: Prometheus
```

After `helm upgrade`, the Grafana pod restarted normally (`3/3 Running`, 0 restarts), but the dashboard didn't show up anywhere.

#### Error Symptoms:
* The "NVIDIA DCGM Exporter Dashboard" didn't appear in Grafana's dashboard list/search (neither UI nor API).
* `curl .../api/search?query=NVIDIA` returned empty, while searching for other known dashboards worked normally (ruling out an auth/API problem).
* `curl .../api/search?query=*` (all dashboards) returned exactly the same **24 default dashboards** as before — none extra.
* **No error anywhere**: the main Grafana container logs only showed `level=info msg="starting to provision dashboards"` / `"finished to provision dashboards"`, with no mention of failure. Logs from the initContainer responsible for the download (`download-dashboards`, image `curlimages/curl`) were **completely empty** — no success, no error.
* `helm get values` confirmed Helm had correctly registered our `dashboards.default.dcgm-exporter` entry — it wasn't a typo in `values.yaml`.

---

### 2. Investigation & Discovery Process

I first suspected a Helm merge problem (as had already happened earlier with Alertmanager — see [troubleshooting/alertmanager_helm_receiver_list_replace.md](alertmanager_helm_receiver_list_replace.md)), but `helm get values` ruled that out: the value was there, correctly set.

Next, `helm get manifest` showed something revealing: the ConfigMap the chart generates from the `dashboards:` key (`kube-prometheus-stack-grafana-dashboards-default`) had `data: {}` — **empty**. And, in the same manifest, a shell script fragment (part of an initContainer) showed up doing a `curl` straight to `https://grafana.com/api/dashboards/12239/revisions/2/download`, with a `sed` to replace the `datasource` field with the literal datasource name, saving the result to `/var/lib/grafana/dashboards/default/dcgm-exporter.json`.

In other words: **dashboards defined via `gnetId` don't go through the ConfigMap + sidecar path** (the mechanism we already knew worked, confirmed by the 24 default dashboards) — they use a **separate initContainer** (`download-dashboards`) that downloads the JSON straight from the internet at pod startup and writes it to a local directory inside the container, without creating any intermediate Kubernetes resource.

I tried inspecting that directory directly inside the pod to confirm whether the file had been downloaded, but the Grafana image used here has neither `ls` nor `cat` (a very slim image, no coreutils) — `kubectl exec` failed with `executable file not found in $PATH` for both.

Going back to the main Grafana container's logs, filtered by "dashboard", a seemingly positive clue showed up: the internal search index build (`bleve-backend`) reported `doc_count=25` — one more than the previous 24. That count turned out to be **misleading**: the actual search via the API (`/api/search?query=*`) confirmed only 24 dashboards were truly searchable; `doc_count=25` was probably counting some other internal object from Grafana's new "unified storage" backend, not the new dashboard.

#### Root Cause: old dashboard schema (from 2020) + a download mechanism with no per-item error log
Dashboard 12239 revision 2 is from 2020 (`schemaVersion: 22`, `"datasource": "${DS_PROMETHEUS}"` as a literal string — a format meant for Grafana's old "import with inputs" UI flow, which doesn't apply when the JSON is dropped straight into a provisioning folder). The cluster runs a fairly recent Grafana version (confirmed by logs showing the new `dashboard.grafana.app`/"unified storage" backend). The initContainer's `curl`+`sed` likely downloaded and "fixed" the `datasource` field, but Grafana silently rejected or ignored that specific dashboard while processing it — with no individual error log, only the generic `"finished to provision dashboards"` (which only confirms the *process* finished, not that *each* dashboard loaded successfully).

**Key point**: `info`-level logs like "finished to provision dashboards" aren't proof of per-item success — they confirm the loop/process ran to completion, even if one specific item was silently dropped along the way without producing a visible `warn`/`error`-level log.

---

### 3. Commands Used for Diagnosis

```bash
# Confirms Helm has the custom value registered (rules out a typo/merge error)
helm get values kube-prometheus-stack -n monitoring -a | grep -A6 "dashboards:"

# Looks for the dashboard ConfigMap under any name (not just the default label)
kubectl get configmap -n monitoring | grep -i dcgm

# Shows what Helm actually rendered for this dashboard - revealed the curl-based
# initContainer, a mechanism different from the ConfigMap+sidecar one
helm get manifest kube-prometheus-stack -n monitoring | grep -B5 -A30 "dcgm-exporter"

# Confirms which initContainers the Grafana pod has and whether they already ran
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].metadata.name}{"\n"}{.items[0].status.initContainerStatuses[*].name}{"\n"}'

# Download initContainer log (came back empty - neither success nor error)
kubectl logs -n monitoring <grafana-pod> -c download-dashboards

# InitContainer exit code/state
kubectl describe pod -n monitoring <grafana-pod> | grep -A15 "^  download-dashboards:"

# Attempt to inspect the pod filesystem directly - failed (no ls/cat in the image)
kubectl exec -n monitoring <grafana-pod> -c grafana -- ls -la /var/lib/grafana/dashboards/default/
kubectl exec -n monitoring <grafana-pod> -c grafana -- cat /var/lib/grafana/dashboards/default/dcgm-exporter.json

# Main container log filtered by "dashboard" - only generic info-level messages
kubectl logs -n monitoring <grafana-pod> -c grafana | grep -i dashboard

# Definitive confirmation via Grafana's own search API
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
curl -s -u admin:'<password>' "http://dashboard.home/api/search?query=NVIDIA"       # empty
curl -s -u admin:'<password>' "http://dashboard.home/api/search?query=*" \
  | grep -o '"title":"[^"]*"'                                                        # only the 24 default ones
```

---

### 4. The Definitive Fix

Abandoned the chart's `gnetId` mechanism. Instead, the dashboard was downloaded manually, fixed, and applied as a **plain ConfigMap** using the same label (`grafana_dashboard: "1"`) the chart's sidecar already uses for the 24 default dashboards — a mechanism proven to work in this cluster (confirmed in the `grafana-sc-dashboard` container logs).

```bash
# Downloads the dashboard JSON directly from the grafana.com API
curl -fsSL "https://grafana.com/api/dashboards/12239/revisions/2/download" -o dcgm_dashboard.json

# Replaces the (old-format) datasource placeholder with the literal datasource name
# already configured in Grafana - the same thing the broken initContainer was trying to do
sed '/-- .* --/! s/"datasource":[[:space:]]*"\${DS_PROMETHEUS}"/"datasource": "Prometheus"/g' \
  dcgm_dashboard.json > dcgm_dashboard_fixed.json

# JSON embedded as a ConfigMap (infrastructure/gpu-monitoring/dcgm-exporter-dashboard-configmap.yaml),
# labels: { grafana_dashboard: "1" }, applied directly (outside the Helm release, same pattern
# already used for infrastructure/monitoring/truenas-external-target.yaml and apps/ntfy/)
kubectl apply -f infrastructure/gpu-monitoring/dcgm-exporter-dashboard-configmap.yaml
```

The `grafana.dashboards.default.dcgm-exporter.gnetId` key was removed from the kube-prometheus-stack `values.yaml` — it wasn't doing anything other than triggering the broken initContainer.

#### Final validation
```bash
# Confirms the sidecar (same mechanism as the default dashboards) picked up the new ConfigMap
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard --tail=10
# -> "Writing /tmp/dashboards/dcgm-exporter.json"

# Confirms via the search API: 25 dashboards now, including the new one
curl -s -u admin:'<password>' "http://dashboard.home/api/search?query=*" | grep -o '"title":"[^"]*"'
```
The "NVIDIA DCGM Exporter Dashboard" was confirmed visible and functional in the Grafana UI.

---

### 5. Lessons Learned for the Homelab

1. **An `info`-level log like "finished to provision dashboards" does not guarantee per-item success** — it only confirms the process/loop finished. To confirm a specific dashboard actually loaded, the source of truth is Grafana's own search API (`/api/search`), not the provisioning logs.
2. **The `dashboards.<provider>.<name>.gnetId` mechanism of the `grafana/grafana` chart uses a completely different path than ConfigMap+sidecar** (a `curl`-based initContainer, writing straight to the pod's filesystem) — even though the key looks like a sibling of an inline JSON entry, the debugging experience is different (no ConfigMap to inspect, no sidecar log). Prefer the manual path (a ConfigMap labeled `grafana_dashboard: "1"`) since it's more predictable and observable.
3. **Dashboards downloaded by ID from grafana.com can have an outdated schema** (`"datasource": "${DS_PROMETHEUS}"` as a string, meant for the UI's manual-import flow) and fail silently on recent Grafana versions — worth checking the dashboard's date/version before trusting the automatic import, and always validating the result via search, not just "no error in the log".
4. **Minimal container images (no `ls`/`cat`/shell) make debugging via `kubectl exec` harder** — when that happens, use the application's own logs and HTTP API (if it has one) as the source of truth, instead of insisting on inspecting the pod's filesystem directly.
