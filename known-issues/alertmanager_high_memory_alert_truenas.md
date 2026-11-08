# Known Issue: Alertmanager dispara alerta de alto consumo de memória pro TrueNAS / Alertmanager fires a high memory usage alert for TrueNAS

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Causa identificada — ação pendente (ajustar/desativar o alerta especificamente pro TrueNAS).
**Componente(s)**: Alertmanager, TrueNAS (ZFS ARC).
**Identificado em**: 2026-11-08.
**Relacionado**: [Observability - config.md](../clusters/observability/config.md) (seção de resources, menciona um incidente anterior de "memory pressure" no TrueNAS em 2026-10-26, resolvido na época reduzindo a RAM da VM worker2 — ver [Worker2 - config.md](../clusters/worker2/config.md)). Esse histórico é sobre pressão de memória real; o caso atual é diferente — é o alerta disparando para um consumo de memória que é esperado.

O Alertmanager está disparando alertas de alto consumo de memória para o host TrueNAS. Investigação inicial (feita pelo operador, não totalmente validada tecnicamente ainda) indica que isso é **comportamento esperado**: o ZFS usa o ARC (Adaptive Replacement Cache) e por padrão tenta consumir toda a memória disponível no host para cache — não é um vazamento nem pressão real de memória. O operador está de acordo com esse comportamento; o que falta é ajustar o alerta para não disparar mais falsamente para o TrueNAS.

### Causa identificada

- **ZFS ARC consome memória disponível por padrão** — comportamento normal e esperado do ZFS, não um problema de recursos real no host.
- O alerta provavelmente é uma das regras padrão de memória do `kube-prometheus-stack`/node-exporter mixin (nome exato ainda não identificado — não há regra customizada de memória pro TrueNAS versionada em `apps/grafana/alerts/` neste repositório), que não leva em conta esse comportamento do ZFS ao avaliar o threshold.

### Ação sugerida (ainda não aplicada)

- Identificar o `alertname` exato disparando (via UI do Alertmanager/Prometheus, `http://alert.home`) antes de qualquer mudança.
- Desativar ou ajustar o alerta **só para a instância do TrueNAS** (não globalmente, pra não perder o alerta de memória real nos outros nodes) — opções a avaliar: silence permanente no Alertmanager filtrado por `instance`/`job` do TrueNAS, ou uma exceção na expressão PromQL da regra excluindo essa instância.
- Evitar desabilitar a regra inteira via `kubePrometheus.defaultRules` em `infrastructure/monitoring/values.yaml`, já que isso desativaria o alerta pra todos os nodes, não só o TrueNAS.

---

## English

**Status**: 🔴 Cause identified — action pending (adjust/disable the alert specifically for TrueNAS).
**Component(s)**: Alertmanager, TrueNAS (ZFS ARC).
**Identified on**: 2026-11-08.
**Related**: [Observability - config.md](../clusters/observability/config.md) (resources section mentions an earlier "memory pressure" incident on TrueNAS in 2026-10-26, resolved at the time by reducing the worker2 VM's RAM — see [Worker2 - config.md](../clusters/worker2/config.md)). That history was about real memory pressure; the current case is different — it's the alert firing for memory usage that is expected.

Alertmanager is firing high memory usage alerts for the TrueNAS host. Initial investigation (done by the operator, not fully technically validated yet) indicates this is **expected behavior**: ZFS uses the ARC (Adaptive Replacement Cache) and by default tries to consume all available memory on the host for caching — it's not a leak or real memory pressure. The operator is fine with this behavior; what's missing is tuning the alert so it stops falsely firing for TrueNAS.

### Identified cause

- **ZFS ARC consumes available memory by default** — normal, expected ZFS behavior, not a real resource problem on the host.
- The alert is likely one of the default memory rules from `kube-prometheus-stack`/the node-exporter mixin (exact name not yet identified — there's no custom memory rule for TrueNAS versioned in `apps/grafana/alerts/` in this repository), which doesn't account for this ZFS behavior when evaluating its threshold.

### Suggested action (not yet applied)

- Identify the exact `alertname` firing (via the Alertmanager/Prometheus UI, `http://alert.home`) before making any change.
- Disable or adjust the alert **only for the TrueNAS instance** (not globally, to avoid losing the real memory alert on the other nodes) — options to evaluate: a permanent silence in Alertmanager filtered by TrueNAS's `instance`/`job`, or an exception in the rule's PromQL expression excluding that instance.
- Avoid disabling the whole rule via `kubePrometheus.defaultRules` in `infrastructure/monitoring/values.yaml`, since that would disable the alert for every node, not just TrueNAS.
