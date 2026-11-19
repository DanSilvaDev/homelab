# Troubleshooting: Alertmanager disparava alerta de memória pro TrueNAS por causa do ARC do ZFS / Alertmanager fired a memory alert for TrueNAS because of ZFS's ARC

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

O Alertmanager disparava `NodeMemoryHighUtilization` repetidamente pro host TrueNAS. Causa raiz: comportamento normal e esperado do ARC (cache) do ZFS, que por padrão consome toda a memória disponível do host — não uma pressão real de memória. Corrigido silenciando o alerta só pra essa instância, sem desabilitá-lo pros outros nodes.

---

### 1. O Cenário Inicial e Sintomas

Registrado em [known-issues/alertmanager_high_memory_alert_truenas.md](../known-issues/alertmanager_high_memory_alert_truenas.md) (2026-11-08): o Alertmanager notificava (Telegram/ntfy) repetidamente sobre alto consumo de memória no host TrueNAS.

#### Sintomas do Erro:
* Alerta de memória disparando pro TrueNAS, com consumo de memória real acima de 90%.
* Diferente de um incidente anterior real de "memory pressure" documentado em [Worker2 - config.md](../clusters/worker2/config.md) (2026-10-26, resolvido reduzindo RAM da VM worker2) — esse caso era claramente esperado: o operador já sabia que era o ARC do ZFS, só faltava ajustar o alerta pra parar de notificar falsamente.

---

### 2. Processo de Investigação & Descoberta

Identificado o `alertname` exato via UI do Prometheus (`/alerts`), no grupo de regras `node-exporter` (regra padrão do `kube-prometheus-stack`/node-exporter mixin, não customizada neste repositório):

```
alertname: NodeMemoryHighUtilization
expr: 100 - (node_memory_MemAvailable_bytes{job="node-exporter"} / node_memory_MemTotal_bytes{job="node-exporter"} * 100) > 90
for: 15m
severity: warning
```

Disparando especificamente para `instance="192.168.3.103:9100"` (`service="truenas-node-exporter"` — o alvo externo do TrueNAS configurado manualmente, ver [Observability - config.md](../clusters/observability/config.md)), em 95.7% de utilização no momento da investigação.

#### Causa Raiz: ARC do ZFS consumindo memória disponível por padrão, regra padrão sem exceção por instância
O ZFS usa o ARC (Adaptive Replacement Cache) para cache de disco, e por padrão tenta usar toda a memória disponível do host para isso — comportamento intencional, não um vazamento nem pressão real de recursos. A regra `NodeMemoryHighUtilization`, vinda do mixin padrão do `kube-prometheus-stack`, avalia o mesmo threshold (90%) pra **qualquer** instância com `job="node-exporter"`, sem distinguir hosts onde esse padrão de uso é esperado (TrueNAS/ZFS) dos onde não é (nodes do cluster k3s).

**Ponto-chave**: o chart `kube-prometheus-stack` não expõe uma forma granular de excluir só uma instância de uma regra padrão — os toggles de `kubePrometheus.defaultRules` operam por grupo de regra inteiro (ex: todas as regras de `node-exporter` de uma vez), não por `alertname` individual nem por instância. Pra esse nível de granularidade, a solução fica no roteamento do Alertmanager, não na regra do Prometheus.

---

### 3. Comandos Aplicados para o Diagnóstico

```
# UI do Prometheus (http://192.168.3.203/alerts) - localizar o alertname exato,
# grupo de regras, expressão PromQL e labels da instância disparando
```

```
# UI do Alertmanager (http://192.168.3.204/#/status) - confirmar a config
# efetiva (route/routes/receivers) já aplicada, antes de editar
```

---

### 4. A Correção Definitiva (ou Workaround)

Em vez de uma silence manual na UI do Alertmanager (não fica versionada no Git) ou desabilitar a regra inteira via `kubePrometheus.defaultRules` (perderia o alerta real de memória nos outros nodes), seguido o mesmo padrão já usado neste repositório pro alerta sintético `Watchdog`: uma sub-rota em `alertmanager.config.route.routes`, em `infrastructure/monitoring/values.yaml`, direcionando esse `alertname` + `instance` específicos pro receiver `"null"` (silencioso):

```yaml
route:
  receiver: notifications
  group_by: ["alertname"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  # O Helm substitui a lista "routes" inteira (não faz merge item a item). O chart
  # por padrão já inclui uma sub-rota pro alerta sintético "Watchdog" -> receiver
  # "null" - precisa ser reincluída aqui manualmente, senão some.
  routes:
    - receiver: "null"
      matchers:
        - alertname="Watchdog"
      continue: false
    - receiver: "null"
      matchers:
        - alertname="NodeMemoryHighUtilization"
        - instance="192.168.3.103:9100"
      continue: false
```

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1
```

A regra continua ativa e visível no Prometheus/Alertmanager pra todos os nodes (não é uma silence que esconde o alerta, nem uma regra desabilitada) — só deixa de gerar notificação (Telegram/ntfy) especificamente pra essa instância.

#### Validação final
Aplicado e confirmado pelo operador: o alerta parou de notificar pro TrueNAS, sem afetar `NodeMemoryHighUtilization` de outros nodes.

---

### 5. Lições Aprendidas para o Homelab

1. **Regras de alerta padrão de um chart (`kubePrometheus.defaultRules`) geralmente não têm granularidade por instância** — só por grupo de regra inteiro. Quando um host específico tem um padrão de uso legitimamente diferente (ex: ZFS ARC no TrueNAS), a exceção pertence ao **roteamento** do Alertmanager, não à regra do Prometheus.
2. **Reaproveitar o padrão já estabelecido (`Watchdog` → receiver `null`) em vez de inventar um mecanismo novo** — manter consistência facilita entender/estender o roteamento no futuro, e já tinha o detalhe da pegadinha do Helm (substituição de lista inteira) documentado.
3. **Silence manual na UI do Alertmanager não fica versionada** — uma sub-rota no `values.yaml`, apontando pro mesmo receiver `null`, é reversível por `git revert`, visível em `git blame`/PR, e sobrevive a uma reinstalação do zero do Helm release — uma silence feita só pela UI não tem nenhuma dessas garantias.

---

## English

Alertmanager repeatedly fired `NodeMemoryHighUtilization` for the TrueNAS host. Root cause: normal, expected ZFS ARC (cache) behavior, which by default consumes all available memory on the host — not real memory pressure. Fixed by silencing the alert only for that instance, without disabling it for other nodes.

---

### 1. Initial Scenario and Symptoms

Logged in [known-issues/alertmanager_high_memory_alert_truenas.md](../known-issues/alertmanager_high_memory_alert_truenas.md) (2026-11-08): Alertmanager repeatedly notified (Telegram/ntfy) about high memory usage on the TrueNAS host.

#### Error Symptoms:
* Memory alert firing for TrueNAS, with real memory usage above 90%.
* Different from an earlier, real "memory pressure" incident documented in [Worker2 - config.md](../clusters/worker2/config.md) (2026-10-26, resolved by reducing the worker2 VM's RAM) — this case was clearly expected: the operator already knew it was ZFS's ARC, what was missing was tuning the alert to stop falsely notifying.

---

### 2. Investigation & Discovery Process

Identified the exact `alertname` via the Prometheus UI (`/alerts`), in the `node-exporter` rule group (a default `kube-prometheus-stack`/node-exporter mixin rule, not customized in this repository):

```
alertname: NodeMemoryHighUtilization
expr: 100 - (node_memory_MemAvailable_bytes{job="node-exporter"} / node_memory_MemTotal_bytes{job="node-exporter"} * 100) > 90
for: 15m
severity: warning
```

Firing specifically for `instance="192.168.3.103:9100"` (`service="truenas-node-exporter"` — the manually configured external TrueNAS target, see [Observability - config.md](../clusters/observability/config.md)), at 95.7% utilization at investigation time.

#### Root Cause: ZFS's ARC consuming available memory by default, default rule with no per-instance exception
ZFS uses the ARC (Adaptive Replacement Cache) for disk caching, and by default tries to use all available memory on the host for it — intentional behavior, not a leak or real resource pressure. The `NodeMemoryHighUtilization` rule, coming from `kube-prometheus-stack`'s default mixin, evaluates the same threshold (90%) for **any** instance with `job="node-exporter"`, without distinguishing hosts where that usage pattern is expected (TrueNAS/ZFS) from those where it isn't (k3s cluster nodes).

**Key point**: the `kube-prometheus-stack` chart doesn't expose a granular way to exclude just one instance from a default rule — the `kubePrometheus.defaultRules` toggles operate per whole rule group (e.g. every `node-exporter` rule at once), not per individual `alertname` nor per instance. For that level of granularity, the solution belongs in Alertmanager's routing, not in the Prometheus rule.

---

### 3. Commands Used for Diagnosis

```
# Prometheus UI (http://192.168.3.203/alerts) - locate the exact alertname,
# rule group, PromQL expression, and the firing instance's labels
```

```
# Alertmanager UI (http://192.168.3.204/#/status) - confirm the currently
# effective config (route/routes/receivers) before editing
```

---

### 4. The Definitive Fix (or Workaround)

Instead of a manual silence in the Alertmanager UI (not version-controlled in Git) or disabling the whole rule via `kubePrometheus.defaultRules` (would lose the real memory alert on other nodes), followed the same pattern already used in this repository for the synthetic `Watchdog` alert: a sub-route under `alertmanager.config.route.routes`, in `infrastructure/monitoring/values.yaml`, directing that specific `alertname` + `instance` to the `"null"` (silent) receiver:

```yaml
route:
  receiver: notifications
  group_by: ["alertname"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  # Helm replaces the entire "routes" list (no item-by-item merge). The chart's
  # default already includes a sub-route for the synthetic "Watchdog" alert ->
  # receiver "null" - it needs to be re-included here manually, or it disappears.
  routes:
    - receiver: "null"
      matchers:
        - alertname="Watchdog"
      continue: false
    - receiver: "null"
      matchers:
        - alertname="NodeMemoryHighUtilization"
        - instance="192.168.3.103:9100"
      continue: false
```

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1
```

The rule stays active and visible in Prometheus/Alertmanager for every node (this isn't a silence hiding the alert, nor a disabled rule) — it just stops generating a notification (Telegram/ntfy) specifically for that instance.

#### Final validation
Applied and confirmed by the operator: the alert stopped notifying for TrueNAS, without affecting `NodeMemoryHighUtilization` on other nodes.

---

### 5. Lessons Learned for the Homelab

1. **A chart's default alert rules (`kubePrometheus.defaultRules`) generally have no per-instance granularity** — only per whole rule group. When a specific host legitimately has a different usage pattern (e.g. ZFS ARC on TrueNAS), the exception belongs in Alertmanager's **routing**, not in the Prometheus rule.
2. **Reuse the already-established pattern (`Watchdog` → `null` receiver) instead of inventing a new mechanism** — staying consistent makes the routing easier to understand/extend later, and already had the Helm full-list-replacement gotcha documented.
3. **A manual silence in the Alertmanager UI isn't version-controlled** — a sub-route in `values.yaml`, pointing at the same `null` receiver, is revertible via `git revert`, visible in `git blame`/a PR, and survives a from-scratch reinstall of the Helm release — a UI-only silence has none of those guarantees.
