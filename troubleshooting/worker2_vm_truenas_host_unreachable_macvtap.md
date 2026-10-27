# Troubleshooting: VM worker2 não alcança o host TrueNAS (macvtap)

Documenta por que a VM worker2 (hospedada no TrueNAS SCALE) não conseguia alcançar o próprio host TrueNAS em nenhum protocolo de rede, bloqueando o uso de storage NFS para a stack de monitoring. Ocorreu durante a configuração do storage para o Rancher Monitoring (kube-prometheus-stack).

---

## 1. O Cenário Inicial e Sintomas

Ao validar se o export NFS criado no TrueNAS (`ssd-apps/k8s`) estava acessível pelos 3 nodes do cluster antes de instalar o `nfs-subdir-external-provisioner`, o teste falhou especificamente no worker2.

### Sintomas do Erro:
* `showmount -e 192.168.3.103` funcionou normalmente no control-plane e no worker1 (nodes físicos), mas falhou no worker2 com `clnt_create: RPC: Unable to receive`.
* `ping -c 3 192.168.3.103` a partir do worker2 retornava `Destination Host Unreachable` em 100% dos pacotes.
* `nc -zv 192.168.3.103 111` e `nc -zv 192.168.3.103 2049` a partir do worker2 retornavam `No route to host`.
* O worker2 conseguia alcançar normalmente os outros dois nodes do cluster (control-plane, worker1) — o problema era isolado ao host TrueNAS especificamente.
* NFSv3 e NFSv4 já estavam habilitados no TrueNAS, descartando causa de versão de protocolo.

---

## 2. Processo de Investigação & Descoberta

Passos de descarte: confirmado que o NFS estava corretamente configurado e acessível a partir de qualquer node físico, e que o NFSv3 (necessário para `showmount`) estava habilitado — isso isolou o problema à rede específica do worker2, não ao compartilhamento em si.

### Causa Raiz: NIC da VM anexada diretamente à interface física (modo macvtap)
A VM worker2 tinha sua NIC virtual anexada diretamente à interface física `enp6s0` do TrueNAS (em vez de uma bridge dedicada). O TrueNAS SCALE implementa esse tipo de anexo via **macvtap**. Esse modo tem uma limitação conhecida: o tráfego entre a VM e o host que a hospeda (quando ambos compartilham a mesma porta física) precisaria ser refletido pelo switch de volta pela mesma porta de origem ("hairpin") — e a maioria dos switches (incluindo switches domésticos) não faz isso por padrão. Isso explica por que o worker2 alcançava normalmente outros hosts da LAN (tráfego sai e entra por portas diferentes do switch), mas não o próprio TrueNAS.

**Ponto-chave**: essa limitação não é específica de NFS — qualquer protocolo IP entre a VM e o host que a hospeda é afetado quando a NIC está anexada diretamente à interface física.

> Observação: essa é a explicação técnica mais consistente com todos os sintomas observados e foi a hipótese usada para guiar a correção, mas não foi confirmada via captura de pacotes — não é 100% certeza absoluta, apenas a causa mais provável.

---

## 3. Comandos Aplicados para o Diagnóstico

```bash
# No control-plane e worker1 (funcionou normalmente)
showmount -e 192.168.3.103

# No worker2
sudo apt install -y nfs-common      # showmount não estava disponível
showmount -e 192.168.3.103
# -> clnt_create: RPC: Unable to receive

sudo apt install -y iputils-ping    # ping não estava disponível
ping -c 3 192.168.3.103
# -> Destination Host Unreachable (100% packet loss)

nc -zv 192.168.3.103 111            # porta do rpcbind/portmapper
nc -zv 192.168.3.103 2049           # porta do NFS
# -> No route to host (ambos)
```

---

## 4. A Correção Definitiva

Duas tentativas iniciais falharam antes da correção funcionar:
- 1ª tentativa: criar uma Bridge (`br1`) com `enp6s0` como membro, mantendo DHCP — falhou (TrueNAS ficou inacessível durante o "Test Changes" e reverteu automaticamente). Hipótese: atraso de STP + negociação DHCP não coube na janela de teste.
- 2ª tentativa: recriar a bridge com IP estático e sem STP/learning — ainda falhou, com a VM worker2 ligada durante a aplicação da mudança.

Correção que funcionou:
1. **Desligar a VM worker2 antes de qualquer mudança de rede no host** — essa foi a etapa que faltava nas tentativas anteriores; aplicar a reconfiguração de rede do host com a VM ligada (ainda presa à interface física antiga) causava falha e rollback automático.
2. Criar a interface **Bridge** `br1` no TrueNAS (Network → Interfaces), com `enp6s0` como membro e **IP estático** `192.168.3.103/24` (em vez de DHCP).
3. Remover a configuração de IP da interface `enp6s0` (ela passa a atuar só como membro da bridge).
4. Aplicar via "Test Changes" no TrueNAS e confirmar com "Keep Changes" dentro da janela de segurança.
5. Editar o dispositivo de rede da VM worker2, trocando o anexo de `enp6s0` diretamente para a nova bridge `br1`.
6. Ligar a VM novamente.
7. A interface virtual da VM passou a expor um **endereço MAC diferente** após essa reconfiguração — foi necessário atualizar a reserva de DHCP no roteador para o novo MAC, mantendo o IP `192.168.3.105` para o worker2.

### Validação final
```bash
# No worker2, após reiniciar com a NIC anexada à br1
ping -c 3 192.168.3.103     # OK
showmount -e 192.168.3.103  # lista /mnt/ssd-apps/k8s normalmente
```

---

## 5. Lições Aprendidas para o Homelab

1. **NIC de VM anexada direto a uma interface física (macvtap) não permite comunicação com o próprio host que a hospeda** — mesmo alcançando o resto da LAN normalmente. Qualquer VM que precise falar com o host (TrueNAS, Proxmox, etc.) deve usar uma Bridge Linux de verdade.
2. **Mudanças de rede no host devem ser aplicadas com a VM desligada**, quando essa VM está anexada à interface que está sendo reconfigurada — aplicar com a VM ligada causou falhas repetidas aqui.
3. **Reconfigurar a NIC de uma VM pode trocar o endereço MAC exposto** — sempre conferir e atualizar reservas de DHCP no roteador depois de mudanças desse tipo.
4. **A proteção "Test Changes" (rollback automático) do TrueNAS é essencial** antes de aplicar mudanças de rede no host — evitou perda de acesso permanente nas duas tentativas que falharam.
5. **Essa limitação bloqueia qualquer workload no worker2 que dependa do TrueNAS**, não só NFS — relevante para planejar serviços futuros (ex: Jellyfin acessando mídia do TrueNAS) que rodem nesse node.
