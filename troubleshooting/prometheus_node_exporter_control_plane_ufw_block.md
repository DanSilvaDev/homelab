# Troubleshooting: Prometheus não conseguia raspar o node-exporter do control-plane

Documenta por que o target do node-exporter do control-plane aparecia `DOWN` no Prometheus (e ausente nos dashboards do Grafana) logo após a instalação do Rancher Monitoring (kube-prometheus-stack), enquanto worker1 e worker2 funcionavam normalmente.

---

## 1. O Cenário Inicial e Sintomas

Após instalar o `kube-prometheus-stack` e validar os dashboards no Grafana, o dashboard "Node Exporter / Nodes" não listava o control-plane no seletor de node — só apareciam worker1 e worker2.

### Sintomas do Erro:
* O seletor de node do dashboard no Grafana não incluía o control-plane.
* Os pods do node-exporter estavam confirmadamente `Running` nos 3 nodes (`kubectl get pods -n monitoring -o wide`).
* No Prometheus (`/targets`), o target do node-exporter do control-plane aparecia **DOWN**, com o erro:
  ```
  Error scraping target: Get "http://192.168.3.101:9100/metrics": context deadline exceeded
  ```
* worker1 e worker2 apareciam normalmente como `UP`.

---

## 2. Processo de Investigação & Descoberta

Descartado problema de scheduling ou do pod em si, já que o node-exporter estava rodando normalmente nos 3 nodes. O erro "context deadline exceeded" indicava falha de conexão de rede (não um erro de aplicação), e o problema estava isolado ao control-plane — o que levou a checar o firewall local dele.

### Causa Raiz: regra de firewall (ufw) não cobria o tráfego SNAT do cluster
O node-exporter roda com `hostNetwork: true`, ou seja, ele escuta na porta `9100` do **IP real do node** (ex: `192.168.3.101`), não num IP de pod. O `ufw` do control-plane já tinha regras liberando tráfego vindo das CIDRs internas do cluster (`10.42.0.0/16` para pods, `10.43.0.0/16` para serviços) — mas o k3s (via flannel) aplica **SNAT/masquerade** no tráfego que sai de um pod em direção a um destino fora dessas CIDRs, como o IP real de outro node na LAN. Isso significa que o pacote chega no control-plane com o **IP real do node de origem** (ex: o IP do worker1 ou worker2 na LAN), não o IP do pod do Prometheus. Essa origem não batia com nenhuma regra existente no `ufw`, e caía na política padrão `deny (entrada)`.

**Ponto-chave**: isso só afetou o control-plane porque apenas ele tinha o `ufw` ativo com política restritiva — worker1 e worker2 não tinham essa porta bloqueada.

---

## 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirma que o pod do node-exporter está rodando nos 3 nodes
kubectl get pods -n monitoring -o wide | grep node-exporter

# Acessa a UI do Prometheus pra inspecionar os targets
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# http://localhost:9090/targets -> job node-exporter, instância do control-plane em DOWN
# Erro: Get "http://192.168.3.101:9100/metrics": context deadline exceeded

# No control-plane: inspeciona o firewall
sudo ufw status verbose
# Confirmado: política padrão "deny (entrada)", sem nenhuma regra para a porta 9100
```

---

## 4. A Correção Definitiva

Liberar a porta `9100/tcp` no `ufw` do control-plane, a partir da rede local — mesmo padrão já usado para outras portas de comunicação entre nodes do cluster:

```bash
sudo ufw allow from 192.168.3.0/24 to any port 9100 proto tcp comment 'node-exporter'
sudo ufw reload
```

### Validação final
```bash
sudo ufw status verbose   # nova regra para a porta 9100 a partir de 192.168.3.0/24 confirmada
```
No Prometheus (`/targets`), o target do node-exporter do control-plane passou a `UP`. Confirmado também no Grafana: o dashboard "Node Exporter / Nodes" passou a listar os 3 nodes com dados populados.

---

## 5. Lições Aprendidas para o Homelab

1. **Um DaemonSet com `hostNetwork: true` é alcançado pelo IP real do node, não por um IP de pod** — regras de firewall baseadas nas CIDRs internas do cluster (pod/serviço) não cobrem esse tráfego; a porta precisa ser liberada especificamente no firewall do host.
2. **Tráfego de um pod para um destino fora das CIDRs de pod/serviço do cluster é mascarado (SNAT) pelo k3s/flannel** — o firewall de destino enxerga o IP real do node de origem, não o IP do pod. Importante lembrar ao escrever regras de firewall baseadas apenas nas CIDRs do cluster.
3. **Nodes com postura de firewall diferente dentro do mesmo cluster geram falhas assimétricas e difíceis de generalizar** — o mesmo app (node-exporter) falhou só no node com `ufw` ativo. Vale revisar se os demais nodes deveriam ter proteção equivalente, em vez de depender da ausência de firewall.
4. **Pod `Running` não garante que o Prometheus consegue alcançá-lo pela rede** — sempre validar a página `/targets` do Prometheus além do status dos pods ao investigar métricas ausentes.
