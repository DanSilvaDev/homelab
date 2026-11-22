# Worker3 (k3s) - Resumo de Configuração

Host físico: desktop dedicado (bare metal, sem virtualização)
Status: **No cluster como `agent` desde 2026-11-22** (`v1.36.5+k3s1`, ROLES `worker`). GPU exposta no k8s via generic-device-plugin (`devic.es/amdgpu`) e **Jellyfin rodando aqui com VA-API** desde 2026-11-22 (fase 2 concluída).

Papel planejado: nó "GPU de verdade" do cluster — Jellyfin (migra do worker2), streaming de jogos (Wolf/Moonlight) e IA local (Ollama/Open WebUI). Plano completo e decisões: [ideas/worker3_gpu_node.md](../../ideas/worker3_gpu_node.md).

## Hardware

- **CPU**: AMD Ryzen 7 5700X (Zen 3, 8C/16T, sem vídeo integrado, PPT padrão 76 W)
- **Placa-mãe**: ASUS TUF GAMING B550M-PLUS (micro-ATX), BIOS `3636` (2026-01-04)
- **Memória RAM**: 32 GB DDR4
- **GPU**: AMD Radeon RX 9070 16 GB (Navi 48 / RDNA4, PCI `1002:7550`, slot `09:00.0`)
- **Storage**: NVMe Lexar NM790 512 GB (upgrade para disco maior planejado)
- **Fans do gabinete**: 3x Corsair RS120 (PWM) — 2 de entrada na base (`CHA_FAN2`), 1 de exaustão na traseira (pressão positiva)

### BIOS (configurada em 2026-11-21)

| Opção | Caminho | Valor |
|---|---|---|
| CPPC | Advanced → AMD CBS → NBIO Common Options → SMU Common Options | Enabled (confirmado: driver `amd-pstate-epp` ativo no Linux) |
| Above 4G Decoding / Re-Size BAR Support | Advanced → PCI Subsystem Settings | Enabled |
| Launch CSM | Boot → CSM | Disabled (só UEFI) |
| Power On By PCI-E (Wake-on-LAN) | Advanced → APM Configuration | Enabled |
| ErP Ready | Advanced → APM Configuration | **Disabled** — nas placas ASUS, ErP ligado desativa o "Power On By PCI-E" (mata o WoL) |
| Global C-state Control / Power Supply Idle Control | não encontrados nos caminhos esperados (AMD CBS → CPU Common Options) | ficaram em **Auto** — C-states ativos mesmo assim: `cpuidle` expõe `POLL`, `C1`, `C2` (confirmado em 2026-11-22; no Zen 3 o `C2` do ACPI é o estado profundo do core) |
| Q-Fan (fans de chassis) | Advanced → Monitor → Q-Fan Configuration | PWM Mode, fonte **Multiple Sources (CPU + Motherboard)**, Step Up 12 s (menor valor disponível), Step Down 25 s, curva Manual: 45 °C → 35%, 65 °C → 50%, 80 °C → 100% |

Adiados (medir consumo antes): PCIe ASPM (histórico de instabilidade de NIC no worker1), Eco Mode/Curve Optimizer, Restore AC Power Loss (depende do plano de nobreak/NUT).

## Sistema operacional

- Distro: Ubuntu Server 26.04.1 LTS (mesma do control-plane/worker1/worker2) — reinstalado em 2026-11-21: a primeira instalação foi por engano com Ubuntu 24.04.5 (kernel 6.8), em que o `amdgpu` falhava com `Fatal error during GPU init` / `probe ... failed with error -22` (kernel antigo demais para RDNA4)
- Kernel: `7.0.0-44-generic` (os outros nodes estão em `7.0.0-43` — diferença só de patch)
- Hostname: `worker3`
- Usuário: `kubecluster` (mesmo do control-plane)
- IP: `192.168.3.106/24` na `enp6s0` (DHCP + reserva por MAC no roteador)
- NIC: Realtek RTL8125 2.5GbE (`10ec:8125`, slot `06:00.0`), MAC `b0:82:e2:4c:5a:d2`
- Wake-on-LAN no Linux: a placa suporta (`Supports Wake-on: pumbg`), mas vinha em `Wake-on: d` (desativado) mesmo com o WoL ligado na BIOS. Persistido com `wakeonlan: true` no netplan (gera `WakeOnLan=magic` em `/run/systemd/network/10-netplan-enp6s0.link`, aplicado no boot) em 2026-11-22
- Acesso SSH: por chave Ed25519 (`~/.ssh/id_ed25519_homelab` no host administrador); senha desabilitada via drop-in `/etc/ssh/sshd_config.d/00-disable-password.conf` (`PasswordAuthentication no`, `KbdInteractiveAuthentication no`) — o prefixo `00-` garante precedência sobre o `50-cloud-init.conf` (no sshd vale o primeiro valor lido)
- Driver de CPU: `amd-pstate-epp`, EPP padrão `balance_performance` (ajuste para `balance_power` previsto na fase de energia)
- Sem `ufw` (instalação Minimized)

### GPU

- Driver `amdgpu` do kernel (in-tree) — sem DKMS, sem ROCm no host (o userspace ROCm/Mesa vai dentro das imagens dos containers)
- Device files: `/dev/dri/cardN` (o índice **não é estável** entre boots — era `card1` na instalação, sumiu após um reboot; localizar com `ls -l /sys/class/drm/ | grep -E 'card[0-9] '`), `/dev/dri/renderD128` (VA-API/Vulkan), `/dev/kfd` (ROCm) — os dois últimos são `root:render 660`, grupo `render` = **GID 991**
- Firmware carregado: DMUB via PSP (`0x0A000800`), VCN (ENC 1.12 / DEC 9)
- `Failed to setup vendor infoframe on connector HDMI-A-1: -22` no `dmesg` — inofensivo (monitor conectado durante a instalação)
- Diferente do worker2 (NVIDIA): **não precisa** de runtime customizado no containerd

### GPU no Kubernetes (fase 2, 2026-11-22)

- Label do node: **`gpu-vendor=amd`** — não usar `gpu=true` (é o `nodeSelector` do `nvidia-device-plugin` e do `dcgm-exporter`, que subiriam aqui)
- Device plugin: [`infrastructure/amd-gpu/generic-device-plugin.yaml`](../../infrastructure/amd-gpu/generic-device-plugin.yaml) (`squat/generic-device-plugin:f7c3291`, DaemonSet no `kube-system`, restrito a `gpu-vendor=amd`)
  - Recurso **`devic.es/amdgpu`**, `count: 3` (allocatable `3`) — cada alocação injeta `/dev/dri/renderD128` + `/dev/kfd`; até 3 pods compartilham a placa (Jellyfin + Ollama + Wolf), sem isolamento
  - O domínio default do plugin mudou de `squat.ai` (ainda citado no README do projeto) para `devic.es` — fixado com `--domain devic.es` para um bump de imagem não renomear o recurso
  - Escolhido em vez do plugin oficial da AMD (`amd.com/gpu`, 1 GPU = 1 pod) e de pod `privileged` com hostPath
- Pods que não rodam como root precisam de `securityContext.supplementalGroups: [991]`
- Validação VA-API ([`apps/gpu-test/amd-vaapi-test-pod.yaml`](../../apps/gpu-test/amd-vaapi-test-pod.yaml), mesma imagem/usuário do Jellyfin): `Mesa Gallium driver 26.0.8 for AMD Radeon RX 9070 (radeonsi, gfx1201)` — vem dentro da imagem (`jellyfin-ffmpeg`), nada instalado no host
  - Decode: H.264, HEVC 8/10-bit, VP9 8/10-bit, AV1 8/10-bit, JPEG
  - Encode: H.264, HEVC 8/10-bit, AV1 8-bit
- **Monitorar uso de vídeo**: `gpu_busy_percent` mede só o bloco 3D (GFX) e fica ~0 durante transcode. O uso do VCN aparece por processo no fdinfo do `amdgpu`: `sudo grep -hE 'drm-engine|drm-memory-vram' /proc/<pid-do-ffmpeg>/fdinfo/*` (contadores `drm-engine-enc`/`dec` em ns crescendo). Painel de verdade fica para a fase 5 (observabilidade de energia)
- **Jellyfin** migrado para cá — ver [Media Stack - config.md](../media-stack/config.md)

### Storage (LVM)

| Item | Tamanho | Uso |
|---|---|---|
| `nvme0n1p1` | 1 GB | `/boot/efi` |
| `nvme0n1p2` | 2 GB | `/boot` |
| `ubuntu-vg/ubuntu-lv` | 100 GiB | `/` — sistema, k3s, imagens de container (crescer online com `lvextend -r` se precisar) |
| VG livre | ~374 GiB | reservado para um LV separado de Steam + modelos de IA (evita `DiskPressure` no `/`) |

## Contexto do cluster k3s

| Nó | IP | Papel |
|---|---|---|
| control-plane (`kubecluster-lenovo-z40-70`) | `192.168.3.100` (VIP da API: `192.168.3.101`) | server (`control-plane,etcd,master`), datastore etcd embutido (migrado de SQLite em 2026-11-22) |
| worker1 | `192.168.3.102` | agent (`worker`) |
| worker2 | `192.168.3.105` | agent (`worker`, GTX 970, label `gpu=true`) — sem Jellyfin desde 2026-11-22 |
| worker3 | `192.168.3.106` | agent (`worker`, RX 9070, label `gpu-vendor=amd`) — ingressou em 2026-11-22, apontando para o VIP `https://192.168.3.101:6443`; roda o Jellyfin |

Desligamento noturno: o worker3 entra na mesma janela de desligamento da madrugada que worker1, worker2 e o TrueNAS (decisão de 2026-11-22) — ver "Gerenciamento de energia" em [`ia/CLAUDE.md`](../../ia/CLAUDE.md). Religar via Wake-on-LAN (BIOS + netplan; testado em 2026-11-22 com magic packet enviado do control-plane).

Sem taint no worker3 por enquanto: aceita qualquer workload como um worker comum (32 GB dão conta da media stack + o que vier). Decisão de 2026-11-22, na fase 2 — reavaliar nas fases 3/4, quando Ollama e Wolf passarem a disputar VRAM/RAM.

### Próximos passos
- [x] BIOS (CPPC, Above 4G/ReBAR, CSM off, WoL, ErP off, curva dos fans)
- [x] Ubuntu Server 26.04.1, hostname, reserva DHCP, SSH por chave
- [x] GPU reconhecida pelo kernel (`renderD128`, `/dev/kfd`)
- [x] IP `192.168.3.106` autorizado nos 3 exports NFS do TrueNAS (`ssd-apps/k8s`, `data/k8s`, `data/midias`)
- [x] Backup do datastore SQLite no control-plane antes do join (`/root/k3s-backup-pre-worker3/`)
- [x] Instalar `nfs-common` e validar acesso aos exports — mount de teste `OK` nos 3 (`ssd-apps/k8s`, `data/k8s`, `data/midias`) em 2026-11-22
- [x] Join como `agent` (`v1.36.5+k3s1`, binário copiado do control-plane via `scp -3`) — `Ready` em 2026-11-22
- [x] Label `node-role.kubernetes.io/worker` (worker2 e worker3 — o worker2 estava sem, ao contrário do worker1)
- [x] Pods DaemonSet no worker3 todos `Running` (`alloy`, `node-exporter`, `loki-canary`) e `k3s-agent` `active (running)` (2026-11-22)
- [x] Wake-on-LAN: persistido no netplan, `Wake-on: g` confirmado após o reboot (e `k3s-agent` voltou sozinho); testado em 2026-11-22: `poweroff`, ~2 min, magic packet enviado do control-plane → node ligou e voltou a `Ready`
- [x] Drop-in do SSH confirmado após a reinstalação (`sshd -T`: `passwordauthentication no`, `kbdinteractiveauthentication no`)
- [x] Migração SQLite → etcd no control-plane (2026-11-22) — ver [control_plane_1/config.md](../control_plane_1/config.md)
- [x] Confirmar C-states (`cpuidle`: `POLL`, `C1`, `C2`)
- [x] Fase 2: GPU no k8s (generic-device-plugin, `devic.es/amdgpu`) + Jellyfin migrado do worker2 com VA-API e cache local (2026-11-22) — transcode validado (`h264 -> hevc (hevc_vaapi)`, `scale_vaapi`)
- [ ] Fase 3: Ollama + Open WebUI — ver [ideas/worker3_gpu_node.md](../../ideas/worker3_gpu_node.md)

## Log de comandos executados

```bash
# Diagnóstico pós-instalação (2026-11-21)
hostnamectl; uname -r
ip -br a
lsblk -f; sudo vgs; sudo lvs
lspci -nn | grep -Ei 'vga|display'
ls -l /dev/dri /dev/kfd
sudo dmesg | grep -iE 'amdgpu.*(firmware|error|fail)|gc_12' | head
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_driver                     # amd-pstate-epp
cat /sys/devices/system/cpu/cpufreq/policy0/energy_performance_preference      # balance_performance

# SSH: senha desabilitada via drop-in
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/00-disable-password.conf
sudo sshd -t
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication'
sudo systemctl restart ssh

# Join no cluster (2026-11-22)
# [PC, Git Bash] binário copiado do control-plane (ISP bloqueia *.githubusercontent.com)
scp -3 kubecluster@192.168.3.100:/usr/local/bin/k3s kubecluster@192.168.3.106:~/
# [worker3]
sudo install -m 755 ~/k3s /usr/local/bin/k3s && rm ~/k3s
k3s --version
sudo -v
read -rs K3S_TOKEN && export K3S_TOKEN        # token lido de /var/lib/rancher/k3s/server/node-token no control-plane; não fica no histórico
curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.36.5+k3s1"   K3S_URL=https://192.168.3.101:6443 sh -s - agent --node-name worker3
unset K3S_TOKEN
# [control-plane]
kubectl get nodes -o wide
kubectl label node worker2 worker3 node-role.kubernetes.io/worker=
kubectl get pods -A -o wide --field-selector spec.nodeName=worker3   # alloy, node-exporter, loki-canary Running
# [worker3] validação pós-join
systemctl status k3s-agent --no-pager | head -5
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication'
grep . /sys/devices/system/cpu/cpu0/cpuidle/state*/name                  # POLL, C1, C2
dpkg -s nfs-common >/dev/null 2>&1 && echo "nfs-common OK"
for p in /mnt/ssd-apps/k8s /mnt/data/k8s /mnt/data/midias; do
  sudo mount -t nfs 192.168.3.103:$p /mnt && echo "OK $p" && sudo umount /mnt || echo "FAIL $p"
done                                                                     # 3x OK

# Wake-on-LAN (2026-11-22)
sudo ethtool enp6s0 | grep -i wake                                       # Supports Wake-on: pumbg / Wake-on: d
F=/etc/netplan/<arquivo-do-subiquity>.yaml
sudo cp "$F" ~/netplan-backup.yaml
sudo sed -i '/set-name: enp6s0/a\      wakeonlan: true' "$F"
sudo netplan generate
sudo cat /run/systemd/network/10-netplan-enp6s0.link                     # WakeOnLan=magic
sudo reboot
sudo ethtool enp6s0 | grep 'Wake-on:'                                    # Wake-on: g
systemctl is-active k3s-agent                                            # active
sudo poweroff
# [control-plane] magic packet para o MAC do worker3 (python3 do sistema, sem instalar nada)
python3 -c 'import socket;p=b"\xff"*6+bytes.fromhex("b082e24c5ad2")*16;s=socket.socket(socket.AF_INET,socket.SOCK_DGRAM);s.setsockopt(socket.SOL_SOCKET,socket.SO_BROADCAST,1);s.sendto(p,("192.168.3.255",9))'
kubectl get nodes -w                                                     # worker3 NotReady -> Ready

# Fase 2 - GPU no k8s + Jellyfin (2026-11-22)
# [worker3] GID dos device files
stat -c '%n %U:%G %a gid=%g' /dev/dri/renderD128 /dev/kfd               # root:render 660 gid=991
# [PC, Git Bash, raiz do repo]
curl -s "https://hub.docker.com/v2/repositories/squat/generic-device-plugin/tags?page_size=15" | grep -oE '"name":"[^"]+"'
kubectl label node worker3 gpu-vendor=amd
kubectl apply -f infrastructure/amd-gpu/generic-device-plugin.yaml
kubectl get node worker3 -o jsonpath='{.status.allocatable}{"\n"}'      # devic.es/amdgpu: 3
kubectl apply -f apps/gpu-test/amd-vaapi-test-pod.yaml
MSYS_NO_PATHCONV=1 kubectl exec amd-vaapi-test -- /usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
kubectl delete -f apps/gpu-test/amd-vaapi-test-pod.yaml
kubectl apply -f apps/jellyfin/cache-local-pvc.yaml
kubectl apply -f apps/jellyfin/deployment.yaml
kubectl -n media rollout status deploy/jellyfin-deployment --timeout=300s
# Jellyfin UI: Painel -> Reprodução -> Transcodificação -> VAAPI, /dev/dri/renderD128
# Validação do transcode pelo log do ffmpeg
MSYS_NO_PATHCONV=1 kubectl -n media exec deploy/jellyfin-deployment -- sh -c 'f=$(ls -t /config/log/FFmpeg.Transcode-* | head -1); grep -m1 -A6 "Stream mapping" "$f"'
```
