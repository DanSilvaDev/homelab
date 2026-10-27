# Troubleshooting: Alertmanager rejeitava a config customizada (`undefined receiver "null" used in route`)

Documenta por que, ao configurar o roteamento de alertas do Alertmanager (via `alertmanager.config` no `values.yaml` do `kube-prometheus-stack`) para o Telegram, o Prometheus Operator passou a falhar a reconciliação e o Alertmanager continuou usando a config padrão (sem o receiver novo).

---

## 1. O Cenário Inicial e Sintomas

Depois de adicionar `alertmanager.config` no `values.yaml` (com `route.receiver: telegram` e um `receivers` customizado) e rodar `helm upgrade`, o pod do Alertmanager reiniciou (por causa do novo `alertmanagerSpec.secrets` montado), carregou sem erro de parse — mas nenhuma mensagem chegava no Telegram, e o alerta de teste continuava sendo roteado para um receiver chamado `"null"`, não para `telegram`.

### Sintomas do Erro:
* `kubectl exec ... cat /etc/alertmanager/config_out/alertmanager.env.yaml` mostrava a config **padrão** do chart (`route.receiver: "null"`, `receivers: [{name: "null"}]`), sem nenhum traço do `telegram_configs` que havíamos colocado no `values.yaml`.
* `helm get values kube-prometheus-stack -n monitoring` confirmava que o Helm **tinha** registrado corretamente o `alertmanager.config` customizado — ou seja, o problema não era um erro de digitação no `values.yaml`.
* O Secret gerado diretamente pelo Helm (`alertmanager-kube-prometheus-stack-alertmanager`) **continha** o `telegram_configs` certo — ou seja, o Helm tinha renderizado tudo certo.
* O recurso `Alertmanager` (CRD) tinha uma condição `Reconciled: False`:
  ```
  reason: ReconciliationFailed
  message: provision alertmanager configuration: failed to initialize from secret: undefined receiver "null" used in route
  ```
* Os logs do `prometheus-operator` confirmavam a mesma falha, repetida a cada tentativa de sync.

---

## 2. Processo de Investigação & Descoberta

A cadeia de componentes é: `values.yaml` → Helm renderiza um Secret (`alertmanager-<release>-alertmanager`) → Prometheus Operator lê esse Secret e tenta "provisionar" a config final do Alertmanager → se válida, escreve a config final que o `config-reloader` descomprime para dentro do pod. Cada etapa foi inspecionada isoladamente (comando correspondente na seção 3) até achar em qual etapa a cadeia quebrava: o Secret do Helm estava correto, mas o Operator rejeitava a config ao tentar aplicá-la — ou seja, a falha estava na validação feita pelo próprio Alertmanager/Operator, não no template do Helm.

### Causa Raiz: Helm substitui listas inteiras, não faz merge item a item
A config *padrão* do chart `kube-prometheus-stack` já vem com uma sub-rota pronta, para silenciar o alerta sintético `Watchdog` (um "dead man's switch" que fica sempre firing só para provar que o pipeline de alertas está vivo):
```yaml
route:
  routes:
    - receiver: "null"
      matchers:
        - alertname = "Watchdog"
receivers:
  - name: "null"
```
Nosso `values.yaml` customizado sobrescreveu `route` (um mapa) e `receivers` (uma lista). O Helm faz **merge profundo em mapas**, então `route.receiver`, `route.group_by` etc. foram sobrescritos como esperado, mas como não especificamos `route.routes`, essa chave **do mapa** permaneceu com o valor padrão (ainda apontando pro `Watchdog` → `"null"`). Já `receivers` é uma **lista**, e o Helm **substitui listas inteiras** em vez de mesclar item a item — então nossa lista (só com `telegram`) apagou completamente o receiver `"null"` do default. Resultado: a sub-rota órfã referenciava um receiver que não existia mais, e o Alertmanager rejeitou a config inteira (não é um "degradar com aviso", é tudo ou nada).

**Ponto-chave**: ao sobrescrever `alertmanager.config.receivers` neste chart, é preciso **reincluir manualmente** qualquer receiver padrão que ainda seja referenciado por uma sub-rota não sobrescrita (como o `"null"` do `Watchdog`) — mesmo que você não tenha tocado nessa sub-rota diretamente.

---

## 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirma que o pod subiu e a config "carregou" sem erro de parse (não prova que é a config certa)
kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager
kubectl logs -n monitoring -l app.kubernetes.io/name=alertmanager -c alertmanager --tail=50

# Confirma os valores que o Helm tem registrados pra release (sem checar o que de fato chegou no pod)
helm get values kube-prometheus-stack -n monitoring

# Confirma o que o Secret gerado diretamente pelo Helm contém (ainda não passou pelo Operator)
kubectl get secret -n monitoring alertmanager-kube-prometheus-stack-alertmanager \
  -o jsonpath='{.data.alertmanager\.yaml}' | base64 -d

# Confirma o que o pod REALMENTE carregou no final (depois do Operator processar)
# Nome do arquivo pode variar por versão do Prometheus Operator ("alertmanager.yaml" vs "alertmanager.env.yaml")
MSYS_NO_PATHCONV=1 kubectl exec -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0 \
  -c alertmanager -- ls -la /etc/alertmanager/config_out/
MSYS_NO_PATHCONV=1 kubectl exec -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0 \
  -c alertmanager -- cat /etc/alertmanager/config_out/alertmanager.env.yaml

# Isolou a falha na etapa do Operator: condição Reconciled=False com a mensagem de erro
kubectl get alertmanager -n monitoring kube-prometheus-stack-alertmanager -o jsonpath='{.status.conditions}'

# Confirmação adicional nos logs do próprio operator
kubectl logs -n monitoring -l app.kubernetes.io/name=kube-prometheus-stack-prometheus-operator --tail=200 \
  | grep -i alertmanager
```

> Nota (Windows/Git Bash): comandos com caminhos absolutos iniciados em `/` (ex: `/etc/alertmanager/...`) passados para `kubectl exec` são auto-convertidos para caminhos do Windows pelo MSYS2, quebrando o comando dentro do container. Contorna-se com `MSYS_NO_PATHCONV=1 kubectl exec ...` ou dobrando a barra inicial (`//etc/...`).

---

## 4. A Correção Definitiva

Reincluir um receiver `"null"` vazio (silencioso, sem nenhuma integração) na lista customizada, ao lado do `telegram`/`notifications`:

```yaml
receivers:
  - name: notifications
    telegram_configs: [...]
    webhook_configs: [...]
  - name: "null"
```

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1
```

### Validação final
```bash
# Reconciled deve virar True
kubectl get alertmanager -n monitoring kube-prometheus-stack-alertmanager -o jsonpath='{.status.conditions}'

# Config final deve conter o receiver customizado
MSYS_NO_PATHCONV=1 kubectl exec -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0 \
  -c alertmanager -- cat /etc/alertmanager/config_out/alertmanager.env.yaml

# Teste ponta a ponta
curl -i -H "Content-Type: application/json" \
  -d '[{"labels":{"alertname":"TesteHomelab","severity":"info"}}]' \
  http://192.168.3.204/api/v2/alerts
```
Confirmado: `Reconciled: True`, config final com `telegram_configs`/`webhook_configs`, e a notificação de teste chegou no Telegram e no ntfy.

---

## 5. Lições Aprendidas para o Homelab

1. **Em charts Helm, mapas fazem merge profundo, mas listas são substituídas inteiras** — qualquer `values.yaml` que sobrescreva uma lista que o chart já popula por padrão (como `alertmanager.config.receivers`) apaga qualquer item default daquela lista. Antes de sobrescrever, vale checar `helm show values <chart>` ou os defaults do chart pra saber se algum outro campo (como uma sub-rota) depende de um item que está prestes a desaparecer.
2. **"Pod rodando e sem erro de parse nos logs" não prova que a config nova foi aplicada** — nesse caso o Alertmanager continuou de pé com a config *antiga* (default), porque quem rejeitou a mudança foi uma camada acima (o Operator), não o próprio Alertmanager tentando carregar um arquivo malformado. Validar sempre a config final (`config_out/`) e o status de reconciliação do recurso Operator, não só "o pod não crashou".
3. **Com o Prometheus Operator, existe uma cadeia de 3 etapas entre o `values.yaml` e o que o pod realmente usa** (Helm → Secret base → Operator reconcilia e gera a config final → config-reloader escreve no pod) — ao debugar, isolar em qual etapa a cadeia quebra (inspecionando cada uma separadamente) é bem mais rápido do que assumir que o problema está sempre no `values.yaml`.
4. **O alerta `Watchdog` do kube-prometheus-stack é intencionalmente silenciado (`receiver: "null"`)** — é um "dead man's switch" pensado para integração com serviços externos de heartbeat (ex: Healthchecks.io), não para notificar. Isso é esperado, não um bug a corrigir.
