# Diário de Troubleshooting: Recuperação e Downgrade de Cluster K3s + Rancher + Kube-VIP

Este documento serve como um guia de referência pós-incidente para consultas futuras e documentação de arquitetura do Homelab. Ele detalha o ciclo de upgrade malsucedido, os conflitos de compatibilidade encontrados e os procedimentos passo a passo adotados para estabilizar o ambiente na versão estável correta.

---

## 1. O Cenário Inicial e o Gatilho do Problema
O incidente começou após a tentativa de atualização do cluster **K3s para a versão v1.37**. Ao tentar rodar manifestos YAML a partir da máquina de trabalho local, o cliente `kubectl` passou a rejeitar os comandos com a seguinte mensagem de erro de validação local:
```bash
error validating data: failed to download openapi
```

### Causa Raiz do Erro de OpenAPI
O cache local de esquemas OpenAPI do `kubectl` na máquina de desenvolvimento ficou corrompido ou dessincronizado em relação às novas definições de API trazidas pela v1.37 do plano de controle.

---

## 2. Linha do Tempo de Problemas Encontrados e Resoluções

### Problema 1: Invalidação de Credenciais Locais (`401 Unauthorized`)
Após forçar a limpeza do cache de esquemas local com o comando `rm -rf ~/.kube/cache/schema`, o cliente passou a reportar um erro de falta de credenciais do servidor:
```bash
the server has asked for the client to provide credentials
```
* **Causa:** Durante a reemissão e reconfiguração de certificados TLS nos nós do cluster (Master e Worker), as chaves criptográficas contidas no arquivo `config` da máquina local tornaram-se inválidas perante o novo plano de controle do K3s.
* **Resolução:** Foi necessário acessar o nó Master via SSH, extrair o arquivo de configuração atualizado em `/etc/rancher/k3s/k3s.yaml`, substituir o IP de loopback (`127.0.0.1`) pelo IP real do nó e atualizar o arquivo local `~/.kube/config`.

### Problema 2: Incompatibilidade entre Rancher e Kubernetes v1.37
Com a conectividade restaurada, a interface web do Rancher (`v2.12.2`) ficou completamente offline. O Helm Chart mostrava o status do Rancher como `failed`, e componentes como o `system-upgrade-controller` entraram em loop de travamento (`CrashLoopBackOff`).
* **Causa:** O Rancher Manager possui uma matriz estrita de compatibilidade de APIs. A versão v2.12/v2.15 não suporta o Kubernetes v1.37 como cluster local/host (o suporte estendia-se apenas até a v1.36). O Helm barrou o upgrade devido à trava de segurança `kubeVersion: < 1.37.0-0`.
* **Resolução:** Foi necessário realizar um **downgrade controlado do K3s** de volta para a linha estável **v1.36.5** em todos os nós do ecossistema, permitindo que os pacotes do Rancher fossem atualizados para a versão estável compatível **v2.15.2**.

### Problema 3: Falha em Hooks de Pré-Upgrade do Helm e Ingress TLS Inválido
Ao tentar rodar o comando `helm upgrade` para atualizar o Rancher para a versão v2.15.2 utilizando os valores salvos (`--reuse-values`), o processo falhou em duas frentes:
1. `failed calling webhook "rancher.cattle.io.secrets": context deadline exceeded` (O webhook antigo quebrado bloqueava a validação de novos Secrets do Helm).
2. `Ingress.networking.k8s.io "rancher" is invalid: spec.tls[0].hosts[0]: Invalid value: ""` (As versões novas do Rancher exigem obrigatoriamente um hostname ou domínio válido definido no objeto Ingress).

* **Resolução:** 
  1. Remoção temporária das regras estritas de interceptação de webhooks da API do Kubernetes para destravar o motor do Helm.
  2. Substituição do reaproveitamento cego de variáveis por um arquivo de valores explícito e injeção do hostname usando o serviço de DNS dinâmico para IPs privados `.sslip.io`.

### Problema 4: Timeout na Rede Interna de Pods (CNI/Flannel) e Kubelet Oculto
Com o Rancher instalado na versão correta, a interface web congelou em uma tela azul de carregamento infinito. O console de desenvolvedor (F12) reportava erros `504 Gateway Timeout` e os logs do pod do Rancher exibiam repetidos:
```bash
[ERROR] Failed to connect to peer wss://10.42.1.44/v3/connect ... i/o timeout
```
Ao tentar verificar logs mais profundos, o servidor retornava:
```bash
proxy error from 127.0.0.1:6443 while dialing 192.168.3.102:10250: 502 Bad Gateway
```
* **Causa:** As idas e vindas de versões de sistema operacional deixaram regras órfãs e corrompidas no kernel do Linux e nas tabelas de IPTables do plugin de rede **Flannel**. O nó Master perdeu a capacidade de enxergar o nó Worker (`192.168.3.102`) na porta interna do Kubelet (`10250`), quebrando a comunicação cruzada de pods do Rancher.
* **Resolução:** Foi executado o script nativo de purga de tabelas de rede do K3s (`k3s-killall.sh`) nos nós físicos para forçar os kernels a limparem as tabelas IPTables e reconstruírem as interfaces virtuais de forma limpa.

### Problema 5: Serviço do Traefik Travado em `<pending>` no Pool do Kube-VIP
Após restaurar a estabilidade interna, o tráfego externo tentava acessar a interface pelo IP virtual `192.168.3.202` do pool do Kube-VIP, mas recebia *Connection Refused*. O comando `kubectl get svc traefik` exibia o status de IP externo eternamente travado em `<pending>`.
* **Causa:** Como o Kube-VIP roda como **Static Pods** (arquivos locais lidos em disco pelo agente do K3s) e não como um DaemonSet convencional, o cluster não recarregou o seu provedor de nuvem de rede (Cloud Provider) automaticamente, deixando o serviço do Traefik LoadBalancer sem destino.
* **Resolução:** Identificação do ID estático do pod do Kube-VIP no nó Master e deleção forçada em memória, obrigando o K3s a ler o `ConfigMap` de faixa de IP atualizado (`192.168.3.200 - 192.168.3.255`) e injetar com sucesso o IP `.202` de volta ao Traefik.

---

## 3. Consolidação dos Comandos Utilizados

Abaixo está o compêndio de comandos utilizados em cada etapa para referência rápida em futuras manutenções:

### Limpeza de Caches Locais e Diagnósticos Iniciais
```bash
# Limpar cache de OpenAPI local do cliente
rm -rf ~/.kube/cache/schema

# Verificar se os pods e serviços vitais de entrada existem e mapear seus nós
kubectl get ingress -n cattle-system
kubectl get svc traefik -n kube-system
kubectl get pods -n kube-system -l app.kubernetes.io/name=traefik -o wide
```

### Script de Downgrade Controlado do K3s (Executados nos Hosts via SSH)
```bash
# No Nó Master (Control Plane) - Forçar downgrade para v1.36 estável
curl -sfL https://k3s.io | INSTALL_K3S_VERSION="v1.36.5+k3s1" sh -

# No Nó Worker - Sincronizar agente na mesma versão estável
curl -sfL https://k3s.io | INSTALL_K3S_VERSION="v1.36.5+k3s1" K3S_URL="https://<IP_DO_MASTER>:6443" K3S_TOKEN="<SEU_TOKEN>" sh -
```

### Limpeza Preventiva de Webhooks Travados (Desbloqueio de API)
```bash
# Remover travas do validador do Rancher
kubectl delete mutatingwebhookconfiguration rancher.cattle.io --ignore-not-found=true
kubectl delete validatingwebhookconfiguration rancher.cattle.io --ignore-not-found=true

# Remover travas do validador do Cert-Manager
kubectl delete validatingwebhookconfiguration cert-manager-webhook --ignore-not-found=true
kubectl delete mutatingwebhookconfiguration cert-manager-webhook --ignore-not-found=true
```

### Aplicação e Correção com o Helm (Atualização de Versão do Rancher)
```bash
# Exportar configurações ativas antigas para mesclagem de templates
helm get values rancher -n cattle-system -o yaml > rancher-values.yaml

# Executar upgrade forçando a injeção do DNS sslip com o IP virtual válido do pool
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.15.2 \
  -f rancher-values.yaml \
  --set hostname="192.168.3.202.sslip.io"
```

### Reset Bruto de Regras de Rede nos Nós (Executado via SSH em caso de corrupção de Kernel)
```bash
# 1. Parar os processos do K3s
sudo systemctl stop k3s          # (No Master)
sudo systemctl stop k3s-agent    # (No Worker)

# 2. Executar script de purga do IPTables e tabelas de roteamento do Flannel
sudo k3s-killall.sh

# 3. Reiniciar os serviços de orquestração
sudo systemctl start k3s          # (No Master)
sudo systemctl start k3s-agent    # (No Worker)
```

### Reinício Forçado de Componentes Estáticos e Cloud Providers
```bash
# Identificar o nome do Pod Estático do Kube-VIP no Master
kubectl get pods -n kube-system | grep kube-vip

# Forçar a deleção do pod para recarga de ConfigMap de IPs
kubectl delete pod kube-vip-<NOME-DO-NODE> -n kube-system --force --grace-period=0

# Reciclar o provedor de nuvem e controladores auxiliares
kubectl rollout restart deployment kube-vip-cloud-provider -n kube-system
kubectl rollout restart deployment rancher -n cattle-system --replicas=1
```

---

## 4. Lições Aprendidas para a Gestão do Homelab
1. **Verificar a Matriz de Suporte do Rancher antes de Upgrades de Sistema Operacional/Kubernetes:** O Rancher é muito sensível a mudanças na API interna do Kubernetes. Nunca atualize o Kubernetes Host (Local Cluster) sem antes garantir que a versão instalada do Rancher suporta oficialmente a nova *minor version* (ex: v1.37).
2. **Kube-VIP e Static Pods:** Componentes declarados como Static Pods em `/var/lib/rancher/k3s/agent/pod-manifests/` ignoram comandos clássicos como `kubectl rollout restart`. O ciclo de vida deles deve ser gerenciado derrubando o pod físico ou reiniciando o serviço do `k3s` no host.
