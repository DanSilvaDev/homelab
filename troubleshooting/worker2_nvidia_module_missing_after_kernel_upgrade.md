# Troubleshooting: Módulo NVIDIA ausente após upgrade de kernel no worker2 / Missing NVIDIA module after kernel upgrade on worker2

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

O `nvidia-device-plugin` entrou em `CrashLoopBackOff` no node `worker2` sem nenhum log ou evento útil. A causa raiz estava no host (VM worker2), não no pod: um `apt upgrade` trouxe um kernel novo antes do próximo reboot da VM, e o pacote de módulo do driver NVIDIA travado via `apt-mark hold` não acompanhou essa mudança de kernel.

---

### 1. O Cenário Inicial e Sintomas

Durante a retomada do trabalho na stack de mídia (Jellyfin), foi notado que o pod `nvidia-device-plugin-mxjwh` estava em `CrashLoopBackOff` no cluster k3s. O usuário havia rodado `sudo apt update && sudo apt upgrade` no worker2 antes do reboot mais recente da VM (feito para aplicar um aumento de RAM documentado em `clusters/worker2/config.md`).

#### Sintomas do Erro:
* Pod `nvidia-device-plugin-mxjwh` em `CrashLoopBackOff`.
* `kubectl logs` sem conteúdo útil (container nem chegava a iniciar o processo).
* `kubectl describe pod` sem Events relevantes (expiraram pelo TTL padrão de ~1h do cluster).
* Erro reportado pelo containerd ao tentar criar o container:
  ```
  failed to create containerd task: failed to create shim task: OCI runtime create failed: could not apply required modification to OCI specification: error modifying OCI spec: failed to create the automatic CDI modifier: failed to generate CDI spec for mode "auto": failed to construct device spec generators: failed to initialize NVML: Driver Not Loaded
  ```
* `nvidia-smi` rodado diretamente no host (worker2, via SSH) falhando com:
  ```
  NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.
  ```

---

### 2. Processo de Investigação & Descoberta

O erro do containerd (`failed to initialize NVML: Driver Not Loaded`) já indicava que o problema não estava no pod nem no device-plugin em si, mas na camada do host, antes mesmo do container iniciar — a falha ocorre no NVIDIA Container Toolkit ao tentar gerar a CDI spec. Reproduzir `nvidia-smi` direto no host confirmou isso.

A partir daí, a investigação seguiu três frentes: (1) o kernel atual batia com o kernel para o qual o módulo foi compilado? (2) o driver usa DKMS (recompila automaticamente a cada kernel novo) ou pacotes pré-compilados por versão? (3) o que mudou recentemente no host?

- `uname -r` retornou `7.0.0-42-generic`, diferente do `7.0.0-38-generic` documentado em `clusters/worker2/config.md`.
- `dkms status` retornou `dkms: command not found` — ou seja, esse driver **não** usa DKMS. O Ubuntu, nesse caso, usa pacotes `.deb` de módulo versionados por ABI de kernel (`linux-modules-nvidia-580-server-<versão-do-kernel>`), não um módulo recompilado dinamicamente.
- `dpkg -l | grep -i nvidia` mostrou `linux-modules-nvidia-580-server-7.0.0-38-generic` e `linux-modules-nvidia-580-server-generic` instalados e **em hold** (flag `hi`), ambos travados na versão `7.0.0-38.38+2` — nenhum pacote de módulo existia para o kernel `7.0.0-42`.
- `sudo dmesg | grep -iE "nvidia|xid"` não trouxe nenhuma linha `NVRM` (só entradas de áudio HDMI da própria placa, que usa um driver de som genérico separado) — confirmando que o módulo gráfico/compute nunca chegou a tentar carregar, porque o arquivo `.ko` correspondente ao kernel `7.0.0-42` simplesmente não existia no disco.

#### Causa Raiz: pacote de módulo de kernel travado por `apt-mark hold`, sem acompanhar upgrade de kernel
O driver NVIDIA `580-server` foi instalado nesse host sem DKMS, usando os pacotes pré-compilados do Ubuntu (`linux-modules-nvidia-580-server-*`), que são versionados por ABI de kernel. Na instalação original, **todos** os pacotes contendo `nvidia` e `580` foram travados via `apt-mark hold $(dpkg-query -W -f='${Package}\n' | grep -E 'nvidia.*-580')` — uma precaução deliberada para evitar que a GPU (GTX 970, Maxwell) fosse upgradeada para a branch `590+`, que não suporta esse hardware (ver `ia/CLAUDE.md`, seção "Convenções importantes").

Esse hold, porém, também travou o pacote `linux-modules-nvidia-580-server-generic` (e sua versão específica para o kernel `7.0.0-38`) na versão instalada originalmente. O pacote `linux-image-generic` (kernel em si) **não** está em hold, então um `apt upgrade` trouxe um kernel novo (`7.0.0-42`) sem trazer junto o módulo NVIDIA correspondente — porque o pacote de módulo ficou congelado na versão antiga pelo hold. Depois do reboot que aplicou esse kernel novo, não existia nenhum módulo `nvidia.ko` para ele, e o driver ficou efetivamente "desligado" até alguém instalar manualmente o pacote certo.

**Ponto-chave**: travar pacotes `nvidia-*-580` via `apt-mark hold` para fixar a branch do driver também trava, sem intenção explícita, os pacotes de módulo de kernel versionados por ABI — que precisam ser atualizados manualmente (dentro da mesma branch) toda vez que o kernel do host sobe de versão.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Reproduzir o erro direto no host, fora de qualquer container
nvidia-smi

# Ver o kernel atualmente rodando
uname -r

# Checar se o driver usa DKMS (recompilação automática por kernel)
dkms status

# Ver módulo carregado atualmente (nenhum resultado = não carregado)
lsmod | grep nvidia

# Procurar erros de inicialização do driver no log do kernel
sudo dmesg | grep -iE "nvidia|xid" | tail -50

# Listar todos os pacotes relacionados a nvidia e seu status de hold (flag "hi" = held + installed)
dpkg -l | grep -i nvidia

# Confirmar que o pacote de módulo pro kernel novo existe no repositório, na mesma branch (580)
sudo apt update
apt-cache madison linux-modules-nvidia-580-server-7.0.0-42-generic
```

---

### 4. A Correção Definitiva (ou Workaround)

Instalado diretamente o pacote de módulo específico para o kernel em execução (`7.0.0-42-generic`), sem precisar destravar o hold geral — o `apt install` de um pacote até então não instalado não é bloqueado pelo hold de outros pacotes:

```bash
sudo apt install -y linux-modules-nvidia-580-server-7.0.0-42-generic linux-objects-nvidia-580-server-7.0.0-42-generic
sudo depmod -a
sudo modprobe nvidia
```

#### Validação final
```bash
nvidia-smi
# Driver Version: 580.178.04, CUDA Version: 13.0
# GPU 0: NVIDIA GeForce GTX 970, 0MiB / 4096MiB em uso, nenhum processo rodando
```

O pod `nvidia-device-plugin` voltou a `Running` automaticamente no retry seguinte do `CrashLoopBackOff`, sem necessidade de deletar o pod manualmente.

---

### 5. Lições Aprendidas para o Homelab

1. **Um erro do containerd/OCI runtime ao criar o container (não um erro do próprio pod) aponta pra fora do Kubernetes** — quando a falha menciona `NVML`, `CDI spec` ou "OCI runtime create failed" antes mesmo do container logar algo, o problema está no host (driver, toolkit, kernel), não no manifesto do pod ou no device-plugin.
2. **Eventos do Kubernetes expiram (~1h por padrão)** — a ausência de `Events` relevantes no `kubectl describe pod` não significa que nada de errado aconteceu; pode só significar que o evento é mais antigo que o TTL. Não descartar a causa raiz só porque a seção de Events está vazia.
3. **`apt-mark hold` por nome de pacote (`grep` amplo) pode travar mais do que o pretendido** — travar `nvidia-*-580` pra fixar a branch do driver também trava os pacotes de módulo de kernel versionados por ABI, que *precisam* ser atualizados manualmente (dentro da mesma branch) a cada upgrade de kernel do host. Isso é esperado se acontecer de novo: depois de qualquer `apt upgrade` que traga kernel novo no worker2, checar se o pacote `linux-modules-nvidia-580-server-$(uname -r)` existe e está instalado **antes** de reiniciar a VM, evitando a janela de GPU indisponível.
4. **Driver sem DKMS usa pacotes de módulo versionados por ABI de kernel** — `dkms status` retornando "command not found" é o sinal de que a recompilação não é automática; cada bump de kernel exige o pacote `linux-modules-nvidia-<branch>-<kernel-abi>` correspondente já publicado no repositório (confirmável via `apt-cache madison`) e instalado manualmente.

---

## English

The `nvidia-device-plugin` pod entered `CrashLoopBackOff` on the `worker2` node with no useful logs or events. The root cause was on the host (worker2 VM), not the pod: an `apt upgrade` brought in a new kernel before the VM's next reboot, and the NVIDIA driver's kernel-module package — held via `apt-mark hold` — never caught up with that kernel change.

---

### 1. Initial Scenario and Symptoms

While resuming work on the media stack (Jellyfin), the `nvidia-device-plugin-mxjwh` pod was found in `CrashLoopBackOff` on the k3s cluster. The user had run `sudo apt update && sudo apt upgrade` on worker2 before the VM's most recent reboot (done to apply a RAM increase documented in `clusters/worker2/config.md`).

#### Error Symptoms:
* `nvidia-device-plugin-mxjwh` pod in `CrashLoopBackOff`.
* `kubectl logs` with no useful content (the container never got far enough to start its process).
* `kubectl describe pod` with no relevant Events (expired due to the cluster's default ~1h TTL).
* Error reported by containerd when trying to create the container:
  ```
  failed to create containerd task: failed to create shim task: OCI runtime create failed: could not apply required modification to OCI specification: error modifying OCI spec: failed to create the automatic CDI modifier: failed to generate CDI spec for mode "auto": failed to construct device spec generators: failed to initialize NVML: Driver Not Loaded
  ```
* `nvidia-smi` run directly on the host (worker2, via SSH) failing with:
  ```
  NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.
  ```

---

### 2. Investigation & Discovery Process

The containerd error (`failed to initialize NVML: Driver Not Loaded`) already pointed away from the pod or the device-plugin itself and toward the host layer, before the container even started — the failure happens in the NVIDIA Container Toolkit while generating the CDI spec. Reproducing `nvidia-smi` directly on the host confirmed this.

From there, the investigation followed three threads: (1) did the running kernel match the kernel the module was built for? (2) does the driver use DKMS (automatic rebuild on every new kernel) or version-pinned precompiled packages? (3) what had recently changed on the host?

- `uname -r` returned `7.0.0-42-generic`, different from the `7.0.0-38-generic` documented in `clusters/worker2/config.md`.
- `dkms status` returned `dkms: command not found` — meaning this driver does **not** use DKMS. In this case, Ubuntu uses kernel-ABI-versioned `.deb` module packages (`linux-modules-nvidia-580-server-<kernel-version>`), not a dynamically rebuilt module.
- `dpkg -l | grep -i nvidia` showed `linux-modules-nvidia-580-server-7.0.0-38-generic` and `linux-modules-nvidia-580-server-generic` installed and **on hold** (`hi` flag), both pinned at version `7.0.0-38.38+2` — no module package existed for kernel `7.0.0-42`.
- `sudo dmesg | grep -iE "nvidia|xid"` showed no `NVRM` lines at all (only the card's own HDMI audio entries, handled by a separate generic sound driver) — confirming the graphics/compute module never even attempted to load, because the `.ko` file matching kernel `7.0.0-42` simply didn't exist on disk.

#### Root Cause: kernel-module package held by `apt-mark hold`, out of sync with a kernel upgrade
The `580-server` NVIDIA driver was installed on this host without DKMS, using Ubuntu's precompiled packages (`linux-modules-nvidia-580-server-*`), which are versioned per kernel ABI. During the original install, **every** package containing both `nvidia` and `580` was held via `apt-mark hold $(dpkg-query -W -f='${Package}\n' | grep -E 'nvidia.*-580')` — a deliberate precaution to prevent the GPU (GTX 970, Maxwell) from being upgraded to the `590+` branch, which doesn't support this hardware (see `ia/CLAUDE.md`, "Convenções importantes" section).

That hold, however, also pinned the `linux-modules-nvidia-580-server-generic` package (and its specific `7.0.0-38` kernel variant) at the originally installed version. The `linux-image-generic` package (the kernel itself) is **not** held, so an `apt upgrade` brought in a new kernel (`7.0.0-42`) without bringing along the matching NVIDIA module — because the module package stayed frozen at the old version due to the hold. After the reboot that applied this new kernel, no `nvidia.ko` module existed for it, and the driver was effectively "off" until someone manually installed the right package.

**Key point**: holding `nvidia-*-580` packages via `apt-mark hold` to pin the driver branch also unintentionally pins the kernel-ABI-versioned module packages — which need to be updated manually (within the same branch) every time the host's kernel version bumps.

---

### 3. Commands Used for Diagnosis

```bash
# Reproduce the error directly on the host, outside any container
nvidia-smi

# Check the currently running kernel
uname -r

# Check whether the driver uses DKMS (automatic rebuild per kernel)
dkms status

# Check the currently loaded module (no output = not loaded)
lsmod | grep nvidia

# Look for driver initialization errors in the kernel log
sudo dmesg | grep -iE "nvidia|xid" | tail -50

# List all nvidia-related packages and their hold status ("hi" flag = held + installed)
dpkg -l | grep -i nvidia

# Confirm the module package for the new kernel exists in the repo, on the same branch (580)
sudo apt update
apt-cache madison linux-modules-nvidia-580-server-7.0.0-42-generic
```

---

### 4. The Definitive Fix (or Workaround)

Installed the specific module package for the currently running kernel (`7.0.0-42-generic`) directly, without needing to unhold anything else — installing a package that wasn't previously installed isn't blocked by other packages' hold status:

```bash
sudo apt install -y linux-modules-nvidia-580-server-7.0.0-42-generic linux-objects-nvidia-580-server-7.0.0-42-generic
sudo depmod -a
sudo modprobe nvidia
```

#### Final validation
```bash
nvidia-smi
# Driver Version: 580.178.04, CUDA Version: 13.0
# GPU 0: NVIDIA GeForce GTX 970, 0MiB / 4096MiB in use, no running processes
```

The `nvidia-device-plugin` pod returned to `Running` automatically on the next `CrashLoopBackOff` retry, with no need to manually delete the pod.

---

### 5. Lessons Learned for the Homelab

1. **A containerd/OCI runtime error when creating the container (not an error from the pod itself) points outside Kubernetes** — when the failure mentions `NVML`, `CDI spec`, or "OCI runtime create failed" before the container even logs anything, the problem is on the host (driver, toolkit, kernel), not in the pod manifest or the device-plugin.
2. **Kubernetes events expire (~1h by default)** — the absence of relevant `Events` in `kubectl describe pod` doesn't mean nothing went wrong; it may just mean the event is older than the TTL. Don't rule out a root cause just because the Events section is empty.
3. **`apt-mark hold` by broad package-name `grep` can hold more than intended** — pinning `nvidia-*-580` to fix the driver branch also holds the kernel-ABI-versioned module packages, which *need* to be updated manually (within the same branch) on every host kernel upgrade. Expect this to happen again: after any `apt upgrade` that brings a new kernel on worker2, check whether `linux-modules-nvidia-580-server-$(uname -r)` exists and is installed **before** rebooting the VM, to avoid the GPU-unavailable window.
4. **A driver without DKMS uses kernel-ABI-versioned module packages** — `dkms status` returning "command not found" is the signal that rebuilding isn't automatic; every kernel bump requires the matching `linux-modules-nvidia-<branch>-<kernel-abi>` package to already be published in the repo (confirmable via `apt-cache madison`) and installed manually.
