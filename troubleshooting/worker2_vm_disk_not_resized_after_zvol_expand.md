# Troubleshooting: VM worker2 não enxergava o zvol ampliado, filesystem raiz quase cheio / worker2 VM didn't see the expanded zvol, root filesystem nearly full

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

O zvol `vm-k3s-disk` (disco da VM worker2) havia sido ampliado de 40GiB para 50GiB no TrueNAS, mas o Linux dentro da VM nunca refletiu esse espaço — o filesystem raiz chegou a 82% de uso (19GiB no total) e disparou `FreeDiskSpaceFailed`/`ImageGCFailed` no kubelet. Resolvido estendendo partição, LVM e filesystem dentro da VM, numa cadeia com duas causas, não só uma.

---

### 1. O Cenário Inicial e Sintomas

Esse problema já estava registrado como [known-issue](../known-issues/worker2_vm_disk_not_resized_after_zvol_expand.md) desde a ampliação do zvol no TrueNAS (40→50GiB), sem diagnóstico detalhado ainda. Ele voltou à tona quando o kubelet do worker2 passou a disparar eventos repetidos de espaço insuficiente no filesystem de imagens (ver [known-issue relacionado](../known-issues/worker2_image_filesystem_low_disk_space.md)), e os dois foram investigados juntos.

#### Sintomas do Erro:
* `df -h /` mostrava o filesystem raiz com só **19G de tamanho total**, 14G usados, 82% de uso — bem menor que os 50GiB já disponíveis no zvol.
* Eventos do Kubernetes no node `worker2`:
  ```
  Warning   FreeDiskSpaceFailed   Node/worker2   Insufficient free disk space on the node's image filesystem (88% of 18.0 GiB used)...
  Warning   ImageGCFailed         Node/worker2   Insufficient free disk space on the node's image filesystem (88% of 18.0 GiB used)...
  ```

---

### 2. Processo de Investigação & Descoberta

`lsblk` revelou a cadeia completa de camadas entre o disco físico (zvol) e o filesystem montado: `vda` (disco, partição GPT) → `vda3` (partição, LVM PV) → `ubuntu-vg` (Volume Group) → `ubuntu-lv` (Logical Volume, ext4). Cada camada precisa ser estendida individualmente — redimensionar só uma não propaga pra próxima.

```
vda                       50G  (zvol já reconhecido corretamente no bloco)
|-vda1                     1G  /boot/efi
|-vda2                     2G  /boot
`-vda3                  36.9G  LVM2_member
  `-ubuntu--vg-ubuntu--lv 18.5G  ext4, montado em /
```

#### Causa Raiz: duas lacunas de espaço empilhadas, não uma só
1. **~10GiB nunca particionados**: `vda1` (1G) + `vda2` (2G) + `vda3` (36.9G) somam só ~39.9GiB — a partição `vda3` nunca foi estendida depois da ampliação do zvol de 40 para 50GiB, deixando os ~10GiB novos como espaço bruto não particionado no fim do disco.
2. **~18.4GiB já não alocados à LV, independente do zvol**: dentro do próprio `vda3` (36.9G de PV), a LV `ubuntu-vg/ubuntu-lv` só usava 18.5G — sobravam ~18.4G dentro do Volume Group que nunca tinham sido atribuídos à Logical Volume, nem antes da ampliação do zvol. Esse é o padrão comum do instalador do Ubuntu Server (não aloca 100% do VG pra LV raiz por padrão).

**Ponto-chave**: a ampliação do zvol no TrueNAS só resolve a camada mais externa (o disco virtual). Entre o disco e o filesystem montado existem a partição, o PV, o VG e a LV — cada uma precisa do seu próprio comando de resize, em sequência, de fora pra dentro.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirmar o tamanho real do filesystem raiz e identificar as camadas (partição/LVM)
df -h /
lsblk -f
```

---

### 4. A Correção Definitiva (ou Workaround)

Cadeia completa de resize, de fora pra dentro — partição → PV (LVM) → LV (com resize do filesystem embutido via `-r`):

```bash
# 1. Estende a partição 3 pra usar o espaço bruto não particionado no fim do disco
sudo growpart /dev/vda 3

# 2. Estende o Physical Volume (LVM) pra reconhecer a partição maior
sudo pvresize /dev/vda3

# 3. Estende a Logical Volume pra usar TODO o espaço livre do Volume Group
#    (o que já estava livre + o que veio da ampliação do zvol) e já
#    redimensiona o filesystem ext4 automaticamente (-r)
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

#### Validação final
```bash
df -h /
# Filesystem                         Size  Used Avail Use% Mounted on
# /dev/mapper/ubuntu--vg-ubuntu--lv   47G   14G   31G  32% /

kubectl describe node worker2 | grep -A 5 "DiskPressure"
# DiskPressure   False   ...   KubeletHasNoDiskPressure   kubelet has no disk pressure
```
Filesystem raiz foi de 19G (82% de uso) para 47G (32% de uso). `DiskPressure` confirmado `False`, sem mais eventos de `FreeDiskSpaceFailed`/`ImageGCFailed` — resolveu também o known-issue do filesystem de imagens, já que eram a mesma causa raiz.

---

### 5. Lições Aprendidas para o Homelab

1. **Ampliar o zvol no TrueNAS é só o primeiro passo, não o último** — o disco virtual cresce na hora, mas o Linux guest precisa de uma cadeia própria de comandos (`growpart` → `pvresize`/`pvresize` → `lvextend`/`resize2fs`) pra cada camada entre o bloco e o filesystem montado. Sem isso, o espaço novo fica "invisível" indefinidamente.
2. **Em VMs com LVM, sempre checar quanto do VG está realmente alocado à LV antes de assumir que o problema é só o tamanho do disco** — nesse caso, ~18GiB já estavam disponíveis dentro do próprio VG, sem relação nenhuma com o zvol, só porque o instalador do Ubuntu nunca aloca 100% do VG pra LV raiz por padrão.
3. **`lvextend -l +100%FREE -r` resolve LV e filesystem num comando só** — mais seguro que rodar `lvextend` e `resize2fs`/`xfs_growfs` separados, já que o `-r` detecta o filesystem certo e evita o erro de rodar o comando de resize errado pro tipo de filesystem.
4. **Um known-issue de storage pode ser a causa raiz escondida de outro sintoma aparentemente não relacionado** — o alerta de `FreeDiskSpaceFailed`/`ImageGCFailed` do kubelet parecia um problema isolado de gerenciamento de imagens, mas era só um efeito colateral do filesystem raiz nunca ter sido redimensionado.

---

## English

The `vm-k3s-disk` zvol (worker2 VM's disk) had been expanded from 40GiB to 50GiB on TrueNAS, but the Linux guest never reflected that space — the root filesystem reached 82% usage (19GiB total) and triggered `FreeDiskSpaceFailed`/`ImageGCFailed` on the kubelet. Fixed by extending the partition, LVM, and filesystem inside the VM, in a chain with two stacked causes, not just one.

---

### 1. Initial Scenario and Symptoms

This problem was already tracked as a [known issue](../known-issues/worker2_vm_disk_not_resized_after_zvol_expand.md) since the zvol expansion on TrueNAS (40→50GiB), without detailed diagnosis yet. It resurfaced when the worker2 kubelet started firing repeated insufficient-space events on the image filesystem (see [related known issue](../known-issues/worker2_image_filesystem_low_disk_space.md)), and both were investigated together.

#### Error Symptoms:
* `df -h /` showed the root filesystem with only **19G total size**, 14G used, 82% usage — much smaller than the 50GiB already available on the zvol.
* Kubernetes events on node `worker2`:
  ```
  Warning   FreeDiskSpaceFailed   Node/worker2   Insufficient free disk space on the node's image filesystem (88% of 18.0 GiB used)...
  Warning   ImageGCFailed         Node/worker2   Insufficient free disk space on the node's image filesystem (88% of 18.0 GiB used)...
  ```

---

### 2. Investigation & Discovery Process

`lsblk` revealed the full chain of layers between the physical disk (zvol) and the mounted filesystem: `vda` (disk, GPT partition) → `vda3` (partition, LVM PV) → `ubuntu-vg` (Volume Group) → `ubuntu-lv` (Logical Volume, ext4). Each layer needs to be extended individually — resizing just one does not propagate to the next.

```
vda                       50G  (zvol already correctly recognized at the block level)
|-vda1                     1G  /boot/efi
|-vda2                     2G  /boot
`-vda3                  36.9G  LVM2_member
  `-ubuntu--vg-ubuntu--lv 18.5G  ext4, mounted on /
```

#### Root Cause: two stacked space gaps, not just one
1. **~10GiB never partitioned**: `vda1` (1G) + `vda2` (2G) + `vda3` (36.9G) add up to only ~39.9GiB — partition `vda3` was never extended after the zvol was expanded from 40 to 50GiB, leaving the new ~10GiB as raw unpartitioned space at the end of the disk.
2. **~18.4GiB already not allocated to the LV, unrelated to the zvol**: inside `vda3` itself (36.9G of PV), the LV `ubuntu-vg/ubuntu-lv` only used 18.5G — about ~18.4G inside the Volume Group had never been assigned to the Logical Volume, even before the zvol expansion. This is Ubuntu Server's installer default pattern (it doesn't allocate 100% of the VG to the root LV by default).

**Key point**: expanding the zvol on TrueNAS only fixes the outermost layer (the virtual disk). Between the disk and the mounted filesystem sit the partition, the PV, the VG, and the LV — each needs its own resize command, in sequence, from the outside in.

---

### 3. Commands Used for Diagnosis

```bash
# Confirm the real root filesystem size and identify the layers (partition/LVM)
df -h /
lsblk -f
```

---

### 4. The Definitive Fix (or Workaround)

Full resize chain, from the outside in — partition → PV (LVM) → LV (with filesystem resize built in via `-r`):

```bash
# 1. Extend partition 3 to use the raw unpartitioned space at the end of the disk
sudo growpart /dev/vda 3

# 2. Extend the Physical Volume (LVM) to recognize the larger partition
sudo pvresize /dev/vda3

# 3. Extend the Logical Volume to use ALL free space in the Volume Group
#    (what was already free + what came from the zvol expansion) and
#    automatically resize the ext4 filesystem at the same time (-r)
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

#### Final validation
```bash
df -h /
# Filesystem                         Size  Used Avail Use% Mounted on
# /dev/mapper/ubuntu--vg-ubuntu--lv   47G   14G   31G  32% /

kubectl describe node worker2 | grep -A 5 "DiskPressure"
# DiskPressure   False   ...   KubeletHasNoDiskPressure   kubelet has no disk pressure
```
The root filesystem went from 19G (82% usage) to 47G (32% usage). `DiskPressure` confirmed `False`, with no more `FreeDiskSpaceFailed`/`ImageGCFailed` events — this also resolved the image filesystem known issue, since both shared the same root cause.

---

### 5. Lessons Learned for the Homelab

1. **Expanding the zvol on TrueNAS is only the first step, not the last** — the virtual disk grows immediately, but the Linux guest needs its own chain of commands (`growpart` → `pvresize` → `lvextend`/`resize2fs`) for each layer between the block device and the mounted filesystem. Without that, the new space stays "invisible" indefinitely.
2. **On LVM-based VMs, always check how much of the VG is actually allocated to the LV before assuming the problem is just disk size** — in this case, ~18GiB were already available inside the VG itself, unrelated to the zvol, simply because Ubuntu's installer doesn't allocate 100% of the VG to the root LV by default.
3. **`lvextend -l +100%FREE -r` resolves the LV and the filesystem in a single command** — safer than running `lvextend` and `resize2fs`/`xfs_growfs` separately, since `-r` detects the right filesystem and avoids running the wrong resize command for the filesystem type.
4. **A storage known issue can be the hidden root cause of another, seemingly unrelated symptom** — the kubelet's `FreeDiskSpaceFailed`/`ImageGCFailed` alert looked like an isolated image-management problem, but it was just a side effect of the root filesystem never having been resized.
