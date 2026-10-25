# Observability (Logging + Monitoring) - Resumo de Configuração

Propósito: stack de métricas, dashboards e alertas para o cluster k3s, mais o storage dinâmico via NFS (TrueNAS) que a sustenta.

Ver também: [TrueNAS - config.md](../truenas/config.md) | [Worker2 - config.md](../worker2/config.md)

Status: Prometheus + Grafana + Alertmanager instalados e validados (métricas dos 3 nodes confirmadas). Pendente: roteamento de alertas (Discord/Telegram) e stack de logs (Loki + Alloy).

## Storage dinâmico (pré-requisito)

Dois provisioners NFS dedicados, apontando para datasets do TrueNAS (ver [truenas/config.md](../truenas/config.md) para detalhes de cada dataset):

| StorageClass | Provisioner (release Helm) | Backend TrueNAS | Uso |
|---|---|---|---|
| `truenas-nfs-ssd` | `nfs-subdir-provisioner-ssd` | `ssd-apps/k8s` (SSD, quota 40GiB) | Dados sensíveis a I/O (Prometheus, Grafana, Alertmanager) |
| `truenas-nfs-hdd` | `nfs-subdir-provisioner-hdd` | `data/k8s` (HD WD Blue 1TB, quota 100GiB) | Reservado para uso futuro (ex: Loki), dados maiores/menos sensíveis a latência |

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
- **Resources** (requests/limits) dimensionados para o hardware do cluster (control-plane e worker1: 4 vCPU/15GiB cada; worker2: 5 vCPU/12GiB, GPU) — ver `infrastructure/monitoring/values.yaml` para os valores exatos.

### Acesso (LoadBalancer via kube-vip)

O cluster usa `kube-vip-cloud-provider` com range de IPs `192.168.3.200-254` para `Service` do tipo `LoadBalancer`. IPs fixos alocados:

| Serviço | IP | Porta | DNS local (Pi-hole) |
|---|---|---|---|
| Grafana | `192.168.3.201` | 80 | `dashboard.home` |
| Rancher | `192.168.3.202` | — | `rancher.home` |
| Prometheus | `192.168.3.203` | 80 (targetPort 9090) | `prometheus.home` |
| Alertmanager | `192.168.3.204` | 80 (targetPort 9093) | `alert.home` |

> **Atenção de segurança**: Prometheus e Alertmanager **não têm autenticação própria** (diferente do Grafana, que tem login). Aceitável dentro da LAN confiável de casa, mas nunca expor publicamente (ex: Cloudflare Tunnel) sem autenticação na frente.

DNS local configurado no Pi-hole (Settings → Local DNS Records), resolvendo apenas para dispositivos que usam o Pi-hole como DNS (padrão da LAN).

### Credenciais
Usuário do Grafana: `admin`. Senha gerada automaticamente pelo chart (não commitada no repositório), recuperável com:
```bash
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
```

### Validação realizada
- Pods de `monitoring` e `nfs-provisioner` todos `Running`; PVCs `Bound` via `truenas-nfs-ssd`.
- Dashboard "Node Exporter / Nodes" no Grafana confirmado com dados dos 3 nodes (control-plane, worker1, worker2) após corrigir bloqueio de firewall no control-plane — ver [troubleshooting/prometheus_node_exporter_control_plane_ufw_block.md](../../troubleshooting/prometheus_node_exporter_control_plane_ufw_block.md).

## Próximos passos
- [x] Storage dinâmico via NFS (TrueNAS) — SSD e HDD
- [x] Instalar kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
- [x] Validar métricas dos 3 nodes no Grafana
- [x] Expor Grafana/Prometheus/Alertmanager via LoadBalancer (kube-vip) + DNS local (Pi-hole)
- [ ] Configurar Alertmanager → Discord (receiver `slack_configs` via webhook `/slack`)
- [ ] Configurar Alertmanager → Telegram (receiver nativo `telegram_configs`)
- [ ] Instalar Loki + Grafana Alloy (logs), usando `truenas-nfs-hdd` como storage
- [ ] (Opcional) `dcgm-exporter` da NVIDIA para métricas de GPU do worker2

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
```
