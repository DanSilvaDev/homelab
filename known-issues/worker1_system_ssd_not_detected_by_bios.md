# Known Issue: SSD do sistema do worker1 não é reconhecido pela placa-mãe / worker1 system SSD not detected by the motherboard

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: 🔴 Em investigação — causa raiz não confirmada; worker1 fora do ar
**Componente(s)**: worker1 (HP EliteDesk 705 G3 Desktop Mini), SSD do sistema
**Identificado em**: 2026-11-24
**Relacionado**: [worker1_tg3_bcm5762_highdma_udev_workaround.md](../troubleshooting/worker1_tg3_bcm5762_highdma_udev_workaround.md) (motivo da troca de rede que antecedeu o reboot) | [worker1_unexpected_poweroff_investigation.md](../troubleshooting/worker1_unexpected_poweroff_investigation.md)

Após um reboot, o worker1 não inicia: o SSD do sistema **não aparece para a placa-mãe**. O SMART tinha sido verificado alguns dias antes, sem nenhum problema apontado.

Contexto imediatamente anterior (mesmo dia):
- O worker1 já estava com `cordon` + `drain` feitos, para a troca da placa de rede onboard (`eno1`, Broadcom `tg3`/BCM5762, instável) por um adaptador USB 3.0 → Gigabit Realtek RTL8153 (`0bda:8153`, driver `r8152`, interface `enx00e04c1a7e38`, MAC `00:e0:4c:1a:7e:38`) — o "plano B" do troubleshooting do `tg3`.
- O roteador ignorou a reserva DHCP do `192.168.3.102` para o MAC novo (o adaptador recebia `.86` mesmo com `dhcp-identifier: mac`, liberação/renovação do lease e reboot do roteador; `SERVER_ADDRESS` do lease confirmado como sendo o roteador). Contornado com **IP fixo `192.168.3.102/24` no netplan** (`/etc/netplan/50-cloud-init.yaml`), que funcionou ao ser aplicado com `netplan apply`.
- O reboot para validar essa configuração foi o boot em que o SSD deixou de ser reconhecido. A troca de rede não foi validada após reboot (`ethtool`, Wake-on-LAN, node `Ready` com `INTERNAL-IP` `.102`).

Impacto no cluster: worker1 `NotReady`; workloads já tinham sido movidos para worker2/worker3 pelo `drain`. O node continua `cordon`.

### Hipóteses / o que já foi tentado

- **Falha súbita do SSD** — SMART limpo dias antes não exclui isso: falhas de controlador em SSD costumam acontecer sem nenhum aviso prévio no SMART. Não confirmado.
- **Mau contato / encaixe** do SSD no slot ou conector — não verificado.
- **Firmware do SSD travado** — alguns SSDs deixam de ser detectados até um ciclo completo de energia (fonte desconectada), não só um reboot. Não testado.
- **Interferência do adaptador USB novo no POST** — improvável (o sintoma é o disco ausente, não ordem de boot), mas é a única mudança de hardware recente e é barato descartar. Não testado.
- Tipo exato do SSD (M.2 NVMe, M.2 SATA ou SATA 2.5") não está registrado neste repositório.

### Diagnóstico sugerido (ainda não executado)

1. Ciclo completo de energia: desligar, **tirar o cabo de energia** por 1-2 min, ligar de novo — com o adaptador USB desconectado.
2. Conferir se o disco aparece no HP Computer Setup (`F10` → Storage → Device Configuration) e, se aparecer, rodar o teste de storage do HP PC Hardware Diagnostics UEFI (`F2`).
3. Abrir o gabinete, reencaixar o SSD e testar de novo.
4. Testar o SSD em outra máquina ou num case/adaptador USB, para separar **SSD com defeito** de **slot/placa-mãe com defeito**.

Se o SSD tiver que ser trocado: reinstalar o Ubuntu e reingressar no k3s (`kubectl delete node worker1` antes do novo join). Reaplicar o que estava só no disco: netplan com IP fixo no adaptador USB (+ `network: {config: disabled}` do cloud-init), regra udev `80-tg3-fix.rules` (se a `eno1` ainda for usada) e `HandlePowerKey=ignore` no `logind.conf`.

---

## English

**Status**: 🔴 Under investigation — root cause not confirmed; worker1 down
**Component(s)**: worker1 (HP EliteDesk 705 G3 Desktop Mini), system SSD
**Identified on**: 2026-11-24
**Related**: [worker1_tg3_bcm5762_highdma_udev_workaround.md](../troubleshooting/worker1_tg3_bcm5762_highdma_udev_workaround.md) (reason for the network change that preceded the reboot) | [worker1_unexpected_poweroff_investigation.md](../troubleshooting/worker1_unexpected_poweroff_investigation.md)

After a reboot, worker1 doesn't boot: the system SSD **is not visible to the motherboard**. SMART had been checked a few days earlier, with no problems reported.

Immediately preceding context (same day):
- worker1 had already been `cordon`ed + `drain`ed to replace the onboard NIC (`eno1`, Broadcom `tg3`/BCM5762, unstable) with a USB 3.0 → Gigabit Realtek RTL8153 adapter (`0bda:8153`, `r8152` driver, interface `enx00e04c1a7e38`, MAC `00:e0:4c:1a:7e:38`) — the "plan B" from the `tg3` troubleshooting doc.
- The router ignored the DHCP reservation of `192.168.3.102` for the new MAC (the adapter kept getting `.86` despite `dhcp-identifier: mac`, releasing/renewing the lease and rebooting the router; the lease's `SERVER_ADDRESS` was confirmed to be the router). Worked around with a **static IP `192.168.3.102/24` in netplan** (`/etc/netplan/50-cloud-init.yaml`), which worked when applied with `netplan apply`.
- The reboot to validate that configuration was the boot in which the SSD stopped being detected. The network change was not validated after a reboot (`ethtool`, Wake-on-LAN, node `Ready` with `INTERNAL-IP` `.102`).

Cluster impact: worker1 `NotReady`; workloads had already been moved to worker2/worker3 by the `drain`. The node remains `cordon`ed.

### Hypotheses / what's already been tried

- **Sudden SSD failure** — a clean SMART check days earlier doesn't rule this out: SSD controller failures often happen with no prior SMART warning. Not confirmed.
- **Loose contact / seating** of the SSD in its slot or connector — not checked.
- **Hung SSD firmware** — some SSDs stop being detected until a full power cycle (power cable unplugged), not just a reboot. Not tested.
- **Interference from the new USB adapter during POST** — unlikely (the symptom is a missing disk, not boot order), but it's the only recent hardware change and cheap to rule out. Not tested.
- The exact SSD type (M.2 NVMe, M.2 SATA or 2.5" SATA) isn't recorded in this repository.

### Suggested diagnostics (not yet run)

1. Full power cycle: shut down, **unplug the power cable** for 1-2 min, power back on — with the USB adapter disconnected.
2. Check whether the disk shows up in HP Computer Setup (`F10` → Storage → Device Configuration) and, if it does, run the storage test in HP PC Hardware Diagnostics UEFI (`F2`).
3. Open the chassis, reseat the SSD and test again.
4. Test the SSD in another machine or in a USB enclosure/adapter, to separate a **faulty SSD** from a **faulty slot/motherboard**.

If the SSD has to be replaced: reinstall Ubuntu and rejoin k3s (`kubectl delete node worker1` before the new join). Reapply what lived only on the disk: netplan with the static IP on the USB adapter (+ cloud-init `network: {config: disabled}`), the `80-tg3-fix.rules` udev rule (if `eno1` is still used) and `HandlePowerKey=ignore` in `logind.conf`.
