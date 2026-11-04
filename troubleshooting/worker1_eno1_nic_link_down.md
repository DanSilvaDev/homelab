# Troubleshooting: Placa de rede onboard (`eno1`) do worker1 sem link / Worker1 onboard NIC (`eno1`) link down

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

> ✅ **Status: RESOLVIDO (2026-11-04)** — um reboot completo do servidor restabeleceu o link normalmente. A causa raiz exata (por que o link ficou travado e por que só um reboot completo resolveu, não um reload do driver) **não foi confirmada em detalhe**, mas o sintoma foi definitivamente corrigido.

Este documento cobre a investigação de um problema de rede no `worker1` (nó worker físico dedicado do cluster k3s, ver `README.md`) — a placa de rede onboard parou de estabelecer link, mesmo após troca de cabo e de porta de switch.

---

### 1. O Cenário Inicial e Sintomas

O `worker1` parou de responder na rede (sem resposta a `ping`, inacessível via SSH/rede). Acesso físico via teclado/monitor confirmou que o sistema operacional estava de pé e o login funcionava normalmente — o problema era especificamente de rede, não uma trava geral do sistema.

#### Sintomas do Erro:
* `ping` para o `worker1` sem nenhuma resposta.
* Tirar e recolocar o cabo de rede não restabeleceu a conectividade.
* `ethtool eno1` reportando `Link detected: no`, `Speed: Unknown!`, `Duplex: Unknown! (255)` — ou seja, a interface nunca completa negociação de velocidade/duplex com o switch.
* `dmesg` registrando a mensagem padrão de driver "Link is Down" para `eno1`, sem nenhum erro mais grave associado.
* LEDs da porta de rede piscando, apesar do link nunca ser efetivamente estabelecido.
* Servidor sem uma segunda porta de rede onboard disponível para teste comparativo.

---

### 2. Processo de Investigação & Descoberta

A investigação seguiu uma eliminação sistemática por camada, da mais simples/barata até a mais profunda:

1. Confirmado via acesso físico (teclado/monitor) que o sistema operacional estava operacional — isolando o problema como especificamente de rede, não uma trava geral do host.
2. `ip link show` e `ethtool eno1` confirmaram o estado de link down a nível de interface, sem nenhuma negociação de velocidade/duplex (sintoma de camada física/PHY, não de roteamento ou DNS).
3. `dmesg -T | grep -iE "eth|link|nic|carrier|reset"` — só registrou a mensagem padrão "Link is Down" do driver, sem nenhum erro de driver/hardware mais específico.
4. Teste de cabo: troca por um cabo diferente, mantendo a mesma porta do switch — sem mudança no resultado.
5. Teste de porta de switch: troca de porta, mantendo o mesmo cabo — sem mudança.
6. Teste combinado (cabo diferente **e** porta de switch diferente ao mesmo tempo) — sem mudança. Isso descarta cabo e switch como causa.
7. Forçar velocidade e duplex fixos (`ethtool -s eno1 speed 100 duplex full autoneg off`) para descartar falha específica de autonegociação — não resolveu.
8. `ethtool -i eno1` identificou o driver (`tg3`) e o firmware da placa (`5762-v1.35`) — chip Broadcom NetXtreme BCM5762.
9. `lspci -vv` (verificação da placa no barramento PCIe) e `dmesg` filtrado por `pcie|aer` — sem nenhum erro de barramento (AER) ou indicação de perda de comunicação a nível de PCIe. A placa é enumerada normalmente e responde a comandos via `ethtool`, ou seja, a comunicação SO ↔ placa pelo barramento está saudável.
10. Servidor não possui uma segunda porta de rede onboard, o que impossibilitou um teste comparativo direto (testar se outra porta do mesmo chip/controlador consegue linkar normalmente).

#### Hipótese testada: gerenciamento de energia
Chips Broadcom `tg3` têm histórico conhecido de bugs relacionados a **EEE (Energy Efficient Ethernet, IEEE 802.3az)** causando falha permanente de negociação de link com determinados switches. Essa hipótese foi testada:

11. `ethtool --show-eee eno1` / `ethtool --set-eee eno1 eee off` — desabilitar EEE não resolveu.
12. Verificação do runtime power management do PCIe (`/sys/bus/pci/devices/<id>/power/control`, forçado para `on`) — não resolveu.
13. Reload do módulo do driver (`rmmod tg3` seguido de `modprobe tg3`) — não resolveu.

#### Causa Raiz: Estado travado na placa/firmware, resolvido por reboot completo (mecanismo exato não confirmado)
Com cabo, porta de switch, autonegociação, EEE, runtime PM do PCIe e o próprio módulo do driver (`rmmod`/`modprobe`) todos descartados como correção, e sem erros de barramento PCIe, a hipótese de **falha permanente de hardware na PHY foi descartada** pelo resultado final: um **reboot completo do servidor restabeleceu o link normalmente**. Isso prova que o hardware em si estava funcional — o problema era algum estado travado (na placa, no firmware dela, ou em alguma camada entre o kernel e o chip) que sobreviveu a um reload do módulo do driver, mas foi limpo por um power cycle completo (POST da placa-mãe reinicializando a NIC do zero).

**Ponto-chave**: o fato de `rmmod`/`modprobe` **não** terem resolvido, mas um reboot completo **ter** resolvido, é um dado importante — mostra que o estado travado estava num nível mais profundo do que o módulo do driver do Linux consegue alcançar sozinho (ex: firmware da própria placa, ou estado de inicialização que só é refeito no POST). O mecanismo exato não foi identificado, mas fica descartada a hipótese de hardware permanentemente danificado.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirmar estado da interface e negociação de link
ip link show eno1
sudo ethtool eno1

# Logs do kernel sobre a placa de rede
sudo dmesg -T | grep -iE "eth|link|nic|carrier|reset" | tail -50

# Forçar velocidade/duplex fixos (descartar falha de autonegociação)
sudo ethtool -s eno1 speed 100 duplex full autoneg off
sleep 3
sudo ethtool eno1

# Identificar driver e firmware da placa
ethtool -i eno1

# Verificar barramento PCIe e possíveis erros de hardware
lspci -vv | grep -A 20 -i ethernet
sudo dmesg -T | grep -iE "pcie|aer|eno1" | tail -80

# Verificar e desabilitar EEE (Energy Efficient Ethernet)
sudo ethtool --show-eee eno1
sudo ethtool --set-eee eno1 eee off
sleep 3
sudo ethtool eno1

# Verificar e forçar runtime power management do dispositivo PCIe
cat /sys/bus/pci/devices/$(ethtool -i eno1 | grep bus-info | awk '{print $2}')/power/control
echo on | sudo tee /sys/bus/pci/devices/$(ethtool -i eno1 | grep bus-info | awk '{print $2}')/power/control
sleep 3
sudo ethtool eno1

# Recarregar o módulo do driver
sudo ip link set eno1 down
sudo rmmod tg3
sleep 2
sudo modprobe tg3
sleep 3
sudo ethtool eno1
```

---

### 4. A Correção Definitiva (ou Workaround)

Nenhuma das intervenções de software testadas em runtime (forçar velocidade, desabilitar EEE, forçar runtime PM, recarregar o driver) resolveu o problema. A correção efetiva foi um **reboot completo do servidor** (`worker1`) — após o power cycle, a placa `eno1` voltou a estabelecer link normalmente e o `worker1` voltou a responder na rede.

```bash
sudo reboot
```

Como workaround geral pra esse tipo de sintoma (link travado em NIC Broadcom `tg3`/BCM5762 que não responde a ajustes via `ethtool` nem a reload do módulo): **tentar um reboot completo antes de assumir falha de hardware permanente** — ele resolve estados travados que um simples `rmmod`/`modprobe` não alcança.

#### Validação final
Após o reboot, o `worker1` voltou a responder a `ping` normalmente e a interface `eno1` voltou a reportar link ativo. Validação recomendada para confirmar de forma mais completa (incluindo a volta do node ao cluster k3s):
```bash
ethtool eno1          # confirmar "Link detected: yes" e Speed/Duplex reportando valores reais
kubectl get nodes -o wide   # confirmar que o worker1 voltou ao estado Ready
```

---

### 5. Lições Aprendidas para o Homelab

1. **Um reboot completo (power cycle) não é equivalente a recarregar o módulo do driver** (`rmmod`/`modprobe`) — um estado travado na placa/firmware pode sobreviver a um reload do driver via software, mas ser limpo pelo POST de um boot completo. Diante de um link que não sobe e não responde a ajustes via `ethtool` nem a reload do driver, vale testar um reboot completo **antes** de concluir falha permanente de hardware.
2. **Eliminação sistemática por camada é valiosa mesmo quando a correção final acaba sendo "simples"** (reboot) — seguir a ordem cabo → porta de switch → velocidade/duplex fixos → driver → gerenciamento de energia → barramento PCIe eliminou um volume grande de causas possíveis e documentou com confiança que **não** era cabo, switch, autonegociação, EEE ou hardware permanentemente danificado.
3. **Chips Broadcom `tg3` têm histórico conhecido de bugs relacionados a EEE** (Energy Efficient Ethernet) causando falha de negociação de link — mesmo não tendo sido a causa raiz neste caso, continua valendo testar isso cedo em qualquer problema de link nesse tipo de NIC, por ser rápido e não-destrutivo.
4. **Ausência de erros AER/PCIe não garante hardware saudável nem indica necessariamente hardware danificado** — garante só que o barramento de comunicação entre a placa e o SO está ok. Nesse caso específico, mesmo sem hardware danificado, a placa ficou com link down até o reboot.
5. **Sem uma segunda porta de rede onboard disponível, o isolamento "placa vs porta específica" fica impossível de confirmar** sem hardware adicional (placa PCIe avulsa) — um ponto a considerar para hardware de cluster crítico no futuro (preferir placas com múltiplas portas, ou manter uma placa PCIe avulsa de reserva).
6. **LED piscando na porta não é garantia de link saudável** — pode refletir apenas tentativas de negociação que nunca fecham, não necessariamente atividade de dados real.

---

## English

> ✅ **Status: RESOLVED (2026-11-04)** — a full server reboot restored the link normally. The exact root cause (why the link got stuck, and why only a full reboot fixed it, not a driver reload) **was not confirmed in detail**, but the symptom was definitively fixed.

This document covers the investigation of a network issue on `worker1` (dedicated physical worker node of the k3s cluster, see `README.md`) — the onboard network card stopped establishing link, even after swapping both the cable and the switch port.

---

### 1. Initial Scenario and Symptoms

`worker1` stopped responding on the network (no response to `ping`, unreachable via SSH/network). Physical access via keyboard/monitor confirmed the operating system was up and login worked normally — the problem was specifically network-related, not a general system hang.

#### Error Symptoms:
* `ping` to `worker1` got no response at all.
* Unplugging and reseating the network cable did not restore connectivity.
* `ethtool eno1` reported `Link detected: no`, `Speed: Unknown!`, `Duplex: Unknown! (255)` — i.e. the interface never completes speed/duplex negotiation with the switch.
* `dmesg` logged the driver's standard "Link is Down" message for `eno1`, with no more serious associated error.
* The network port's LEDs were blinking, despite the link never actually being established.
* The server has no second onboard network port available for comparative testing.

---

### 2. Investigation & Discovery Process

The investigation followed a systematic layer-by-layer elimination, from the simplest/cheapest to the deepest:

1. Confirmed via physical access (keyboard/monitor) that the operating system was operational — isolating the problem as specifically network-related, not a general host hang.
2. `ip link show` and `ethtool eno1` confirmed the link-down state at the interface level, with no speed/duplex negotiation at all (a physical/PHY-layer symptom, not routing or DNS).
3. `dmesg -T | grep -iE "eth|link|nic|carrier|reset"` — only logged the driver's standard "Link is Down" message, with no more specific driver/hardware error.
4. Cable test: swapped for a different cable, keeping the same switch port — no change in result.
5. Switch port test: swapped port, keeping the same cable — no change.
6. Combined test (different cable **and** different switch port at the same time) — no change. This rules out the cable and the switch as the cause.
7. Forced fixed speed and duplex (`ethtool -s eno1 speed 100 duplex full autoneg off`) to rule out a specific autonegotiation failure — did not resolve it.
8. `ethtool -i eno1` identified the driver (`tg3`) and the card's firmware (`5762-v1.35`) — a Broadcom NetXtreme BCM5762 chip.
9. `lspci -vv` (checking the card on the PCIe bus) and `dmesg` filtered by `pcie|aer` — no bus errors (AER) or any indication of lost communication at the PCIe level. The card enumerates normally and responds to commands via `ethtool`, meaning OS ↔ card communication over the bus is healthy.
10. The server has no second onboard network port, which made a direct comparative test impossible (testing whether another port on the same chip/controller links up normally).

#### Hypothesis tested: power management
Broadcom `tg3` chips have a known history of bugs related to **EEE (Energy Efficient Ethernet, IEEE 802.3az)** causing permanent link negotiation failure with certain switches. This hypothesis was tested:

11. `ethtool --show-eee eno1` / `ethtool --set-eee eno1 eee off` — disabling EEE did not resolve it.
12. Checked PCIe runtime power management (`/sys/bus/pci/devices/<id>/power/control`, forced to `on`) — did not resolve it.
13. Reloaded the driver module (`rmmod tg3` followed by `modprobe tg3`) — did not resolve it.

#### Root Cause: Stuck card/firmware state, resolved by a full reboot (exact mechanism unconfirmed)
With the cable, switch port, autonegotiation, EEE, PCIe runtime PM, and the driver module itself (`rmmod`/`modprobe`) all ruled out as a fix, and no PCIe bus errors, the **permanent PHY hardware failure hypothesis was ruled out** by the final outcome: a **full server reboot restored the link normally**. This proves the hardware itself was functional — the problem was some stuck state (on the card, its firmware, or some layer between the kernel and the chip) that survived a driver module reload but was cleared by a full power cycle (the motherboard's POST reinitializing the NIC from scratch).

**Key point**: the fact that `rmmod`/`modprobe` did **not** fix it, but a full reboot **did**, is an important data point — it shows the stuck state lived at a level deeper than the Linux driver module alone can reach (e.g. the card's own firmware, or an initialization state that only gets redone at POST). The exact mechanism was not identified, but a permanently damaged hardware is ruled out.

---

### 3. Commands Used for Diagnosis

```bash
# Confirm interface state and link negotiation
ip link show eno1
sudo ethtool eno1

# Kernel logs about the network card
sudo dmesg -T | grep -iE "eth|link|nic|carrier|reset" | tail -50

# Force fixed speed/duplex (rule out an autonegotiation failure)
sudo ethtool -s eno1 speed 100 duplex full autoneg off
sleep 3
sudo ethtool eno1

# Identify the card's driver and firmware
ethtool -i eno1

# Check the PCIe bus and possible hardware errors
lspci -vv | grep -A 20 -i ethernet
sudo dmesg -T | grep -iE "pcie|aer|eno1" | tail -80

# Check and disable EEE (Energy Efficient Ethernet)
sudo ethtool --show-eee eno1
sudo ethtool --set-eee eno1 eee off
sleep 3
sudo ethtool eno1

# Check and force the PCIe device's runtime power management
cat /sys/bus/pci/devices/$(ethtool -i eno1 | grep bus-info | awk '{print $2}')/power/control
echo on | sudo tee /sys/bus/pci/devices/$(ethtool -i eno1 | grep bus-info | awk '{print $2}')/power/control
sleep 3
sudo ethtool eno1

# Reload the driver module
sudo ip link set eno1 down
sudo rmmod tg3
sleep 2
sudo modprobe tg3
sleep 3
sudo ethtool eno1
```

---

### 4. The Definitive Fix (or Workaround)

None of the runtime software interventions tested (forcing speed, disabling EEE, forcing runtime PM, reloading the driver) resolved the problem. The actual fix was a **full server reboot** (`worker1`) — after the power cycle, `eno1` went back to establishing link normally and `worker1` started responding on the network again.

```bash
sudo reboot
```

As a general workaround for this kind of symptom (stuck link on a Broadcom `tg3`/BCM5762 NIC that does not respond to `ethtool` tweaks or a module reload): **try a full reboot before assuming permanent hardware failure** — it clears stuck states that a plain `rmmod`/`modprobe` cannot reach.

#### Final validation
After the reboot, `worker1` started responding to `ping` normally again and the `eno1` interface reported an active link again. Recommended validation for a more complete confirmation (including the node rejoining the k3s cluster):
```bash
ethtool eno1          # confirm "Link detected: yes" and real Speed/Duplex values
kubectl get nodes -o wide   # confirm worker1 is back to Ready
```

---

### 5. Lessons Learned for the Homelab

1. **A full reboot (power cycle) is not equivalent to reloading the driver module** (`rmmod`/`modprobe`) — a stuck card/firmware state can survive a software-level driver reload but get cleared by a full boot's POST. Faced with a link that won't come up and doesn't respond to `ethtool` tweaks or a driver reload, try a full reboot **before** concluding permanent hardware failure.
2. **Systematic layer-by-layer elimination is valuable even when the eventual fix turns out to be "simple"** (a reboot) — following the order cable → switch port → fixed speed/duplex → driver → power management → PCIe bus ruled out a large set of possible causes and confidently documented that it was **not** the cable, the switch, autonegotiation, EEE, or permanently damaged hardware.
3. **Broadcom `tg3` chips have a known history of EEE-related bugs** (Energy Efficient Ethernet) causing link negotiation failure — even though it wasn't the root cause here, it's still worth testing early in any link issue on this type of NIC, since it's fast and non-destructive.
4. **Absence of AER/PCIe errors neither guarantees healthy hardware nor necessarily indicates damaged hardware** — it only guarantees that the communication bus between the card and the OS is fine. In this specific case, even with no damaged hardware, the card stayed link-down until the reboot.
5. **Without a second onboard network port available, "card vs. specific port" isolation cannot be confirmed** without extra hardware (a standalone PCIe card) — worth considering for critical cluster hardware going forward (prefer multi-port cards, or keep a spare PCIe NIC on hand).
6. **A blinking port LED is not a guarantee of a healthy link** — it may just reflect negotiation attempts that never close, not necessarily real data activity.
