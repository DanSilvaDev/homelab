# Troubleshooting: Lista de servidores Surfshark desatualizada no Gluetun / Stale Surfshark server list in Gluetun

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Durante a Fase 4 da media stack (qBittorrent + Gluetun atrás de VPN Surfshark via WireGuard, namespace `media`, `apps/qbittorrent/`), o túnel WireGuard nunca chegava a funcionar de verdade apesar de "conectar" segundo os logs do Gluetun — e, depois de corrigido isso, um segundo problema (healthcheck com falso-negativo) manteve o túnel reiniciando sem parar mesmo já funcionando de verdade.

---

### 1. O Cenário Inicial e Sintomas

Deployment com 2 containers no mesmo pod (`gluetun` + `qbittorrent`, network namespace compartilhada) mais um `initContainer`. O Gluetun estava configurado com `VPN_SERVICE_PROVIDER=surfshark` e `SERVER_COUNTRIES=Brazil`, usando credenciais WireGuard (chave privada + endereço) geradas no painel da Surfshark.

#### Sintomas do Erro:
* Log do Gluetun dizia `wireguard setup is complete`, mas minutos depois: `restarting VPN because it failed to pass the healthcheck: startup check: all check tries failed: parallel attempt 1/2 failed: dialing: dial tcp4: lookup github.com: i/o timeout, parallel attempt 2/2 failed: dialing: dial tcp4: lookup cloudflare.com: i/o timeout`.
* O ciclo se repetia indefinidamente a cada ~10 segundos, sem nunca estabilizar.
* O Gluetun alternava entre dois IPs de servidor resolvidos para `SERVER_COUNTRIES=Brazil` (`146.70.163.205` e `193.19.205.93`), com o mesmo resultado em ambos.
* Um teste de conectividade bruta por IP (`wget` para `1.1.1.1`, sem nome de domínio) também falhava, descartando problema isolado de DNS.

---

### 2. Processo de Investigação & Descoberta

A investigação seguiu, nesta ordem:

1. `wg show` dentro do container `gluetun` falhou (`wg` não existe no PATH) — a imagem configura o WireGuard via netlink diretamente (kernelspace), não via `wg-quick`/CLI `wg`.
2. Teste de conectividade bruta por IP (`wget` para `1.1.1.1`) falhou com exit code 4 — confirmou que não era um problema isolado de resolução DNS, e sim de conectividade real.
3. Checagem do sysctl `net.ipv4.conf.all.src_valid_mark` no node (worker2) retornou `0`. Hipótese: *reverse-path filtering* descartando pacotes do *policy-routing* (baseado em `fwmark`) do WireGuard. Ponto importante: esse sysctl é por *network namespace* — setar no host não tem nenhum efeito sobre o pod, porque a *netns* do pod não herda esse valor da *netns* do host.
4. Tentativa de corrigir via `initContainer` rodando `sysctl -w net.ipv4.conf.all.src_valid_mark=1` com `securityContext.capabilities.add: ["NET_ADMIN"]` (sem `privileged`) — falhou: `sysctl: error setting key 'net.ipv4.conf.all.src_valid_mark': Read-only file system`. Causa: `/proc/sys` vem montado somente-leitura no container por padrão (parte do `readonlyPaths` da OCI runtime spec), independente de capabilities — `NET_ADMIN` concede a permissão lógica, mas o *mount* em si bloqueia a escrita.
5. Corrigido trocando o `initContainer` para `securityContext.privileged: true` — isso remonta `/proc/sys` como leitura-escrita só para esse container efêmero (roda em frações de segundo antes do Gluetun subir; como todos os containers do pod compartilham a mesma *network namespace*, o efeito do sysctl persiste para o container `gluetun`).
6. Com o sysctl corrigido, verificação de roteamento interno (`ip route show`, `ip rule show`, e as tabelas customizadas `51820`, `200`, `199`) mostrou exatamente a estrutura que o Gluetun documenta/espera (tabela `51820` = rota padrão via `tun0` para tráfego normal via `fwmark`; tabela `200` = rota padrão original via `eth0`, preservada para o próprio tráfego encapsulado do WireGuard não entrar em loop; tabela `199` = exceção para `FIREWALL_OUTBOUND_SUBNETS` configurado). Pesquisa confirmou que essa é a estrutura correta/esperada, não um bug de roteamento.
7. Verificação do firewall interno (`iptables -L -n -v` e `iptables -t nat -L -n -v`) do próprio Gluetun mostrou regra `ACCEPT` específica para UDP destino `<ip-do-servidor-atual>:51820` e `ACCEPT` genérico para saída via `tun0` — nada indicando bloqueio da causa raiz.
8. Contadores de bytes de `tun0` via `/proc/net/dev` (sem precisar de ferramenta extra), comparados antes/depois com ~10s de intervalo — inconclusivo, pois os contadores resetavam a cada ciclo de restart do Gluetun (que recria `tun0` do zero a cada tentativa).
9. **Teste decisivo**: usar o mesmo arquivo `.conf` da Surfshark diretamente no cliente oficial do WireGuard no Windows, totalmente fora do Kubernetes — **conectou normalmente**. Isso isolou definitivamente o problema como específico do ambiente Kubernetes/cluster, descartando credenciais erradas, chave revogada, bloqueio de rede/ISP, ou problema do lado da Surfshark.
10. Checagem de relógio (*clock skew* pode causar rejeição silenciosa de handshake WireGuard por timestamp fora da janela de tolerância) entre o pod e o node — horários batendo exatamente, descartado.
11. `conntrack -L -p udp --dport 51820` no node, rodado várias vezes durante o loop de retry, mostrou múltiplas entradas com a flag `[ASSURED]` para ambos os IPs de servidor — confirmando que o kernel via tráfego de resposta em ambas as direções (ou seja, pacotes UDP realmente iam e voltavam do servidor). Isso descartou de vez NAT/firewall de rede como causa.
12. Hipótese de faltar configurar o campo `DNS` do `.conf` nas variáveis do Gluetun foi descartada rapidamente: o teste de conectividade bruta por IP (passo 2) já tinha falhado antes, e se fosse só falta de DNS esse teste teria funcionado (o Gluetun gerencia DNS internamente, com resolver próprio via DNS-over-TLS).
13. Pesquisa por padrões conhecidos de Gluetun + Kubernetes + WireGuard não encontrou o problema exato documentado, mas revelou a pista de trocar `SERVER_COUNTRIES` por um endpoint fixo (`SERVER_HOSTNAMES` ou modo `custom`).
14. Checagem dos logs completos por erros/`ipv6`/`fatal` não revelou nenhum erro explícito sendo suprimido — só o ciclo repetitivo de "setup completo" + falha de healthcheck.

#### Causa Raiz: Lista interna de servidores Surfshark desatualizada no Gluetun
O usuário forneceu o bloco `[Peer]` do mesmo arquivo `.conf` que funcionou no Windows:
```ini
[Peer]
PublicKey = SURFSHARK_SERVER_PUBLIC_KEY
AllowedIPs = 0.0.0.0/0
Endpoint = br-sao.prod.surfshark.com:51820
```
Resolvendo esse hostname (`nslookup br-sao.prod.surfshark.com`), os IPs retornados foram `146.70.163.211` e `138.199.58.35` — diferentes dos IPs que o Gluetun vinha selecionando automaticamente via `SERVER_COUNTRIES=Brazil` (`146.70.163.205` e `193.19.205.93`).

O Gluetun, com `VPN_SERVICE_PROVIDER=surfshark` + `SERVER_COUNTRIES`, usa uma lista de servidores embutida na própria imagem (mapeando localização → IP + chave pública esperada do servidor). Essa lista estava desatualizada/divergente da infraestrutura real da Surfshark para os servidores de São Paulo — o Gluetun tentava o handshake contra IPs que, mesmo respondendo a nível de rede (daí o `[ASSURED]` no `conntrack` — provavelmente um *cookie-reply*, resposta de baixo nível do protocolo usada para rate-limiting/anti-scanning), não tinham a chave pública que o Gluetun esperava, ou eram servidores diferentes/decomissionados. O cliente WireGuard no Windows usa o `Endpoint` (resolvido via DNS em tempo real) e a `PublicKey` exatos do `.conf` gerado na hora pelo painel da Surfshark — sempre atual.

**Ponto-chave**: uma entrada `[ASSURED]` no `conntrack` confirma troca de pacotes bidirecional a nível de IP/porta, mas **não** confirma que o handshake criptográfico da VPN foi concluído com sucesso.

#### Segunda causa raiz (depois do fix acima): healthcheck com falso-negativo por usar hostnames como alvo
Após trocar para o modo `custom`, o túnel passou a funcionar de verdade (`ping`, `nslookup` via `127.0.0.1` e `wget` manuais confirmaram conectividade real, DNS funcionando e IP de saída da Surfshark), mas o Gluetun continuava reiniciando a VPN internamente a cada ~10s (`kubectl get pods` não mostrava `RESTARTS` porque é um restart interno do processo, não do container/pod).

Hipóteses testadas e descartadas, nesta ordem:
- **CPU throttling**: o log mostrava múltiplos timeouts simultâneos em operações não relacionadas (`restoring IPv4 iptables failed: context deadline exceeded`, `TLS handshake timeout`, `lookup github.com: i/o timeout`) — padrão compatível com throttling. Subir `limits.cpu` de `250m` para `1` não resolveu sozinho (loop de restart continuou idêntico).
- **Download de listas de bloqueio (`BLOCK_MALICIOUS`)**: suspeita de que o download das listas de anúncios/malware pela própria VPN, no boot, travasse a resolução de DNS. Desligar (`BLOCK_MALICIOUS=off`) não resolveu sozinho, embora o download realmente só acontecesse na primeira execução (ficava em cache depois).
- **DNS-over-TLS (DoT)**: suspeita de que a negociação TLS extra com o Cloudflare, renegociada do zero a cada restart do túnel, não coubesse na janela curta do healthcheck. Trocar para DNS "plain" (`DNS_UPSTREAM_RESOLVER_TYPE=plain` + `DNS_UPSTREAM_PLAIN_ADDRESSES=1.1.1.1:53`) causou um efeito colateral revelador: `write udp <ip-do-pod-no-eth0>:<porta>->1.1.1.1:53: write: operation not permitted` — um bloqueio **explícito** (não timeout) do próprio firewall do Gluetun, porque essa consulta específica saía pela interface `eth0` (IP do pod) em vez de pelo túnel (`tun0`). Revertido para DoT (padrão), que já tinha se confirmado funcional via `nslookup` manual repetido 5x com sucesso.

**Causa raiz real**: o alvo padrão do healthcheck do Gluetun (`HEALTH_TARGET_ADDRESSES`, antes `HEALTH_TARGET_ADDRESS`) é `github.com:443` e `cloudflare.com:443` - ambos **hostnames**, exigindo resolução de nome antes do `dial` TCP. Mesmo com DNS comprovadamente funcional (testado manualmente várias vezes com sucesso, em menos de 1 segundo), essa resolução especificamente dentro do healthcheck falhava de forma consistente logo após cada restart "frio" do túnel, disparando um novo restart - um ciclo que nunca se quebrava sozinho. Trocar o alvo do healthcheck para um endereço IP fixo, sem necessidade de lookup de hostname (`HEALTH_TARGET_ADDRESSES=1.1.1.1:443`), eliminou essa variável por completo e o túnel estabilizou imediatamente (0 restarts).

**Ponto-chave**: quando testes manuais confirmam repetidamente que uma funcionalidade (aqui, DNS) funciona, mas o healthcheck automático de uma ferramenta continua reportando falha nela especificamente, vale suspeitar de falso-negativo do próprio mecanismo de healthcheck (timing, ambiente, ou lógica interna diferente do que um teste manual replica) em vez de continuar assumindo que o problema reportado é real.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# wg CLI não existe nessa imagem (configuração via netlink/kernelspace)
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- wg show

# Teste de conectividade bruta por IP, sem DNS
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- wget -qO- --timeout=5 http://1.1.1.1

# Sysctl no host - não tem efeito sobre o pod (netns separada)
sysctl net.ipv4.conf.all.src_valid_mark

# Checagem de rotas e regras de policy-routing dentro do pod
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip addr show tun0
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip route show
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip rule show
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip route show table 51820
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip route show table 200
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip route show table 199

# Firewall interno do Gluetun
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- iptables -L -n -v
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- iptables -t nat -L -n -v

# Contadores de bytes da interface do túnel
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- cat /proc/net/dev

# Comparação de relógio entre pod e node
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- date
date

# Estado real das conexões UDP no node, durante o loop de retry
sudo apt install -y conntrack
sudo conntrack -L -p udp --dport 51820

# Resolução do hostname do servidor (feito no Windows, fora do cluster)
nslookup br-sao.prod.surfshark.com

# Logs completos filtrados por termos-chave
kubectl logs -n media deploy/qbittorrent-deployment -c gluetun --tail=200 | grep -iE "ipv6|error|firewall|fatal"

# Contagem de restarts internos do VPN num intervalo (restart do processo,
# não do pod/container - kubectl get pods não mostra isso)
kubectl logs -n media deploy/qbittorrent-deployment -c gluetun --timestamps --since=5m | grep -c "restarting VPN"

# Teste manual repetido de DNS via o resolvedor interno do próprio gluetun
for i in 1 2 3 4 5; do kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- nslookup github.com 127.0.0.1; sleep 1; done
```

---

### 4. A Correção Definitiva (ou Workaround)

Trocar o Gluetun de `VPN_SERVICE_PROVIDER=surfshark` (+ `SERVER_COUNTRIES`, dependente da lista interna) para `VPN_SERVICE_PROVIDER=custom`, fornecendo manualmente os dados validados do mesmo `.conf` que funcionou no Windows: IP do endpoint (resolvido do hostname), porta, e a chave pública do servidor (`WIREGUARD_PUBLIC_KEY`). A chave privada e o endereço do cliente (`WIREGUARD_PRIVATE_KEY`/`WIREGUARD_ADDRESSES`) já estavam corretos desde o início.

```bash
kubectl delete secret gluetun-surfshark-credentials -n media
kubectl create secret generic gluetun-surfshark-credentials \
  --namespace media \
  --from-literal=wireguard-private-key='...' \
  --from-literal=wireguard-addresses='10.x.x.x/16' \
  --from-literal=wireguard-public-key='SURFSHARK_SERVER_PUBLIC_KEY' \
  --from-literal=wireguard-endpoint-ip='146.70.163.211'

kubectl apply -f apps/qbittorrent/deployment.yaml
kubectl rollout restart deployment/qbittorrent-deployment -n media
```

No `deployment.yaml`, isso significou trocar `VPN_SERVICE_PROVIDER` para `custom`, remover `SERVER_COUNTRIES`, e adicionar `WIREGUARD_PUBLIC_KEY`, `WIREGUARD_ENDPOINT_IP` (via `secretKeyRef`) e `WIREGUARD_ENDPOINT_PORT: "51820"`.

#### Validação final
```bash
kubectl get pods -n media -l app=qbittorrent
# 2/2 Running, 0 restarts (antes reiniciava a cada ~10s)

kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ping -c 4 -W 3 1.1.1.1
# 0% packet loss, ~7ms - conectividade real pela primeira vez

kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- cat /etc/resolv.conf
# nameserver 127.0.0.1 (resolver interno do próprio Gluetun)

kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ping -c 2 -W 3 github.com
# resolveu e respondeu normalmente, 0% loss, ~117ms (latência extra esperada do túnel)

kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- wget -qO- https://ipinfo.io/ip
# 146.70.163.212 - confirmado IP de saída da Surfshark, não o IP residencial
```

Depois do fix acima, o túnel funcionava de verdade mas o healthcheck interno continuava derrubando a VPN a cada ~10s (segunda causa raiz, ver seção 2). A correção definitiva foi trocar o alvo do healthcheck para um IP fixo, sem necessidade de resolver hostname:

```yaml
- name: HEALTH_TARGET_ADDRESSES
  value: "1.1.1.1:443"
```

#### Validação final (segunda correção)
```bash
kubectl logs -n media deploy/qbittorrent-deployment -c gluetun --timestamps --since=3m | grep -c "restarting VPN"
# 0 (antes: 29 reinícios em 5 minutos)

kubectl get pods -n media -l app=qbittorrent
# 2/2 Running, 0 restarts, estável há vários minutos
```

---

### 5. Lições Aprendidas para o Homelab

1. **Seleção automática de servidor por localização pode usar dados desatualizados** — `SERVER_COUNTRIES`/`SERVER_CITIES`/`SERVER_REGIONS` do Gluetun dependem de uma lista de servidores embutida na imagem, que pode ficar divergente da infraestrutura real do provider (IPs/chaves rotacionados do lado deles). Isso falha silenciosamente, porque o WireGuard não retorna erro de handshake malsucedido (só timeout). Quando a seleção automática falha de forma consistente e sem erro aparente, vale testar o modo `custom` com um servidor+chave validados manualmente antes de assumir problema de rede/firewall/Kubernetes.
2. **`[ASSURED]` no `conntrack` não prova que uma VPN está funcionando de ponta a ponta** — só confirma troca de pacotes bidirecional a nível de IP/porta. Um servidor WireGuard pode responder com um *cookie-reply* (rate-limiting/anti-scanning) sem nunca completar o handshake criptográfico real, marcando o fluxo como `ASSURED` mesmo assim.
3. **Testar fora do Kubernetes isola rápido "é do cluster" vs. "é da credencial/rede"** — usar a mesma credencial num cliente nativo, em outra máquina, decide em minutos se vale a pena continuar investigando dentro do cluster ou se o problema é externo a ele.
4. **`/proc/sys` é somente-leitura por padrão em containers, independente de capabilities** — escrever um sysctl de dentro de um container exige `privileged: true` (reescreve todo o mount) ou o mecanismo nativo `securityContext.sysctls` do Kubernetes (mais granular, mas restrito por padrão a uma lista "safe", exigindo `--allowed-unsafe-sysctls` no kubelet para qualquer sysctl fora dela).
5. **Sysctls de rede (`net.ipv4.conf.*`) são por *network namespace*, não herdados do host** — setar no host não tem efeito sobre pods; precisa ser aplicado dentro da mesma *netns* que vai usar a interface (em um pod multi-container, qualquer container do mesmo pod serve, já que compartilham a *netns*).
6. **Um restart interno de processo pode ficar invisível pro Kubernetes** — o Gluetun reiniciava o túnel inteiro a cada falha de healthcheck sem o container ou pod reiniciar de verdade (`kubectl get pods` mostrava `0 RESTARTS` o tempo todo). Quando uma ferramenta tem seu próprio loop de "restart on failure" interno, contar linhas de log específicas (`grep -c "restarting VPN"`) num intervalo de tempo é mais confiável do que olhar o contador de restarts do Kubernetes pra medir estabilidade real.
7. **Healthchecks que dependem de resolução de hostname são mais frágeis do que parecem** — mesmo com DNS funcionando perfeitamente em testes manuais, um healthcheck automático que resolve nome + conecta logo após um "cold start" pode falhar consistentemente por timing/ambiente, sem que a capacidade testada esteja de fato quebrada. Preferir um alvo por IP fixo nesse tipo de checagem elimina essa fonte de falso-negativo.

---

## English

During Phase 4 of the media stack (qBittorrent + Gluetun behind a Surfshark VPN via WireGuard, `media` namespace, `apps/qbittorrent/`), the WireGuard tunnel never actually worked despite the Gluetun logs claiming it had "connected" — and, once that was fixed, a second problem (a healthcheck false negative) kept the tunnel restarting endlessly even though it was genuinely working by then.

---

### 1. Initial Scenario and Symptoms

Deployment with 2 containers in the same pod (`gluetun` + `qbittorrent`, shared network namespace) plus one `initContainer`. Gluetun was configured with `VPN_SERVICE_PROVIDER=surfshark` and `SERVER_COUNTRIES=Brazil`, using WireGuard credentials (private key + address) generated from the Surfshark panel.

#### Error Symptoms:
* Gluetun's log said `wireguard setup is complete`, but minutes later: `restarting VPN because it failed to pass the healthcheck: startup check: all check tries failed: parallel attempt 1/2 failed: dialing: dial tcp4: lookup github.com: i/o timeout, parallel attempt 2/2 failed: dialing: dial tcp4: lookup cloudflare.com: i/o timeout`.
* The cycle repeated indefinitely every ~10 seconds, never stabilizing.
* Gluetun alternated between two server IPs resolved for `SERVER_COUNTRIES=Brazil` (`146.70.163.205` and `193.19.205.93`), with the same result on both.
* A raw IP connectivity test (`wget` to `1.1.1.1`, no hostname) also failed, ruling out an isolated DNS problem.

---

### 2. Investigation & Discovery Process

The investigation proceeded in this order:

1. `wg show` inside the `gluetun` container failed (`wg` is not in `PATH`) — the image configures WireGuard directly via netlink (kernelspace), not via `wg-quick`/the `wg` CLI.
2. A raw IP connectivity test (`wget` to `1.1.1.1`) failed with exit code 4 — confirming this was not an isolated DNS resolution problem, but a real connectivity one.
3. Checking the `net.ipv4.conf.all.src_valid_mark` sysctl on the node (worker2) returned `0`. Hypothesis: reverse-path filtering dropping packets from WireGuard's `fwmark`-based policy routing. Important point: this sysctl is per network namespace — setting it on the host has no effect on the pod, because the pod's netns does not inherit this value from the host's netns.
4. Attempted fix via an `initContainer` running `sysctl -w net.ipv4.conf.all.src_valid_mark=1` with `securityContext.capabilities.add: ["NET_ADMIN"]` (no `privileged`) — failed: `sysctl: error setting key 'net.ipv4.conf.all.src_valid_mark': Read-only file system`. Cause: `/proc/sys` is mounted read-only in containers by default (part of the OCI runtime spec's `readonlyPaths`), regardless of capabilities — `NET_ADMIN` grants the logical permission, but the mount itself blocks the write.
5. Fixed by switching the `initContainer` to `securityContext.privileged: true` — this remounts `/proc/sys` read-write only for that short-lived container (runs for a fraction of a second before Gluetun starts; since all containers in the pod share the same network namespace, the sysctl's effect persists for the `gluetun` container).
6. With the sysctl fixed, checking internal routing (`ip route show`, `ip rule show`, and the custom tables `51820`, `200`, `199`) showed exactly the structure Gluetun documents/expects (table `51820` = default route via `tun0` for normal `fwmark`-based traffic; table `200` = the original default route via `eth0`, preserved so WireGuard's own encapsulated traffic doesn't loop; table `199` = exception for the configured `FIREWALL_OUTBOUND_SUBNETS`). Research confirmed this is the correct/expected structure, not a routing bug.
7. Checking Gluetun's own internal firewall (`iptables -L -n -v` and `iptables -t nat -L -n -v`) showed a specific `ACCEPT` rule for UDP destined to `<current-server-ip>:51820` and a generic `ACCEPT` for egress via `tun0` — nothing indicating the root cause was a firewall block.
8. Byte counters on `tun0` via `/proc/net/dev` (no extra tooling needed), compared before/after with a ~10s gap — inconclusive, since the counters reset on every Gluetun restart cycle (which recreates `tun0` from scratch on each attempt).
9. **Decisive test**: using the same Surfshark `.conf` file directly in the official WireGuard client on Windows, entirely outside Kubernetes — **connected normally**. This definitively isolated the problem as specific to the Kubernetes/cluster environment, ruling out wrong credentials, a revoked key, network/ISP blocking, or a Surfshark-side issue.
10. Clock check (clock skew can cause a WireGuard handshake to be silently rejected due to its timestamp falling outside the tolerance window) between the pod and the node — times matched exactly, ruled out.
11. `conntrack -L -p udp --dport 51820` on the node, run several times during the retry loop, showed multiple entries with the `[ASSURED]` flag for both server IPs — confirming the kernel saw reply traffic in both directions (i.e., UDP packets really were going out and coming back from the server). This definitively ruled out NAT/network firewall as the cause.
12. The hypothesis of needing to also set the `.conf` file's `DNS` field in Gluetun's variables was quickly ruled out: the raw IP connectivity test (step 2) had already failed earlier, and if it were only a missing DNS setting, that test would have succeeded (Gluetun manages DNS internally, with its own DNS-over-TLS resolver).
13. Research for known Gluetun + Kubernetes + WireGuard patterns did not find this exact documented issue, but surfaced a lead: switching from `SERVER_COUNTRIES` to a fixed endpoint (`SERVER_HOSTNAMES` or `custom` mode).
14. Checking the full logs for errors/`ipv6`/`fatal` revealed no explicit error being suppressed — just the repeating cycle of "setup complete" + healthcheck failure.

#### Root Cause: Stale Surfshark server list embedded in Gluetun
The user provided the `[Peer]` block from the same `.conf` file that worked on Windows:
```ini
[Peer]
PublicKey = SURFSHARK_SERVER_PUBLIC_KEY
AllowedIPs = 0.0.0.0/0
Endpoint = br-sao.prod.surfshark.com:51820
```
Resolving that hostname (`nslookup br-sao.prod.surfshark.com`) returned `146.70.163.211` and `138.199.58.35` — different from the IPs Gluetun had been auto-selecting via `SERVER_COUNTRIES=Brazil` (`146.70.163.205` and `193.19.205.93`).

Gluetun, with `VPN_SERVICE_PROVIDER=surfshark` + `SERVER_COUNTRIES`, uses a server list embedded in the image itself (mapping location → server IP + expected public key). That list was stale/diverged from Surfshark's actual infrastructure for the São Paulo servers — Gluetun was attempting the handshake against IPs that, while responding at the network level (hence `[ASSURED]` in `conntrack` — likely a cookie-reply, a low-level protocol response used for rate-limiting/anti-scanning), did not have the public key Gluetun expected, or were different/decommissioned servers. The WireGuard client on Windows, by contrast, uses the `Endpoint` (resolved via live DNS) and `PublicKey` exactly as generated moments earlier by the Surfshark panel — always current.

**Key point**: an `[ASSURED]` entry in `conntrack` confirms bidirectional packet exchange at the IP/port level, but it does **not** confirm the VPN's cryptographic handshake actually succeeded.

#### Second root cause (after the fix above): healthcheck false negative from using hostnames as its target
After switching to `custom` mode, the tunnel actually started working (manual `ping`, `nslookup` via `127.0.0.1`, and `wget` all confirmed real connectivity, working DNS, and the Surfshark egress IP), but Gluetun kept restarting the VPN internally every ~10s (`kubectl get pods` showed no `RESTARTS` because this is an internal process restart, not a container/pod restart).

Hypotheses tested and ruled out, in this order:
- **CPU throttling**: the log showed multiple simultaneous timeouts across unrelated operations (`restoring IPv4 iptables failed: context deadline exceeded`, `TLS handshake timeout`, `lookup github.com: i/o timeout`) — a pattern consistent with throttling. Raising `limits.cpu` from `250m` to `1` did not fix it alone (the restart loop continued identically).
- **Block list download (`BLOCK_MALICIOUS`)**: suspicion that downloading ad/malware block lists through the VPN itself, at boot, was blocking DNS resolution. Turning it off (`BLOCK_MALICIOUS=off`) did not fix it alone, though the download genuinely only happened on the first run (cached afterward).
- **DNS-over-TLS (DoT)**: suspicion that the extra TLS negotiation with Cloudflare, renegotiated from scratch on every tunnel restart, didn't fit within the healthcheck's short window. Switching to plain DNS (`DNS_UPSTREAM_RESOLVER_TYPE=plain` + `DNS_UPSTREAM_PLAIN_ADDRESSES=1.1.1.1:53`) caused a revealing side effect: `write udp <pod-ip-on-eth0>:<port>->1.1.1.1:53: write: operation not permitted` — an **explicit** block (not a timeout) from Gluetun's own firewall, because that specific query was going out via the `eth0` interface (the pod's IP) instead of through the tunnel (`tun0`). Reverted to DoT (default), which had already been confirmed working via 5 successful manual `nslookup` attempts.

**Real root cause**: Gluetun's default healthcheck target (`HEALTH_TARGET_ADDRESSES`, formerly `HEALTH_TARGET_ADDRESS`) is `github.com:443` and `cloudflare.com:443` — both **hostnames**, requiring name resolution before the TCP `dial`. Even with DNS demonstrably working (tested manually multiple times successfully, in under a second), that resolution specifically inside the healthcheck consistently failed right after each "cold" tunnel restart, triggering yet another restart — a cycle that never broke on its own. Switching the healthcheck's target to a fixed IP address, with no hostname lookup needed (`HEALTH_TARGET_ADDRESSES=1.1.1.1:443`), eliminated that variable entirely and the tunnel stabilized immediately (0 restarts).

**Key point**: when manual tests repeatedly confirm a given capability (here, DNS) works, but a tool's own automated healthcheck keeps reporting failure specifically on that capability, it's worth suspecting a false negative in the healthcheck mechanism itself (timing, environment, or internal logic different from what a manual test replicates) rather than continuing to assume the reported problem is real.

---

### 3. Commands Used for Diagnosis

```bash
# wg CLI doesn't exist in this image (configured via netlink/kernelspace)
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- wg show

# Raw IP connectivity test, no DNS involved
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- wget -qO- --timeout=5 http://1.1.1.1

# Host sysctl - has no effect on the pod (separate netns)
sysctl net.ipv4.conf.all.src_valid_mark

# Checking routes and policy-routing rules inside the pod
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip addr show tun0
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip route show
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip rule show
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip route show table 51820
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip route show table 200
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ip route show table 199

# Gluetun's own internal firewall
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- iptables -L -n -v
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- iptables -t nat -L -n -v

# Tunnel interface byte counters
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- cat /proc/net/dev

# Clock comparison between pod and node
kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- date
date

# Real UDP connection state on the node, during the retry loop
sudo apt install -y conntrack
sudo conntrack -L -p udp --dport 51820

# Resolving the server hostname (done on Windows, outside the cluster)
nslookup br-sao.prod.surfshark.com

# Full logs filtered by key terms
kubectl logs -n media deploy/qbittorrent-deployment -c gluetun --tail=200 | grep -iE "ipv6|error|firewall|fatal"

# Count internal VPN restarts over a time window (process restart, not
# pod/container restart - kubectl get pods doesn't show this)
kubectl logs -n media deploy/qbittorrent-deployment -c gluetun --timestamps --since=5m | grep -c "restarting VPN"

# Repeated manual DNS test via Gluetun's own internal resolver
for i in 1 2 3 4 5; do kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- nslookup github.com 127.0.0.1; sleep 1; done
```

---

### 4. The Definitive Fix (or Workaround)

Switch Gluetun from `VPN_SERVICE_PROVIDER=surfshark` (+ `SERVER_COUNTRIES`, dependent on the embedded list) to `VPN_SERVICE_PROVIDER=custom`, manually supplying the validated data from the same `.conf` that worked on Windows: the endpoint IP (resolved from the hostname), port, and the server's public key (`WIREGUARD_PUBLIC_KEY`). The client's private key and address (`WIREGUARD_PRIVATE_KEY`/`WIREGUARD_ADDRESSES`) were already correct from the start.

```bash
kubectl delete secret gluetun-surfshark-credentials -n media
kubectl create secret generic gluetun-surfshark-credentials \
  --namespace media \
  --from-literal=wireguard-private-key='...' \
  --from-literal=wireguard-addresses='10.x.x.x/16' \
  --from-literal=wireguard-public-key='SURFSHARK_SERVER_PUBLIC_KEY' \
  --from-literal=wireguard-endpoint-ip='146.70.163.211'

kubectl apply -f apps/qbittorrent/deployment.yaml
kubectl rollout restart deployment/qbittorrent-deployment -n media
```

In `deployment.yaml`, this meant switching `VPN_SERVICE_PROVIDER` to `custom`, removing `SERVER_COUNTRIES`, and adding `WIREGUARD_PUBLIC_KEY`, `WIREGUARD_ENDPOINT_IP` (via `secretKeyRef`) and `WIREGUARD_ENDPOINT_PORT: "51820"`.

#### Final validation
```bash
kubectl get pods -n media -l app=qbittorrent
# 2/2 Running, 0 restarts (previously restarted every ~10s)

kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ping -c 4 -W 3 1.1.1.1
# 0% packet loss, ~7ms - real connectivity for the first time

kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- cat /etc/resolv.conf
# nameserver 127.0.0.1 (Gluetun's own internal resolver)

kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- ping -c 2 -W 3 github.com
# resolved and responded normally, 0% loss, ~117ms (extra latency expected from the tunnel)

kubectl exec -n media deploy/qbittorrent-deployment -c gluetun -- wget -qO- https://ipinfo.io/ip
# 146.70.163.212 - confirmed Surfshark egress IP, not the residential one
```

After the fix above, the tunnel genuinely worked but the internal healthcheck kept tearing the VPN down every ~10s (second root cause, see section 2). The definitive fix was switching the healthcheck's target to a fixed IP, with no hostname resolution needed:

```yaml
- name: HEALTH_TARGET_ADDRESSES
  value: "1.1.1.1:443"
```

#### Final validation (second fix)
```bash
kubectl logs -n media deploy/qbittorrent-deployment -c gluetun --timestamps --since=3m | grep -c "restarting VPN"
# 0 (previously: 29 restarts in 5 minutes)

kubectl get pods -n media -l app=qbittorrent
# 2/2 Running, 0 restarts, stable for several minutes
```

---

### 5. Lessons Learned for the Homelab

1. **Automatic location-based server selection can use stale data** — Gluetun's `SERVER_COUNTRIES`/`SERVER_CITIES`/`SERVER_REGIONS` depend on a server list embedded in the image, which can diverge from the provider's actual infrastructure (IPs/keys rotated on their end). This fails silently, because WireGuard doesn't return a failed-handshake error (only a timeout). When automatic selection fails consistently with no apparent error, it's worth testing `custom` mode with a manually validated server+key before assuming a network/firewall/Kubernetes problem.
2. **`[ASSURED]` in `conntrack` does not prove a VPN is working end-to-end** — it only confirms bidirectional packet exchange at the IP/port level. A WireGuard server can respond with a cookie-reply (rate-limiting/anti-scanning) without ever completing the real cryptographic handshake, still marking the flow as `ASSURED`.
3. **Testing outside Kubernetes quickly isolates "it's the cluster" vs. "it's the credential/network"** — using the same credential in a native client, on another machine, decides in minutes whether it's worth continuing to investigate inside the cluster or whether the problem is external to it.
4. **`/proc/sys` is read-only by default in containers, regardless of capabilities** — writing a sysctl from inside a container requires `privileged: true` (remounts the whole path) or Kubernetes' native `securityContext.sysctls` mechanism (more granular, but restricted by default to a "safe" list, requiring `--allowed-unsafe-sysctls` on the kubelet for anything outside it).
5. **Network sysctls (`net.ipv4.conf.*`) are per network namespace, not inherited from the host** — setting them on the host has no effect on pods; they must be applied inside the same netns that will use the interface (in a multi-container pod, any container in that pod works, since they share the netns).
6. **An internal process restart can be invisible to Kubernetes** — Gluetun restarted the entire tunnel on every healthcheck failure without the container or pod actually restarting (`kubectl get pods` showed `0 RESTARTS` the whole time). When a tool has its own internal "restart on failure" loop, counting specific log lines (`grep -c "restarting VPN"`) over a time window is more reliable than watching Kubernetes' restart counter to measure real stability.
7. **Healthchecks that depend on hostname resolution are more fragile than they look** — even with DNS working perfectly in manual tests, an automated healthcheck that resolves a name and connects right after a cold start can fail consistently due to timing/environment, without the tested capability actually being broken. Preferring a fixed-IP target for this kind of check eliminates that source of false negatives.
