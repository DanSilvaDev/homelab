# Observability (Logging + Monitoring) - Resumo de Configuração

Propósito: stack de métricas, dashboards e alertas para o cluster k3s, mais o storage dinâmico via NFS (TrueNAS) que a sustenta.

Ver também: [TrueNAS - config.md](../truenas/config.md) | [Worker2 - config.md](../worker2/config.md)

Status: Prometheus + Grafana + Alertmanager instalados e validados (métricas dos 3 nodes confirmadas). Roteamento de alertas (Telegram + ntfy) configurado e validado. Stack de logs (Loki + Alloy) instalada e validada (logs visíveis no Grafana Explore).

## Storage dinâmico (pré-requisito)

Dois provisioners NFS dedicados, apontando para datasets do TrueNAS (ver [truenas/config.md](../truenas/config.md) para detalhes de cada dataset):

| StorageClass | Provisioner (release Helm) | Backend TrueNAS | Uso |
|---|---|---|---|
| `truenas-nfs-ssd` | `nfs-subdir-provisioner-ssd` | `ssd-apps/k8s` (SSD, quota 40GiB) | Dados sensíveis a I/O (Prometheus, Grafana, Alertmanager) |
| `truenas-nfs-hdd` | `nfs-subdir-provisioner-hdd` | `data/k8s` (HD WD Blue 1TB, quota 100GiB) | Loki (dados maiores, menos sensíveis a latência) |

- Chart: `nfs-subdir-external-provisioner/nfs-subdir-external-provisioner`, versão `4.0.18`
- Namespace: `nfs-provisioner`
- Values: `infrastructure/storage/nfs-provisioner-ssd-values.yaml` e `infrastructure/storage/nfs-provisioner-hdd-values.yaml`
- `local-path` continua sendo a StorageClass default do cluster — essas duas são de uso explícito via `storageClassName`.
- Pré-requisito de rede: a VM worker2 precisou de correção de rede (bridge `br1` no TrueNAS) para conseguir alcançar o TrueNAS via NFS — ver [troubleshooting/worker2_vm_truenas_host_unreachable_macvtap.md](../../troubleshooting/worker2_vm_truenas_host_unreachable_macvtap.md).

## kube-prometheus-stack (Prometheus + Grafana + Alertmanager)

- Chart: `prometheus-community/kube-prometheus-stack`, versão `92.13.1`
- Release: `kube-prometheus-stack`
- Namespace: `monitoring`
- Values: `infrastructure/monitoring/values.yaml`

### Decisões de configuração
- **Retenção do Prometheus**: 15 dias
- **Monitores desabilitados**: `kubeControllerManager`, `kubeScheduler`, `kubeEtcd`, `kubeProxy` — o k3s embute esses componentes no próprio binário, expostos só em `127.0.0.1`; os ServiceMonitors padrão do chart não conseguem alcançá-los e ficariam permanentemente `DOWN`.
- **Node affinity**: nenhuma restrição de node para Prometheus/Alertmanager/Grafana. Inicialmente havia uma restrição excluindo o worker2 (disco local pequeno, ~18GB), removida depois que o storage passou a ser NFS (dado não fica mais preso a nenhum node).
- **PVCs** (todos via `truenas-nfs-ssd`):
  - Prometheus: 20Gi
  - Alertmanager: 2Gi
  - Grafana: 5Gi
- **Resources** (requests/limits) dimensionados para o hardware do cluster (control-plane e worker1: 4 vCPU/15GiB cada; worker2: 5 vCPU/10GiB, GPU — reduzido de 12GiB em 2026-10-26 após alertas de memory pressure no host TrueNAS, ver `clusters/worker2/config.md`) — ver `infrastructure/monitoring/values.yaml` para os valores exatos.
- **Grafana**: limite de memória ajustado de `256Mi` para `512Mi` (request de `128Mi` para `192Mi`) após o pod entrar em `CrashLoopBackOff` por `OOMKilled` em uso normal — ver [troubleshooting/grafana_oomkilled_memory_limit.md](../../troubleshooting/grafana_oomkilled_memory_limit.md).

### Acesso (LoadBalancer via kube-vip)

O cluster usa `kube-vip-cloud-provider` com range de IPs `192.168.3.200-254` para `Service` do tipo `LoadBalancer`. IPs fixos alocados:

| Serviço | IP | Porta | DNS local (Pi-hole) |
|---|---|---|---|
| Grafana | `192.168.3.201` | 80 | `dashboard.home` |
| ~~Rancher~~ | ~~`192.168.3.202`~~ | — | Rancher removido em 2026-10-28, IP liberado para reuso futuro — ver [troubleshooting/rancher_removal_traefik_crd_conflict.md](../../troubleshooting/rancher_removal_traefik_crd_conflict.md) |
| Prometheus | `192.168.3.203` | 80 (targetPort 9090) | `prometheus.home` |
| Alertmanager | `192.168.3.204` | 80 (targetPort 9093) | `alert.home` |
| ntfy | `192.168.3.205` | 80 | `ntfy.home` |

> **Atenção de segurança**: Prometheus, Alertmanager e ntfy **não têm autenticação própria**. Aceitável dentro da LAN confiável de casa, mas nunca expor publicamente (ex: Cloudflare Tunnel) sem autenticação na frente.

DNS local configurado no Pi-hole (Settings → Local DNS Records), resolvendo apenas para dispositivos que usam o Pi-hole como DNS (padrão da LAN).

### Credenciais
Usuário do Grafana: `admin`. Senha gerada automaticamente pelo chart (não commitada no repositório), recuperável com:
```bash
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

### Validação realizada
- Pods de `monitoring` e `nfs-provisioner` todos `Running`; PVCs `Bound` via `truenas-nfs-ssd`.
- Dashboard "Node Exporter / Nodes" no Grafana confirmado com dados dos 3 nodes (control-plane, worker1, worker2) após corrigir bloqueio de firewall no control-plane — ver [troubleshooting/prometheus_node_exporter_control_plane_ufw_block.md](../../troubleshooting/prometheus_node_exporter_control_plane_ufw_block.md).

## TrueNAS (host externo ao cluster)

O TrueNAS não é um node do k3s, então não é coberto automaticamente pelo `node-exporter` (DaemonSet) do `kube-prometheus-stack`. Métricas coletadas à parte:

- **Coleta**: `node_exporter` (`prom/node-exporter:v1.8.2`) rodando via **Docker direto no host** do TrueNAS (não pelo catálogo de Apps da UI — a Community Edition 25.10.7 restringe host path mounts customizados como `/proc`/`/sys` pela interface gráfica). Persistência garantida por `--restart unless-stopped` do próprio Docker.
  ```bash
  sudo docker run -d \
    --name node-exporter \
    --restart unless-stopped \
    --net=host \
    -v "/proc:/host/proc:ro" \
    -v "/sys:/host/sys:ro" \
    -v "/:/host/root:ro,rslave" \
    prom/node-exporter:v1.8.2 \
    --path.procfs=/host/proc \
    --path.sysfs=/host/sys \
    --path.rootfs=/host/root \
    --collector.filesystem.mount-points-exclude='^/(dev|proc|sys|var/lib/docker/.+)($|/)'
  ```
  Coletor de ZFS vem habilitado por padrão (sem flag extra) — expõe também saúde/uso dos pools.
- **Alvo externo no Prometheus**: manifesto `infrastructure/monitoring/truenas-external-target.yaml` (Service sem selector + Endpoints manual apontando para `192.168.3.103:9100` + ServiceMonitor com label `release: kube-prometheus-stack`), aplicado via `kubectl apply` (não faz parte do Helm release).
- **`jobLabel`**: o Service e o ServiceMonitor usam `jobLabel: node-exporter` (label) + `spec.jobLabel: jobLabel` (apontando para esse label), igual ao padrão do ServiceMonitor do node-exporter do cluster. Sem isso, o Prometheus Operator usaria o nome do Service (`truenas-node-exporter`) como label `job`, e o alvo ficaria fora do dashboard "Node Exporter / Nodes" do Grafana (filtrado por `job="node-exporter"`).
- **Validação**: target `serviceMonitor/monitoring/truenas-node-exporter/0` confirmado `UP` no Prometheus; com o `jobLabel` ajustado, o host aparece no seletor de node do dashboard "Node Exporter / Nodes" junto dos 3 nodes do cluster.

## Alertmanager - Roteamento de alertas

Um único receiver (`notifications`) com dois canais, mais um receiver `"null"` (silencioso) reservado pro alerta sintético `Watchdog` do chart (dead man's switch, não deve notificar ninguém). Configurado em `alertmanager.config` dentro de `infrastructure/monitoring/values.yaml`.

### Telegram (`telegram_configs`)
- Bot próprio criado via BotFather. Token e `chat_id` guardados como Secrets separados (criados manualmente via `kubectl create secret`, **não commitados**): `alertmanager-telegram-api-key` e `alertmanager-telegram-chat-id` (namespace `monitoring`), ambos com a key `token`.
- Montados no pod via `alertmanagerSpec.secrets` (lista de nomes de Secret) — o Prometheus Operator monta cada um em `/etc/alertmanager/secrets/<nome-do-secret>/<key>`. Referenciados na config via `bot_token_file`/`chat_id_file` (lidos a cada notificação, não só na inicialização — atualizar o Secret não exige reiniciar o pod nem reaplicar o Helm).
- `parse_mode: HTML` + campo `message` customizado (template Go) pra formatar a notificação (status, severidade, instância, resumo por alerta).
- Token original exposto acidentalmente durante um diagnóstico e rotacionado por segurança — ver [troubleshooting/kubectl_jsonpath_error_secret_exposure.md](../../troubleshooting/kubectl_jsonpath_error_secret_exposure.md).

### ntfy self-hosted (`webhook_configs`)
- App de push notification, deployado no próprio cluster via manifestos simples (`apps/ntfy/`: `deployment.yaml`, `service.yaml`, `pvc.yaml`), namespace `monitoring`, imagem `binwiederhier/ntfy:v2.34.0`.
- Storage: PVC 1Gi via `truenas-nfs-ssd` (cache de mensagens, retenção `NTFY_CACHE_DURATION=12h`).
- Exposto só na LAN por enquanto (mesmo padrão dos outros serviços: LoadBalancer kube-vip + DNS local no Pi-hole) — push só chega com o celular na rede de casa. Migração pra Tailscale (acesso de qualquer lugar) é possível depois sem mudar o deployment do ntfy em si.
- App ntfy no celular configurado com "Default server" apontando pro servidor próprio (`http://ntfy.home`), inscrito no tópico `homelab-alerts`.
- O Alertmanager alcança o ntfy via DNS interno do cluster (`http://ntfy.monitoring.svc.cluster.local/homelab-alerts`), não pelo IP do LoadBalancer — não depende do Pi-hole nem da LAN pra esse tráfego interno.
- **Limitação conhecida**: `webhook_configs` sempre manda o payload JSON fixo do Alertmanager como corpo da requisição (sem um template simples/suportado nesta versão, diferente do `telegram_configs`). O ntfy trata esse corpo cru como texto da notificação — funcional, mas não formatado. Possível melhoria futura: um pequeno serviço "tradutor" no meio do caminho.

### Gotcha de configuração (Helm + Prometheus Operator)
Ao sobrescrever `alertmanager.config.receivers`, é preciso reincluir manualmente o receiver `"null"` que o chart usa por padrão pro alerta `Watchdog` — o Helm substitui listas inteiras (não faz merge item a item), então uma lista `receivers` customizada apaga o `"null"` default, e a sub-rota (não sobrescrita) fica órfã, rejeitando a config inteira. Ver [troubleshooting/alertmanager_helm_receiver_list_replace.md](../../troubleshooting/alertmanager_helm_receiver_list_replace.md).

### Validação realizada
Alerta de teste disparado via `POST /api/v2/alerts` do Alertmanager chegou tanto no Telegram quanto no ntfy (push no celular).

## Alertas nativos do Grafana (export versionado)

Além do roteamento feito pelo Alertmanager (seção acima), também existem regras de alerta criadas diretamente na UI do Grafana (Alerting → Alert rules), usando o datasource Prometheus já configurado pelo chart. Essas regras são exportadas manualmente pela própria UI (Alerting → Alert rules → Export → formato de provisionamento) e versionadas em `apps/grafana/alerts/` como backup/histórico — a exportação **não é reaplicada automaticamente** no cluster (o Helm release não tem o sidecar de provisionamento de alertas do chart habilitado); recriar a regra exige reimportá-la pela própria UI, ou futuramente habilitar `grafana.sidecar.alerts` apontando pra essa pasta, caso essa automação seja adotada.

### `apps/grafana/alerts/alert_node_is_down.yaml`
- Regra: **Node unresponsive**, pasta `Cluster`.
- Condição: `up{job="node-exporter"} < 1` por 5 minutos (query instantânea reduzida contra o threshold).
- `noDataState: NoData` + `execErrState: Error` — ausência de dados e erro de execução da query também contam como alerta.
- Notificação: contact point `Alert Manager` (encaminha pro Alertmanager nativo do kube-prometheus-stack, que então roteia pro Telegram/ntfy conforme a seção acima).
- Intervalo de avaliação do grupo: 5m.

## Loki + Grafana Alloy (logs)

Agregação de logs de todos os pods do cluster, consultável no Grafana (Explore).

- **Loki**: chart `grafana-community/loki` (repo migrou de `grafana/helm-charts` em 2026-03-16), versão `18.7.6`, release `loki`, namespace `monitoring`. Values: `infrastructure/logging/loki-values.yaml`.
- **Alloy**: chart `grafana/alloy` (continua no repo clássico), release `alloy`, namespace `monitoring`. Values: `infrastructure/logging/alloy-values.yaml`. Roda como DaemonSet (todos os 3 nodes), descobre pods via API do Kubernetes (`loki.source.kubernetes`, sem precisar montar `/var/log/pods` do host) e envia pro Loki via DNS interno (`http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push`).

### Decisões de configuração
- **Modo**: `deploymentMode: Monolithic` (nome atual de "SingleBinary", chart >= 12.0.0) — um único pod, adequado ao volume de log de um homelab de 3 nodes.
- **Storage**: filesystem (sem S3/minio), PVC `10Gi` via `truenas-nfs-hdd`.
- **Retenção**: 7 dias, via `compactor.retention_enabled` (mecanismo atual; `tableManager` está deprecado).
- **Caches desabilitados** (`chunksCache`/`resultsCache`): o default do chart pede até 8GB de RAM pra memcached — inviável no homelab e desnecessário pro volume de log baixo esperado aqui.
- **Gateway (nginx) desabilitado**: só tráfego interno do cluster, sem necessidade de proxy/auth na frente — mesmo modelo de confiança já usado em Prometheus/Alertmanager/ntfy (sem autenticação própria, aceitável só dentro da LAN confiável).
- **Service do Loki**: `ClusterIP` (não `LoadBalancer`/kube-vip) — os únicos clientes (Alloy grava, Grafana lê) rodam dentro do cluster; ninguém de fora precisa alcançar o Loki diretamente, você sempre acessa via Grafana.
- **Resources do Alloy**: request `64Mi`/limit `256Mi` de memória. Valor inicial (`128Mi`/`512Mi`, com folga de propósito por falta de um baseline oficial de overhead por node na documentação do Alloy) validado via `kubectl top pod` — uso real ficou entre 47-88Mi nos 3 pods — e reduzido pela metade, mantendo ~3x de margem sobre o pico observado.
- **Datasource no Grafana**: adicionado via `grafana.additionalDataSources` em `infrastructure/monitoring/values.yaml` (não um ConfigMap separado), apontando pro Service `loki` na porta `3100`.

### Validação realizada
- `kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy` — 3 pods `Running` (um por node).
- `kubectl top pod` confirmou uso de memória bem abaixo do limit configurado.
- Grafana Explore → datasource Loki → query `{namespace="monitoring"}` retornou logs dos pods do namespace.

## dcgm-exporter (métricas de GPU do worker2)

Métricas da GPU (GTX 970, Maxwell) via NVIDIA DCGM, scrapeadas pelo Prometheus.

- Chart: `gpu-helm-charts/dcgm-exporter` (`https://nvidia.github.io/dcgm-exporter/helm-charts`), versão `4.8.4`, release `dcgm-exporter`, namespace `nvidia-device-plugin` (mesmo namespace do `nvidia-device-plugin`, pra agrupar infra de GPU). Values: `infrastructure/gpu-monitoring/dcgm-exporter-values.yaml`.
- **Sem o gotcha de `nodeAffinity` oculta** que pegou o `nvdp/nvidia-device-plugin` (ver [troubleshooting/nvidia_device_plugin_nodeaffinity_requirement.md](../../troubleshooting/nvidia_device_plugin_nodeaffinity_requirement.md)) — confirmado lendo o template do chart; `nodeSelector: {gpu: "true"}` sozinho já restringe ao worker2.
- `resources.limits["nvidia.com/gpu"]: "1"` adicionado manualmente (replica o padrão já validado no pod de teste CUDA), além dos defaults de cpu/memory do chart (já adequados ao homelab).
- `serviceMonitor.enabled: true` + `additionalLabels: {release: kube-prometheus-stack}` — sem essa label o Prometheus Operator ignora o ServiceMonitor (mesmo motivo já documentado na seção do TrueNAS).
- **Limitação da GPU (Maxwell, compute capability 5.2)**: o módulo de *profiling* do DCGM (métricas `DCGM_FI_PROF_*`, incluindo o painel "Tensor Core Utilization" do dashboard) exige Volta+ e não funciona nessa GPU. Métricas básicas (utilização, memória, temperatura, potência, clocks, ECC) funcionam normalmente — validado via `/metrics` do exporter.

### Dashboard no Grafana
O dashboard oficial (grafana.com ID 12239) **não pôde ser importado pelo mecanismo automático** do chart (`grafana.dashboards.<provider>.<nome>.gnetId`) — falhava silenciosamente nessa versão do Grafana, sem nenhum log de erro. Aplicado como ConfigMap manual (`infrastructure/gpu-monitoring/dcgm-exporter-dashboard-configmap.yaml`, label `grafana_dashboard: "1"`, mesmo mecanismo de sidecar já usado pelos dashboards padrão do chart), aplicado direto via `kubectl apply` (fora do Helm release). Detalhes completos do diagnóstico: [troubleshooting/grafana_dashboard_gnetid_silent_failure.md](../../troubleshooting/grafana_dashboard_gnetid_silent_failure.md).

### Validação realizada
- `kubectl logs ... -c download-dashboards` e `/metrics` do exporter confirmaram GPU real sendo lida (temperatura, potência, framebuffer).
- Dashboard "NVIDIA DCGM Exporter Dashboard" confirmado visível e funcional no Grafana (painéis de profiling sem dado, como esperado).

## Próximos passos
- [x] Storage dinâmico via NFS (TrueNAS) — SSD e HDD
- [x] Instalar kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
- [x] Validar métricas dos 3 nodes no Grafana
- [x] Expor Grafana/Prometheus/Alertmanager via LoadBalancer (kube-vip) + DNS local (Pi-hole)
- [x] Coletar métricas do host TrueNAS (fora do cluster) via node_exporter + ServiceMonitor manual
- [x] Configurar Alertmanager → Telegram (receiver nativo `telegram_configs`)
- [x] Configurar Alertmanager → ntfy self-hosted (push no celular, `webhook_configs`)
- [x] Instalar Loki + Grafana Alloy (logs), usando `truenas-nfs-hdd` como storage
- [x] `dcgm-exporter` da NVIDIA para métricas de GPU do worker2
- [x] Primeira regra de alerta nativa do Grafana versionada (`apps/grafana/alerts/alert_node_is_down.yaml`)
- [ ] Configurar `grafana.sidecar.alerts` para provisionar `apps/grafana/alerts/` automaticamente (hoje é só export manual via UI)
- [x] Remover o Rancher do cluster (conflito de CRD com Traefik) e desabilitar o Traefik — ver [troubleshooting/rancher_removal_traefik_crd_conflict.md](../../troubleshooting/rancher_removal_traefik_crd_conflict.md)

## Log de comandos executados

```bash
# Storage dinâmico (NFS via TrueNAS)
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm upgrade --install nfs-subdir-provisioner-ssd nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner --create-namespace \
  -f infrastructure/storage/nfs-provisioner-ssd-values.yaml \
  --version 4.0.18

helm upgrade --install nfs-subdir-provisioner-hdd nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner \
  -f infrastructure/storage/nfs-provisioner-hdd-values.yaml \
  --version 4.0.18

# kube-prometheus-stack
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1

# Validação
kubectl get pods -n monitoring
kubectl get pvc -n monitoring
kubectl get svc -n monitoring kube-prometheus-stack-grafana kube-prometheus-stack-prometheus kube-prometheus-stack-alertmanager

kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d

# Fix: firewall do control-plane bloqueando scrape do node-exporter (ver troubleshooting doc)
sudo ufw allow from 192.168.3.0/24 to any port 9100 proto tcp comment 'node-exporter'
sudo ufw reload

# Métricas do host TrueNAS (fora do cluster)
# No TrueNAS, via SSH:
sudo docker run -d \
  --name node-exporter \
  --restart unless-stopped \
  --net=host \
  -v "/proc:/host/proc:ro" \
  -v "/sys:/host/sys:ro" \
  -v "/:/host/root:ro,rslave" \
  prom/node-exporter:v1.8.2 \
  --path.procfs=/host/proc \
  --path.sysfs=/host/sys \
  --path.rootfs=/host/root \
  --collector.filesystem.mount-points-exclude='^/(dev|proc|sys|var/lib/docker/.+)($|/)'

# No cluster: registra o alvo externo no Prometheus
kubectl apply -f infrastructure/monitoring/truenas-external-target.yaml

# Fix: jobLabel do alvo do TrueNAS ajustado para cair na mesma série "job=node-exporter"
# (ver nova versão de infrastructure/monitoring/truenas-external-target.yaml)
kubectl apply -f infrastructure/monitoring/truenas-external-target.yaml

# Fix: Grafana em CrashLoopBackOff por OOMKilled (ver troubleshooting doc)
# Ajuste de resources.limits.memory em infrastructure/monitoring/values.yaml (256Mi -> 512Mi)
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1

# Alertmanager -> Telegram: Secrets criados manualmente (nao commitados)
kubectl create secret generic alertmanager-telegram-api-key \
  --namespace monitoring --from-literal=token='<token do bot via BotFather>'
kubectl create secret generic alertmanager-telegram-chat-id \
  --namespace monitoring --from-literal=token='<chat_id>'

# ntfy self-hosted (apps/ntfy/)
kubectl apply -f apps/ntfy/pvc.yaml -f apps/ntfy/deployment.yaml -f apps/ntfy/service.yaml

# Alertmanager -> Telegram + ntfy: alertmanager.config em infrastructure/monitoring/values.yaml
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1

# Validacao: dispara alerta de teste direto na API do Alertmanager
curl -i -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TesteHomelab","severity":"info"}}]' \
  http://192.168.3.204/api/v2/alerts

# Loki + Grafana Alloy (logs)
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install loki grafana-community/loki \
  --namespace monitoring \
  -f infrastructure/logging/loki-values.yaml \
  --version 18.7.6

# Confirma nome/porta do Service do Loki antes de apontar o Alloy/datasource pra ele
kubectl get svc -n monitoring | grep loki

helm upgrade --install alloy grafana/alloy \
  --namespace monitoring \
  -f infrastructure/logging/alloy-values.yaml

# Reaplica o kube-prometheus-stack com o datasource Loki no Grafana
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1

# Validacao
kubectl get pods -n monitoring -l app.kubernetes.io/name=alloy
kubectl top pod -n monitoring -l app.kubernetes.io/name=alloy

# dcgm-exporter (metricas de GPU)
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update

helm upgrade --install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace nvidia-device-plugin \
  -f infrastructure/gpu-monitoring/dcgm-exporter-values.yaml \
  --version 4.8.4

# Validacao: GPU real sendo lida (ver troubleshooting doc p/ diagnostico do dashboard)
kubectl get pods -n nvidia-device-plugin -l app.kubernetes.io/name=dcgm-exporter -o wide
kubectl run curl-test --rm -it --image=curlimages/curl --restart=Never -n nvidia-device-plugin -- \
  curl -s http://dcgm-exporter.nvidia-device-plugin.svc.cluster.local:9400/metrics

# Fix: dashboard via gnetId falhava silenciosamente (ver troubleshooting doc) - aplicado
# como ConfigMap manual em vez da chave grafana.dashboards.default.dcgm-exporter.gnetId
kubectl apply -f infrastructure/gpu-monitoring/dcgm-exporter-dashboard-configmap.yaml
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard --tail=10
```
