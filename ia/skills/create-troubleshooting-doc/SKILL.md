---
name: create-troubleshooting-doc
description: Cria um documento de troubleshooting/post-mortem no padrão deste repositório (homelab), dentro do diretório troubleshooting/ na raiz do projeto. Use sempre que um problema não-óbvio (bug, configuração quebrada, comportamento inesperado de uma ferramenta) tiver sido diagnosticado e resolvido durante a sessão, e valer a pena documentar para que outro agente ou o próprio usuário não precise re-investigar do zero no futuro.
---

# Criar documento de troubleshooting (padrão homelab)

Esta skill documenta como escrever um novo arquivo de troubleshooting em `troubleshooting/` seguindo exatamente o padrão já estabelecido nos arquivos existentes deste repositório (`downgrade_kubernetes.md`, `kubevip_lost_ip_attribution_capability.md`, `nvidia_device_plugin_nodeaffinity_requirement.md`, `k3s_install_download_failure_isp.md`).

## Quando usar

Use esta skill quando, durante a sessão, você (o agente) tiver:
- Diagnosticado a causa raiz de um problema que não era óbvio à primeira vista
- Aplicado uma correção ou workaround que funcionou
- Identificado que o mesmo problema (ou um parecido) provavelmente vai se repetir, ou que o processo de diagnóstico consumiu tempo/passos não triviais

Não use para documentar simples decisões de configuração (isso vai nos `config.md` de cada `clusters/<nome>/`), nem para passos de instalação padrão sem nenhuma descoberta não-óbvia envolvida.

## Passo 1: Determine o nome do arquivo

- Caminho: `troubleshooting/<nome_descritivo>.md`
- Nome em `snake_case`, minúsculo, descritivo do problema (não da solução), em português ou nomes técnicos em inglês quando for termo de ferramenta (ex: `k3s_install_download_failure_isp.md`, `nvidia_device_plugin_nodeaffinity_requirement.md`)
- Evite nomes genéricos como `bug1.md` ou `fix.md`

## Passo 2: Estrutura obrigatória do documento

Siga exatamente esta estrutura de seções, nesta ordem:

```markdown
# Troubleshooting: <título curto e específico do sintoma observado>

<1-2 frases de resumo do que o documento cobre e em qual componente/ambiente ocorreu>

---

## 1. O Cenário Inicial e Sintomas

<Contexto: o que estava sendo feito quando o problema apareceu>

### Sintomas do Erro:
* <sintoma observável 1 - mensagem de erro, comportamento incorreto, etc.>
* <sintoma observável 2>
* <...>

---

## 2. Processo de Investigação & Descoberta

<Como a investigação foi conduzida - o que foi testado, o que foi descartado>

### Causa Raiz: <nome curto da causa>
<Explicação técnica precisa da causa raiz, com trechos de config/código relevantes quando aplicável>

**Ponto-chave**: <se houver um insight não-óbvio que vale destacar, coloque em negrito>

---

## 3. Comandos Aplicados para o Diagnóstico

```bash
# comandos reais usados para diagnosticar, com comentários explicando o propósito de cada um
```

---

## 4. A Correção Definitiva (ou Workaround)

<Explicação da correção aplicada>

```bash
# comandos reais da correção
```

### Validação final
```bash
# comandos usados para confirmar que a correção funcionou
```

---

## 5. Lições Aprendidas para o Homelab

1. **<lição 1 em negrito curto>** — <explicação de por que isso importa e quando se aplica de novo>
2. **<lição 2>** — <...>
3. <...>
```

## Passo 3: Regras de conteúdo

- **Use apenas informação real da sessão** — nunca invente comandos, saídas ou causas raiz que não foram de fato observados/confirmados na conversa. Se algo não foi confirmado (ex: causa raiz suspeita mas não validada), diga isso explicitamente em vez de apresentar como certeza.
- **Comandos devem ser copiáveis e corretos** — copie exatamente os comandos usados na sessão, não paráfrases.
- **Lições aprendidas devem ser generalizáveis** — cada lição deve ajudar a reconhecer ou evitar problemas parecidos no futuro, não apenas recapitular o que aconteceu.
- **Idioma**: português, mantendo termos técnicos (nomes de comandos, flags, nomes de ferramentas) no idioma original.
- **Tamanho**: documento deve ser objetivo — não é necessário replicar o nível de detalhe do post-mortem mais extenso do repositório (`kubevip_lost_ip_attribution_capability.md`) se o problema for mais simples; adapte a profundidade de cada seção à complexidade real do caso.

## Passo 4: Depois de criar

- Se o problema resolvido também mudou o estado de algum componente documentado em `clusters/<nome>/config.md`, atualize esse arquivo também (checklist e/ou log de comandos), mantendo os dois documentos consistentes entre si.
- Não duplique informação de estado (config atual) dentro do arquivo de troubleshooting — o troubleshooting documenta o *problema e sua resolução*, o `config.md` documenta o *estado atual*.
