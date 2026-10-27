# Troubleshooting: erro do `kubectl -o jsonpath` imprimiu o Secret inteiro (incluindo valor sensível) no terminal

Documenta um incidente de exposição acidental de um token (bot do Telegram) durante um comando de diagnóstico pensado para ser inofensivo (só listar os *nomes* das keys de um Secret, sem mostrar o valor).

---

## 1. O Cenário Inicial e Sintomas

Durante a configuração do Alertmanager → Telegram, foi pedido rodar o seguinte comando (só leitura, pensado para listar apenas os nomes das keys dentro do Secret, sem expor valores):
```bash
kubectl get secret -n monitoring alertmanager-telegram-api-key -o jsonpath='{.data}' | jq 'keys'
```

### Sintomas do Erro:
* `jq` não estava instalado no ambiente (Git Bash/MSYS2 no Windows): `bash: jq: command not found`.
* Sem `jq` pra consumir a saída, o `kubectl` reportou um erro de execução do template jsonpath (`Error executing template... write /dev/stdout: The pipe is being closed.`).
* Como parte da mensagem de erro, o `kubectl` imprimiu, **para fins de debug do próprio erro**, a representação completa do objeto Go que ele estava processando — incluindo o campo `data` inteiro, com o valor do Secret em base64 (`"token":"<base64 do token do bot>"`).
* Essa saída foi colada no chat/terminal, deixando o valor em base64 (reversível com um simples `base64 -d`) visível fora do Secret.

---

## 2. Processo de Investigação & Descoberta

O comando em si não tinha a intenção de expor o valor — `-o jsonpath='{.data}'` sempre retorna os valores (em base64), não só as keys; a ideia era deixar o `jq 'keys'` filtrar só os nomes antes de qualquer coisa chegar à tela. Como o `jq` não existia no PATH, o pipe quebrou, e o `kubectl` caiu no caminho de erro — que, para ajudar a debugar *problemas de template jsonpath*, despeja o objeto inteiro (não filtrado) como parte da mensagem.

### Causa Raiz: dependência de uma ferramenta externa (`jq`) não verificada antes de montar o pipe
O comando assumia implicitamente que `jq` estava disponível no ambiente. Quando a suposição falhou, o "plano B" do `kubectl` (mensagem de erro verbosa) expôs exatamente o dado que o pipe deveria ter filtrado.

**Ponto-chave**: qualquer comando que dependa de uma ferramenta externa para *esconder* ou *filtrar* um dado sensível é tão seguro quanto o caminho de erro dessa combinação — se o pipe quebrar antes do filtro, o dado cru pode vazar pela mensagem de erro, não pela saída "normal".

---

## 3. Comandos Aplicados para o Diagnóstico

Não houve diagnóstico adicional necessário — a causa já estava completa na primeira mensagem de erro. A ação foi de contenção/correção, não de investigação.

---

## 4. A Correção Definitiva

1. **Tratar o token exposto como comprometido**, independentemente de ter sido só um ambiente local — revogar e gerar um novo token via BotFather (`/revoke` ou `/token`) no bot do Telegram afetado.
2. **Atualizar o Secret no cluster** com o valor novo:
   ```bash
   kubectl create secret generic alertmanager-telegram-api-key \
     --namespace monitoring \
     --from-literal=token='<NOVO_TOKEN>' \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
3. **Alternativa mais segura pra essa inspeção** (sem depender de `jq` nem de um pipe que pode quebrar): pedir o objeto completo em JSON formatado e olhar manualmente só os nomes das keys em `data`, sem tentar extrair/filtrar via jsonpath:
   ```bash
   kubectl get secret -n monitoring alertmanager-telegram-api-key -o json
   ```
   (os valores continuam em base64 na tela, mas pelo menos o comando não depende de nenhuma ferramenta externa ausente, e o risco vira "ler a tela com cuidado" em vez de "um erro imprevisível despeja tudo").

### Validação final
Bot do Telegram passou a usar o token novo (validado nas etapas seguintes de configuração do Alertmanager); o token antigo, mesmo que alguém tivesse capturado a saída exposta, não tem mais validade.

---

## 5. Lições Aprendidas para o Homelab

1. **O caminho de erro de uma ferramenta pode ser menos cuidadoso com dados sensíveis do que o caminho de sucesso** — `kubectl`, ao falhar em executar um `jsonpath`, imprime o objeto subjacente inteiro para ajudar a debugar o próprio erro do template, sem saber que esse objeto continha um Secret. Vale lembrar disso sempre que um comando de inspeção de Secret depender de um pipe frágil.
2. **Qualquer valor que apareça em texto num terminal (mesmo em base64) deve ser tratado como comprometido** se esse terminal for colado em qualquer lugar fora do seu controle direto (chat, log compartilhado, ticket) — base64 não é criptografia, é só codificação; reverter é trivial (`base64 -d`).
3. **Antes de montar um pipe pra "esconder" um dado (`| jq ...`, `| grep -v ...`), verificar que a ferramenta do pipe existe no ambiente** — ou preferir um comando que não dependa de nada externo para ser seguro (ex: pedir o JSON bruto e inspecionar manualmente, em vez de confiar que o filtro sempre vai rodar).
4. **Rotacionar a credencial é sempre mais barato do que tentar avaliar "será que alguém viu"** — mesmo numa sessão local, assim que um segredo aparece em texto fora do Secret do Kubernetes, o caminho mais simples e seguro é gerar um novo.
