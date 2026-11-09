# Troubleshooting: Jellyfin não conseguia ler legendas externas via NFS rodando como root / Jellyfin couldn't read external subtitles over NFS while running as root

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

Durante a investigação de [known-issues/jellyfin_frequent_pod_restarts.md](../known-issues/jellyfin_frequent_pod_restarts.md), foi descoberto que o Jellyfin não conseguia ler arquivos de legenda externa (`.srt`) no `media-pvc` (NFS/TrueNAS), mesmo rodando como `root` dentro do container. Corrigido alinhando o Jellyfin à mesma convenção `1000:1000` já usada no resto da media stack.

---

### 1. O Cenário Inicial e Sintomas

Durante o diagnóstico de restarts frequentes do pod do Jellyfin, os logs (`kubectl logs --previous`) mostravam, repetidamente, para dezenas de episódios em sequência durante um scan de biblioteca:

```
Unsupported codec with id 98304 for input stream 5
file:/media/tv/.../<episódio>.srt: Permission denied
MediaBrowser.Common.FfmpegException: ffprobe failed - streams and format are both null.
```

#### Sintomas do Erro:
* `ffprobe` falhando ao analisar arquivos `.srt` externos durante o scan de metadados, para toda uma temporada de uma vez.
* Mensagem explícita de `Permission denied` ao tentar ler o arquivo, não um erro de parsing do conteúdo.
* Vídeos (`.mkv`) continuavam sendo reproduzidos normalmente — só as legendas externas eram afetadas.

---

### 2. Processo de Investigação & Descoberta

`kubectl exec -n media deploy/jellyfin-deployment -- id` confirmou que o Jellyfin rodava como `uid=0(root) gid=0(root)` — a imagem oficial `jellyfin/jellyfin` (diferente dos apps `linuxserver.io` da stack) não usa `PUID`/`PGID`, e o `deployment.yaml` não definia nenhum `securityContext`.

Checando o dono/permissão real dos arquivos no NFS (via `kubectl exec` num pod com acesso RW, como o Sonarr):
```
-rw-r--r-- 2 1000 1000 ... Initial D ... .mkv
-rw-r----- 1 1000 1000 ... Initial D ... .srt
```
Os `.mkv` são `644` (legível por qualquer um) — por isso o vídeo sempre funcionou. Os `.srt` são `640` (legível só pelo dono e pelo grupo `1000`) — o container do Jellyfin, rodando como `root`, não conseguia ler esses arquivos (`cat` confirmou `Permission denied` diretamente).

#### Causa Raiz: UID 0 sem acesso de leitura a arquivos restritos a dono/grupo via NFS
Confirmado via teste direto (`cat` no arquivo como `root` → `Permission denied`) que o `root` do container não tinha acesso de leitura a esses arquivos através do NFS, apesar de normalmente um processo root ignorar completamente a checagem de permissão POSIX num filesystem local. **O mecanismo exato não foi confirmado com certeza** — um teste de `chown` como `root` no mesmo pod funcionou normalmente (o que descarta um `root_squash` "clássico" e uniforme, já que esse bloquearia `chown` também) — mas o comportamento observado e reproduzível é claro: como `root`, leitura nega; como `1000` (dono/grupo do arquivo), leitura funciona.

**Ponto-chave**: a imagem oficial do Jellyfin não segue a convenção `PUID`/`PGID` usada pelos apps `linuxserver.io` da stack — sem um `securityContext` explícito, ela roda como `root` por padrão, o que pode se comportar de forma inesperada (e difícil de prever) contra permissões de arquivo via NFS, mesmo sendo teoricamente "mais privilegiado".

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirma qual UID o Jellyfin realmente usa dentro do container
kubectl exec -n media deploy/jellyfin-deployment -- id

# Dono/permissões reais dos arquivos no NFS (via outro pod com acesso RW a /media)
kubectl exec -n media deploy/sonarr-deployment -- ls -la "/media/tv/Initial D/Season 1/"

# Reproduz o erro de permissão diretamente, como root
kubectl exec -n media deploy/jellyfin-deployment -- cat "/media/tv/Initial D/Season 1/<episódio>.srt" > /dev/null

# Testa se o chown como root funciona (confirma que não é root_squash clássico/uniforme)
kubectl exec -n media deploy/jellyfin-deployment -- chown 1000:1000 /config/.jellyfin-data
```

---

### 4. A Correção Definitiva (ou Workaround)

Alinhado o Jellyfin à mesma convenção `1000:1000` já usada no resto da media stack, via `securityContext` em `apps/jellyfin/deployment.yaml`:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
```

Como o Jellyfin sempre rodou como `root` até então, os dados existentes em `/config` e `/cache` (PVCs próprios, donos `root:root`) precisaram de `chown` **antes** de aplicar a mudança, senão o Jellyfin subiria sem conseguir ler o próprio banco/config:

```bash
kubectl exec -n media deploy/jellyfin-deployment -- chown -R 1000:1000 /config /cache
kubectl apply -f apps/jellyfin/deployment.yaml
```

#### Validação final
```bash
# Confirma que o Jellyfin agora roda como 1000, não root
kubectl exec -n media deploy/jellyfin-deployment -- id
# uid=1000 gid=1000 groups=1000

# Confirma que agora consegue ler o .srt que antes dava Permission denied
kubectl exec -n media deploy/jellyfin-deployment -- cat "/media/tv/Initial D/Season 1/<episódio>.srt" > /dev/null
echo "exit code: $?"
# exit code: 0
```

---

### 5. Lições Aprendidas para o Homelab

1. **`root` dentro de um container não é garantia de acesso total a um filesystem via NFS** — diferente de um filesystem local, onde `root` ignora checagem de permissão POSIX, o comportamento observado aqui foi o oposto do esperado (leitura negada como `root`, liberada como o UID/GID dono real do arquivo). Não assuma que rodar como `root` simplifica acesso a storage de rede.
2. **Apps que não seguem a convenção `PUID`/`PGID` do resto da stack merecem atenção extra** — a imagem oficial do Jellyfin roda como `root` por padrão sem isso ser óbvio, diferente dos apps `linuxserver.io` já padronizados em `1000:1000`. Vale checar `id` dentro do container de qualquer app novo antes de assumir que ele já está alinhado.
3. **Migrar um app já em produção pra outro UID exige migrar o dono dos dados existentes primeiro** — um `securityContext.runAsUser` novo não "adapta" dados já gravados por outro usuário; é preciso um `chown` explícito dos PVCs antes da mudança, ou o app sobe quebrado.

---

## English

While investigating [known-issues/jellyfin_frequent_pod_restarts.md](../known-issues/jellyfin_frequent_pod_restarts.md), it was discovered that Jellyfin couldn't read external subtitle (`.srt`) files on `media-pvc` (NFS/TrueNAS), even while running as `root` inside the container. Fixed by aligning Jellyfin to the same `1000:1000` convention already used across the rest of the media stack.

---

### 1. Initial Scenario and Symptoms

While diagnosing frequent Jellyfin pod restarts, the logs (`kubectl logs --previous`) repeatedly showed, for dozens of episodes in a row during a library scan:

```
Unsupported codec with id 98304 for input stream 5
file:/media/tv/.../<episode>.srt: Permission denied
MediaBrowser.Common.FfmpegException: ffprobe failed - streams and format are both null.
```

#### Error Symptoms:
* `ffprobe` failing to analyze external `.srt` files during a metadata scan, across an entire season at once.
* An explicit `Permission denied` message trying to read the file, not a content-parsing error.
* Videos (`.mkv`) kept playing normally — only external subtitles were affected.

---

### 2. Investigation & Discovery Process

`kubectl exec -n media deploy/jellyfin-deployment -- id` confirmed Jellyfin ran as `uid=0(root) gid=0(root)` — the official `jellyfin/jellyfin` image (unlike the stack's `linuxserver.io` apps) doesn't use `PUID`/`PGID`, and `deployment.yaml` had no `securityContext` at all.

Checking the files' actual owner/permissions on NFS (via `kubectl exec` into a pod with RW access, like Sonarr):
```
-rw-r--r-- 2 1000 1000 ... Initial D ... .mkv
-rw-r----- 1 1000 1000 ... Initial D ... .srt
```
`.mkv` files are `644` (world-readable) — that's why video always worked. `.srt` files are `640` (readable only by owner and group `1000`) — Jellyfin's container, running as `root`, couldn't read them (`cat` confirmed `Permission denied` directly).

#### Root Cause: UID 0 had no read access to owner/group-restricted files over NFS
Confirmed via a direct test (`cat` the file as `root` → `Permission denied`) that the container's `root` had no read access to these files over NFS, despite a root process normally bypassing POSIX permission checks entirely on a local filesystem. **The exact mechanism wasn't conclusively confirmed** — a `chown` test as `root` in the same pod worked normally (which rules out a "classic", uniform `root_squash`, since that would also block `chown`) — but the observed, reproducible behavior is clear: as `root`, reads are denied; as `1000` (the file's actual owner/group), reads work.

**Key point**: the official Jellyfin image doesn't follow the `PUID`/`PGID` convention used by the stack's `linuxserver.io` apps — without an explicit `securityContext`, it runs as `root` by default, which can behave unexpectedly (and unpredictably) against file permissions over NFS, even though it's theoretically "more privileged."

---

### 3. Commands Used for Diagnosis

```bash
# Confirms which UID Jellyfin actually runs as inside the container
kubectl exec -n media deploy/jellyfin-deployment -- id

# Actual owner/permissions of the files on NFS (via another pod with RW access to /media)
kubectl exec -n media deploy/sonarr-deployment -- ls -la "/media/tv/Initial D/Season 1/"

# Reproduces the permission error directly, as root
kubectl exec -n media deploy/jellyfin-deployment -- cat "/media/tv/Initial D/Season 1/<episode>.srt" > /dev/null

# Tests whether chown as root works (confirms this isn't a classic/uniform root_squash)
kubectl exec -n media deploy/jellyfin-deployment -- chown 1000:1000 /config/.jellyfin-data
```

---

### 4. The Definitive Fix (or Workaround)

Aligned Jellyfin to the same `1000:1000` convention already used across the rest of the media stack, via `securityContext` in `apps/jellyfin/deployment.yaml`:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
```

Since Jellyfin had always run as `root` until then, the existing data in `/config` and `/cache` (its own PVCs, owned `root:root`) needed a `chown` **before** applying the change, otherwise Jellyfin would come up unable to read its own database/config:

```bash
kubectl exec -n media deploy/jellyfin-deployment -- chown -R 1000:1000 /config /cache
kubectl apply -f apps/jellyfin/deployment.yaml
```

#### Final validation
```bash
# Confirms Jellyfin now runs as 1000, not root
kubectl exec -n media deploy/jellyfin-deployment -- id
# uid=1000 gid=1000 groups=1000

# Confirms it can now read the .srt that previously returned Permission denied
kubectl exec -n media deploy/jellyfin-deployment -- cat "/media/tv/Initial D/Season 1/<episode>.srt" > /dev/null
echo "exit code: $?"
# exit code: 0
```

---

### 5. Lessons Learned for the Homelab

1. **`root` inside a container is no guarantee of full access to an NFS filesystem** — unlike a local filesystem, where `root` bypasses POSIX permission checks entirely, the behavior observed here was the opposite of expected (read denied as `root`, granted as the file's actual owning UID/GID). Don't assume running as `root` simplifies access to network storage.
2. **Apps that don't follow the stack's `PUID`/`PGID` convention deserve extra attention** — the official Jellyfin image runs as `root` by default without that being obvious, unlike the stack's already-standardized `linuxserver.io` apps (`1000:1000`). Worth checking `id` inside any new app's container before assuming it's already aligned.
3. **Migrating an already-running app to a different UID requires migrating the existing data's ownership first** — a new `securityContext.runAsUser` doesn't "adapt" data already written by another user; an explicit `chown` of the PVCs is needed before the change, or the app comes up broken.
