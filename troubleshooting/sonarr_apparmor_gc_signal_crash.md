# Troubleshooting: AppArmor bloqueando o auto-sinal do GC do .NET derrubava o Sonarr / AppArmor blocking .NET's GC self-signal was crashing Sonarr

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Após o reset do `/config` do Sonarr (ver [`sonarr_reset_stale_apikey_integrations.md`](sonarr_reset_stale_apikey_integrations.md)) e a correção do problema de API key, o Sonarr continuou instável, reiniciando repetidamente de forma aparentemente aleatória, sem nenhum rastro de erro em lugar nenhum — até a causa ser rastreada ao log de auditoria do kernel do node: o AppArmor padrão do containerd bloqueando o sinal interno que o Garbage Collector do .NET usa pra suspender threads.

---

### 1. O Cenário Inicial e Sintomas

Com o Sonarr já com a API key corrigida em todos os integradores (Bazarr/Prowlarr/Seerr), o pod continuava a reiniciar internamente (o processo do Sonarr, supervisionado pelo s6, não o container) em intervalos irregulares — de poucos segundos a vários minutos — sem relação clara com nenhuma ação específica.

#### Sintomas do Erro:
* `kubectl -n media logs <pod> --timestamps` mostrava a linha `Bootstrap: Starting Sonarr` reaparecendo dezenas de vezes ao longo de horas, sem nenhuma linha de erro, exceção ou aviso imediatamente antes.
* `kubectl get pod` sempre mostrava `RestartCount: 0` e o mesmo `containerID` desde o início — o container nunca foi recriado, só o processo interno do Sonarr.
* O intervalo entre quedas variava muito: às vezes o processo morria ~2 segundos depois de terminar de subir (antes até de logar a verificação de rede), às vezes sobrevivia 5+ minutos normalmente.
* Aumentar o limite de memória de `512Mi` para `1.5Gi` não mudou o padrão — o processo caiu de novo usando apenas ~112Mi de uso real, bem longe do novo limite.
* `/sys/fs/cgroup/memory.events` sempre com `oom_kill: 0` e `oom: 0`, mesmo monitorando ao vivo durante uma queda reproduzida.
* Nenhuma `livenessProbe`/`readinessProbe`/`lifecycle hook` configurada no Deployment nem no Pod real (confirmado via `kubectl get pod -o yaml`, inclusive checando se algum webhook de admissão tinha injetado algo).

---

### 2. Processo de Investigação & Descoberta

A investigação eliminou, em ordem, cada camada óbvia: CPU e memória via métricas do Prometheus (sem pico anormal), OOM a nível de cgroup (`memory.events` zerado mesmo monitorado ao vivo durante uma reprodução do problema), probes/sidecars/webhooks do Kubernetes (spec do Pod limpo), cron/periodic scripts customizados (`crontab -l` e `/etc/periodic/*` vazios), serviços extras do s6 (`/run/service/` só com os serviços padrão) e o mecanismo de auto-update do Sonarr (desabilitado em modo Docker, não é a causa).

O ponto de virada foi perceber que o histórico de `kubectl logs` do pod era **contínuo desde a criação do container** (nunca reiniciou de verdade), e que, apesar disso, o processo do Sonarr simplesmente parava de logar e reaparecia com `Bootstrap: Starting Sonarr` sem absolutamente nenhuma linha entre os dois eventos — nem no nível `Debug`. Isso descartou qualquer exceção gerenciada (.NET sempre loga `Unhandled exception` com stack trace quando isso acontece) e apontou pra algo abaixo do código da aplicação.

Como o container não tem acesso a `dmesg` (`Operation not permitted`), a investigação precisou ir até o **log de auditoria do kernel do próprio node** via SSH:

```
sudo journalctl -k --since "16:35" --until "16:50" | grep -iE "sonarr|oom|segfault|panic|killed process|out of memory"
```

Isso revelou a causa raiz:

```
kernel: audit: type=1400 ... apparmor="DENIED" operation="signal" class="signal" profile="cri-containerd.apparmor.d" ... comm="Sonarr" requested_mask="send" denied_mask="send" signal=rtmin+3 peer="cri-containerd.apparmor.d//&unconfined"
kernel: audit: type=1400 ... apparmor="DENIED" operation="signal" ... comm="Sonarr" requested_mask="send" denied_mask="send" signal=abrt peer="cri-containerd.apparmor.d//&unconfined"
```

Os timestamps dessas negações batiam exatamente com os timestamps das quedas já documentadas.

#### Causa Raiz: AppArmor bloqueando o auto-sinal de suspensão de thread do GC do .NET
No Linux, o CoreCLR (runtime do .NET) emula a suspensão de threads que o GC precisa fazer pra um "stop-the-world" (equivalente ao `SuspendThread` do Windows) usando **sinais de tempo real POSIX** (`SIGRTMIN+N`), entregues via `tgkill` entre as próprias threads do processo. Isso acontece toda vez que o GC roda uma coleta que precisa suspender threads — a frequência depende da pressão de alocação de memória no momento, por isso o timing das quedas parecia aleatório.

O perfil padrão de AppArmor aplicado pelo containerd (`cri-containerd.apparmor.d`) nega o envio de sinal quando o alvo resolve pra um peer `unconfined` sob as regras de mediação de sinal desse perfil. Quando isso bloqueia o `SIGRTMIN+3` do GC, o CoreCLR tenta seu caminho de **fail-fast** (abortar de forma controlada em vez de continuar com estado potencialmente inconsistente), que é implementado mandando `SIGABRT` pra si mesmo — e esse sinal **também** é negado. O processo morre logo depois por um caminho que nunca passa pelo pipeline de exceção/log gerenciado (Serilog) do Sonarr, daí a ausência total de rastro na aplicação.

**Ponto-chave**: um processo que morre sem absolutamente nenhum log, sem OOM no cgroup e sem sinal visível no Kubernetes não está necessariamente crashando em código gerenciado — pode estar sendo impedido de usar um mecanismo interno do próprio runtime por uma camada de segurança do SO (AppArmor/SELinux), o que só aparece no log de auditoria do kernel do node, não em nada acessível de dentro do container.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirmar que o container nunca reiniciou de verdade (histórico contínuo
# de log, RestartCount sempre 0)
kubectl -n media get pod <pod> -o jsonpath="RestartCount: {.status.containerStatuses[0].restartCount}{'\n'}State: {.status.containerStatuses[0].state}{'\n'}"
kubectl -n media get pod <pod> -o yaml

# Listar todas as reinicializações internas do processo com timestamp
kubectl -n media logs <pod> --timestamps | grep -n "Bootstrap: Starting Sonarr"

# Monitorar memória do cgroup ao vivo durante uma reprodução do problema
kubectl -n media exec <pod> -- sh -c 'i=0; while [ $i -lt 90 ]; do date +%T; cat /sys/fs/cgroup/memory.current; cat /sys/fs/cgroup/memory.events; echo ---; sleep 1; i=$((i+1)); done'

# Descartar cron/serviços extras dentro do container
kubectl -n media exec <pod> -- crontab -l
kubectl -n media exec <pod> -- ls -la /run/service/

# dmesg não acessível de dentro do container (confirma a necessidade de ir ao node)
kubectl -n media exec <pod> -- dmesg
# dmesg: klogctl: Operation not permitted

# No node (via SSH), o log de auditoria do kernel que revelou a causa raiz
sudo journalctl -k --since "<horário da queda>" --until "<horário+alguns min>" | grep -iE "sonarr|oom|segfault|panic|killed process|out of memory"
sudo dmesg -T | grep -iE "segfault|killed process|oom|panic"
```

---

### 4. A Correção Definitiva (ou Workaround)

A correção é remover o confinamento do AppArmor especificamente pro container do Sonarr, já que ele legitimamente precisa do auto-sinal pro GC funcionar. No Kubernetes 1.30+, isso é feito via o campo estável `securityContext.appArmorProfile` (a anotação antiga `container.apparmor.security.beta.kubernetes.io/<container>: unconfined` ainda funciona, mas emite aviso de depreciação):

```yaml
containers:
  - name: sonarr
    securityContext:
      appArmorProfile:
        type: Unconfined
```

Como Radarr, Lidarr e Prowlarr são da mesma família de código (.NET/CoreCLR, mesmo mecanismo de GC), a mesma correção foi aplicada preventivamente a eles em `apps/servarr/radarr/deployment.yaml`, `apps/servarr/lidarr/deployment.yaml` e `apps/servarr/prowlarr/deployment.yaml`. O Jellyfin (também .NET/ASP.NET Core) também recebeu a correção em `apps/jellyfin/deployment.yaml` — e o resultado **confirmou** que seu problema registrado anteriormente (`known-issues/jellyfin_frequent_pod_restarts.md`, atribuído a um "bug upstream de WebSocket") era, na verdade, o mesmo bloqueio de AppArmor; esse known-issue foi removido depois dessa confirmação.

#### Validação final
```bash
kubectl apply -f apps/servarr/sonarr/deployment.yaml
# acompanhar kubectl -n media logs -f <pod> --timestamps por um período
# prolongado e repetir a reprodução (abrir a UI do Sonarr) pra confirmar
# ausência de novas linhas "Bootstrap: Starting Sonarr" inesperadas
```

**Confirmado.** Após aplicar a correção em todos os apps (Sonarr, Radarr, Lidarr, Prowlarr, Jellyfin), a stack ficou estável por 7+ horas seguidas, com o Jellyfin sendo usado ativamente (pessoas assistindo) sem nenhum restart registrado, e o Sonarr completando sincronizações e jobs normalmente sem cair nenhuma vez.

---

### 5. Lições Aprendidas para o Homelab

1. **Um processo que morre sem nenhum log não é necessariamente OOM ou uma exceção gerenciada** — verifique o log de auditoria do kernel do node (`journalctl -k`/`dmesg`) por entradas `DENIED` de AppArmor/SELinux, não só os contadores de cgroup do container; uma camada de segurança do SO pode quebrar silenciosamente um mecanismo interno do runtime antes que qualquer log de aplicação tenha chance de rodar.
2. **O GC do CoreCLR no Linux depende de sinais de tempo real auto-enviados (`SIGRTMIN+N` via `tgkill`)** pra suspender threads — qualquer camada que medeie entrega de sinal (AppArmor, alguns perfis de seccomp) pode quebrar isso silenciosamente, sem exceção, sem log Fatal, sem assinatura de OOM.
3. **Aumentar um limite de recurso como experimento é um passo de diagnóstico válido, e um resultado negativo vale tanto quanto um positivo** — o problema persistir usando uma fração do novo limite (1.5Gi vs ~112Mi de uso real) foi o que definitivamente descartou pressão de memória como causa, em vez de simplesmente atribuir à "imprevisibilidade do GC".
4. **Quando vários componentes da stack compartilham o mesmo runtime/linguagem**, uma causa raiz encontrada em um é um forte indício pros outros, mesmo que o sintoma de cada um já tivesse sido atribuído a causas diferentes (um "bug upstream") — vale revisitar known-issues antigos quando uma causa raiz mais fundamental é encontrada.
5. **Os próprios sinais do Kubernetes (`RestartCount`, eventos do pod, horário de início do container) podem parecer perfeitamente normais** mesmo com algo em crash loop dentro do container, porque um supervisor de processo como o s6-overlay absorve a queda e sobe o processo filho de novo sem o container em si jamais reiniciar.

---

## English

After resetting Sonarr's `/config` (see [`sonarr_reset_stale_apikey_integrations.md`](sonarr_reset_stale_apikey_integrations.md)) and fixing the stale API key cascade, Sonarr remained unstable, restarting repeatedly in an apparently random pattern with absolutely no trace of an error anywhere — until the cause was traced all the way to the node's own kernel audit log: the default containerd AppArmor profile blocking the internal signal .NET's Garbage Collector uses to suspend threads.

---

### 1. Initial Scenario and Symptoms

With Sonarr's API key already fixed across all integrators (Bazarr/Prowlarr/Seerr), the pod kept restarting internally (the Sonarr process itself, supervised by s6, not the container) at irregular intervals — from a few seconds to several minutes — with no clear relationship to any specific action.

#### Error Symptoms:
* `kubectl -n media logs <pod> --timestamps` showed the `Bootstrap: Starting Sonarr` line reappearing dozens of times over several hours, with no error, exception, or warning line immediately before it.
* `kubectl get pod` always showed `RestartCount: 0` and the same `containerID` since the start — the container itself was never recreated, only Sonarr's internal process.
* The interval between crashes varied wildly: sometimes the process died ~2 seconds after finishing startup (before even logging the network check), sometimes it ran normally for 5+ minutes.
* Raising the memory limit from `512Mi` to `1.5Gi` didn't change the pattern at all — the process crashed again using only ~112Mi of actual memory, nowhere near the new limit.
* `/sys/fs/cgroup/memory.events` always showed `oom_kill: 0` and `oom: 0`, even while monitoring it live during a reproduced crash.
* No `livenessProbe`/`readinessProbe`/`lifecycle hook` configured on either the Deployment or the live Pod (confirmed via `kubectl get pod -o yaml`, including checking whether an admission webhook had injected anything).

---

### 2. Investigation & Discovery Process

The investigation eliminated, in order, every obvious layer: CPU and memory via Prometheus metrics (no abnormal spike), cgroup-level OOM (`memory.events` stayed at zero even while monitored live during a reproduced crash), Kubernetes probes/sidecars/webhooks (clean live Pod spec), custom cron/periodic scripts (`crontab -l` and `/etc/periodic/*` both empty), extra s6 services (`/run/service/` only had the default services), and Sonarr's own auto-update mechanism (disabled under Docker mode, not the cause).

The turning point was realizing the pod's `kubectl logs` history was **continuous since the container's creation** (it never actually restarted), yet the Sonarr process would simply stop logging and reappear with `Bootstrap: Starting Sonarr` with absolutely no line in between — not even at `Debug` level. That ruled out any managed exception (.NET always logs `Unhandled exception` with a stack trace when that happens) and pointed to something below the application's own code.

Since the container has no access to `dmesg` (`Operation not permitted`), the investigation had to go all the way to the **node's own kernel audit log** via SSH:

```
sudo journalctl -k --since "16:35" --until "16:50" | grep -iE "sonarr|oom|segfault|panic|killed process|out of memory"
```

This revealed the root cause:

```
kernel: audit: type=1400 ... apparmor="DENIED" operation="signal" class="signal" profile="cri-containerd.apparmor.d" ... comm="Sonarr" requested_mask="send" denied_mask="send" signal=rtmin+3 peer="cri-containerd.apparmor.d//&unconfined"
kernel: audit: type=1400 ... apparmor="DENIED" operation="signal" ... comm="Sonarr" requested_mask="send" denied_mask="send" signal=abrt peer="cri-containerd.apparmor.d//&unconfined"
```

The timestamps of these denials matched exactly the timestamps of the already-documented crashes.

#### Root Cause: AppArmor blocking .NET's GC thread-suspension self-signal
On Linux, CoreCLR (the .NET runtime) emulates the thread suspension the GC needs for a "stop-the-world" pause (the equivalent of Windows' `SuspendThread`) using **POSIX realtime signals** (`SIGRTMIN+N`), delivered via `tgkill` between the process's own threads. This happens every time the GC runs a collection that needs to suspend threads — the frequency depends on allocation pressure at the time, which is why the crash timing looked random.

The default AppArmor profile applied by containerd (`cri-containerd.apparmor.d`) denies a signal send when the target resolves to an `unconfined` peer under that profile's signal-mediation rules. When this blocks the GC's `SIGRTMIN+3`, CoreCLR attempts its **fail-fast** path (aborting in a controlled way rather than continuing with potentially inconsistent state), implemented by sending itself `SIGABRT` — and that signal is **also** denied. The process dies shortly after through a path that never goes through Sonarr's managed exception/logging pipeline (Serilog), hence the total absence of any application-level trace.

**Key point**: a process that dies with absolutely no log, no cgroup OOM signature, and no visible Kubernetes signal is not necessarily crashing in managed code — it may be getting blocked from using an internal runtime mechanism by an OS-level security layer (AppArmor/SELinux), which only shows up in the node's own kernel audit log, not in anything reachable from inside the container.

---

### 3. Commands Used for Diagnosis

```bash
# Confirm the container never actually restarted (continuous log history,
# RestartCount always 0)
kubectl -n media get pod <pod> -o jsonpath="RestartCount: {.status.containerStatuses[0].restartCount}{'\n'}State: {.status.containerStatuses[0].state}{'\n'}"
kubectl -n media get pod <pod> -o yaml

# List every internal process restart with its timestamp
kubectl -n media logs <pod> --timestamps | grep -n "Bootstrap: Starting Sonarr"

# Watch cgroup memory live while reproducing the problem
kubectl -n media exec <pod> -- sh -c 'i=0; while [ $i -lt 90 ]; do date +%T; cat /sys/fs/cgroup/memory.current; cat /sys/fs/cgroup/memory.events; echo ---; sleep 1; i=$((i+1)); done'

# Rule out cron/extra services inside the container
kubectl -n media exec <pod> -- crontab -l
kubectl -n media exec <pod> -- ls -la /run/service/

# dmesg isn't reachable from inside the container (confirms the need to go to the node)
kubectl -n media exec <pod> -- dmesg
# dmesg: klogctl: Operation not permitted

# On the node (via SSH), the kernel audit log that revealed the root cause
sudo journalctl -k --since "<crash time>" --until "<crash time + a few min>" | grep -iE "sonarr|oom|segfault|panic|killed process|out of memory"
sudo dmesg -T | grep -iE "segfault|killed process|oom|panic"
```

---

### 4. The Definitive Fix (or Workaround)

The fix is to remove AppArmor confinement specifically for Sonarr's container, since it legitimately needs the self-signal for the GC to work. On Kubernetes 1.30+, this is done via the stable `securityContext.appArmorProfile` field (the old `container.apparmor.security.beta.kubernetes.io/<container>: unconfined` annotation still works but emits a deprecation warning):

```yaml
containers:
  - name: sonarr
    securityContext:
      appArmorProfile:
        type: Unconfined
```

Since Radarr, Lidarr, and Prowlarr are the same codebase family (.NET/CoreCLR, same GC mechanism), the same fix was proactively applied to them in `apps/servarr/radarr/deployment.yaml`, `apps/servarr/lidarr/deployment.yaml`, and `apps/servarr/prowlarr/deployment.yaml`. Jellyfin (also .NET/ASP.NET Core) also received the fix in `apps/jellyfin/deployment.yaml` — and the result **confirmed** that its previously tracked known issue (`known-issues/jellyfin_frequent_pod_restarts.md`, attributed to an "upstream WebSocket bug") was actually the same AppArmor block; that known issue was removed after this confirmation.

#### Final validation
```bash
kubectl apply -f apps/servarr/sonarr/deployment.yaml
# follow kubectl -n media logs -f <pod> --timestamps for an extended period
# and repeat the reproduction (open Sonarr's UI) to confirm no unexpected
# new "Bootstrap: Starting Sonarr" lines appear
```

**Confirmed.** After applying the fix to all affected apps (Sonarr, Radarr, Lidarr, Prowlarr, Jellyfin), the stack stayed stable for 7+ hours straight, with Jellyfin actively in use (people watching) and no restarts recorded, and Sonarr completing syncs and jobs normally without crashing once.

---

### 5. Lessons Learned for the Homelab

1. **A process that dies with no log at all is not necessarily OOM or a managed exception** — check the node's own kernel audit log (`journalctl -k`/`dmesg`) for AppArmor/SELinux `DENIED` entries, not just the container's own cgroup counters; an OS security layer can silently break an internal runtime mechanism before any application log has a chance to run.
2. **CoreCLR's GC on Linux relies on self-sent realtime signals (`SIGRTMIN+N` via `tgkill`)** to suspend threads — any layer that mediates signal delivery (AppArmor, some seccomp profiles) can silently break this, with no exception, no Fatal log, and no OOM signature.
3. **Raising a resource limit as an experiment is a valid diagnostic step, and a negative result is just as valuable as a positive one** — the problem persisting while using a tiny fraction of the new limit (1.5Gi vs ~112Mi actually used) is what definitively ruled out memory pressure as the cause, instead of just attributing it to "unpredictable GC behavior".
4. **When several components of a stack share the same runtime/language**, a root cause found in one is a strong prior for the others, even if each one's symptom had already been attributed to a different cause (an "upstream bug") — it's worth revisiting old known-issues when a more fundamental root cause is found elsewhere.
5. **Kubernetes' own signals (`RestartCount`, pod events, container start time) can look perfectly clean** even while something is crash-looping inside the container, because a process supervisor like s6-overlay absorbs the crash and respawns the child process without the container itself ever restarting.
