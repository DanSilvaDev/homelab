# Gaming (Wolf / Moonlight) - Estado Atual

**Status: Implantado em 2026-11-22.** Streaming de jogos da Steam do worker3 (RX 9070) para clientes Moonlight na LAN, via **Wolf (Games on Whales)** rodando no k3s. Steam e Wolf UI validados com encode por hardware (VA-API, "zero copy pipeline on AMD").

Ver também: [Worker3 - config.md](../worker3/config.md) (pré-requisitos no host) | [ideas/worker3_gpu_node.md](../../ideas/worker3_gpu_node.md) (plano do node) | [troubleshooting/wolf_dind_sidecar_apps_crash_tmp_shm.md](../../troubleshooting/wolf_dind_sidecar_apps_crash_tmp_shm.md) (problemas da implantação)

Manifests: [`apps/wolf/`](../../apps/wolf/) — `namespace.yaml` (namespace `gaming`) e `deployment.yaml`.

## Decisões já tomadas

- **Wolf no k3s, não Sunshine no host** — mantém o worker3 sem ambiente gráfico instalado (convenção 5 do [`ia/CLAUDE.md`](../../ia/CLAUDE.md)). O Wolf cria um compositor Wayland virtual por sessão: sem monitor nem dummy plug (o Ryzen 7 5700X não tem vídeo integrado). Sunshine no host continua sendo o fallback.
- **Opção A: sidecar `docker:dind` no mesmo pod** (2026-11-22). O Wolf não roda o jogo dentro do próprio container: a cada sessão ele cria containers novos (Steam, PulseAudio, Wolf UI) pela API do Docker. O k3s usa containerd e o host não tem Docker, então o pod traz o próprio `dockerd`. Não há suporte upstream a Kubernetes nativo (wolf#82; o operator "fenrir" se declara "NOT IN A USEABLE STATE"). Plano B, se o dind der problema: Podman socket no host (caminho documentado pelo Wolf, mas quebra a convenção de nodes limpos).
- **`hostNetwork: true`** — o protocolo do Moonlight usa portas fixas TCP 47984, 47989, 48010 (RTSP) e UDP 47999, 48100, 48200. No Moonlight o host é adicionado pelo IP do worker3 (`192.168.3.106`). Mais simples que um LoadBalancer do Kube-VIP com protocolos mistos, já que o Wolf só roda neste node.
- **Estado no LV dedicado `/srv/games`** (200 GiB, ver worker3) via `hostPath`, não PVC: `/srv/games/wolf` (config, pareamentos, homes dos apps, biblioteca da Steam) e `/srv/games/docker` (imagens/camadas do dind). Começar com um jogo por vez; mais armazenamento depois, se o uso pedir.
- **Imagem fixada por digest** — o Wolf não publica tags de versão de imagem (só `stable`, `beta` e branches): `ghcr.io/games-on-whales/wolf:stable@sha256:47ce7a0b...`. Para atualizar, pegar o digest novo do `stable` e trocar no manifest.
- **GPU**: `devic.es/amdgpu: "1"` (1 dos 3 slots do generic-device-plugin, junto com Jellyfin e o futuro Ollama). O acesso real vem do `privileged` + `/dev` do host — o pedido serve para o scheduler contabilizar o compartilhamento. Regra de convivência: jogo e LLM não rodam ao mesmo tempo (ver [worker3_gpu_node.md](../../ideas/worker3_gpu_node.md)).

## Arquitetura do pod

| Container | Tipo | Papel |
|---|---|---|
| `dind` | native sidecar (`initContainer` com `restartPolicy: Always` + `startupProbe`) | `docker:29.9.3-dind`, só socket unix em `/var/run/dind/docker.sock`. Sobe antes do `wolf` e só libera o `wolf` quando o socket responde. Sem limit de CPU/memória: os jogos rodam como filhos deste `dockerd` |
| `wolf` | container principal | Wolf `stable`, `WOLF_DOCKER_SOCKET` apontando para o socket do dind, `livenessProbe` `exec` na porta RTSP |

**Regra que faz o dind funcionar**: o Wolf manda o Docker montar caminhos "do host" nos apps (`/etc/wolf`, `/tmp/sockets`, `/dev`, `/run/udev`). Quem resolve esses caminhos é o `dockerd` do dind, então **os dois containers precisam enxergar os mesmos diretórios nos mesmos caminhos**. Os volumes são montados igualmente no `wolf` e no `dind`.

Detalhes do `deployment.yaml` que não são óbvios (todos com comentário no manifest; histórico em [troubleshooting](../../troubleshooting/wolf_dind_sidecar_apps_crash_tmp_shm.md)):

| Item | Por quê |
|---|---|
| `command: ["dind", "dockerd"]` | pula o `dockerd-entrypoint.sh`, que com `DOCKER_TLS_CERTDIR` vazio abria `tcp://0.0.0.0:2375` sem TLS — com `hostNetwork`, uma API Docker (= root no worker3) aberta para a LAN |
| `--bridge=none --iptables=false --ip6tables=false` | o pod está na rede do host; um `dockerd` com bridge/iptables mexeria nas regras do próprio worker3 (flannel/kube-proxy). Consequência: apps usam `"NetworkMode": "host"` (ver config.toml abaixo) |
| volume `dind-tmp` em `/tmp` (só no dind) | o wrapper `dind` monta um tmpfs em `/tmp` quando `/tmp` não é mountpoint, e esse tmpfs escondia o `/tmp/sockets` compartilhado |
| volume `shm` (`emptyDir` `Memory`, `sizeLimit: 2Gi`) em `/dev/shm` (wolf e dind) | os apps usam `"IpcMode": "host"`, que no dind é o `/dev/shm` do pod (64 MB por padrão) — pouco para o Chromium da Steam |
| `livenessProbe` `exec` (`grep` por LISTEN na porta 48010 em `/proc/net/tcp*`) | se o Wolf sobe com a porta RTSP ocupada ele não morre, só segue sem RTSP; o probe reinicia o container. `exec` em vez de `tcpSocket` porque a conexão do probe gerava `[RTSP] error during transmission: End of file` a cada 20 s |
| `strategy: Recreate` | um único Wolf por node (portas fixas no host, uinput, estado em `/etc/wolf`) |

Avisos esperados no log do Wolf (inofensivos): `No valid container ID found in mountinfo, are we running in Docker?` / `automatic mounts matching is disabled` (o Wolf não está rodando dentro de Docker) e `Unable to become drm master, assuming unprivileged mode`.

## `config.toml` do Wolf (`/srv/games/wolf/cfg/config.toml`)

Fica **fora do repositório**: contém os certificados dos clientes pareados. O Wolf gera o arquivo no primeiro start e o reescreve (com aspas simples) a cada pareamento. Backups das edições ficam ao lado (`config.toml.bak*-2026-11-22`).

Edições feitas à mão (2026-11-22):

- **`"NetworkMode": "host",`** adicionado depois de cada `"IpcMode": "host",` no `base_create_json` de todos os apps (consequência do `--bridge=none` do dind — sem isso os apps ficam sem rede).
- **Bloco da Steam copiado para o perfil `moonlight-profile-id`**. Só os apps desse perfil aparecem direto na lista do Moonlight (Wolf UI, Test ball, Steam); os demais apps do perfil `user` (Firefox, RetroArch, Steam, Pegasus, Lutris, Prismlauncher, Desktop (xfce), EmulationStation, Kodi) aparecem só dentro do Wolf UI.

Edições temporárias de depuração (app "Debug shell" e `WOLF_UI_ARGS=--display-driver wayland --verbose` no Wolf UI) foram removidas em 2026-11-22.

## Riscos e pontos em aberto

- **NetworkManager/Bluez da imagem da Steam com `NET_ADMIN` na rede do host**: a imagem `steam:fedora` sobe esses serviços, e com `NetworkMode: host` num pod `hostNetwork` eles enxergam a rede real do worker3. Rede do host conferida sem alterações até agora. Alternativa (adiada, porque o Wolf só é documentado com rede do host): pod sem `hostNetwork` + `hostPort`, dind com bridge/iptables normais dentro do netns do pod, e sem `NetworkMode: host` nos apps.
- **Wolf abortou uma vez** (`SIGABRT`, `invalid unclassed pointer in cast to 'GstAppSink'`) ao desmontar o pipeline de um app que morreu durante a inicialização (antes da correção do `/tmp`). O supervisord interno da imagem reiniciou o processo e o pod não caiu. Não se repetiu depois das correções.
- **Moonlight "Wake PC"**: o worker3 já acorda por WoL (testado com magic packet do control-plane), mas o "Wake PC" do Moonlight ainda não foi testado.
- **Anti-cheat**: jogos com anti-cheat podem não funcionar via Proton (limitação geral de jogos no Linux, não do Wolf).
- **Só LAN** — acesso remoto seguiria a ideia de [Tailscale](../../ideas/tailscale_remote_access.md).

## Log de comandos executados

```bash
# Pré-requisitos no host: ver clusters/worker3/config.md ("Wolf - exceções no host")

# [PC, Git Bash, raiz do repo] Deploy
kubectl apply -f apps/wolf/namespace.yaml
kubectl apply -f apps/wolf/deployment.yaml
kubectl -n gaming rollout status deploy/wolf

# Atalho para a API do dind (apps criados pelo Wolf)
D="kubectl -n gaming exec deploy/wolf -c dind -- docker -H unix:///var/run/dind/docker.sock"
$D ps -a --format '{{.Names}}  {{.Status}}'
$D logs --tail 60 WolfSteam_<id-do-cliente>
# Log do Wolf tem bytes binários: grep precisa de -a. (Em `docker events --format`, Docker 29 usa .Actor.ID, não .ID)
kubectl -n gaming logs deploy/wolf -c wolf --since=10m | grep -a -iE "error|warn|fail|gst|pipeline|exit" | tail -60
# Pareamento: a URL com o PIN só aparece no log do Wolf DEPOIS de iniciar o pareamento no Moonlight

# [worker3] config.toml: NetworkMode host em todos os apps
F=/srv/games/wolf/cfg/config.toml
sudo sed -i 's/^\(\s*\)"IpcMode": "host",$/\1"IpcMode": "host",\n\1"NetworkMode": "host",/' "$F"
sudo grep -c '"NetworkMode": "host"' "$F"
# [worker3] config.toml: copia o bloco da Steam para o perfil moonlight-profile-id (antes do perfil 'user')
sudo python3 -c 'import sys;p=sys.argv[1];s=open(p).read();k=s.index("title = \x27Steam\x27");a=s.rindex("\n",0,s.rindex("[[profiles.apps]]",0,k))+1;b=s.rindex("\n",0,s.index("[[profiles.apps]]",k))+1;blk=s[a:b];u=s.index("[[profiles]]\nid = \x27user\x27");s=s[:u]+blk+s[u:];open(p,"w").write(s);print("ok")' "$F"
sudo grep -n -E "title =" "$F"

# Validação
MSYS_NO_PATHCONV=1 kubectl -n gaming exec deploy/wolf -c dind -- df -h /dev/shm         # ~2.0G
MSYS_NO_PATHCONV=1 kubectl -n gaming exec deploy/wolf -c wolf -- ls -l /tmp/sockets      # srwx... wayland-1, pulse-socket, wolf.sock
kubectl -n gaming logs deploy/wolf -c wolf --since=1m | grep -ac "RTSP] error"            # 0
```
