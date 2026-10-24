# Troubleshooting: NVIDIA Device Plugin não agenda Pod mesmo com nodeSelector correto

Esta documentação detalha o diagnóstico de um problema onde o Helm chart `nvdp/nvidia-device-plugin` não agendava nenhum Pod no nó GPU (`worker2`) do cluster K3s, mesmo com o `nodeSelector` customizado corretamente aplicado e sem nenhum taint bloqueando o node.

---

## 1. O Cenário Inicial e Sintomas

Após instalar o driver NVIDIA (580-server), o NVIDIA Container Toolkit, e configurar o runtime `nvidia` no template de containerd do K3s, o passo final era instalar o **NVIDIA device plugin** via Helm para expor o recurso `nvidia.com/gpu` ao cluster.

### Sintomas do Erro:
* `kubectl get pods -n nvidia-device-plugin` retornava `No resources found`, mesmo com o release do Helm marcado como `deployed`.
* `kubectl get daemonset -n nvidia-device-plugin` mostrava `DESIRED: 0`, `CURRENT: 0` — ou seja, o controller do DaemonSet entendia que **nenhum node** era elegível.
* O node (`worker2`) não tinha nenhum taint (`Taints: <none>`), estava `Ready` e `Unschedulable: false`.
* O label customizado (`gpu=true`) aplicado via `kubectl label node worker2 gpu=true` batia exatamente com o `nodeSelector` do chart.
* Um Pod de teste simples (`busybox`) com o mesmo `nodeSelector: {gpu: "true"}` **agendou normalmente** no worker2 — descartando problema geral de scheduling no cluster.
* `kubectl get events -n nvidia-device-plugin` não mostrava nenhum evento, nem de rejeição (`FailedScheduling`) — indicando que o controller do DaemonSet nunca sequer tentou criar o Pod.

---

## 2. Processo de Investigação & Descoberta

Como um Pod comum com o mesmo `nodeSelector` funcionava, mas o DaemonSet gerenciado pelo Helm não, o problema tinha que estar em alguma diferença entre o manifesto renderizado pelo chart e um Pod simples.

### Causa Raiz: nodeAffinity obrigatória embutida no chart
Inspecionando o YAML completo do DaemonSet (`kubectl get daemonset ... -o yaml`, não apenas `describe`, que pode omitir seções), foi encontrada uma `nodeAffinity` obrigatória no `spec.template.spec.affinity`:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: feature.node.kubernetes.io/pci-10de.present
          operator: In
          values: ["true"]
      - matchExpressions:
        - key: feature.node.kubernetes.io/cpu-model.vendor_id
          operator: In
          values: ["NVIDIA"]
      - matchExpressions:
        - key: nvidia.com/gpu.present
          operator: In
          values: ["true"]
```

Essa `nodeAffinity` exige que o node tenha **pelo menos um** desses três labels (cada item da lista `nodeSelectorTerms` é uma condição alternativa — lógica OR entre os termos). Os dois primeiros labels (`feature.node.kubernetes.io/*`) são normalmente setados automaticamente pelo **Node Feature Discovery (NFD)**, e o terceiro (`nvidia.com/gpu.present`) pelo **NVIDIA GPU Operator** — nenhum dos dois estava instalado nesse ambiente, que usa a instalação manual do driver + container toolkit (mais leve, sem exigir esses operadores completos).

**Ponto-chave**: `nodeSelector` (campo customizado que configuramos via Helm `--set`) e `nodeAffinity` (embutida no chart, não documentada nos values padrão) são exigências que **se somam (AND)**, não se substituem. Mesmo com o `nodeSelector` 100% correto, a ausência de qualquer um dos três labels de affinity bloqueia o agendamento silenciosamente — sem gerar evento de erro, porque o controller do DaemonSet nem considera o node como candidato.

---

## 3. Comandos Aplicados para o Diagnóstico

```bash
# 1. Confirmar ausência de taints e status do node
kubectl describe node worker2 | grep -A5 "Taints"

# 2. Testar se nodeSelector funciona para um Pod comum (isola o problema)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-gpu-selector
spec:
  nodeSelector:
    gpu: "true"
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
EOF
kubectl get pod test-gpu-selector -o wide

# 3. Inspecionar o YAML COMPLETO do DaemonSet (não só o describe)
kubectl get daemonset nvidia-device-plugin -n nvidia-device-plugin -o yaml

# 4. Confirmar os values realmente aplicados pelo Helm
helm get values nvidia-device-plugin -n nvidia-device-plugin
```

---

## 4. A Correção Definitiva

Como o ambiente não usa NFD nem GPU Operator, a solução mais simples foi aplicar manualmente o terceiro label exigido pela affinity:

```bash
kubectl label node worker2 nvidia.com/gpu.present=true
```

Imediatamente após o label ser aplicado, o DaemonSet reconciliou e o `DESIRED` passou de `0` para `1`, com o Pod subindo normalmente no worker2.

### Validação final
```bash
kubectl get daemonset -n nvidia-device-plugin
kubectl get pods -n nvidia-device-plugin -o wide
kubectl describe node worker2 | grep -i gpu   # confirma nvidia.com/gpu em Capacity/Allocatable
```

---

## 5. Lições Aprendidas para o Homelab

1. **`describe` pode esconder detalhes** — ao investigar um DaemonSet/Deployment gerenciado por Helm que não se comporta como esperado, sempre confira o `-o yaml` completo, não só o `kubectl describe`. Seções como `affinity` podem estar presentes no spec mas não saltar aos olhos no describe se não forem a primeira coisa checada.
2. **Charts de GPU normalmente assumem NFD/GPU Operator** — vários charts da NVIDIA (device plugin, GPU Feature Discovery, DCGM exporter) têm affinities padrão que assumem um ambiente com Node Feature Discovery rodando. Em setups manuais e enxutos (sem esses operadores), é preciso inspecionar o template renderizado e aplicar manualmente os labels equivalentes.
3. **`nodeSelector` custom via Helm não sobrescreve affinity do chart** — os dois mecanismos coexistem e ambos precisam ser satisfeitos. Um `nodeSelector` correto não é garantia de scheduling se o chart também define `affinity.nodeAffinity` nos bastidores.
4. **Ausência de eventos de erro é um sinal, não um "tudo ok"** — quando o DaemonSet controller decide que `desiredNumberScheduled = 0`, ele nunca tenta criar o Pod, então não há `FailedScheduling` nem qualquer evento registrado. "Nenhum erro" nesse caso significa "nem tentou", não "funcionou".
