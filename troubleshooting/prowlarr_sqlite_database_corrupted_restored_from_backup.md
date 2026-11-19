# Troubleshooting: Banco SQLite do Prowlarr corrompido, restaurado via backup interno / Prowlarr's SQLite database got corrupted, restored from an internal backup

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

O Prowlarr passou a falhar com um erro de corrupção no seu banco SQLite (`database disk image is malformed`) no mesmo dia de um incidente sério de rede/NFS no cluster (ver [`troubleshooting/kubevip_lost_vips_after_interface_bounce.md`](kubevip_lost_vips_after_interface_bounce.md)). Resolvido restaurando um backup automático interno do Prowlarr do dia anterior, sem precisar de recuperação manual do SQLite nem de reset completo do `/config`.

---

### 1. O Cenário Inicial e Sintomas

No mesmo dia em que o cluster passou por uma instabilidade significativa de rede/NFS (perda dos IPs virtuais do Kube-VIP, cluster temporariamente inacessível — ver documento relacionado acima), o Prowlarr começou a apresentar erro de banco de dados corrompido.

#### Sintomas do Erro:
* Erro exibido pelo Prowlarr:
  ```
  code = Corrupt (11), message = System.Data.SQLite.SQLiteException (0x87AF054E): database disk image is malformed
  database disk image is malformed
  ```
* O `/config` do Prowlarr (onde fica o `prowlarr.db`) está num PVC `ReadWriteOnce` sobre NFS (`storageClassName: truenas-nfs-ssd`, ver `apps/servarr/prowlarr/pvc.yaml`) — a mesma camada de storage afetada pela instabilidade de rede do dia.

---

### 2. Processo de Investigação & Descoberta

Antes de partir para uma recuperação manual do SQLite (via `sqlite3 .recover`, usando um pod auxiliar temporário montando o mesmo PVC — plano preparado mas não chegou a ser necessário) ou um reset completo do `/config`, foi verificado primeiro se existia um backup automático recente e utilizável em `/config/Backups/scheduled/` (funcionalidade nativa do Prowlarr, que gera backups periódicos). Um backup do dia anterior estava disponível e foi restaurado diretamente pela própria UI do Prowlarr (**Settings → System → Backup → Restore**).

#### Causa Raiz: não confirmada em detalhe — hipótese forte ligada à instabilidade de rede/NFS do mesmo dia
A causa raiz exata da corrupção **não foi investigada a fundo** (o problema foi contornado pelo restore, não diagnosticado). A hipótese mais provável, dado o contexto: bancos SQLite são sensíveis a corrupção quando uma escrita é interrompida no meio do caminho, e isso é significativamente mais provável em storage de rede (NFS) do que em disco local — justamente o tipo de falha que ocorreu no cluster no mesmo dia (Kube-VIP perdendo todos os IPs virtuais, API inacessível, nodes `NotReady`, ver [`troubleshooting/kubevip_lost_vips_after_interface_bounce.md`](kubevip_lost_vips_after_interface_bounce.md)). Não há confirmação de que esse foi o gatilho exato, mas a coincidência temporal é forte e o mecanismo é plausível.

**Ponto-chave**: antes de qualquer recuperação manual de banco corrompido (`sqlite3`) ou reset de `/config`, **sempre checar primeiro os backups automáticos internos do próprio app** (`/config/Backups/scheduled/` nos apps da família Servarr) — é o caminho mais rápido e menos arriscado, e resolveu o problema aqui sem precisar de nenhuma das alternativas mais trabalhosas.

---

### 3. Comandos Aplicados para o Diagnóstico

```bash
# Confirmar se existe backup automático recente, utilizável, antes de
# qualquer tentativa de recuperação manual
kubectl exec -n media deploy/prowlarr-deployment -- ls -la /config/Backups/scheduled/
```

Restauração feita diretamente pela UI do Prowlarr (**Settings → System → Backup → Restore**), sem necessidade dos comandos de recuperação manual de SQLite que tinham sido preparados como plano B (ver `troubleshooting/sonarr_reset_stale_apikey_integrations.md`/histórico de sessão pra um exemplo do padrão geral de pod auxiliar montando o mesmo PVC pra manutenção de banco).

---

### 4. A Correção Definitiva (ou Workaround)

Restore do backup automático do dia anterior via a própria UI do Prowlarr (**Settings → System → Backup → Restore**). Não foi necessário:
* Recuperação manual via `sqlite3 .recover` (plano B preparado, não executado).
* Reset completo do `/config` com reconfiguração manual de indexers e conexões com Sonarr/Radarr/Lidarr.

#### Validação final
Prowlarr voltou a funcionar normalmente após o restore, com a configuração (indexers, conexões de app) do dia anterior preservada.

---

### 5. Lições Aprendidas para o Homelab

1. **Manter os backups automáticos do Prowlarr (e dos demais apps Servarr) habilitados é o que salvou esse incidente** — sem um backup recente disponível, a alternativa seria uma recuperação manual de SQLite (incerta) ou um reset completo com reconfiguração manual.
2. **Bancos SQLite em PVCs sobre NFS (`truenas-nfs-ssd`/`truenas-nfs-hdd`) são estruturalmente mais vulneráveis a corrupção por interrupção de escrita do que em disco local** — um risco latente compartilhado por todos os apps da media stack que guardam config em SQLite (Sonarr, Radarr, Lidarr, Prowlarr), não só o Prowlarr.
3. **Antes de qualquer recuperação manual de banco corrompido, checar primeiro os backups automáticos internos do próprio app** — é o caminho mais rápido e menos arriscado, e deveria ser sempre o primeiro passo, não um plano B.
4. **Correlacionar falhas de aplicação com incidentes de infraestrutura concorrentes** — essa corrupção não foi tratada como um evento isolado por acontecer no mesmo dia de uma instabilidade séria de rede/NFS no cluster; vale sempre checar o que mais estava acontecendo na infraestrutura antes de assumir um bug isolado da aplicação.

---

## English

Prowlarr started failing with a SQLite database corruption error (`database disk image is malformed`) on the same day as a serious network/NFS incident on the cluster (see [`troubleshooting/kubevip_lost_vips_after_interface_bounce.md`](kubevip_lost_vips_after_interface_bounce.md)). Fixed by restoring an internal automatic Prowlarr backup from the previous day, without needing manual SQLite recovery or a full `/config` reset.

---

### 1. Initial Scenario and Symptoms

On the same day the cluster went through significant network/NFS instability (Kube-VIP losing its virtual IPs, the cluster temporarily unreachable — see the related document above), Prowlarr started showing a corrupted-database error.

#### Error Symptoms:
* Error shown by Prowlarr:
  ```
  code = Corrupt (11), message = System.Data.SQLite.SQLiteException (0x87AF054E): database disk image is malformed
  database disk image is malformed
  ```
* Prowlarr's `/config` (where `prowlarr.db` lives) sits on a `ReadWriteOnce` PVC over NFS (`storageClassName: truenas-nfs-ssd`, see `apps/servarr/prowlarr/pvc.yaml`) — the same storage layer affected by that day's network instability.

---

### 2. Investigation & Discovery Process

Before resorting to manual SQLite recovery (via `sqlite3 .recover`, using a temporary helper pod mounting the same PVC — a plan that was prepared but ended up not being needed) or a full `/config` reset, the first check was whether a recent, usable automatic backup existed in `/config/Backups/scheduled/` (Prowlarr's built-in periodic backup feature). A backup from the previous day was available and was restored directly through Prowlarr's own UI (**Settings → System → Backup → Restore**).

#### Root Cause: not confirmed in detail — strong hypothesis tied to that day's network/NFS instability
The exact root cause of the corruption **was not investigated in depth** (the problem was worked around via the restore, not diagnosed). The most likely hypothesis, given the context: SQLite databases are prone to corruption when a write is interrupted mid-way, and that's significantly more likely on network storage (NFS) than on local disk — exactly the kind of failure that happened on the cluster the same day (Kube-VIP losing every virtual IP, the API unreachable, nodes `NotReady`, see [`troubleshooting/kubevip_lost_vips_after_interface_bounce.md`](kubevip_lost_vips_after_interface_bounce.md)). There's no confirmation that was the exact trigger, but the timing coincidence is strong and the mechanism is plausible.

**Key point**: before any manual recovery of a corrupted database (`sqlite3`) or a `/config` reset, **always check the app's own internal automatic backups first** (`/config/Backups/scheduled/` on Servarr-family apps) — it's the fastest, least risky path, and it resolved the problem here without needing either of the more involved alternatives.

---

### 3. Commands Used for Diagnosis

```bash
# Confirm whether a recent, usable automatic backup exists before
# attempting any manual recovery
kubectl exec -n media deploy/prowlarr-deployment -- ls -la /config/Backups/scheduled/
```

The restore itself was done directly through Prowlarr's UI (**Settings → System → Backup → Restore**), without needing the manual SQLite recovery commands that had been prepared as a fallback plan (see `troubleshooting/sonarr_reset_stale_apikey_integrations.md`/session history for an example of the general pattern of a helper pod mounting the same PVC for database maintenance).

---

### 4. The Definitive Fix (or Workaround)

Restoring the previous day's automatic backup through Prowlarr's own UI (**Settings → System → Backup → Restore**). Not needed:
* Manual recovery via `sqlite3 .recover` (fallback plan prepared, not executed).
* A full `/config` reset with manual reconfiguration of indexers and Sonarr/Radarr/Lidarr connections.

#### Final validation
Prowlarr went back to working normally after the restore, with the previous day's configuration (indexers, app connections) preserved.

---

### 5. Lessons Learned for the Homelab

1. **Keeping Prowlarr's (and the other Servarr apps') automatic backups enabled is what saved this incident** — without a recent backup available, the alternative would have been an uncertain manual SQLite recovery or a full reset with manual reconfiguration.
2. **SQLite databases on NFS-backed PVCs (`truenas-nfs-ssd`/`truenas-nfs-hdd`) are structurally more vulnerable to write-interruption corruption than on local disk** — a latent risk shared by every media-stack app that stores config in SQLite (Sonarr, Radarr, Lidarr, Prowlarr), not just Prowlarr.
3. **Before any manual recovery of a corrupted database, always check the app's own internal automatic backups first** — it's the fastest, least risky path, and should always be step one, not a fallback plan.
4. **Correlate application failures with concurrent infrastructure incidents** — this corruption wasn't treated as an isolated event because it happened the same day as serious network/NFS instability on the cluster; it's always worth checking what else was going on in the infrastructure before assuming an isolated application bug.
