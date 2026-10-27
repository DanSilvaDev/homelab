# Troubleshooting: Grafana reiniciando repetidamente (OOMKilled) por limite de memória subdimensionado

Documenta por que o pod do Grafana (instalado via `kube-prometheus-stack`) ficava reiniciando com frequência, entrando em `CrashLoopBackOff`, logo após a instalação do stack de monitoring.

---

## 1. O Cenário Inicial e Sintomas

Após a instalação do `kube-prometheus-stack` e validação inicial do Grafana, o pod passou a reiniciar com frequência ao longo do uso normal (navegação entre dashboards, carregamento de plugins).

### Sintomas do Erro:
* `kubectl get events -n monitoring` mostrava repetidos `Unhealthy` (liveness e readiness probe falhando) e `BackOff` para o pod do Grafana:
  ```
  Warning   Unhealthy   Liveness probe failed: Get "http://10.42.1.30:3000/api/health": read tcp 10.42.1.1:40500->10.42.1.30:3000: read: connection reset by peer
  Warning   Unhealthy   Readiness probe failed: Get "http://10.42.1.30:3000/api/health": dial tcp 10.42.1.30:3000: connect: connection refused
  Warning   BackOff     Back-off restarting failed container grafana in pod ...
  ```
* `restartCount` do container `grafana` em `15` em pouco tempo de uso.
* `kubectl top pod` mostrava uso de memória (~155Mi) já próximo do limite configurado (256Mi) mesmo em regime estável, antes de qualquer pico.

---

## 2. Processo de Investigação & Descoberta

Os sintomas (probe falhando por "connection reset"/"connection refused", não por timeout de rede) e o padrão de `BackOff` logo após `Started` indicavam o processo morrendo sozinho dentro do container, não um problema de rede/conectividade. Isso levou a inspecionar o motivo da última terminação do container.

### Causa Raiz: limite de memória (`resources.limits.memory`) insuficiente para o Grafana
```bash
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].status.containerStatuses[?(@.name=="grafana")].lastState}'
```
retornou:
```json
{"terminated":{"exitCode":137,"reason":"OOMKilled","finishedAt":"2026-10-26T17:00:03Z","startedAt":"2026-10-26T16:55:20Z"}}
```
`exitCode: 137` + `reason: OOMKilled` confirmam que o kernel matou o processo por estourar o cgroup de memória do container — não é falha de aplicação nem de rede. O limite configurado em `infrastructure/monitoring/values.yaml` era `256Mi`, baixo para o Grafana em uso real (já ~155Mi em regime estável, sem contar picos de carregamento de dashboards/plugins).

**Ponto-chave**: probes de liveness/readiness falhando com "connection reset"/"connection refused" (em vez de timeout) é um sinal característico de o processo ter morrido abruptamente — vale checar `lastState` do container (`OOMKilled`/`exitCode`) antes de investigar rede ou configuração da aplicação.

---

## 3. Comandos Aplicados para o Diagnóstico

```bash
# Eventos recentes do namespace monitoring
kubectl get events -n monitoring

# Último estado de terminação do container do Grafana (confirma OOMKilled)
kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].status.containerStatuses[?(@.name=="grafana")].lastState}'

# Uso de memória atual do pod, para comparar com o limite configurado
kubectl top pod -n monitoring -l app.kubernetes.io/name=grafana
```

---

## 4. A Correção Definitiva

Aumentar o limite (e o request) de memória do Grafana em `infrastructure/monitoring/values.yaml`:

```diff
   resources:
     requests:
       cpu: 100m
-      memory: 128Mi
+      memory: 192Mi
     limits:
       cpu: 500m
-      memory: 256Mi
+      memory: 512Mi
```

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1
```

### Validação final
```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana
# Novo pod Running, RESTARTS = 0

kubectl top pod -n monitoring -l app.kubernetes.io/name=grafana
# Uso de memória (~316Mi) dentro do novo limite (512Mi), com folga

kubectl get pod -n monitoring -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].status.containerStatuses[?(@.name=="grafana")].lastState}'
# Vazio ({}) - sem nova terminação por OOMKilled
```

---

## 5. Lições Aprendidas para o Homelab

1. **Probe falhando com "connection reset"/"connection refused" é sinal de processo morto, não de rede** — antes de investigar firewall/conectividade (como no caso do node-exporter, ver [prometheus_node_exporter_control_plane_ufw_block.md](prometheus_node_exporter_control_plane_ufw_block.md)), checar `lastState` do container via `kubectl get pod -o jsonpath=...` para descartar `OOMKilled`/crash da aplicação.
2. **`exitCode: 137` é o indicador direto de OOM kill** (128 + sinal 9/SIGKILL) — junto com `reason: OOMKilled`, é a confirmação definitiva, sem precisar adivinhar pela ausência de outros sintomas.
3. **Valores de `resources.limits` copiados de exemplos genéricos de chart nem sempre refletem o uso real de uma aplicação específica** — vale validar com `kubectl top pod` após a primeira instalação e ajustar com folga, em vez de assumir que o default do chart é suficiente para o cenário de uso real.
