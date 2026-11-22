# Troubleshooting: Wolf no k3s com sidecar dind — apps morrendo (`/tmp` escondido, `/dev/shm` de 64 MB) e outros tropeços / Wolf on k3s with a dind sidecar — apps dying (hidden `/tmp`, 64 MB `/dev/shm`) and other pitfalls

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Implantação do Wolf (Games on Whales) no worker3 (RX 9070) com um sidecar `docker:dind` no mesmo pod, em 2026-11-22. O pareamento com o Moonlight e o app "Test ball" funcionavam, mas Steam e Wolf UI morriam em menos de 1 s. No caminho apareceram também uma API Docker exposta sem TLS e uma porta RTSP presa após restart.

---

### 1. O Cenário Inicial e Sintomas

O Wolf cria um container novo por app via API do Docker. Como o k3s usa containerd e o host não tem Docker, o pod `wolf` (namespace `gaming`, `hostNetwork: true`) ganhou um sidecar `docker:dind`, com os volumes (`/etc/wolf`, `/tmp/sockets`, `/dev`, `/run/udev`) montados nos mesmos caminhos nos dois containers — ver [clusters/gaming/config.md](../clusters/gaming/config.md).

#### Sintomas do Erro:
* Logs do dind: `API listen on [::]:2375` — API Docker sem TLS escutando em todas as interfaces do worker3.
* Wolf logando `[DOCKER] Unable to retrieve docker API version` no start (subia ~20 s antes do `dockerd`).
* Depois de um `kubectl rollout restart`: Moonlight com **erro 111 / RTSP handshake**, e no log do Wolf `Unable to create RTSP server on port: 48010 ... Address already in use` — o Wolf continuou rodando, só sem RTSP.
* **Wolf UI**: o Moonlight conectava e fechava; container saindo com `exit 139` e `segfault at 968 ... in libX11.so.6.4.0` no `dmesg`.
* **Steam**: container saindo com `exit=1` ~0,6 s depois de subir; o log parava em `[Sway] - Starting: steam -bigpicture`.
* **Test ball** (fonte `videotestsrc`) funcionava normalmente, com encode VA-API ("Using zero copy pipeline on AMD").
* Depois da correção do `/tmp`: Steam abria e mostrava **"steamwebhelper is not responding — a critical Steam component is not responding, the Steam UI will not be usable"**.

---

### 2. Processo de Investigação & Descoberta

**API 2375 exposta**: com `DOCKER_TLS_CERTDIR` vazio, o `dockerd-entrypoint.sh` da imagem `docker:dind` acrescenta `tcp://0.0.0.0:2375` aos hosts, mesmo com `--host=unix://...` passado. Com `hostNetwork`, isso equivale a root no worker3 para qualquer máquina da LAN. Corrigido chamando o wrapper direto (`command: ["dind", "dockerd"]`), sem o entrypoint.

**Ordem de start**: dind virou *native sidecar* (`initContainer` com `restartPolicy: Always` + `startupProbe` no socket) — o `wolf` só inicia depois que o `dockerd` responde.

**Porta RTSP presa**: com `hostNetwork` as portas são do node; o Wolf não morre quando o bind falha. Adicionado `livenessProbe` na porta 48010. O `tcpSocket` gerava `[RTSP] error during transmission: End of file` a cada execução, então virou um `exec` que só confere o LISTEN em `/proc/net/tcp*`.

**Apps morrendo**: a hipótese inicial de falta do socket X11 (`/tmp/.X11-unix`) foi descartada — o compositor do Wolf é só Wayland, sem `DISPLAY`. O "Test ball" não cria a tela Wayland, então não servia para investigar. Foi criado um app temporário "Debug shell" (mesma imagem/caps/devices da Steam, com `"Entrypoint": ["sleep", "infinity"]`) para manter uma sessão aberta e entrar no container. Dentro dele:

```
/tmp/sockets:
drwxr-xr-x 2 root root 40 Nov 22 16:59 pulse-socket
drwxr-xr-x 2 root root 40 Nov 22 16:59 wayland-1
```

`wayland-1` e `pulse-socket` eram **diretórios vazios** (tamanho 40 = tmpfs vazio), não sockets. No container do Wolf, o socket `wayland-1` existia de verdade.

#### Causa Raiz 1: o wrapper `dind` monta um tmpfs por cima de `/tmp`
O script `dind` (moby `hack/dind`), executado antes do `dockerd`, faz:

```sh
# Mount /tmp (conditionally)
if ! mountpoint -q /tmp; then
	mount -t tmpfs none /tmp
fi
```

No container dind, `/tmp` não era mountpoint (só `/tmp/sockets` era), então o script montava um tmpfs novo em `/tmp` — **escondendo o emptyDir `/tmp/sockets`** compartilhado com o Wolf. O `dockerd` enxergava um `/tmp` vazio; quando o Wolf pedia o bind de `/tmp/sockets/wayland-1`, o Docker não achava a origem e criava um diretório vazio no lugar. Sem Wayland, o sway saía com erro e o Wolf UI (Godot) caía para X11 e dava segfault no `libX11`. O Test ball não usa esses sockets, por isso funcionava.

**Ponto-chave**: um volume montado num subcaminho (`/tmp/sockets`) pode ser escondido por algo que o próprio processo do container monta depois no caminho pai (`/tmp`). O kubelet/containerd monta os volumes na ordem certa, mas não controla o que o entrypoint faz em seguida.

#### Causa Raiz 2: `/dev/shm` de 64 MB
Com o `/tmp` corrigido, a Steam subia mas o `steamwebhelper` (Chromium) reiniciava a cada 10 s (`console-linux.txt`), com `CommandBufferHelper::AllocateRingBuffer() failed` no `cef_log.txt`. Os apps do Wolf usam `"IpcMode": "host"`; dentro do dind, "host" é o próprio pod, cujo `/dev/shm` padrão no Kubernetes tem **64 MB** — estava 97% cheio.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# [PC, Git Bash] atalho para a API do dind
D="kubectl -n gaming exec deploy/wolf -c dind -- docker -H unix:///var/run/dind/docker.sock"
$D ps -a --format '{{.Names}}  {{.Status}}'

# Dentro da sessão "Debug shell": sockets e devices vistos pelo app
MSYS_NO_PATHCONV=1 $D exec WolfDebug_12163859283594782795 sh -c 'id; env | grep -E "WAYLAND|XDG|DISPLAY"; ls -l /tmp/sockets /dev/dri'

# Mesmos sockets vistos pelo container do Wolf
MSYS_NO_PATHCONV=1 kubectl -n gaming exec deploy/wolf -c wolf -- sh -c 'env | grep -iE "sock|wayland"; ls -l /var/run/wolf /tmp/sockets'

# Comparar env/mounts que o Wolf passou para cada app
W=Wolf-UI_12163859283594782795; S=WolfSteam_12163859283594782795
for c in $W $S; do echo "== $c"; $D inspect $c --format '{{json .Config.User}} {{json .Config.Env}}{{"\n"}}{{json .HostConfig.Binds}}{{"\n"}}{{json .HostConfig.Mounts}}'; done

# Script do wrapper dind (via API do GitHub; o ISP bloqueia raw.githubusercontent.com)
curl -s -H "Accept: application/vnd.github.raw" "https://api.github.com/repos/moby/moby/contents/hack/dind" | head -80

# Steam: tamanho do shm e erros do webhelper
MSYS_NO_PATHCONV=1 $D exec -u retro $S sh -c 'df -h /dev/shm; cd ~/.local/share/Steam/logs; grep -aiE "shm|fatal|crash|exited|restart|error|failed" console-linux.txt cef_log.txt 2>/dev/null | grep -v crash_reporting | tail -50'
```

---

### 4. A Correção Definitiva (ou Workaround)

Tudo em [`apps/wolf/deployment.yaml`](../apps/wolf/deployment.yaml), com comentários explicando cada item:

```yaml
# dind: sem o dockerd-entrypoint.sh (não abre 2375), só socket unix
command: ["dind", "dockerd"]
args: [--host=unix:///var/run/dind/docker.sock, --bridge=none, --iptables=false, --ip6tables=false]

# dind: /tmp vira mountpoint -> o wrapper não monta tmpfs por cima de /tmp/sockets
- name: dind-tmp
  mountPath: /tmp

# wolf e dind: /dev/shm maior que os 64 MB padrão
- name: shm
  emptyDir:
    medium: Memory
    sizeLimit: 2Gi

# wolf: liveness sem conectar na porta RTSP (48010 = 0xBB8A, 0A = LISTEN)
livenessProbe:
  exec:
    command: ["sh", "-c", "grep -qE ':BB8A [0-9A-F]+:0000 0A' /proc/net/tcp /proc/net/tcp6"]
```

```bash
kubectl apply -f apps/wolf/deployment.yaml
kubectl -n gaming rollout status deploy/wolf
```

#### Validação final
```bash
MSYS_NO_PATHCONV=1 kubectl -n gaming exec deploy/wolf -c dind -- sh -c 'grep " /tmp" /proc/mounts'   # /tmp não é mais "none /tmp tmpfs"
MSYS_NO_PATHCONV=1 kubectl -n gaming exec deploy/wolf -c dind -- df -h /dev/shm                       # ~2.0G
kubectl -n gaming logs deploy/wolf -c wolf --since=1m | grep -ac "RTSP] error"                        # 0
kubectl -n gaming get pod -l app=wolf                                                                 # 2/2 Running, 0 restarts
```

Steam e Wolf UI abriram e funcionaram pelo Moonlight.

---

### 5. Lições Aprendidas para o Homelab

1. **Ler o entrypoint de imagens "de infraestrutura" antes de usar com `hostNetwork`** — o `docker:dind` abre uma porta TCP sem TLS por padrão em certas combinações de env; com `hostNetwork` isso vira exposição na LAN. Conferir com `ss -tlnp` depois do primeiro start.
2. **Volume em subcaminho pode ser escondido pelo entrypoint** — se um container monta algo no caminho pai depois do start (como o `dind` faz com `/tmp`), o volume do subcaminho some para aquele processo. Sintoma típico: o Docker cria diretórios vazios onde deveria haver sockets/arquivos.
3. **Diretório de 40 bytes onde deveria haver um socket = bind mount de origem inexistente** — o Docker cria a origem como diretório vazio em vez de falhar. Vale para qualquer setup dind/Docker-out-of-Docker com caminhos compartilhados.
4. **`/dev/shm` de pods é 64 MB** — apps com Chromium/Electron (Steam, navegadores headless) precisam de `emptyDir` `medium: Memory` em `/dev/shm`. Com `IpcMode: host` dentro de dind, o shm dos apps é o do pod.
5. **App de depuração com `sleep infinity`** — quando um container criado por outro sistema morre rápido demais para investigar, criar uma variante com o mesmo ambiente e entrypoint `sleep infinity` permite entrar com `docker exec` (lembrando que o entrypoint original, que cria usuários etc., não roda).
6. **`tcpSocket` probe tem efeito colateral** — a aplicação vê uma conexão abrindo e fechando; quando isso gera ruído ou erro, um probe `exec` que só inspeciona o estado (ex: `/proc/net/tcp`) evita o problema.

---

## English

Deployment of Wolf (Games on Whales) on worker3 (RX 9070) with a `docker:dind` sidecar in the same pod, on 2026-11-22. Moonlight pairing and the "Test ball" app worked, but Steam and Wolf UI died in under 1 s. Along the way there was also an unauthenticated Docker API exposed without TLS and an RTSP port left busy after a restart.

---

### 1. Initial Scenario and Symptoms

Wolf creates a new container per app through the Docker API. Since k3s uses containerd and the host has no Docker, the `wolf` pod (`gaming` namespace, `hostNetwork: true`) got a `docker:dind` sidecar, with the volumes (`/etc/wolf`, `/tmp/sockets`, `/dev`, `/run/udev`) mounted at the same paths in both containers — see [clusters/gaming/config.md](../clusters/gaming/config.md).

#### Error Symptoms:
* dind logs: `API listen on [::]:2375` — Docker API without TLS listening on every worker3 interface.
* Wolf logging `[DOCKER] Unable to retrieve docker API version` at start (it came up ~20 s before `dockerd`).
* After a `kubectl rollout restart`: Moonlight showing **error 111 / RTSP handshake**, and Wolf logging `Unable to create RTSP server on port: 48010 ... Address already in use` — Wolf kept running, just without RTSP.
* **Wolf UI**: Moonlight connected and closed; the container exited with `exit 139` and `segfault at 968 ... in libX11.so.6.4.0` in `dmesg`.
* **Steam**: the container exited with `exit=1` ~0.6 s after starting; the log stopped at `[Sway] - Starting: steam -bigpicture`.
* **Test ball** (`videotestsrc` source) worked normally, with VA-API encoding ("Using zero copy pipeline on AMD").
* After the `/tmp` fix: Steam opened and showed **"steamwebhelper is not responding — a critical Steam component is not responding, the Steam UI will not be usable"**.

---

### 2. Investigation & Discovery Process

**Exposed 2375 API**: with an empty `DOCKER_TLS_CERTDIR`, the `docker:dind` image's `dockerd-entrypoint.sh` appends `tcp://0.0.0.0:2375` to the hosts, even with `--host=unix://...` passed. With `hostNetwork`, that is root on worker3 for any machine on the LAN. Fixed by calling the wrapper directly (`command: ["dind", "dockerd"]`), skipping the entrypoint.

**Start order**: dind became a *native sidecar* (`initContainer` with `restartPolicy: Always` + a `startupProbe` on the socket) — `wolf` only starts once `dockerd` answers.

**Busy RTSP port**: with `hostNetwork` the ports belong to the node; Wolf doesn't die when the bind fails. A `livenessProbe` on port 48010 was added. The `tcpSocket` probe produced `[RTSP] error during transmission: End of file` on every run, so it became an `exec` probe that only checks for LISTEN in `/proc/net/tcp*`.

**Apps dying**: the initial hypothesis of a missing X11 socket (`/tmp/.X11-unix`) was ruled out — Wolf's compositor is Wayland-only, with no `DISPLAY`. "Test ball" doesn't create the Wayland display, so it was useless for investigating. A temporary "Debug shell" app was created (same image/caps/devices as Steam, with `"Entrypoint": ["sleep", "infinity"]`) to keep a session open and get into the container. Inside it:

```
/tmp/sockets:
drwxr-xr-x 2 root root 40 Nov 22 16:59 pulse-socket
drwxr-xr-x 2 root root 40 Nov 22 16:59 wayland-1
```

`wayland-1` and `pulse-socket` were **empty directories** (size 40 = empty tmpfs), not sockets. In the Wolf container, the `wayland-1` socket really existed.

#### Root Cause 1: the `dind` wrapper mounts a tmpfs over `/tmp`
The `dind` script (moby `hack/dind`), run before `dockerd`, does:

```sh
# Mount /tmp (conditionally)
if ! mountpoint -q /tmp; then
	mount -t tmpfs none /tmp
fi
```

In the dind container, `/tmp` was not a mountpoint (only `/tmp/sockets` was), so the script mounted a fresh tmpfs on `/tmp` — **hiding the `/tmp/sockets` emptyDir** shared with Wolf. `dockerd` saw an empty `/tmp`; when Wolf asked to bind `/tmp/sockets/wayland-1`, Docker couldn't find the source and created an empty directory in its place. Without Wayland, sway exited with an error and Wolf UI (Godot) fell back to X11 and segfaulted in `libX11`. Test ball doesn't use those sockets, which is why it worked.

**Key point**: a volume mounted at a subpath (`/tmp/sockets`) can be hidden by something the container's own process mounts later on the parent path (`/tmp`). The kubelet/containerd mounts volumes in the right order, but has no control over what the entrypoint does afterwards.

#### Root Cause 2: 64 MB `/dev/shm`
With `/tmp` fixed, Steam started but `steamwebhelper` (Chromium) restarted every 10 s (`console-linux.txt`), with `CommandBufferHelper::AllocateRingBuffer() failed` in `cef_log.txt`. Wolf's apps use `"IpcMode": "host"`; inside dind, "host" is the pod itself, whose default Kubernetes `/dev/shm` is **64 MB** — it was 97% full.

---

### 3. Commands Used for Diagnosis

```bash
# [PC, Git Bash] atalho para a API do dind
D="kubectl -n gaming exec deploy/wolf -c dind -- docker -H unix:///var/run/dind/docker.sock"
$D ps -a --format '{{.Names}}  {{.Status}}'

# Dentro da sessão "Debug shell": sockets e devices vistos pelo app
MSYS_NO_PATHCONV=1 $D exec WolfDebug_12163859283594782795 sh -c 'id; env | grep -E "WAYLAND|XDG|DISPLAY"; ls -l /tmp/sockets /dev/dri'

# Mesmos sockets vistos pelo container do Wolf
MSYS_NO_PATHCONV=1 kubectl -n gaming exec deploy/wolf -c wolf -- sh -c 'env | grep -iE "sock|wayland"; ls -l /var/run/wolf /tmp/sockets'

# Comparar env/mounts que o Wolf passou para cada app
W=Wolf-UI_12163859283594782795; S=WolfSteam_12163859283594782795
for c in $W $S; do echo "== $c"; $D inspect $c --format '{{json .Config.User}} {{json .Config.Env}}{{"\n"}}{{json .HostConfig.Binds}}{{"\n"}}{{json .HostConfig.Mounts}}'; done

# Script do wrapper dind (via API do GitHub; o ISP bloqueia raw.githubusercontent.com)
curl -s -H "Accept: application/vnd.github.raw" "https://api.github.com/repos/moby/moby/contents/hack/dind" | head -80

# Steam: tamanho do shm e erros do webhelper
MSYS_NO_PATHCONV=1 $D exec -u retro $S sh -c 'df -h /dev/shm; cd ~/.local/share/Steam/logs; grep -aiE "shm|fatal|crash|exited|restart|error|failed" console-linux.txt cef_log.txt 2>/dev/null | grep -v crash_reporting | tail -50'
```

---

### 4. The Definitive Fix (or Workaround)

All in [`apps/wolf/deployment.yaml`](../apps/wolf/deployment.yaml), with comments explaining each item:

```yaml
# dind: sem o dockerd-entrypoint.sh (não abre 2375), só socket unix
command: ["dind", "dockerd"]
args: [--host=unix:///var/run/dind/docker.sock, --bridge=none, --iptables=false, --ip6tables=false]

# dind: /tmp vira mountpoint -> o wrapper não monta tmpfs por cima de /tmp/sockets
- name: dind-tmp
  mountPath: /tmp

# wolf e dind: /dev/shm maior que os 64 MB padrão
- name: shm
  emptyDir:
    medium: Memory
    sizeLimit: 2Gi

# wolf: liveness sem conectar na porta RTSP (48010 = 0xBB8A, 0A = LISTEN)
livenessProbe:
  exec:
    command: ["sh", "-c", "grep -qE ':BB8A [0-9A-F]+:0000 0A' /proc/net/tcp /proc/net/tcp6"]
```

```bash
kubectl apply -f apps/wolf/deployment.yaml
kubectl -n gaming rollout status deploy/wolf
```

#### Final validation
```bash
MSYS_NO_PATHCONV=1 kubectl -n gaming exec deploy/wolf -c dind -- sh -c 'grep " /tmp" /proc/mounts'   # /tmp não é mais "none /tmp tmpfs"
MSYS_NO_PATHCONV=1 kubectl -n gaming exec deploy/wolf -c dind -- df -h /dev/shm                       # ~2.0G
kubectl -n gaming logs deploy/wolf -c wolf --since=1m | grep -ac "RTSP] error"                        # 0
kubectl -n gaming get pod -l app=wolf                                                                 # 2/2 Running, 0 restarts
```

Steam and Wolf UI opened and worked through Moonlight.

---

### 5. Lessons Learned for the Homelab

1. **Read the entrypoint of "infrastructure" images before using them with `hostNetwork`** — `docker:dind` opens a TCP port without TLS by default under certain env combinations; with `hostNetwork` that becomes LAN exposure. Check with `ss -tlnp` after the first start.
2. **A subpath volume can be hidden by the entrypoint** — if a container mounts something on the parent path after start (as `dind` does with `/tmp`), the subpath volume disappears for that process. Typical symptom: Docker creates empty directories where sockets/files should be.
3. **A 40-byte directory where a socket should be = bind mount with a missing source** — Docker creates the source as an empty directory instead of failing. Applies to any dind/Docker-out-of-Docker setup with shared paths.
4. **Pod `/dev/shm` is 64 MB** — apps with Chromium/Electron (Steam, headless browsers) need a `medium: Memory` `emptyDir` on `/dev/shm`. With `IpcMode: host` inside dind, the apps' shm is the pod's.
5. **A debug app with `sleep infinity`** — when a container created by another system dies too fast to investigate, a variant with the same environment and a `sleep infinity` entrypoint lets you get in with `docker exec` (keeping in mind the original entrypoint, which creates users etc., doesn't run).
6. **A `tcpSocket` probe has side effects** — the application sees a connection open and close; when that causes noise or errors, an `exec` probe that only inspects state (e.g. `/proc/net/tcp`) avoids the problem.
