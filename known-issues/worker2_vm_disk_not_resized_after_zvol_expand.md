# Known Issue: VM worker2 não reconhece zvol ampliado / worker2 VM doesn't see the expanded zvol

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Comandos sugeridos, ainda não executados.
**Componente(s)**: worker2 (VM), TrueNAS (zvol `vm-k3s-disk`).
**Identificado em**: não registrado explicitamente.
**Relacionado**: [TrueNAS - config.md](../clusters/truenas/config.md), [Worker2 - config.md](../clusters/worker2/config.md).

O zvol `vm-k3s-disk` (disco da VM worker2) foi ampliado de 40GiB para 50GiB no TrueNAS, mas o Linux dentro da VM ainda não reconhece o espaço novo — a partição/filesystem não cresceram.

### Comandos sugeridos (ainda não executados)

```bash
df -h /
lsblk
sudo growpart /dev/vda 1
sudo resize2fs /dev/vda1
```

⚠️ Confirmar os nomes de device reais (`lsblk`) antes de rodar `growpart`/`resize2fs` — os exemplos acima assumem `/dev/vda`/`/dev/vda1`, mas isso precisa ser validado na VM antes de executar.

---

## English

**Status**: 🔴 Commands suggested, not yet executed.
**Component(s)**: worker2 (VM), TrueNAS (`vm-k3s-disk` zvol).
**Identified on**: not explicitly recorded.
**Related**: [TrueNAS - config.md](../clusters/truenas/config.md), [Worker2 - config.md](../clusters/worker2/config.md).

The `vm-k3s-disk` zvol (worker2 VM's disk) was expanded from 40GiB to 50GiB on TrueNAS, but the Linux guest inside the VM doesn't see the new space yet — the partition/filesystem haven't grown.

### Suggested commands (not yet run)

```bash
df -h /
lsblk
sudo growpart /dev/vda 1
sudo resize2fs /dev/vda1
```

⚠️ Confirm the actual device names (`lsblk`) before running `growpart`/`resize2fs` — the examples above assume `/dev/vda`/`/dev/vda1`, but this needs to be validated on the VM before executing.
