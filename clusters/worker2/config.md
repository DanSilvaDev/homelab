# Worker2 (k3s) - Resumo de Configuração

Host físico: servidor TrueNAS dedicado (único propósito: TrueNAS + VM worker2)
Status: Worker2 ativo no cluster k3s, com GPU validada e funcional (CUDA testado com sucesso)

Ver também: [TrueNAS - config.md](../truenas/config.md)

## Host físico

### Hardware
- **CPU**: AMD FX-8350 (8 Cores, 8 Threads)
- **Placa-mãe**: ASRock 990FX Extreme3
- **Memória RAM**: 24GB DDR3 Non-ECC
- **GPU boot do host**: NVIDIA GeForce GT 210 (apenas para boot/console do TrueNAS)
- **GPU passthrough**: NVIDIA GeForce GTX 970 (isolada via PCI passthrough, dedicada à VM worker2)

### BIOS
- Overclock Mode: Manual
- CPU Frequency Multiplier: x14.0 (2.8 GHz) ou x15.0 (3.0 GHz)
- AMD Turbo Core: Disabled
- CPU Voltage Undervolt: 1.20V (ajustável)
- Economia de energia: Cool'n'Quiet, C1E, Core C6 State, APM habilitados
- IOMMU (AMD-Vi): Enabled (necessário para GPU passthrough)

### Storage TrueNAS SCALE

**Pool `ssd-apps`** (Stripe, SSD 120GB)

| Nome | Tipo | Tamanho/Recordsize | Compressão | Uso |
|---|---|---|---|---|
| `vm-k3s-disk` | zvol | 40 GiB (volblocksize 16K) | LZ4 | Disco virtual exclusivo da VM worker2 |
| `fast-data` | dataset | recordsize 128K | LZ4 | Storage rápido genérico (NFS), inclui pasta `os_images` com ISOs |
| `databases` | dataset | recordsize 16K | LZ4 | Reservado para Postgres/Redis (recordsize ajustado ao page size) |

**Pool `data`** (Stripe, HD WD Blue 1TB — upgrade futuro para IronWolf 4TB em Mirror)

| Nome | Compressão | Observação |
|---|---|---|
| `Midias` | off / lz4 | Armazenamento geral/mídias |
| `Backups` | ZSTD | Split futuro planejado em sub-datasets por fonte (ex. `smb-pc`, `nfs-k3s`) |

## VM worker2

- **Nome**: `vm_k3s_worker_2`
- **Hypervisor**: TrueNAS SCALE (KVM/QEMU)

### Sistema operacional
- Distro: Ubuntu Server 26.04.1
- Variante de instalação: Minimized
- Hostname: `worker2`
- Kernel: 7.0.0-38-generic
- IP: `192.168.3.105/24` (DHCP + reserva por MAC no roteador — sem IP estático configurado na VM, por decisão deliberada)
- Acesso SSH: por chave Ed25519 (`~/.ssh/id_ed25519_homelab` no host administrador), `PasswordAuthentication no`
- Driver GPU: `nvidia-driver-580-server`
  - Versão instalada: `580.178.04` (confirmado via `nvidia-smi`)
  - Observação: branch 580 é a última com suporte a GPUs Maxwell (GTX 970/GM204); evitar upgrade para branch 590+
  - Todos os pacotes relacionados (`libnvidia-*-580-server`, `nvidia-driver-580-server`, `nvidia-kernel-*-580-server`, etc.) travados com `apt-mark hold`
- NVIDIA Container Toolkit: `1.20.1` instalado

### CPU
- Sockets: 1
- Cores: 5
- Threads por core: 1
- CPU Mode: Host Model

### Memória
- 12 GiB

### Disco
- Tipo: Existing zvol
- Caminho: `/dev/zvol/ssd-apps/vm-k3s-disk`
- Barramento: VirtIO
- Tamanho: 40 GiB

### Rede
- Adapter Type: VirtIO
- NIC Attach: bridge `br1` (não mais `enp6s0` diretamente — ver observação abaixo)
- IP: `192.168.3.105/24` (DHCP + reserva por MAC no roteador)
- Observação: inicialmente a NIC estava anexada direto a `enp6s0` (modo macvtap), o que impedia a VM de alcançar o próprio host TrueNAS (qualquer protocolo, não só NFS). Corrigido criando a bridge `br1` no TrueNAS e reanexando a NIC da VM a ela. O endereço MAC da interface virtual mudou nessa reconfiguração — a reserva de DHCP no roteador foi atualizada para o novo MAC. Detalhes: [troubleshooting/worker2_vm_truenas_host_unreachable_macvtap.md](../../troubleshooting/worker2_vm_truenas_host_unreachable_macvtap.md)

### Display
- Tipo: VNC
- Bind Address: `0.0.0.0`
- Acesso: via cliente VNC externo (ex. TigerVNC) — não há console web nesta versão do TrueNAS SCALE

### GPU Passthrough
- Dispositivo: NVIDIA GeForce GTX 970
- Status: isolada no host (System Settings > Isolated GPU PCI Device(s))
- Observação: único consumidor da GPU neste host — sem disputa com outras VMs

### Boot
- Método: UEFI
- Mídia de instalação: `/mnt/ssd-apps/fast-data/os_images/ubuntu-26.04.1-live-server-amd64.iso`

## Contexto do cluster k3s

Topologia: cluster k3s multi-servidor, cada nó em host físico separado

| Nó | Localização |
|---|---|
| control-plane | Servidor físico separado (fora deste host) |
| worker1 | Servidor físico separado (fora deste host) |
| worker2 | Este host (TrueNAS) — em configuração |

### Próximos passos
- [x] Finalizar instalação do Ubuntu Server na VM
- [x] Configurar reserva DHCP (feita no roteador) e hostname (`worker2`)
- [x] Instalar driver NVIDIA (580-server) e travar pacotes
- [x] Instalar NVIDIA Container Toolkit
- [x] Executar `k3s agent` para ingressar no cluster existente (`v1.36.5+k3s1`, instalado manualmente via scp devido a bloqueio do provedor de internet ao domínio `githubusercontent.com`)
- [x] Configurar containerd (template k3s) para runtime NVIDIA (`--set-as-default`)
- [x] Instalar NVIDIA device plugin no cluster (via control-plane, Helm chart `nvidia-device-plugin-0.20.1`)
- [x] Validar recurso `nvidia.com/gpu` no node via `kubectl describe node worker2`
- [x] Teste funcional de GPU em Pod (CUDA n-body benchmark) — **concluído e validado**

### Validação final
Pod de teste com `resources.limits["nvidia.com/gpu"]: 1` rodou com sucesso no worker2:
```
GPU Device 0: "Maxwell" with compute capability 5.2
Compute 5.2 CUDA device: [NVIDIA GeForce GTX 970]
13312 bodies, total time for 10 iterations: 13.896 ms
= 127.524 billion interactions per second
= 2550.474 single-precision GFLOP/s
```
Pipeline completo validado: TrueNAS (zvol) → VM (Ubuntu) → driver NVIDIA 580-server → NVIDIA Container Toolkit → containerd (k3s) → device plugin → Pod com GPU alocada e funcional.

Pods de teste (`gpu-test`, `test-gpu-selector`) removidos do cluster após validação.

### NVIDIA device plugin — observação importante
O chart Helm `nvdp/nvidia-device-plugin` exige, além do `nodeSelector` customizado, que o node tenha **pelo menos um** destes labels (nodeAffinity embutida no chart, normalmente setados por NFD/GPU Operator, que não usamos aqui):
- `feature.node.kubernetes.io/pci-10de.present=true`
- `feature.node.kubernetes.io/cpu-model.vendor_id=NVIDIA`
- `nvidia.com/gpu.present=true` ← label escolhido manualmente (mais simples sem precisar instalar NFD/GPU Operator)

### Nó no cluster
- k3s version: `v1.36.5+k3s1` (igual ao control-plane — requisito do Rancher, que ainda não suporta v1.37)
- Join confirmado via `kubectl get nodes` no control-plane

## Log de comandos executados

```bash
# Setup inicial do sistema
sudo apt update && sudo apt upgrade -y
hostnamectl
sudo hostnamectl set-hostname worker2
ip a
lspci -nn | grep -i nvidia

# SSH por chave (sem senha)
sudo apt install -y nano
sudo nano /etc/ssh/sshd_config   # PasswordAuthentication no
sudo systemctl restart ssh

# Driver NVIDIA
sudo apt update
ubuntu-drivers devices
sudo apt install -y nvidia-driver-580-server
sudo reboot
nvidia-smi   # confirmado: driver 580.178.04, GTX 970 reconhecida

# Travar pacotes da branch 580 (evitar upgrade para 590+)
sudo apt-mark hold $(dpkg-query -W -f='${Package}\n' | grep -E 'nvidia.*-580')
apt-mark showhold

# NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update
sudo apt install -y nvidia-container-toolkit
nvidia-ctk --version   # confirmado: 1.20.1

# Join no cluster (binário baixado manualmente via scp, bloqueio de ISP a githubusercontent.com)
sha256sum --check <(grep k3s$ sha256sum-amd64.txt)
sudo mv k3s /usr/local/bin/k3s
sudo chmod +x /usr/local/bin/k3s
curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.36.5+k3s1" K3S_URL=https://192.168.3.101:6443 K3S_TOKEN=*** sh -s - agent --node-name worker2

# Containerd: runtime NVIDIA no template do k3s
sudo cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
sudo nvidia-ctk runtime configure --runtime=containerd --config=/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl --set-as-default
sudo systemctl restart k3s-agent
sudo grep -A3 "nvidia" /var/lib/rancher/k3s/agent/etc/containerd/config.toml   # confirmado: runtime nvidia presente

# Device plugin NVIDIA (executado no control-plane)
kubectl label node worker2 gpu=true
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash   # se Helm não instalado
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm install nvidia-device-plugin nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --create-namespace \
  --set-string nodeSelector.gpu="true"

# Fix: chart exige nodeAffinity extra (NFD/GPU Operator labels) - adicionado manualmente
kubectl label node worker2 nvidia.com/gpu.present=true

# Validação final
kubectl describe node worker2 | grep -i "gpu"   # confirmado: nvidia.com/gpu em Capacity/Allocatable
kubectl apply -f gpu-test.yaml                  # pod CUDA n-body benchmark
kubectl logs gpu-test                           # confirmado: 2550.474 GFLOP/s na GTX 970
kubectl delete pod gpu-test test-gpu-selector   # limpeza dos pods de teste
```
