# Troubleshooting: worker1 ficou `NotReady`/inacessível sem nenhum log de erro — investigação completa até a causa real / worker1 became `NotReady`/unreachable with no error logs — full investigation down to the real cause

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Documenta a investigação de ponta a ponta de um node (worker1, servidor físico dedicado) que ficou `NotReady` e totalmente inacessível (sem ping, sem SSH, fisicamente desligado) — cobrindo o processo de descarte de hipóteses (OOM, job agendado, falha de hardware de rede) até a causa real confirmada.

---

### 1. O Cenário Inicial e Sintomas

Durante uma sessão de trabalho normal no cluster, o worker1 parou de responder. `kubectl get nodes` mostrava `NotReady`, e a máquina não respondia nem a `ping` nem a SSH — indicando um problema fora do software do Kubernetes, em alguma camada mais baixa.

#### Sintomas do Erro:
* `kubectl get nodes` mostrava `worker1` como `NotReady`.
* `ping` e `ssh` para o IP do worker1 (`192.168.3.102`) sem resposta nenhuma.
* Fisicamente, a máquina estava com os LEDs apagados (parecia desligada).
* Pods que rodavam no worker1 ficaram presos (ex: `loki-0`, StatefulSet de réplica única, preso em `Terminating` por horas).
* Nenhum alerta do Alertmanager chegou especificamente avisando sobre o node antes da investigação manual.

---

### 2. Processo de Investigação & Descoberta

A investigação seguiu uma cadeia de hipóteses, cada uma descartada ou confirmada com evidência antes de seguir pra próxima — importante porque a primeira impressão ("parece uma queda abrupta") acabou se mostrando errada em mais de um ponto.

**Hipótese 1 — Queda abrupta de energia (kernel panic, PSU falhando)**: ao ligar a máquina de novo e rodar `journalctl -b -1` (log do boot anterior), a sequência encontrada era uma **sequência completamente normal de desligamento via systemd** (`systemd-poweroff.service`, `Reached target poweroff.target - System Power Off`) — não um corte abrupto. Hipótese descartada.

**Causa do atraso aparente**: durante esse desligamento "normal", vários containers com volume NFS montado (`loki-canary`, `alertmanager`, `loki`) travaram tentando parar, porque o **TrueNAS (servidor NFS, `192.168.3.103`) não estava respondendo** nesse exato momento (`nfs: server 192.168.3.103 not responding, still trying`). Isso atrasou o desligamento em ~3,5 minutos (containers precisaram ser mortos com `SIGKILL` por timeout, e até a sincronização final de disco travou e precisou ser forçada) — o que, visto de fora, parecia uma trava/queda súbita.

**Hipótese 2 — Job agendado (cron, systemd timer, `unattended-upgrades` com reboot automático)**: como o usuário relatou que isso "não era a primeira vez" acontecendo perto desse horário, era uma hipótese forte. Checagem sistemática:
* `crontab -l` (root e usuário): vazio.
* `/etc/cron.d/`, `/etc/cron.daily/`, `/etc/cron.hourly/`, `/etc/cron.weekly/`: só os jobs padrão do Ubuntu, nada suspeito.
* `systemctl list-timers --all`: nenhum dos 18 timers batia com o horário do desligamento.
* `unattended-upgrades`: `Automatic-Reboot "false"` (comentado = default, desabilitado).
* `atq`: `at` nem estava instalado.
* `/var/log/apt/history.log`: nenhuma atividade de APT no dia do incidente.

Hipótese descartada — nada agendado dentro do próprio sistema explicava o horário.

**Hipótese 3 — Confirmação do gatilho real**: investigando a janela exata antes do `shutdown.target` ser alcançado (`journalctl -b -1 --since "13:55:00" --until "13:57:30"`), apareceu a linha decisiva:
```
systemd-logind[918]: Power key pressed short.
systemd-logind[918]: Powering off...
systemd-logind[918]: System is powering down.
```
O botão físico de power foi pressionado. Esse era o gatilho real do desligamento.

**Hipótese 4 — Botão com defeito / mau contato**: antes de confirmar que foi uma ação humana deliberada, cogitou-se um botão fisicamente com problema. Essa dúvida foi resolvida diretamente pelo usuário: ele **pressionou o botão pessoalmente**, acreditando que a máquina já estava desligada (LEDs apagados, sem resposta de rede — ver próximo ponto). Precisou segurar o botão depois porque o desligamento gracioso ainda estava preso no timeout do NFS (não tinha terminado de verdade). Isso explica tanto o toque curto registrado quanto a percepção de "precisei forçar".

**Hipótese 5 (paralela) — Falha de hardware de rede**: antes do toque no botão (`13:55`-`13:57`), os logs mostravam uma sequência de erros do `k3s`:
```
error="dial tcp 192.168.3.101:6443: connect: network is unreachable"
"failed to validate nodeIP: node IP: \"192.168.3.102\" not found in the host's network interfaces"
```
Isso indicava que a própria interface de rede do worker1 tinha perdido o IP configurado por um curto período — não um problema de "o control-plane não respondeu", mas "o worker1 não tinha rota pra lugar nenhum". Suspeita de placa de rede degradando. Verificação com dados reais:
* `ip -s link show`: 0 erros, 0 colisões na interface `eno1`.
* `ethtool -S eno1` (driver `tg3`, Broadcom): todos os contadores de erro zerados (`rx_fcs_errors`, `rx_align_errors`, `tx_errors`, etc.).
* Busca no histórico completo do journal (não só o último boot) por eventos de link: só achou o evento de hoje, negociação limpa em `1000 Mbps, full duplex` — nenhum padrão de "Link is Down"/"Link is Up" repetido.

A evidência **não sustenta** uma placa de rede falhando. A causa desse blip específico de ~2 minutos ficou **sem explicação definitiva** — o Pi-hole (que rodava no próprio worker1, via node affinity/PV local) não serviu de testemunha independente, porque ele mesmo caiu junto com o node, só registrando uma interrupção abrupta sem detalhe. Hipótese mais provável, mas não confirmada: um soluço pontual de DHCP/switch, não relacionado à placa do worker1.

#### Causa Raiz: desligamento manual (botão de power), motivado por uma leitura (razoável, mas equivocada) de que a máquina já estava morta
A sequência completa foi: (1) um blip de rede não totalmente explicado deixou o worker1 temporariamente sem IP válido, causando perda de conectividade visível de fora (sem ping/SSH); (2) o usuário, vendo os LEDs apagados e nenhuma resposta, concluiu que a máquina já estava desligada e apertou o botão de power pra religá-la; (3) esse toque, na verdade, iniciou um desligamento gracioso normal (a máquina ainda estava ligada); (4) esse desligamento ficou preso ~3,5 minutos por causa do TrueNAS/NFS não respondendo, reforçando a impressão de que algo estava travado; (5) o usuário segurou o botão, forçando o desligamento de vez.

**Ponto-chave**: um desligamento "gracioso" (via `systemd-poweroff.service`) pode levar vários minutos e parecer, de fora, uma trava/queda súbita — principalmente se algum volume de rede (NFS) estiver indisponível no momento, travando a finalização dos containers. Não assumir "não respondeu rápido = crash" sem antes olhar a sequência completa do `journalctl -b -1`.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Status do node e rede básica
kubectl get nodes -o wide
ping -c 4 192.168.3.102
ssh <usuario>@192.168.3.102

# Log do boot anterior (depois de religar a máquina) - primeira leitura ampla
journalctl -b -1 --no-pager | tail -150

# Isolando a janela exata do início do shutdown.target
sudo journalctl -b -1 --no-pager --since "13:55:00" --until "13:57:30"

# Log específico do componente que trata botão de power / pedidos de desligamento
sudo journalctl -b -1 -u systemd-logind --no-pager | tail -30

# Descartando job agendado
sudo crontab -l
crontab -l
sudo ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/
cat /etc/crontab
systemctl list-timers --all
atq
cat /etc/apt/apt.conf.d/50unattended-upgrades | grep -i -A2 -B2 reboot
grep -i "Oct 27" /var/log/apt/history.log

# Investigando o blip de rede anterior ao shutdown
ip -s link show
ethtool -S eno1
sudo journalctl -k --no-pager | grep -i -E "link is down|link is up|nic|eth.*reset" | tail -50

# Efeitos colaterais no cluster (pods que ficaram presos por causa do node morto)
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki -o wide
kubectl get pod -n monitoring loki-0 -o jsonpath='{.status.containerStatuses[*].lastState}'
kubectl get pod -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -o wide
kubectl describe pod -n monitoring prometheus-kube-prometheus-stack-prometheus-0 | tail -40
```

---

### 4. A Correção Definitiva

Duas ações, uma pra destravar o cluster e outra pra evitar que o mesmo mal-entendido aconteça de novo.

**Destravar o pod preso em `Terminating`** (seguro porque o worker1 estava confirmadamente desligado, sem risco de dois processos escrevendo ao mesmo tempo, e o storage do Loki é NFS — não preso a um node específico):
```bash
kubectl delete pod -n monitoring loki-0 --grace-period=0 --force
```

**Evitar desligamento acidental por toque curto no botão de power** (já que "parece desligado" pode ser enganoso, como foi aqui):
```bash
sudo sed -i 's/^#\?HandlePowerKey=.*/HandlePowerKey=ignore/' /etc/systemd/logind.conf
grep -q "^HandlePowerKey=ignore" /etc/systemd/logind.conf || echo "HandlePowerKey=ignore" | sudo tee -a /etc/systemd/logind.conf
sudo systemctl restart systemd-logind
```

#### Validação final
```bash
# Confirma que o Loki voltou a rodar em outro node
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki -o wide

# Confirma a nova configuração do botão de power
grep "^HandlePowerKey" /etc/systemd/logind.conf
```
Confirmado: Loki voltou a `Running` em outro node após o force-delete; `HandlePowerKey=ignore` aplicado.

---

### 5. Lições Aprendidas para o Homelab

1. **Um desligamento "gracioso" pode parecer uma queda abrupta de fora** — `journalctl -b -1` sempre deve ser a primeira fonte de verdade antes de assumir kernel panic ou perda de energia; procurar especificamente por `systemd-poweroff.service`/`shutdown.target` alcançados com sucesso versus um corte brusco de log sem nenhuma mensagem de desligamento.
2. **Um serviço de storage de rede indisponível (NFS) pode travar o desligamento de qualquer node que monte volumes dele** — isso não é exclusivo do boot/inicialização; containers presos tentando sincronizar com um NFS fora do ar podem atrasar um `shutdown` normal em minutos, criando a falsa impressão de uma trava.
3. **Descartar metodicamente job agendado (cron, systemd timers, `unattended-upgrades`) antes de suspeitar de hardware/ação externa** é rápido (poucos comandos) e evita investigar na direção errada.
4. **Antes de suspeitar de hardware de rede degradando, confirmar com contadores reais** (`ethtool -S`, `ip -s link show`) e histórico de eventos de link no kernel — um blip isolado sem erros acumulados não é evidência de placa/cabo ruim.
5. **Um pod rodando no mesmo node afetado (via node affinity/PV local, como o Pi-hole aqui) não serve de testemunha independente de um incidente** — ele é vítima do mesmo evento, não um observador externo.
6. **Um `StatefulSet` de réplica única preso em `Terminating` após o node morrer precisa de `kubectl delete --grace-period=0 --force`** pra ser recriado em outro node — especialmente seguro de fazer quando o storage é de rede (NFS), não preso ao node morto.
7. **`HandlePowerKey=ignore` no `systemd-logind` é uma proteção barata** contra desligamento acidental por toque curto no botão — vale aplicar em qualquer servidor onde "parece desligado" pode levar alguém a apertar o botão sem necessidade.

---

## English

Documents the end-to-end investigation of a node (worker1, dedicated physical server) that went `NotReady` and fully unreachable (no ping, no SSH, physically powered off) — covering the hypothesis-elimination process (OOM, scheduled job, failing network hardware) down to the confirmed real cause.

---

### 1. Initial Scenario and Symptoms

During a normal working session on the cluster, worker1 stopped responding. `kubectl get nodes` showed `NotReady`, and the machine didn't respond to `ping` or SSH — pointing to a problem below the Kubernetes software layer.

#### Error Symptoms:
* `kubectl get nodes` showed `worker1` as `NotReady`.
* `ping` and `ssh` to worker1's IP (`192.168.3.102`) got no response at all.
* Physically, the machine's LEDs were off (looked powered down).
* Pods that ran on worker1 got stuck (e.g. `loki-0`, a single-replica StatefulSet, stuck in `Terminating` for hours).
* No Alertmanager alert specifically flagged the node before the manual investigation started.

---

### 2. Investigation & Discovery Process

The investigation followed a chain of hypotheses, each ruled out or confirmed with evidence before moving to the next — important because the first impression ("looks like an abrupt crash") turned out to be wrong on more than one point.

**Hypothesis 1 — Abrupt power loss (kernel panic, failing PSU)**: after powering the machine back on and running `journalctl -b -1` (previous boot's log), the sequence found was a **completely normal systemd shutdown sequence** (`systemd-poweroff.service`, `Reached target poweroff.target - System Power Off`) — not an abrupt cutoff. Hypothesis ruled out.

**Cause of the apparent delay**: during that "normal" shutdown, several containers with NFS-mounted volumes (`loki-canary`, `alertmanager`, `loki`) hung while trying to stop, because **TrueNAS (the NFS server, `192.168.3.103`) was not responding** at that exact moment (`nfs: server 192.168.3.103 not responding, still trying`). That delayed the shutdown by ~3.5 minutes (containers had to be force-killed with `SIGKILL` on timeout, and even the final filesystem sync hung and had to be forced) — which, seen from outside, looked like a sudden freeze/crash.

**Hypothesis 2 — Scheduled job (cron, systemd timer, `unattended-upgrades` auto-reboot)**: since the user reported this "wasn't the first time" happening around this time of day, this was a strong hypothesis. Systematic check:
* `crontab -l` (root and user): empty.
* `/etc/cron.d/`, `/etc/cron.daily/`, `/etc/cron.hourly/`, `/etc/cron.weekly/`: only Ubuntu's standard jobs, nothing suspicious.
* `systemctl list-timers --all`: none of the 18 timers matched the shutdown time.
* `unattended-upgrades`: `Automatic-Reboot "false"` (commented out = default, disabled).
* `atq`: `at` wasn't even installed.
* `/var/log/apt/history.log`: no APT activity on the day of the incident.

Hypothesis ruled out — nothing scheduled inside the system itself explained the timing.

**Hypothesis 3 — Confirming the real trigger**: investigating the exact window before `shutdown.target` was reached (`journalctl -b -1 --since "13:55:00" --until "13:57:30"`), the decisive line appeared:
```
systemd-logind[918]: Power key pressed short.
systemd-logind[918]: Powering off...
systemd-logind[918]: System is powering down.
```
The physical power button had been pressed. That was the real trigger for the shutdown.

**Hypothesis 4 — Faulty/sticky power button**: before confirming it was a deliberate human action, a physically faulty button was considered. That question was resolved directly by the user: he **pressed the button himself**, believing the machine was already off (LEDs out, no network response — see next point). He had to hold the button afterward because the graceful shutdown was still stuck on the NFS timeout (hadn't actually finished yet). This explains both the logged short press and the perception of "I had to force it."

**Hypothesis 5 (parallel) — Failing network hardware**: before the button press (`13:55`-`13:57`), the logs showed a sequence of `k3s` errors:
```
error="dial tcp 192.168.3.101:6443: connect: network is unreachable"
"failed to validate nodeIP: node IP: \"192.168.3.102\" not found in the host's network interfaces"
```
This indicated worker1's own network interface had lost its configured IP for a short period — not a "the control-plane didn't respond" problem, but "worker1 had no route anywhere." Suspicion fell on a degrading NIC. Verification with real data:
* `ip -s link show`: 0 errors, 0 collisions on the `eno1` interface.
* `ethtool -S eno1` (`tg3` driver, Broadcom): every error counter at zero (`rx_fcs_errors`, `rx_align_errors`, `tx_errors`, etc.).
* Search of the full journal history (not just the last boot) for link events: only found today's event, a clean `1000 Mbps, full duplex` negotiation — no repeated "Link is Down"/"Link is Up" pattern.

The evidence **does not support** a failing NIC. The cause of that specific ~2-minute blip was left **without a definitive explanation** — Pi-hole (which ran on worker1 itself, via node affinity/a local PV) couldn't serve as an independent witness, since it went down along with the node, only logging an abrupt interruption with no detail. Most likely (but unconfirmed) explanation: an isolated DHCP/switch hiccup, unrelated to worker1's own NIC.

#### Root Cause: manual shutdown (power button), prompted by a reasonable-but-mistaken read that the machine was already dead
The full sequence was: (1) a not-fully-explained network blip left worker1 temporarily without a valid IP, causing a connectivity loss visible from outside (no ping/SSH); (2) the user, seeing the LEDs off and no response, concluded the machine was already powered down and pressed the power button to turn it back on; (3) that press actually started a normal graceful shutdown (the machine was still on); (4) that shutdown got stuck for ~3.5 minutes because of TrueNAS/NFS not responding, reinforcing the impression that something was hung; (5) the user held the button down, forcing the shutdown through.

**Key point**: a "graceful" shutdown (via `systemd-poweroff.service`) can take several minutes and look, from the outside, like a sudden freeze/crash — especially if some network volume (NFS) is unavailable at that moment, hanging container teardown. Don't assume "didn't respond quickly = crash" without first checking the full `journalctl -b -1` sequence.

---

### 3. Commands Used for Diagnosis

```bash
# Node status and basic network reachability
kubectl get nodes -o wide
ping -c 4 192.168.3.102
ssh <user>@192.168.3.102

# Previous boot's log (after powering the machine back on) - first broad read
journalctl -b -1 --no-pager | tail -150

# Isolating the exact window shutdown.target started being reached
sudo journalctl -b -1 --no-pager --since "13:55:00" --until "13:57:30"

# Log specific to the component handling the power button / shutdown requests
sudo journalctl -b -1 -u systemd-logind --no-pager | tail -30

# Ruling out a scheduled job
sudo crontab -l
crontab -l
sudo ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/
cat /etc/crontab
systemctl list-timers --all
atq
cat /etc/apt/apt.conf.d/50unattended-upgrades | grep -i -A2 -B2 reboot
grep -i "Oct 27" /var/log/apt/history.log

# Investigating the network blip that preceded the shutdown
ip -s link show
ethtool -S eno1
sudo journalctl -k --no-pager | grep -i -E "link is down|link is up|nic|eth.*reset" | tail -50

# Side effects on the cluster (pods stuck because of the dead node)
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki -o wide
kubectl get pod -n monitoring loki-0 -o jsonpath='{.status.containerStatuses[*].lastState}'
kubectl get pod -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -o wide
kubectl describe pod -n monitoring prometheus-kube-prometheus-stack-prometheus-0 | tail -40
```

---

### 4. The Definitive Fix

Two actions: one to unstick the cluster, another to prevent the same misunderstanding from happening again.

**Unsticking the pod stuck in `Terminating`** (safe because worker1 was confirmedly powered off, with no risk of two processes writing at once, and Loki's storage is NFS — not tied to a specific node):
```bash
kubectl delete pod -n monitoring loki-0 --grace-period=0 --force
```

**Preventing an accidental shutdown from a short power-button press** (since "looks powered off" can be misleading, as it was here):
```bash
sudo sed -i 's/^#\?HandlePowerKey=.*/HandlePowerKey=ignore/' /etc/systemd/logind.conf
grep -q "^HandlePowerKey=ignore" /etc/systemd/logind.conf || echo "HandlePowerKey=ignore" | sudo tee -a /etc/systemd/logind.conf
sudo systemctl restart systemd-logind
```

#### Final validation
```bash
# Confirms Loki came back up on another node
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki -o wide

# Confirms the new power button configuration
grep "^HandlePowerKey" /etc/systemd/logind.conf
```
Confirmed: Loki returned to `Running` on another node after the force-delete; `HandlePowerKey=ignore` applied.

---

### 5. Lessons Learned for the Homelab

1. **A "graceful" shutdown can look like an abrupt crash from the outside** — `journalctl -b -1` should always be the first source of truth before assuming a kernel panic or power loss; specifically look for `systemd-poweroff.service`/`shutdown.target` being successfully reached versus a log that just cuts off abruptly with no shutdown message at all.
2. **An unavailable network storage service (NFS) can hang the shutdown of any node mounting its volumes** — this isn't unique to boot/startup; containers stuck trying to sync with a down NFS server can delay a normal `shutdown` by minutes, creating a false impression of a hang.
3. **Methodically ruling out a scheduled job (cron, systemd timers, `unattended-upgrades`) before suspecting hardware/external action** is quick (a handful of commands) and avoids investigating in the wrong direction.
4. **Before suspecting degrading network hardware, confirm with real counters** (`ethtool -S`, `ip -s link show`) and the kernel's link-event history — an isolated blip with no accumulated errors is not evidence of a bad NIC/cable.
5. **A pod running on the same affected node (via node affinity/a local PV, like Pi-hole here) can't serve as an independent witness to an incident** — it's a victim of the same event, not an outside observer.
6. **A single-replica `StatefulSet` stuck in `Terminating` after its node dies needs `kubectl delete --grace-period=0 --force`** to get recreated on another node — especially safe to do when the storage is network-based (NFS), not tied to the dead node.
7. **`HandlePowerKey=ignore` in `systemd-logind` is a cheap safeguard** against an accidental shutdown from a short power-button press — worth applying on any server where "looks powered off" could lead someone to press the button unnecessarily.
