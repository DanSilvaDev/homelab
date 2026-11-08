# Known Issue: Loki registra um app não identificado, logando só "ppppppppp" / Loki logs an unidentified app, only recording "ppppppppp"

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Em investigação — causa raiz não confirmada (suspeita de qual app é a origem, ainda não validada).
**Componente(s)**: Loki, Grafana Alloy (observability) — suspeita: Pi-hole.
**Identificado em**: 2026-11-08.
**Relacionado**: [Observability - config.md](../clusters/observability/config.md) (seção "Loki + Grafana Alloy").

No Grafana Explore (datasource Loki), aparece um serviço/app **não identificado** na lista de services — em vez de um label reconhecível (nome do app), as linhas de log registradas para esse serviço são só uma sequência repetida do caractere "p" (`ppppppppp`), sem nenhum conteúdo real de log.

### Hipóteses / o que já foi tentado

- **Suspeita: Pi-hole.** É o único app rodando no cluster que não aparece identificado corretamente na lista de services do Loki — todos os outros têm labels reconhecíveis. Essa suspeita ainda **não foi validada** diretamente (não foi confirmado nos logs brutos do pod do Pi-hole, nem nos labels que o Alloy atribuiu à entrada "desconhecida").
- Não está confirmado se a causa é de **parsing** (o Alloy falhando em extrair labels corretos do pod de origem) ou se o **próprio container** está de fato emitindo essa string nos logs brutos.

### Diagnóstico sugerido (ainda não executado)

- `kubectl logs -n default -l app=pihole` (confirmar o label real do Deployment antes) — ver se a string `ppppppppp` aparece nos logs brutos do container do Pi-hole, o que confirmaria ou descartaria a suspeita diretamente.
- No Grafana Explore, inspecionar os labels completos (`namespace`, `pod`, `container`, `job`) da entrada "desconhecida" — o Alloy sempre atribui esses labels via `loki.source.kubernetes`, então eles devem apontar exatamente pro pod de origem, suspeito ou não.
- Comparar a configuração de descoberta de labels do Alloy (`infrastructure/logging/alloy-values.yaml`) entre o pod do Pi-hole e os pods dos apps identificados corretamente, caso os labels acima apontem pra ele — ver se falta algum label padrão que os outros têm.
- Verificar se o Pi-hole emite alguma saída binária ou sequência de controle de terminal (ex: barra de progresso, log colorido via códigos ANSI) que poderia ser mal interpretada/exibida como `ppppppppp` repetido.

---

## English

**Status**: 🔴 Under investigation — root cause not confirmed (suspected source not yet validated).
**Component(s)**: Loki, Grafana Alloy (observability) — suspected: Pi-hole.
**Identified on**: 2026-11-08.
**Related**: [Observability - config.md](../clusters/observability/config.md) ("Loki + Grafana Alloy" section).

In Grafana Explore (Loki datasource), an **unidentified** service/app shows up in the services list — instead of a recognizable label (app name), the log lines recorded for that service are just a repeated sequence of the "p" character (`ppppppppp`), with no actual log content.

### Hypotheses / what's already been tried

- **Suspected: Pi-hole.** It's the only app running in the cluster that doesn't show up correctly identified in Loki's services list — every other app has a recognizable label. This suspicion has **not been validated** directly yet (not confirmed against the Pi-hole pod's raw logs, nor against the labels Alloy assigned to the "unidentified" entry).
- Not confirmed whether the cause is **parsing** (Alloy failing to extract correct labels from the source pod) or whether the **container itself** is actually emitting that string in its raw logs.

### Suggested diagnostics (not yet run)

- `kubectl logs -n default -l app=pihole` (confirm the Deployment's actual label first) — check whether the `ppppppppp` string shows up in the Pi-hole container's raw logs, which would directly confirm or rule out the suspicion.
- In Grafana Explore, inspect the full labels (`namespace`, `pod`, `container`, `job`) of the "unidentified" entry — Alloy always assigns these via `loki.source.kubernetes`, so they should point exactly to the source pod, suspected or not.
- Compare Alloy's label discovery config (`infrastructure/logging/alloy-values.yaml`) between the Pi-hole pod and the pods of correctly identified apps, if the labels above point to it — check whether some standard label the others have is missing.
- Check whether Pi-hole emits any binary output or terminal control sequence (e.g. a progress bar, colored log via ANSI codes) that could be misinterpreted/displayed as repeated `ppppppppp`.
