# Troubleshooting: Log de consultas do Pi-hole não chegava ao stdout/Loki / Pi-hole query log never reached stdout/Loki

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Nenhuma linha de consulta DNS do Pi-hole aparecia no Loki, mesmo com o Loki/Alloy funcionando normalmente para os outros apps do cluster. A causa raiz era a ausência da capability Linux `CAP_SYS_PTRACE` no container do Pi-hole, necessária para o script interno da imagem conseguir espelhar o log de consultas no stdout do container.

---

### 1. O Cenário Inicial e Sintomas

Durante a investigação de [troubleshooting/loki_canary_misidentified_as_unknown_app.md](loki_canary_misidentified_as_unknown_app.md) (que descartou o Pi-hole como origem do log `ppppppppp`), surgiu a dúvida complementar de por que nenhum log do Pi-hole aparecia no Loki — nem mesmo entradas legítimas de consulta DNS.

#### Sintomas do Erro:
* Nenhuma linha de consulta DNS do Pi-hole (namespace `dns-services`) aparecia no Grafana Explore (datasource Loki), mesmo filtrando por `namespace="dns-services"`.
* `kubectl logs -n dns-services -l app=pihole-app` mostrava repetidamente a linha `/usr/bin/prefix-time.sh: line 5: /proc/1/fd/1: Permission denied`.
* Em paralelo, `kubectl exec -n dns-services deploy/pihole-deployment -- tail /var/log/pihole/pihole.log` mostrava o log de consultas real sendo escrito normalmente dentro do container, incluindo tráfego real vindo do `coredns-bypass`.

---

### 2. Processo de Investigação & Descoberta

Primeiro foi confirmado que **não era um problema do Loki/Alloy**: o Alloy só coleta stdout/stderr de cada pod via `loki.source.kubernetes` (ver [clusters/observability/config.md](../clusters/observability/config.md)), nunca arquivos arbitrários dentro do container — então o fato do `pihole.log` ter conteúdo real já descartava qualquer problema de coleta. O problema era anterior: essas linhas nunca chegavam ao stdout do container.

A imagem oficial do Pi-hole usa um script (`/usr/bin/prefix-time.sh`) cuja função é ler `/var/log/pihole/pihole.log` continuamente e escrever cada linha (prefixada com timestamp) em `/proc/1/fd/1` — o descritor de arquivo do stdout do processo PID 1 do container. É esse mecanismo que normalmente permite ver o log de consultas via `docker logs`/`kubectl logs`.

Para descartar um mismatch simples de UID entre processos, foram coletados dentro do pod:
```
cat /proc/1/status   # PID 1 (FTL) roda como UID 1000, mas tem CapEff com várias capabilities
ps aux                # prefix-time.sh e o processo PID 1 rodam ambos como root/pihole
ls -la /proc/1/fd/1   # symlink de /proc/1/fd/1 existe e é acessível só por quem tem a capability certa
```

#### Causa Raiz: capability `CAP_SYS_PTRACE` ausente no container
Ler o alvo de um symlink `/proc/<pid>/fd/<n>` de **outro** processo (aqui, `prefix-time.sh` lendo `/proc/1/fd/1`, onde `1` é outro processo dentro do mesmo pod) exige a capability Linux `CAP_SYS_PTRACE` — mesmo quando ambos os processos rodam como root/UID 0, e mesmo sem haver uma relação de ptrace real entre eles. O conjunto de capabilities padrão de um container OCI/Docker **não inclui** `CAP_SYS_PTRACE`, e o `Deployment` deste repositório (`apps/pihole/deployment.yaml`) só adicionava `SYS_NICE` explicitamente — faltava `SYS_PTRACE`.

**Ponto-chave**: um processo rodando como root dentro de um container não tem automaticamente todas as permissões de um root "de verdade" — o conjunto de Linux capabilities do container é o que de fato determina o que é permitido, e `CAP_SYS_PTRACE` é frequentemente esquecido por não ser uma capability "óbvia" (associada a debugging/ptrace, não a leitura de arquivos).

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirma que o log de consultas real está sendo escrito corretamente dentro do container
# (descarta problema de geração do log, aponta para o mecanismo de forward ao stdout)
kubectl exec -n dns-services deploy/pihole-deployment -- tail /var/log/pihole/pihole.log

# Mostra o erro repetido do mecanismo de forward ao stdout
kubectl logs -n dns-services -l app=pihole-app --tail=50

# Descarta mismatch de UID entre o prefix-time.sh e o PID 1 do container
kubectl exec -n dns-services deploy/pihole-deployment -- cat /proc/1/status
kubectl exec -n dns-services deploy/pihole-deployment -- ps aux
kubectl exec -n dns-services deploy/pihole-deployment -- ls -la /proc/1/fd/1
```

---

### 4. A Correção Definitiva (ou Workaround)

Adicionada a capability `SYS_PTRACE` à lista já existente de `securityContext.capabilities.add` no container do Pi-hole, em `apps/pihole/deployment.yaml`:

```yaml
containers:
  - name: pihole
    image: pihole/pihole:2026.10.1
    envFrom:
      - configMapRef:
          name: pihole-configs
    securityContext:
      capabilities:
        add: ["SYS_NICE","SYS_PTRACE"]
```

```bash
kubectl apply -f apps/pihole/deployment.yaml
```

#### Validação final
```bash
# Pod reiniciado com a nova capability
kubectl get pods -n dns-services -l app=pihole-app

# O erro "/proc/1/fd/1: Permission denied" não aparece mais no kubectl logs
kubectl logs -n dns-services -l app=pihole-app --tail=50

# Gera uma consulta DNS real e confirma que ela passa a aparecer no kubectl logs
# (não só no tail direto do arquivo), e consequentemente no Loki
nslookup bottoflix.home 192.168.3.200
```
Resultado: as linhas de consulta DNS do Pi-hole passaram a aparecer normalmente no `kubectl logs` e, por consequência, foram coletadas pelo Alloy e ficaram visíveis no Grafana Explore (datasource Loki).

---

### 5. Lições Aprendidas para o Homelab

1. **`CAP_SYS_PTRACE` é necessária para ler `/proc/<pid>/fd/<n>` de outro processo, mesmo como root** — não é intuitivo, já que parece uma operação de simples leitura de arquivo, mas o kernel trata o acesso ao file descriptor table de outro processo como uma operação sensível equivalente a ptrace.
2. **Imagens de terceiros podem depender de capabilities não óbvias pro mecanismo de log funcionar** — a imagem oficial do Pi-hole usa `/proc/1/fd/1` internamente pra espelhar o log de consultas no stdout; isso não está documentado como pré-requisito óbvio, e sem ele o container roda normalmente (sem crash), só perde silenciosamente esse log específico.
3. **Descartar mismatch de UID antes de assumir problema de capability** — nem toda falha de permissão em `/proc` é causada por UID diferente; aqui ambos os processos rodavam como o mesmo usuário, e o problema real era o conjunto de capabilities do container.

---

## English

No Pi-hole DNS query log lines ever showed up in Loki, even though Loki/Alloy worked normally for every other app in the cluster. The root cause was the missing Linux capability `CAP_SYS_PTRACE` in the Pi-hole container, required for the image's internal script to mirror the query log onto the container's stdout.

---

### 1. Initial Scenario and Symptoms

While investigating [troubleshooting/loki_canary_misidentified_as_unknown_app.md](loki_canary_misidentified_as_unknown_app.md) (which ruled out Pi-hole as the source of the `ppppppppp` log), a follow-up question came up: why did no Pi-hole logs show up in Loki at all — not even legitimate DNS query entries?

#### Error Symptoms:
* No Pi-hole DNS query log lines (namespace `dns-services`) showed up in Grafana Explore (Loki datasource), even when filtering by `namespace="dns-services"`.
* `kubectl logs -n dns-services -l app=pihole-app` repeatedly showed the line `/usr/bin/prefix-time.sh: line 5: /proc/1/fd/1: Permission denied`.
* In parallel, `kubectl exec -n dns-services deploy/pihole-deployment -- tail /var/log/pihole/pihole.log` showed the real query log being written normally inside the container, including real traffic coming from `coredns-bypass`.

---

### 2. Investigation & Discovery Process

First it was confirmed this was **not a Loki/Alloy problem**: Alloy only collects stdout/stderr from each pod via `loki.source.kubernetes` (see [clusters/observability/config.md](../clusters/observability/config.md)), never arbitrary files inside the container — so the fact that `pihole.log` had real content already ruled out any collection issue. The problem was upstream: those lines never reached the container's stdout in the first place.

The official Pi-hole image uses a script (`/usr/bin/prefix-time.sh`) whose job is to continuously read `/var/log/pihole/pihole.log` and write each line (prefixed with a timestamp) to `/proc/1/fd/1` — the stdout file descriptor of the container's PID 1 process. This is the mechanism that normally lets you see the query log via `docker logs`/`kubectl logs`.

To rule out a simple UID mismatch between processes, the following were collected inside the pod:
```
cat /proc/1/status   # PID 1 (FTL) runs as UID 1000, but has several capabilities in CapEff
ps aux                # prefix-time.sh and the PID 1 process both run as root/pihole
ls -la /proc/1/fd/1   # the /proc/1/fd/1 symlink exists and is only accessible to whoever has the right capability
```

#### Root Cause: missing `CAP_SYS_PTRACE` capability in the container
Reading the target of a `/proc/<pid>/fd/<n>` symlink belonging to **another** process (here, `prefix-time.sh` reading `/proc/1/fd/1`, where `1` is a different process in the same pod) requires the Linux capability `CAP_SYS_PTRACE` — even when both processes run as root/UID 0, and even with no actual ptrace relationship between them. A standard OCI/Docker container's default capability set **does not include** `CAP_SYS_PTRACE`, and this repository's `Deployment` (`apps/pihole/deployment.yaml`) only explicitly added `SYS_NICE` — `SYS_PTRACE` was missing.

**Key point**: a process running as root inside a container doesn't automatically have every permission of a "real" root — the container's Linux capability set is what actually determines what's allowed, and `CAP_SYS_PTRACE` is often overlooked because it isn't an "obvious" capability (it's associated with debugging/ptrace, not with reading files).

---

### 3. Commands Used for Diagnosis

```bash
# Confirms the real query log is being written correctly inside the container
# (rules out log generation, points to the stdout-forwarding mechanism)
kubectl exec -n dns-services deploy/pihole-deployment -- tail /var/log/pihole/pihole.log

# Shows the repeated error from the stdout-forwarding mechanism
kubectl logs -n dns-services -l app=pihole-app --tail=50

# Rules out a UID mismatch between prefix-time.sh and the container's PID 1
kubectl exec -n dns-services deploy/pihole-deployment -- cat /proc/1/status
kubectl exec -n dns-services deploy/pihole-deployment -- ps aux
kubectl exec -n dns-services deploy/pihole-deployment -- ls -la /proc/1/fd/1
```

---

### 4. The Definitive Fix (or Workaround)

Added the `SYS_PTRACE` capability to the already-existing `securityContext.capabilities.add` list on the Pi-hole container, in `apps/pihole/deployment.yaml`:

```yaml
containers:
  - name: pihole
    image: pihole/pihole:2026.10.1
    envFrom:
      - configMapRef:
          name: pihole-configs
    securityContext:
      capabilities:
        add: ["SYS_NICE","SYS_PTRACE"]
```

```bash
kubectl apply -f apps/pihole/deployment.yaml
```

#### Final validation
```bash
# Pod restarted with the new capability
kubectl get pods -n dns-services -l app=pihole-app

# The "/proc/1/fd/1: Permission denied" error no longer shows up in kubectl logs
kubectl logs -n dns-services -l app=pihole-app --tail=50

# Trigger a real DNS query and confirm it now shows up in kubectl logs
# (not just via a direct tail of the file), and consequently in Loki
nslookup bottoflix.home 192.168.3.200
```
Result: Pi-hole DNS query log lines started showing up normally in `kubectl logs` and, as a result, were picked up by Alloy and became visible in Grafana Explore (Loki datasource).

---

### 5. Lessons Learned for the Homelab

1. **`CAP_SYS_PTRACE` is needed to read another process's `/proc/<pid>/fd/<n>`, even as root** — this isn't intuitive, since it looks like a simple file read, but the kernel treats access to another process's file descriptor table as a sensitive operation equivalent to ptrace.
2. **Third-party images can depend on non-obvious capabilities for their logging mechanism to work** — the official Pi-hole image uses `/proc/1/fd/1` internally to mirror the query log to stdout; this isn't documented as an obvious prerequisite, and without it the container runs fine (no crash), it just silently loses that specific log.
3. **Rule out a UID mismatch before assuming a capability problem** — not every `/proc` permission failure is caused by a different UID; here both processes ran as the same user, and the real problem was the container's capability set.
