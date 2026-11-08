# Idea: Acesso remoto via Tailscale / Remote access via Tailscale

🇧🇷 [Português](#português) | 🇺🇸 [English](#english)

---

## Português

**Status**: Ideia — não implementada.
**Componente(s) relacionado(s)**: media stack (Jellyfin, `*arr`), ntfy — potencialmente outros serviços hoje expostos só na LAN (Grafana, Prometheus, Pi-hole).
**Origem**: mencionada em `clusters/observability/config.md` (ntfy, 2026-10-27) e em `clusters/media-stack/config.md` (acesso remoto à media stack, 2026-11-08).

Hoje todos os serviços do cluster são expostos só na LAN (`Service` tipo `LoadBalancer` via Kube-VIP + DNS local no Pi-hole). Acesso de fora de casa foi explicitamente deixado fora de escopo até validar o básico só na rede local.

A ideia é usar **Tailscale** (ou um reverse proxy equivalente) para habilitar acesso remoto sem expor portas diretamente na internet:

- **ntfy**: hoje as notificações push só chegam com o celular na rede de casa. Migrar pra Tailscale permitiria receber alertas de qualquer lugar, **sem precisar mudar o deployment do ntfy em si**.
- **Media stack**: Jellyfin e os apps `*arr` também foram desenhados só para LAN por enquanto; acesso remoto (assistir fora de casa, gerenciar downloads remotamente) é a "próxima etapa futura" já mencionada no plano da media stack.

### Perguntas em aberto

- Tailscale instalado em qual nível — por pod/sidecar, no node, ou um subnet router único expondo a rede `192.168.3.0/24` inteira?
- Expor todos os serviços de uma vez ou começar só por um (ntfy, por ser mais simples)?
- Algum serviço deve continuar LAN-only de propósito por motivo de segurança (ex: FlareSolverr, que o próprio projeto recomenda nunca expor)?

---

## English

**Status**: Idea — not implemented.
**Related component(s)**: media stack (Jellyfin, `*arr`), ntfy — potentially other services currently LAN-only (Grafana, Prometheus, Pi-hole).
**Origin**: mentioned in `clusters/observability/config.md` (ntfy, 2026-10-27) and in `clusters/media-stack/config.md` (media stack remote access, 2026-11-08).

Today every cluster service is exposed LAN-only (`LoadBalancer` `Service` via Kube-VIP + local DNS in Pi-hole). Access from outside the house was explicitly left out of scope until the basics are validated on the local network.

The idea is to use **Tailscale** (or an equivalent reverse proxy) to enable remote access without exposing ports directly to the internet:

- **ntfy**: push notifications currently only arrive with the phone on the home network. Migrating to Tailscale would allow receiving alerts from anywhere, **without needing to change the ntfy deployment itself**.
- **Media stack**: Jellyfin and the `*arr` apps were also designed LAN-only for now; remote access (watching away from home, managing downloads remotely) is the "future next step" already mentioned in the media stack plan.

### Open questions

- At what level should Tailscale run — per pod/sidecar, on the node, or a single subnet router exposing the whole `192.168.3.0/24` network?
- Expose every service at once, or start with just one (ntfy, being the simplest)?
- Should any service stay LAN-only on purpose for security reasons (e.g. FlareSolverr, which the upstream project itself recommends never exposing)?
