# Troubleshooting: Download do instalador k3s trava ao buscar assets no GitHub

Esta documentação detalha o diagnóstico de uma falha de download durante a instalação do k3s agent (worker2), onde a requisição travava indefinidamente ao buscar os arquivos de release no GitHub, apesar da conectividade básica com `github.com` funcionar normalmente.

---

## 1. O Cenário Inicial e Sintomas

Era necessário instalar uma versão específica do k3s (`v1.36.5+k3s1`) no worker2, para que a versão batesse exatamente com o control-plane gerenciado pelo Rancher (que ainda não suporta a minor version seguinte, `v1.37`).

### Sintomas do Erro:
* O comando de instalação travava logo após a mensagem:
  ```
  [INFO]  Using v1.36.5+k3s1 as release
  [INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.36.5%2Bk3s1/sha256sum-amd64.txt
  ```
  sem erro explícito, sem timeout — simplesmente parava de responder.
* A release existia normalmente na página do GitHub (confirmado visualmente no navegador do PC administrador).
* `curl -I` nos headers do asset parecia "quebrar" a URL.

---

## 2. Processo de Investigação & Descoberta

### Teste com `curl -v` (verbose) direto na VM
```bash
curl -v -fL https://github.com/k3s-io/k3s/releases/download/v1.36.5%2Bk3s1/sha256sum-amd64.txt
```

O resultado mostrou que:
1. A conexão TLS com `github.com` completava normalmente (handshake OK, certificado válido).
2. O `github.com` respondia com um `HTTP/2 302` redirecionando para o domínio real de armazenamento dos assets: `release-assets.githubusercontent.com` (CDN baseado em blob storage, assinado via query string com token temporário).
3. O curl então tentava abrir uma nova conexão TLS com `release-assets.githubusercontent.com` — e essa conexão **nunca completava**, travando indefinidamente no handshake.

### Causa Raiz
O domínio principal `github.com` funcionava, mas o domínio do CDN de assets (`release-assets.githubusercontent.com`, faixa de IPs Fastly `185.199.108-111.133`, compartilhada com `raw.githubusercontent.com` e GitHub Pages) estava inacessível a partir da rede do homelab. Após investigação, a causa raiz era um **problema pontual no provedor de internet (ISP)** — resolvido com uma simples reconexão (reset de modem/conexão), sem necessidade de alterar DNS ou configuração de rede da VM.

Esse padrão (`github.com` acessível, `*.githubusercontent.com` travando) é comum o suficiente para valer a pena reconhecer rapidamente, já que pode ser confundido com um problema de sintaxe no comando de instalação.

---

## 3. Comandos Aplicados para o Diagnóstico

```bash
# 1. Testar o download com verbose, pra ver exatamente onde trava (github.com vs CDN de assets)
curl -v -fL https://github.com/k3s-io/k3s/releases/download/v1.36.5%2Bk3s1/sha256sum-amd64.txt

# 2. Isolar teste de conectividade só com o domínio do CDN
curl -v -o /dev/null https://release-assets.githubusercontent.com/
```

---

## 4. Workaround (independe da causa raiz)

Enquanto a causa raiz (ISP) não estava confirmada/resolvida, o download manual via um host com internet funcional (o PC administrador) + `scp` para a VM contornou o problema:

```bash
# No PC administrador, baixar manualmente pelo navegador:
#   https://github.com/k3s-io/k3s/releases/download/v1.36.5+k3s1/k3s
#   https://github.com/k3s-io/k3s/releases/download/v1.36.5+k3s1/sha256sum-amd64.txt

# Enviar para a VM
scp k3s sha256sum-amd64.txt worker2:~/

# Na VM: validar checksum e instalar sem download automático
sha256sum --check <(grep k3s$ sha256sum-amd64.txt)
sudo mv k3s /usr/local/bin/k3s
sudo chmod +x /usr/local/bin/k3s

curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_VERSION="v1.36.5+k3s1" \
  K3S_URL=https://<IP_CONTROL_PLANE>:6443 K3S_TOKEN=<TOKEN> sh -s - agent --node-name worker2
```

A flag `INSTALL_K3S_SKIP_DOWNLOAD=true` faz o script de instalação do k3s pular a etapa de download e usar o binário já presente em `/usr/local/bin/k3s`, configurando normalmente o serviço systemd e executando o join no cluster.

---

## 5. Lições Aprendidas para o Homelab

1. **`github.com` acessível não significa que todo o GitHub está acessível** — downloads de release assets são redirecionados para domínios de CDN separados (`*.githubusercontent.com`), que podem ter conectividade/filtragem diferente da página principal. Ao diagnosticar downloads travados, sempre usar `curl -v` pra ver pra qual domínio o redirecionamento realmente aponta.
2. **`INSTALL_K3S_SKIP_DOWNLOAD=true` é um workaround confiável** para instalar uma versão específica do k3s quando o download automático falha por qualquer motivo de rede — basta ter o binário correto (validado por checksum) já no destino.
3. **Version skew não exige patch idêntico** — tecnicamente o Kubernetes tolera diferença de até duas minor versions entre agent e control-plane. Mas quando uma ferramenta de gestão (como o Rancher) ainda não suporta a minor version mais nova, confirmar a versão exata suportada evita retrabalho.
