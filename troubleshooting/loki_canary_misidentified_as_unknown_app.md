# Troubleshooting: App "desconhecido" logando `ppppppppp` no Loki era o `loki-canary` / "Unknown" app logging `ppppppppp` in Loki was `loki-canary`

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Um app aparecia como "desconhecido" no Loki, registrando só a string repetida `ppppppppp`. A causa raiz não era um app quebrado nem um problema de parsing do Alloy: era o `loki-canary`, um sub-componente legítimo do próprio Helm chart do Loki, habilitado por padrão e nunca referenciado explicitamente nos values deste repositório — por isso não foi reconhecido de cara.

---

### 1. O Cenário Inicial e Sintomas

Foi registrado um [known-issue](../known-issues.md) relatando um app "desconhecido" no Grafana Explore (datasource Loki), cujas únicas linhas de log eram uma sequência repetida do caractere `p`. A suspeita inicial era o Pi-hole, por ser (aparentemente) o único app sem um label claramente reconhecível na lista de services do Loki — suspeita que nunca foi validada diretamente.

#### Sintomas do Erro:
* No Grafana Explore, uma entrada sem identificação clara aparecia na lista de services do Loki.
* As linhas de log registradas para essa entrada eram só `ppppppppp` repetido, sem conteúdo real.
* Nenhum app conhecido do repositório (`apps/`, `infrastructure/logging/loki-values.yaml`) parecia corresponder a essa origem.

---

### 2. Processo de Investigação & Descoberta

Em vez de investigar a suspeita (Pi-hole) isoladamente, a consulta `{job=~".+"} |= "ppppppppp"` foi rodada direto no Grafana Explore — o Loki guarda os labels completos (`namespace`, `pod`, `container`, `job`) junto de cada linha, então essa busca aponta a origem real sem precisar adivinhar. O resultado mostrou três pods: `loki-canary-95w5h`, `loki-canary-dzsfv`, `loki-canary-czdlm` — um por node do cluster, padrão típico de `DaemonSet`.

Confirmado em seguida que `infrastructure/logging/loki-values.yaml` não tem nenhuma menção a `canary` — ou seja, esse `DaemonSet` nunca foi configurado explicitamente neste repositório, veio habilitado pelo **default do chart** `grafana-community/loki`.

#### Causa Raiz: `loki-canary` é um sub-componente de auto-verificação do próprio chart do Loki
O `loki-canary` é uma ferramenta oficial do ecossistema Loki: ele escreve continuamente entradas de log sintéticas (conteúdo de preenchimento, sem significado real — por isso a aparência de `ppppppppp` repetido) e depois consulta o próprio Loki pra confirmar que essas entradas chegaram corretamente, medindo latência de ingestão, entradas perdidas e entregas fora de ordem. Como ele é um sub-componente automático do chart (não algo definido em `apps/` nem explicitamente nos values deste repo), nunca foi reconhecido como um componente legítimo — parecia um app misterioso.

**Ponto-chave**: Helm charts frequentemente trazem sub-componentes auxiliares (canaries, exporters, etc.) habilitados por padrão, que nunca aparecem no `values.yaml` do próprio usuário — presenças "desconhecidas" no cluster valem uma checagem rápida se não são sub-componentes default de algum chart já instalado, antes de assumir que é um app externo ou um problema.

---

### 3. Comandos Aplicados para o Diagnóstico

```
# No Grafana Explore (datasource Loki) - aponta direto a origem via os labels
# completos de cada linha de log, sem precisar adivinhar qual app é
{job=~".+"} |= "ppppppppp"
```

```bash
# Confirma que o canary nunca foi configurado explicitamente neste repo
# (ou seja, veio do default do chart grafana-community/loki)
grep -n -i "canary" infrastructure/logging/loki-values.yaml
```

---

### 4. A Correção Definitiva (ou Workaround)

Nenhuma correção necessária — é comportamento esperado e intencional de um componente legítimo do Loki. A ação tomada foi só a identificação e esta documentação, pra evitar que o mesmo "app misterioso" seja re-investigado do zero no futuro.

Se o ruído extra no Loki (ou os 3 pods adicionais, um por node) não for desejado, o `loki-canary` pode ser desabilitado explicitamente via `monitoring.lokiCanary.enabled: false` nos values do chart — não foi feito aqui, já que é uma ferramenta útil de auto-verificação da própria stack de logging.

#### Validação final
```
# Re-rodar a mesma query no Grafana Explore e confirmar que os labels da
# origem batem com os pods loki-canary-* (não um Deployment desconhecido)
{job=~".+"} |= "ppppppppp"
```

---

### 5. Lições Aprendidas para o Homelab

1. **Busque a origem real via os labels do próprio sistema de logs antes de seguir uma suspeita** — uma consulta direta (`{job=~".+"} |= "<texto>"`) no Loki aponta exatamente `namespace`/`pod`/`container`/`job` de onde a linha veio, evitando investigar na direção errada (aqui, a suspeita inicial no Pi-hole nunca teria sido confirmada, porque não era ele).
2. **Helm charts instalam sub-componentes auxiliares por padrão, mesmo sem configuração explícita** — `loki-canary` nunca apareceu em `infrastructure/logging/loki-values.yaml` deste repo, mas existe porque é parte do chart `grafana-community/loki`. Antes de tratar uma presença "desconhecida" no cluster como bug ou app externo, vale checar se não é um sub-componente default de algo já instalado.
3. **`loki-canary` é um nome de componente a reconhecer no ecossistema Loki** — escreve entradas sintéticas de propósito (sem significado semântico) pra validar o próprio pipeline de ingestão de logs; não é um app de usuário nem uma fonte de log "real".

---

## English

An app showed up as "unknown" in Loki, only ever logging the repeated string `ppppppppp`. The root cause wasn't a broken app or an Alloy parsing issue: it was `loki-canary`, a legitimate sub-component of the Loki Helm chart itself, enabled by default and never explicitly referenced in this repository's values — which is why it wasn't recognized at first glance.

---

### 1. Initial Scenario and Symptoms

A [known-issue](../known-issues.md) had been logged reporting an "unknown" app in Grafana Explore (Loki datasource), whose only log lines were a repeated sequence of the `p` character. The initial suspicion was Pi-hole, as it was (apparently) the only app without a clearly recognizable label in Loki's services list — a suspicion that was never directly validated.

#### Error Symptoms:
* In Grafana Explore, an entry with no clear identification showed up in Loki's services list.
* The log lines recorded for that entry were just repeated `ppppppppp`, with no actual content.
* No app known from the repository (`apps/`, `infrastructure/logging/loki-values.yaml`) seemed to match that source.

---

### 2. Investigation & Discovery Process

Instead of investigating the suspected source (Pi-hole) in isolation, the query `{job=~".+"} |= "ppppppppp"` was run directly in Grafana Explore — Loki stores the full labels (`namespace`, `pod`, `container`, `job`) alongside every log line, so this search points straight to the real source without guessing. The result showed three pods: `loki-canary-95w5h`, `loki-canary-dzsfv`, `loki-canary-czdlm` — one per cluster node, a typical `DaemonSet` pattern.

It was then confirmed that `infrastructure/logging/loki-values.yaml` has no mention of `canary` at all — meaning this `DaemonSet` was never explicitly configured in this repository, it came enabled by the **chart's default** (`grafana-community/loki`).

#### Root Cause: `loki-canary` is a self-verification sub-component of the Loki chart itself
`loki-canary` is an official tool from the Loki ecosystem: it continuously writes synthetic log entries (filler content, with no real meaning — hence the repeated `ppppppppp` appearance) and then queries Loki itself to confirm those entries arrived correctly, measuring ingestion latency, lost entries, and out-of-order delivery. Since it's an automatic sub-component of the chart (not something defined under `apps/` nor explicitly set in this repo's values), it was never recognized as a legitimate component — it looked like a mystery app.

**Key point**: Helm charts frequently ship auxiliary sub-components (canaries, exporters, etc.) enabled by default, which never show up in the user's own `values.yaml` — an "unknown" presence in the cluster is worth a quick check for whether it's a default sub-component of something already installed, before assuming it's an external app or a problem.

---

### 3. Commands Used for Diagnosis

```
# In Grafana Explore (Loki datasource) - points straight to the source via the
# full labels on each log line, without needing to guess which app it is
{job=~".+"} |= "ppppppppp"
```

```bash
# Confirms the canary was never explicitly configured in this repo
# (i.e. it came from the grafana-community/loki chart's default)
grep -n -i "canary" infrastructure/logging/loki-values.yaml
```

---

### 4. The Definitive Fix (or Workaround)

No fix needed — this is expected, intentional behavior of a legitimate Loki component. The action taken was just identification and this documentation, to prevent the same "mystery app" from being re-investigated from scratch in the future.

If the extra noise in Loki (or the 3 extra pods, one per node) isn't wanted, `loki-canary` can be explicitly disabled via `monitoring.lokiCanary.enabled: false` in the chart's values — not done here, since it's a useful self-verification tool for the logging stack itself.

#### Final validation
```
# Re-run the same query in Grafana Explore and confirm the source's labels
# match the loki-canary-* pods (not some unknown Deployment)
{job=~".+"} |= "ppppppppp"
```

---

### 5. Lessons Learned for the Homelab

1. **Find the real source via the logging system's own labels before chasing a suspicion** — a direct query (`{job=~".+"} |= "<text>"`) in Loki points exactly to the `namespace`/`pod`/`container`/`job` the line came from, avoiding investigating in the wrong direction (here, the initial Pi-hole suspicion would never have been confirmed, because it wasn't Pi-hole).
2. **Helm charts install auxiliary sub-components by default, even with no explicit configuration** — `loki-canary` never showed up in this repo's `infrastructure/logging/loki-values.yaml`, but it exists because it's part of the `grafana-community/loki` chart. Before treating an "unknown" presence in the cluster as a bug or an external app, it's worth checking whether it's a default sub-component of something already installed.
3. **`loki-canary` is a component name worth recognizing in the Loki ecosystem** — it writes synthetic entries on purpose (with no semantic meaning) to validate the log ingestion pipeline itself; it's not a user app nor a "real" log source.
