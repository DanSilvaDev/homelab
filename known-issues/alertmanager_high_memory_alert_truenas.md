# Known Issue: Alertmanager dispara alerta de alto consumo de memória pro TrueNAS / Alertmanager fires a high memory usage alert for TrueNAS

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Causa confirmada, correção preparada em `infrastructure/monitoring/values.yaml` — pendente de `helm upgrade` e validação.
**Componente(s)**: Alertmanager, TrueNAS (ZFS ARC).
**Identificado em**: 2026-11-08.
**Relacionado**: [Observability - config.md](../clusters/observability/config.md) (seção de resources, menciona um incidente anterior de "memory pressure" no TrueNAS em 2026-10-26, resolvido na época reduzindo a RAM da VM worker2 — ver [Worker2 - config.md](../clusters/worker2/config.md)). Esse histórico é sobre pressão de memória real; o caso atual é diferente — é o alerta disparando para um consumo de memória que é esperado.

O Alertmanager está disparando alertas de alto consumo de memória para o host TrueNAS. Investigação inicial (feita pelo operador, não totalmente validada tecnicamente ainda) indica que isso é **comportamento esperado**: o ZFS usa o ARC (Adaptive Replacement Cache) e por padrão tenta consumir toda a memória disponível no host para cache — não é um vazamento nem pressão real de memória. O operador está de acordo com esse comportamento; o que falta é ajustar o alerta para não disparar mais falsamente para o TrueNAS.

### Causa identificada

- **ZFS ARC consome memória disponível por padrão** — comportamento normal e esperado do ZFS, não um problema de recursos real no host.
- Confirmado via UI do Prometheus (`/alerts`, grupo de regras `node-exporter`, regra padrão do `kube-prometheus-stack`/mixin — não customizada neste repo): `alertname="NodeMemoryHighUtilization"`, expressão `100 - (node_memory_MemAvailable_bytes{job="node-exporter"} / node_memory_MemTotal_bytes{job="node-exporter"} * 100) > 90`, `for: 15m`. Disparando pra `instance="192.168.3.103:9100"` (`service="truenas-node-exporter"`), em 95.7% no momento da investigação.

### Correção aplicada (pendente de `helm upgrade` + validação)

Em vez de uma silence manual na UI (não fica versionada) ou desabilitar a regra inteira via `kubePrometheus.defaultRules` (perderia o alerta real nos outros nodes), seguido o mesmo padrão já usado neste repo pro `Watchdog`: uma sub-rota em `alertmanager.config.route.routes` direcionando esse `alertname` + `instance` específicos pro receiver `"null"` (silencioso), em `infrastructure/monitoring/values.yaml`. A regra continua ativa e visível no Prometheus/Alertmanager pra todos os nodes — só não gera mais notificação (Telegram/ntfy) pra essa instância específica.

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1
```

Validação pendente: confirmar em `http://192.168.3.204/#/alerts` que o alerta continua listado, e que não chega mais notificação pro TrueNAS especificamente, sem afetar `NodeMemoryHighUtilization` de outros nodes.

---

## English

**Status**: 🔴 Cause confirmed, fix prepared in `infrastructure/monitoring/values.yaml` — pending `helm upgrade` and validation.
**Component(s)**: Alertmanager, TrueNAS (ZFS ARC).
**Identified on**: 2026-11-08.
**Related**: [Observability - config.md](../clusters/observability/config.md) (resources section mentions an earlier "memory pressure" incident on TrueNAS in 2026-10-26, resolved at the time by reducing the worker2 VM's RAM — see [Worker2 - config.md](../clusters/worker2/config.md)). That history was about real memory pressure; the current case is different — it's the alert firing for memory usage that is expected.

Alertmanager is firing high memory usage alerts for the TrueNAS host. Initial investigation (done by the operator, not fully technically validated yet) indicates this is **expected behavior**: ZFS uses the ARC (Adaptive Replacement Cache) and by default tries to consume all available memory on the host for caching — it's not a leak or real memory pressure. The operator is fine with this behavior; what's missing is tuning the alert so it stops falsely firing for TrueNAS.

### Identified cause

- **ZFS ARC consumes available memory by default** — normal, expected ZFS behavior, not a real resource problem on the host.
- Confirmed via the Prometheus UI (`/alerts`, `node-exporter` rule group, a default `kube-prometheus-stack`/mixin rule — not customized in this repo): `alertname="NodeMemoryHighUtilization"`, expression `100 - (node_memory_MemAvailable_bytes{job="node-exporter"} / node_memory_MemTotal_bytes{job="node-exporter"} * 100) > 90`, `for: 15m`. Firing for `instance="192.168.3.103:9100"` (`service="truenas-node-exporter"`), at 95.7% at investigation time.

### Fix applied (pending `helm upgrade` + validation)

Instead of a manual silence in the UI (not version-controlled) or disabling the whole rule via `kubePrometheus.defaultRules` (would lose the real alert on other nodes), followed the same pattern already used in this repo for `Watchdog`: a sub-route under `alertmanager.config.route.routes` directing that specific `alertname` + `instance` to the `"null"` (silent) receiver, in `infrastructure/monitoring/values.yaml`. The rule stays active and visible in Prometheus/Alertmanager for every node — it just stops generating a notification (Telegram/ntfy) for that specific instance.

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f infrastructure/monitoring/values.yaml \
  --version 92.13.1
```

Validation pending: confirm at `http://192.168.3.204/#/alerts` that the alert is still listed, and that no notification reaches TrueNAS specifically anymore, without affecting `NodeMemoryHighUtilization` on other nodes.
