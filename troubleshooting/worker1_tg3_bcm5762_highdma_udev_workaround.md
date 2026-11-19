# Troubleshooting: Oscilação de rede recorrente no worker1 (Broadcom tg3/BCM5762) — workaround `highdma off` via udev / Recurring network flakiness on worker1 (Broadcom tg3/BCM5762) — `highdma off` workaround via udev

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

> ⚠️ **Status: Workaround aplicado, sobrevive a reboot — efetividade de longo prazo ainda não confirmada.** A causa exata do bug no driver/firmware não é conhecida (é um problema upstream de longa data, sem fix oficial), mas um workaround comunitário específico pra esse chip foi aplicado com sucesso técnico (persiste após reboot). Falta confirmar, ao longo do tempo, se a frequência de oscilação realmente diminuiu.

A placa de rede onboard do `worker1` (`eno1`, chip Broadcom `tg3`/BCM5762, já identificado em [`troubleshooting/worker1_eno1_nic_link_down.md`](worker1_eno1_nic_link_down.md)) passou a oscilar com frequência maior que a de um incidente isolado, levantando a suspeita de necessidade de substituição de hardware. Pesquisa identificou um bug conhecido, de longa data, específico desse chip em hardware HP EliteDesk — com um workaround comunitário (`ethtool -K highdma off`) aplicado via regra `udev`.

---

### 1. O Cenário Inicial e Sintomas

Depois do incidente isolado já documentado (link down, resolvido por reboot completo, causa raiz não confirmada), o `worker1` passou a oscilar com frequência bem maior do que antes, levantando a suspeita de que a placa de rede precisaria ser fisicamente substituída.

#### Sintomas do Erro:
* Oscilações de rede recorrentes no `worker1`, mais frequentes que o incidente isolado anterior.
* Mesmo chip já identificado antes: `ethtool -i eno1` → driver `tg3`, chip Broadcom BCM5762.
* Modelo exato do hardware confirmado via `dmidecode`: **HP EliteDesk 705 G3 Desktop Mini** (`system-product-name`), chassi `chassis-type` = `Space-saving`.

---

### 2. Processo de Investigação & Descoberta

Pesquisa pelo PCI ID exato do chip (`14e4:1687`, Broadcom BCM5762) revelou que esse problema é **conhecido e documentado há anos, especificamente em hardware HP EliteDesk 705** rodando o driver `tg3` no Linux — a placa desconecta sob carga de rede e não se recupera sozinha, só com reboot completo (bate exatamente com o padrão já visto no incidente anterior). Sem fix oficial upstream confirmado ([Launchpad #1447664](https://bugs.launchpad.net/bugs/1447664), [Launchpad #1689383](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1689383)), mas com um workaround comunitário relatado: desabilitar `highdma` na interface via `ethtool`, tornado permanente com uma regra `udev` que detecta o dispositivo pelo vendor/device ID do chip e reaplica o ajuste toda vez que a interface sobe.

Em paralelo, `dmidecode` confirmou que o `worker1` é um **HP EliteDesk 705 G3 Desktop Mini** — essa variante de chassi **não tem nenhum slot PCIe de expansão** interno (só conectores M.2 2230 pra Wi-Fi e M.2 2280 pra SSD), o que descarta de cara a opção de substituir por uma placa PCIe; se o workaround não for suficiente, o plano B é um adaptador USB 3.0 → Gigabit (chipset ASIX AX88179 ou Realtek RTL8153, ambos com driver nativo no kernel Linux), não uma placa interna.

#### Causa Raiz: bug de longa data no driver/firmware `tg3` pra esse chip específico (BCM5762) em hardware HP EliteDesk, sem fix oficial confirmado
Múltiplos relatos desde 2015 (Launchpad) documentam o mesmo sintoma nesse chip/hardware, sem um fix definitivo identificado upstream — o status da issue é inconsistente entre páginas do próprio Launchpad ("Fix Released" em algumas, "Triaged" em outras). O workaround de desabilitar `highdma` tem relatos mistos de sucesso na comunidade (funcionou pra alguns, não pra outros) — não é uma correção garantida, só a mitigação mais concreta disponível.

**Primeira tentativa de aplicar o workaround falhou silenciosamente**: a regra `udev` inicial usava `ATTR{vendor}`/`ATTR{device}` (singular), que só verifica atributos do **próprio** dispositivo que disparou o evento (a interface de rede). Mas `vendor`/`device` são atributos do **dispositivo PCI pai** (a placa em si), não da interface de rede — pra alcançar isso, a regra precisa usar `ATTRS{}` (plural, com S), que sobe a árvore de dispositivos até achar um pai com esse atributo. Com `ATTR{}` singular, a regra nunca casava com nada — sem nenhum erro visível, só falhando silenciosamente a cada boot.

**Ponto-chave**: uma regra `udev` que não casa não gera erro nenhum — ela só nunca executa. `udevadm test /sys/class/net/<interface>` é a forma de confirmar, **antes** de depender de um reboot real, se a regra realmente vai disparar.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirmar chipset/driver da placa onboard (já identificado em incidente anterior)
ethtool -i eno1

# Confirmar modelo exato e form factor do chassi (decide se cabe placa PCIe ou só USB)
sudo dmidecode -s system-product-name
sudo dmidecode -s chassis-type

# Testar o workaround ao vivo, antes de tornar permanente
sudo ethtool -K eno1 highdma off
sudo ethtool -k eno1 | grep highdma

# Validar se a regra udev realmente casa com o dispositivo, sem precisar reiniciar
sudo udevadm test /sys/class/net/eno1 2>&1 | grep -i highdma
```

---

### 4. A Correção Definitiva (ou Workaround)

Regra `udev` que reaplica `highdma off` na interface sempre que ela é adicionada (ex: no boot):

```bash
sudo tee /etc/udev/rules.d/80-tg3-fix.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="net", DRIVERS=="tg3", ATTRS{vendor}=="0x14e4", ATTRS{device}=="0x1687", RUN+="/sbin/ethtool -K %k highdma off"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=net --action=add
ethtool -k eno1 | grep highdma
```

A primeira versão dessa regra usava `ATTR{vendor}`/`ATTR{device}` (singular) e nunca disparava — corrigida pra `ATTRS{vendor}`/`ATTRS{device}` (plural, sobe até o dispositivo PCI pai).

#### Validação final
```bash
sudo reboot
# depois do boot:
ethtool -k eno1 | grep highdma   # confirmado: highdma: off, sobreviveu ao reboot
```

A regra confirmadamente sobrevive a um reboot completo. **Ainda não confirmado**: se isso de fato reduz/elimina a frequência de oscilação observada — requer acompanhamento contínuo ao longo dos próximos dias/semanas antes de considerar o problema resolvido.

---

### 5. Lições Aprendidas para o Homelab

1. **`ATTR{}` (singular) e `ATTRS{}` (plural) em regras `udev` não são intercambiáveis** — singular só olha atributos do dispositivo que disparou o evento; plural sobe a árvore de dispositivos até achar um pai com aquele atributo. `vendor`/`device` de uma interface de rede ficam no dispositivo PCI pai, nunca no próprio dispositivo de rede — por isso sempre precisam de `ATTRS{}`.
2. **Uma regra `udev` que não casa não gera nenhum erro visível** — ela simplesmente nunca executa. Sempre validar com `udevadm test /sys/class/net/<interface>` antes de confiar num reboot real pra provar que a regra funciona; o ciclo de feedback é muito mais rápido.
3. **O PCI ID exato do chip (`vendor:device`) é uma chave de busca mais eficaz que o nome genérico do driver** — pesquisar por `14e4:1687` levou direto a relatos específicos de hardware HP EliteDesk 705, em vez de resultados genéricos sobre bugs do `tg3` em qualquer hardware.
4. **Checar as restrições físicas do chassi (`dmidecode`) antes de planejar uma troca de hardware** — modelos "Desktop Mini"/USDT da HP não têm slot PCIe de expansão nenhum, só conectores M.2 (Wi-Fi/SSD); um plano de troca de NIC nesse tipo de máquina precisa ser via USB, não placa interna.
5. **Um workaround comunitário pra um bug upstream não resolvido ainda vale a pena tentar**, mesmo com relatos mistos de sucesso — mas, justamente por isso, a aplicação bem-sucedida da regra (sobreviver ao reboot) não deve ser confundida com confirmação de que o problema real (oscilação) foi resolvido; exige observação continuada.

---

## English

> ⚠️ **Status: Workaround applied, survives reboot — long-term effectiveness not yet confirmed.** The exact root cause of the driver/firmware bug is unknown (it's a long-standing upstream issue with no official fix), but a community workaround specific to this chip was applied with technical success (it persists after a reboot). Whether the oscillation frequency actually decreased still needs to be confirmed over time.

The onboard NIC on `worker1` (`eno1`, Broadcom `tg3`/BCM5762 chip, already identified in [`troubleshooting/worker1_eno1_nic_link_down.md`](worker1_eno1_nic_link_down.md)) started flaking out more often than a single isolated incident, raising suspicion that the card might need physical replacement. Research identified a known, long-standing bug specific to this chip on HP EliteDesk hardware — with a community workaround (`ethtool -K highdma off`) applied via a `udev` rule.

---

### 1. Initial Scenario and Symptoms

After the previously documented isolated incident (link down, fixed by a full reboot, root cause unconfirmed), `worker1` started flaking out noticeably more often than before, raising the suspicion that the network card might need physical replacement.

#### Error Symptoms:
* Recurring network flakiness on `worker1`, more frequent than the earlier isolated incident.
* Same chip already identified before: `ethtool -i eno1` → `tg3` driver, Broadcom BCM5762 chip.
* Exact hardware model confirmed via `dmidecode`: **HP EliteDesk 705 G3 Desktop Mini** (`system-product-name`), `chassis-type` = `Space-saving`.

---

### 2. Investigation & Discovery Process

Researching the chip's exact PCI ID (`14e4:1687`, Broadcom BCM5762) revealed this is a **known issue documented for years, specifically on HP EliteDesk 705 hardware** running the `tg3` driver on Linux — the card disconnects under network load and doesn't recover on its own, only via a full reboot (matching exactly the pattern already seen in the earlier incident). No confirmed official upstream fix exists ([Launchpad #1447664](https://bugs.launchpad.net/bugs/1447664), [Launchpad #1689383](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1689383)), but there's a reported community workaround: disabling `highdma` on the interface via `ethtool`, made permanent with a `udev` rule that detects the device by the chip's vendor/device ID and reapplies the setting every time the interface comes up.

In parallel, `dmidecode` confirmed `worker1` is an **HP EliteDesk 705 G3 Desktop Mini** — this chassis variant has **no internal PCIe expansion slot at all** (only M.2 2230 for WiFi and M.2 2280 for SSD), which rules out a PCIe replacement card outright; if the workaround isn't enough, the fallback plan is a USB 3.0 → Gigabit adapter (ASIX AX88179 or Realtek RTL8153 chipset, both with native Linux kernel drivers), not an internal card.

#### Root Cause: long-standing bug in the `tg3` driver/firmware for this specific chip (BCM5762) on HP EliteDesk hardware, no confirmed official fix
Multiple reports since 2015 (Launchpad) document the same symptom on this chip/hardware, with no definitive upstream fix identified — the issue's status is inconsistent across Launchpad's own pages ("Fix Released" on some, "Triaged" on others). The `highdma`-off workaround has mixed success reports in the community (worked for some, not for others) — it's not a guaranteed fix, just the most concrete mitigation available.

**The first attempt to apply the workaround failed silently**: the initial `udev` rule used `ATTR{vendor}`/`ATTR{device}` (singular), which only checks attributes of the device that **triggered** the event (the network interface itself). But `vendor`/`device` are attributes of the **parent PCI device** (the card itself), not the network interface — reaching those requires `ATTRS{}` (plural, with an S), which walks up the device tree until it finds a parent with that attribute. With singular `ATTR{}`, the rule never matched anything — with no visible error, just silently failing on every boot.

**Key point**: a `udev` rule that doesn't match produces no error at all — it simply never runs. `udevadm test /sys/class/net/<interface>` is how to confirm, **before** relying on an actual reboot, whether a rule will actually fire.

---

### 3. Commands Used for Diagnosis

```bash
# Confirm the onboard card's chipset/driver (already identified in the earlier incident)
ethtool -i eno1

# Confirm the exact hardware model and chassis form factor (decides PCIe card vs. USB-only)
sudo dmidecode -s system-product-name
sudo dmidecode -s chassis-type

# Test the workaround live, before making it permanent
sudo ethtool -K eno1 highdma off
sudo ethtool -k eno1 | grep highdma

# Validate that the udev rule actually matches the device, without needing a reboot
sudo udevadm test /sys/class/net/eno1 2>&1 | grep -i highdma
```

---

### 4. The Definitive Fix (or Workaround)

A `udev` rule that reapplies `highdma off` on the interface every time it's added (e.g., on boot):

```bash
sudo tee /etc/udev/rules.d/80-tg3-fix.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="net", DRIVERS=="tg3", ATTRS{vendor}=="0x14e4", ATTRS{device}=="0x1687", RUN+="/sbin/ethtool -K %k highdma off"
EOF

sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=net --action=add
ethtool -k eno1 | grep highdma
```

The first version of this rule used `ATTR{vendor}`/`ATTR{device}` (singular) and never fired — fixed to `ATTRS{vendor}`/`ATTRS{device}` (plural, walks up to the parent PCI device).

#### Final validation
```bash
sudo reboot
# after boot:
ethtool -k eno1 | grep highdma   # confirmed: highdma: off, survived the reboot
```

The rule confirmedly survives a full reboot. **Still not confirmed**: whether this actually reduces/eliminates the observed oscillation frequency — requires continued monitoring over the following days/weeks before considering the problem resolved.

---

### 5. Lessons Learned for the Homelab

1. **`ATTR{}` (singular) and `ATTRS{}` (plural) in `udev` rules are not interchangeable** — singular only checks attributes of the device that triggered the event; plural walks up the device tree until it finds a parent with that attribute. A network interface's `vendor`/`device` live on the parent PCI device, never on the network device itself — which is why they always need `ATTRS{}`.
2. **A `udev` rule that doesn't match produces no visible error at all** — it simply never runs. Always validate with `udevadm test /sys/class/net/<interface>` before trusting an actual reboot to prove a rule works; the feedback loop is far faster.
3. **A chip's exact PCI ID (`vendor:device`) is a much more effective search key than the generic driver name** — searching for `14e4:1687` led straight to reports specific to HP EliteDesk 705 hardware, instead of generic results about `tg3` bugs on arbitrary hardware.
4. **Check the chassis's physical constraints (`dmidecode`) before planning a hardware swap** — HP "Desktop Mini"/USDT-class machines have no PCIe expansion slot at all, only M.2 connectors (WiFi/SSD); a NIC replacement plan on that kind of machine has to go through USB, not an internal card.
5. **A community workaround for an unresolved upstream bug is still worth trying**, even with mixed success reports — but precisely because of that, successfully applying the rule (surviving a reboot) should not be mistaken for confirmation that the actual problem (flakiness) is fixed; it requires continued observation.
