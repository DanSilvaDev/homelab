# Known Issue: Pod do Jellyfin reiniciando com frequência anormal / Jellyfin pod restarting with abnormal frequency

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Causa raiz não confirmada.
**Componente(s)**: Jellyfin (media stack), GPU (worker2).
**Identificado em**: 2026-11-04 (16 restarts em 35 horas).
**Relacionado**: [arr_stack_general_instability.md](arr_stack_general_instability.md) (possível relação com contenção de recursos no node).

Pod do Jellyfin reiniciando com frequência anormal — **16 restarts em 35 horas**, observado em 2026-11-04. Causa raiz ainda não confirmada; não deve ser assumido como esperado.

### Diagnóstico sugerido (ainda não executado)

- `kubectl describe pod` — motivo do último restart (`OOMKilled`, `CrashLoopBackOff`, liveness probe falhando, etc.).
- `kubectl logs --previous` do container antes do restart.
- Se os `limits` de CPU/memória (ver tabela de recursos da Fase 3 em [media-stack/config.md](../clusters/media-stack/config.md)) estão sendo estourados durante transcodificação via GPU.

---

## English

**Status**: 🔴 Root cause not confirmed.
**Component(s)**: Jellyfin (media stack), GPU (worker2).
**Identified on**: 2026-11-04 (16 restarts in 35 hours).
**Related**: [arr_stack_general_instability.md](arr_stack_general_instability.md) (possibly related to node resource contention).

The Jellyfin pod is restarting with abnormal frequency — **16 restarts in 35 hours**, observed on 2026-11-04. Root cause not yet confirmed; should not be assumed to be expected behavior.

### Suggested diagnostics (not yet run)

- `kubectl describe pod` — reason for the last restart (`OOMKilled`, `CrashLoopBackOff`, failing liveness probe, etc.).
- `kubectl logs --previous` of the container before the restart.
- Whether the CPU/memory `limits` (see the Phase 3 resources table in [media-stack/config.md](../clusters/media-stack/config.md)) are being exceeded during GPU transcoding.
