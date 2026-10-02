# Chatbot de clima no Telegram com n8n

O bot recebe uma cidade brasileira no formato `Cidade,UF,BR`, consulta a temperatura atual na OpenWeather e responde em português. Aceita também `Cidade,UF`. Normaliza acentos, letras e espaços e informa mínima/máxima atuais na área quando disponíveis. O Google Gemini é opcional e reescreve a mensagem com fallback determinístico.

## Arquivos para entrega

- `workflow-chatbot-telegram.json`: workflow pronto para importar, baseado no último export revisado. Trigger habilitado, workflow inativo e Gemini desativado por padrão.
- `README.md`: importação, configuração e testes.
- `docker-compose.yml`: ambiente Docker opcional, com banco SQLite e volume persistente.
- `.env.example`: somente valores ilustrativos, sem segredos. O `.env` real não faz parte da entrega.

## Importar e configurar

1. Crie um workflow novo e vazio no n8n. No menu, selecione **Import from File** e importe `workflow-chatbot-telegram.json`. Não cole os nodes em outro workflow: isso pode duplicar nomes e quebrar referências.
2. Crie um bot pelo **@BotFather**, usando `/newbot`. Guarde o token como `TELEGRAM_BOT_TOKEN` em local seguro.
3. Crie no n8n uma credencial **Telegram API**. Preencha **Access Token** com o token e mantenha **Base URL** em `https://api.telegram.org`. Selecione essa mesma credencial em **Telegram Trigger**, **Enviar temperatura** e **Enviar erro**.
4. Obtenha a chave em [OpenWeather API Keys](https://home.openweathermap.org/api_keys) e confirme o e-mail da conta. Configure `OPENWEATHER_API_KEY` no ambiente do serviço n8n. O HTTP Request já usa `={{ $env.OPENWEATHER_API_KEY }}`; não substitua por uma chave literal.
5. Configure também `N8N_BLOCK_ENV_ACCESS_IN_NODE=false` para permitir `$env` no workflow. Em instalações com workers, configure o processo que executa os nodes. Essa configuração libera acesso ao ambiente aos editores da instância.
6. Configure uma URL pública HTTPS para o webhook. Para Docker local, siga a próxima seção. Um endereço localhost isolado não recebe chamadas do Telegram.
7. Salve e execute o workflow em teste. Envie `São Paulo,SP,BR` ao bot. Não mantenha outro workflow usando o mesmo bot ativo: um bot aceita um webhook por vez, inclusive entre teste e produção.
8. Depois dos testes, publique/ative o workflow se desejar operação contínua. O arquivo entregue fica inativo para não registrar webhooks automaticamente durante a importação.

## Variáveis esperadas

| Nome | Onde configurar e finalidade |
| --- | --- |
| `OPENWEATHER_API_KEY` | Ambiente do n8n; utilizada pelo HTTP Request. |
| `TELEGRAM_BOT_TOKEN` | Valor inserido na credencial Telegram API. Uma variável de ambiente sozinha não cria a credencial. |
| `GEMINI_API_KEY` | Opcional; valor inserido na credencial Google Gemini API. |
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | Ambiente do n8n; `false` permite a expressão `$env`. |
| `WEBHOOK_URL` | URL pública HTTPS com barra final. |
| `N8N_WEBHOOK_URL` | No Compose, recebe a mesma URL pública, para compatibilidade com a versão que usa esse nome na URL de teste. |
| `N8N_ENCRYPTION_KEY` | Chave aleatória própria para criptografar credenciais; mantenha estável e guarde com o backup. |

O export não contém credenciais nem referências às credenciais privadas da instância original. Cadastre e selecione suas credenciais após importar.

## Docker local, porta 5680

O Compose cria uma instalação independente. Não migra o banco, os usuários ou as credenciais do n8n da VPS. Requer Docker com containers Linux.

1. Copie `.env.example` para `.env`: `Copy-Item .env.example .env` no PowerShell ou `cp .env.example .env` no Linux.
2. Preencha `OPENWEATHER_API_KEY` e `N8N_ENCRYPTION_KEY`. Para gerar uma chave aleatória: `python -c "import secrets; print(secrets.token_hex(32))"`. Não use os textos `SUBSTITUA...` como valores reais.
3. Configure um túnel HTTPS, por exemplo com ngrok instalado/autenticado: `ngrok http 5680`. Mantenha-o aberto e copie a URL HTTPS real exibida em **Forwarding**.
4. Coloque essa URL real em `WEBHOOK_URL` e, para acesso público ao editor, também em `N8N_EDITOR_BASE_URL`. O domínio `seu-n8n.example.com` do exemplo é ilustrativo e não funciona. Não use o domínio da VPS para um túnel que deve chegar ao computador.
5. Execute `docker compose config --quiet` e `docker compose up -d`. Abra `http://localhost:5680`, crie o usuário proprietário e importe o workflow. O editor também pode ser acessado pelo HTTPS do túnel. Não altere a porta interna do n8n, que é 5678.
6. Quando mudar o `.env`, aplique `docker compose up -d --force-recreate n8n`; os dados permanecem no volume. Um restart não reaplica variáveis novas.
7. No Telegram Trigger, confira **Test URL**: deve começar com o domínio HTTPS e conter `/webhook-test/`. A **Production URL** usa `/webhook/`. Esses campos são gerados automaticamente, não editados no node.

Se a porta 5680 estiver ocupada, altere `N8N_LOCAL_PORT`, a URL local do editor e o destino do túnel. O Compose escuta somente em `127.0.0.1`. Em uma VPS, localhost significa a VPS; em proxies separados em containers, configure uma rede compartilhada e o destino `n8n:5678`. Ajuste `N8N_PROXY_HOPS` ao número real de proxies (padrão 1). O proxy deve encaminhar os cabeçalhos `X-Forwarded-For`, `X-Forwarded-Host` e `X-Forwarded-Proto`.

O volume `n8n_data` persiste banco e credenciais. Para parar: `docker compose down`. Não use `down -v` se precisar dos dados. Guarde backup do volume e da chave de criptografia. A tag padrão da imagem é `stable`; para reprodução exata, fixe `N8N_VERSION` na versão testada. A versão do servidor não é informada pelo export do workflow.

## Gemini opcional e fallback

Após **Resposta válida?**, o fluxo segue por **Fallback determinístico → Usar Gemini? → Google Gemini → Selecionar mensagem ou fallback → Enviar temperatura**. A saída falsa de **Usar Gemini?** contorna a IA e vai direto ao seletor.

O padrão é `useGemini: false` no Code e Google Gemini desativado. Assim a avaliação pode executar sem credenciais Gemini e sem chamadas de IA. Para ativar:

1. Obtenha uma chave no [Google AI Studio](https://aistudio.google.com/apikey) e cadastre-a na credencial **Google Gemini (PaLM) API** do n8n.
2. Selecione essa credencial no node **Google Gemini** e confirme um modelo disponível na sua conta. O export mantém `models/gemini-2.5-flash`; substitua-o se não estiver disponível.
3. Habilite o node e altere `useGemini: false` para `useGemini: true` no **Fallback determinístico**.
4. Mantenha temperatura `0.1`, JSON ativado, Simplify Output desativado e Maximum Number of Tokens `2048`. O prompt pede português e `{"message":"...","ok":true}`. Temperatura baixa reduz variação, sem garantir determinismo do modelo.

O fallback sempre gera a mensagem antes da chamada. Erros de chamada, saída vazia, JSON inválido, dados alterados ou números adicionais retornam à mensagem original. O seletor indica `source: gemini` ou `source: fallback`. Ele preserva cidade, temperaturas e o sufixo de mínima/máxima. Não basta remover a credencial com o Gemini habilitado: o n8n pode rejeitar a execução antes dos nodes. Para avaliar sem credenciais, mantenha o bypass padrão.

O modelo recebe cidade, temperatura, faixa e mensagem, sem chatId ou segredos. Não consulta clima nem resolve cidades homônimas; apenas melhora a escrita. O uso real depende dos limites e condições da conta Google.

## Mensagens e limites da API

Exemplo ilustrativo: `🌤️ A temperatura em São Paulo é de 23°C. Mínima atual na área: 21°C | Máxima atual na área: 25°C.` Os valores reais vêm da OpenWeather e são arredondados. Se a faixa não vier ou for incoerente, informa somente a temperatura atual.

`main.temp_min` e `main.temp_max` são observações atuais na área da cidade, não previsão do dia. Podem ser iguais à temperatura atual. O parâmetro real da API é `q`; `queue` é a variável interna exigida pelo desafio. O workflow preserva espaços dentro dos nomes das cidades. A busca por nome não garante desambiguação de municípios brasileiros homônimos pela UF.

Erros recebem: `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` Essa mensagem também cobre resposta inválida e autenticação recusada. Para diagnosticar, veja o Output de **Consultar OpenWeather**: 401 indica chave recusada, 404 indica cidade inexistente e `access to env vars denied` indica bloqueio de `$env`. A prévia `not accessible via UI` não confirma falha: execute o node. O validador lê `$json.body ?? $json.data`.

## Checklist de testes

| Mensagem | Esperado |
| --- | --- |
| `São Paulo,SP,BR` | Temperatura atual em °C. |
| `Belo Horizonte,MG,BR` | Temperatura atual em °C. |
| `Curitiba,PR,BR` | Temperatura atual em °C. |
| `CidadeInexistenteXYZ,SP,BR` | Mensagem de erro. |
| `São Paulo. SP, BR` | Entrada inválida; usar vírgula após a cidade. |

Na revisão de 02/10/2026, chamadas diretas à OpenWeather retornaram HTTP 200 para as três cidades e HTTP 404 para a inexistente. Também passaram 25 testes locais com dados simulados e 8 verificações da estrutura de entrega.

Na mesma data, o responsável pelo projeto confirmou os seguintes retornos pelo Telegram no workflow da VPS:

| Caso | Temperatura atual | Mínima atual na área | Máxima atual na área | Resultado informado |
| --- | --- | --- | --- | --- |
| São Paulo | 17°C | 16°C | 17°C | Mensagem recebida. |
| Belo Horizonte | 22°C | 21°C | 22°C | Mensagem recebida. |
| Curitiba | 13°C | 11°C | 13°C | Mensagem recebida. |
| Cidade inexistente | — | — | — | Mensagem de erro esperada recebida. |

Esses valores registram os testes naquele momento; novas consultas retornam temperaturas atualizadas. Os testes no bot confirmam a execução da instância configurada, mas não comprovam por si só a importação do arquivo sanitizado em uma instalação nova. O texto das respostas não comprova uso do Gemini: confira `source: gemini` ou `source: fallback` no seletor. Para testar sem IA, mantenha o padrão; para testar IA, habilite o node e altere `useGemini` para true.

Teste local: `node test-workflow.cjs`. Em ambientes Windows onde o carregamento de arquivos do Node falhar por realpath, use `node --preserve-symlinks-main test-workflow.cjs`. Testes não exigem chamadas externas.

## Publicação sem segredos

O repositório deve ser público para avaliação, mas a publicação só deve ocorrer após os testes. Não publique `.env`, exports de credenciais, tokens ou dados de execução. O `.gitignore` protege `.env`; o exemplo pode ser incluído porque não contém segredos. Os campos de credencial devem ser configurados em cada instância, sem chaves literais no workflow. Ao reexportar, use o nome `workflow-chatbot-telegram.json` e confira dados fixados e segredos antes de publicar. Nenhuma publicação é feita por estes arquivos.

Referências: [OpenWeather Current Weather](https://openweathermap.org/api/current), [Google AI Studio](https://aistudio.google.com/apikey).
