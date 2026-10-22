# Post-Mortem & Documentação de Estudos: Resolução de Conflitos Kube-VIP no K3s

Esta documentação detalha a investigação, o diagnóstico e a resolução de um problema de rede ocorrido no cluster K3s do Homelab, onde o **Kube-VIP** parou de atribuir endereços IP externos (`EXTERNAL-IP`) aos serviços do tipo `LoadBalancer` (como `pihole-svc` e `nginx`), deixando-os travados no status `<pending>`.

---

## 1. O Cenário Inicial e Sintomas
O ambiente é composto por um cluster **K3s** rodando em hardware físico (incluindo um nó Lenovo Z40-70). O Kube-VIP foi implantado para desempenhar dupla função:
1. **Control Plane HA:** Prover um IP virtual flutuante (`192.168.3.101`) para a API do Kubernetes.
2. **Service LoadBalancer (IPAM):** Distribuir IPs de um pool local (`192.168.3.200-192.168.3.254`) via protocolo **Layer 2 (ARP)** para os Pods expostos.

### Sintomas do Erro:
* Serviços novos (ex: `pihole-svc`) permaneciam indefinidamente com o IP `<pending>`.
* Serviços antigos (ex: `nginx`) exibiam como IP externo o endereço IP físico dos próprios nós do cluster, em vez de um IP do pool IPAM.
* Logs do `kube-vip-cloud-provider` exibiam a mensagem de sucesso `Ensured load balancer`, mas nenhuma alteração prática ocorria no status do recurso.
* O console disparava avisos de depreciação de chaves de afinidade: `Warning: ... node-role.kubernetes.io/master is use "node-role.kubernetes.io/control-plane" instead`.

---

## 2. Processo de Investigação & Descobertas

Durante o debug passo a passo, foram identificadas três causas raízes concorrentes que geraram o colapso do ecossistema de rede:

### Causa Raiz A: Conflito com o Klipper (ServiceLB Nativo do K3s)
O K3s possui um balanceador de carga embutido chamado **Klipper**. Como ele não foi desativado explicitamente na inicialização do K3s, ele disputava os manifestos de `LoadBalancer` com o Kube-VIP. Os logs de eventos registraram a criação oculta de DaemonSets do Klipper (`kube-system/svclb-pihole-svc-...`), que sobrescreviam e limpavam as configurações que o Kube-VIP tentava aplicar.

### Causa Raiz B: Erro de Escopo de Protocolo no DaemonSet (BGP vs ARP)
O arquivo de configuração original do DaemonSet do Kube-VIP possuía a flag `bgp_enable: "true"` ativa. No entanto, os roteadores vizinhos (`bgp_peers`) estavam comentados e o ambiente de Homelab operava em switches convencionais de Camada 2. 
O Kube-VIP travava tentando fechar sessões de roteamento dinâmico que não existiam, deixando de anunciar os pacotes via ARP (Layer 2).

### Causa Raiz C: Parâmetros de Lease Agressivos e Chaves Obsoletas
Os tempos de eleição de líder do painel de controle estavam configurados com valores extremamente baixos (`leaseduration: "5"`, `renewdeadline: "3"`). Qualquer oscilação mínima de CPU no hardware do Homelab fazia o Kube-VIP perder e reassumir a liderança em loops, invalidando as tabelas ARP na rede física. Adicionalmente, a chave de afinidade `node-role.kubernetes.io/master` estava gerando os alertas visuais por ter sido descontinuada pelo Kubernetes.

---

## 3. Comandos Aplicados para o Diagnóstico

Os seguintes comandos foram cruciais para mapear o comportamento do cluster:

```bash
# 1. Verificar se o balanceador nativo do K3s (Klipper) estava rodando
kubectl get pods -n kube-system | grep svclb

# 2. Inspecionar o ciclo de vida e erros do serviço travado
kubectl describe service pihole-svc

# 3. Validar se as permissões de RBAC do Cloud Provider contemplavam a escrita de status
kubectl get clusterrole system:kube-vip-cloud-controller-role -o yaml

# 4. Monitorar o comportamento do controlador de IPAM em tempo real (Modo Debug)
kubectl logs -n kube-system -l component=kube-vip-cloud-provider -f
```

---

## 4. A Correção Definitiva (Plano de Ação)

A solução consistiu em desativar o componente concorrente do K3s, ajustar o Cloud Provider para escutar explicitamente o ConfigMap e reconfigurar o DaemonSet do Kube-VIP para operar puramente em Layer 2 (ARP) estável.

### Passo 1: Desativação do Klipper no Sistema Operacional
Em cada nó de controle, o serviço do K3s foi editado para incluir a flag `--disable servicelb`:
```bash
# 1. Abrir o arquivo de inicialização do systemd
sudo nano /etc/systemd/system/k3s.service

# 2. Modificar a linha de execução adicionando o parâmetro:
# ExecStart=/usr/local/bin/k3s server --disable servicelb

# 3. Recarregar o sistema e reiniciar o cluster
sudo systemctl daemon-reload && sudo systemctl restart k3s
```

### Passo 2: Remoção de Travas de Segurança (Finalizers) Antigas
Os serviços que ficaram órfãos da disputa entre os dois controladores tiveram seus finalizers limpos para destravar o agendamento:
```bash
kubectl patch svc nginx -p '{"metadata":{"finalizers":null}}'
kubectl patch svc pihole-svc -p '{"metadata":{"finalizers":null}}'
```

### Passo 3: Ajuste do Cloud Provider (`kube-vip-cloud-provider`)
Injeção da variável de ambiente `vip_cidrconfigmap` no deployment do controlador para forçá-lo a abandonar o modo pass-through (Node IP) e utilizar o ConfigMap global:
```yaml
        env:
        - name: vip_cidrconfigmap
          value: "kubevip"
```

### Passo 4: Manifesto Corrigido do Kube-VIP DaemonSet (Modo ARP Estável)
O manifesto do DaemonSet foi atualizado substituindo as chaves `master` por `control-plane`, alternando `vip_arp` para `"true"`, desativando o BGP e normalizando os tempos de Lease (`15`, `10`, `2`):

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app.kubernetes.io/name: kube-vip-ds
    app.kubernetes.io/version: v1.0.3
  name: kube-vip-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: kube-vip-ds
  template:
    metadata:
      labels:
        app.kubernetes.io/name: kube-vip-ds
        app.kubernetes.io/version: v1.0.3
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/control-plane
                operator: Exists
      containers:
      - args:
        - manager
        env:
        - name: vip_arp
          value: "true"
        - name: port
          value: "6443"
        - name: vip_nodename
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: vip_interface
          value: enp1s0
        - name: vip_subnet
          value: "32"
        - name: dns_mode
          value: first
        - name: dhcp_mode
          value: ipv4
        - name: cp_enable
          value: "true"
        - name: cp_namespace
          value: kube-system
        - name: svc_enable
          value: "true"
        - name: svc_leasename
          value: plndr-svcs-lock
        - name: vip_leaderelection
          value: "true"
        - name: vip_leasename
          value: plndr-cp-lock
        - name: vip_leaseduration
          value: "15"
        - name: vip_renewdeadline
          value: "10"
        - name: vip_retryperiod
          value: "2"
        - name: address
          value: 192.168.3.101 # IP Virtual reservado do painel de controle
        - name: prometheus_server
          value: :2112
        image: ghcr.io/kube-vip/kube-vip:v1.0.3
        imagePullPolicy: IfNotPresent
        name: kube-vip
        resources: {}
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
            drop:
            - ALL
      hostNetwork: true
      serviceAccountName: kube-vip
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
```

### Passo 5: Aplicação e Reciclagem de Cache
```bash
# Aplicar as mudanças do DaemonSet
kubectl apply -f kube-vip-ds.yaml

# Forçar reinício limpo de todos os componentes da cadeia de rede
kubectl rollout restart daemonset kube-vip-ds -n kube-system
kubectl rollout restart deployment kube-vip-cloud-provider -n kube-system
kubectl rollout restart deployment pihole-deployment
```

---

## 5. Lições Aprendidas para o Homelab
1. **Distribuições Opinadas (K3s/RKE2):** Sempre verificar quais componentes "nativos" de rede (como Klipper e Flannel) a distribuição instala por padrão. Eles precisam ser explicitamente desativados se a intenção for adotar soluções customizadas como Kube-VIP, MetalLB ou Cilium.
2. **Especificação de Protocolo:** O Kube-VIP falha silenciosamente na gerência de serviços se o modo BGP estiver ligado sem vizinhos ativos configurados. Para redes locais domésticas simples, o modo Layer 2 (ARP) com `vip_arp: "true"` deve ser a escolha padrão.
3. **Imutabilidade de Status:** Quando serviços ficarem travados com IPs incorretos após mudanças de arquitetura de rede, inspecione e limpe os `finalizers` ou recrie os serviços para expirar o cache do CoreDNS e do Kubelet.
