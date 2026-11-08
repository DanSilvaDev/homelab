# Known Issue: "Failed to Load Indexers from API" no Prowlarr / "Failed to Load Indexers from API" in Prowlarr

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Diagnóstico pendente.
**Componente(s)**: Prowlarr, apps `*arr` (media stack).
**Identificado em**: 2026-11-03.
**Relacionado**: [arr_stack_general_instability.md](arr_stack_general_instability.md) (possível relação).

Erro "Failed to Load Indexers from API" em **todos** os indexers configurados, aparecendo tanto na página de Indexers do Prowlarr quanto na dos apps `*arr` que sincronizam com ele.

O aumento de `limits.cpu` (aplicado por outro motivo na Fase 5 do media stack) não resolveu. O sintoma afetar todos os indexers de uma vez sugere problema de conectividade de saída do pod do Prowlarr (DNS ou rota de internet), não CPU nem indexer específico.

### Diagnóstico sugerido (ainda não executado)

- Status/eventos do pod: `kubectl describe pod` (OOMKilled? outro evento relevante?).
- Logs do Prowlarr.
- Teste de conectividade bruta (`wget` por IP) e de DNS (`nslookup`) de dentro do pod.
- Uso real de CPU: `kubectl top pod`.

---

## English

**Status**: 🔴 Diagnosis pending.
**Component(s)**: Prowlarr, `*arr` apps (media stack).
**Identified on**: 2026-11-03.
**Related**: [arr_stack_general_instability.md](arr_stack_general_instability.md) (possibly related).

"Failed to Load Indexers from API" error on **every** configured indexer, showing up both on Prowlarr's Indexers page and on the `*arr` apps that sync with it.

Increasing `limits.cpu` (applied for a different reason during media stack Phase 5) didn't fix it. The symptom hitting every indexer at once suggests an outbound connectivity problem on the Prowlarr pod (DNS or internet route), not CPU or a specific indexer.

### Suggested diagnostics (not yet run)

- Pod status/events: `kubectl describe pod` (OOMKilled? another relevant event?).
- Prowlarr logs.
- Raw connectivity test (`wget` by IP) and DNS test (`nslookup`) from inside the pod.
- Actual CPU usage: `kubectl top pod`.
