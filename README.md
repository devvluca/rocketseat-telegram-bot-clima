# 🌦️ Chatbot Telegram de Clima com N8N & OpenWeather

Chatbot para o Telegram desenvolvido no **N8N** que informa a temperatura atual de qualquer cidade do Brasil utilizando a API gratuita da **OpenWeather**.

Projeto desenvolvido como parte do desafio prático da pós-graduação Rocketseat.

---

## 📋 Sumário
- [Visão Geral](#-visão-geral)
- [Arquitetura do Workflow](#-arquitetura-do-workflow)
- [Requisitos e Variáveis de Ambiente](#-requisitos-e-variáveis-de-ambiente)
- [Passo a Passo de Instalação e Execução](#-passo-a-passo-de-instalação-e-execução)
  - [1. Executando o N8N com Docker](#1-executando-o-n8n-com-docker-recomendado)
  - [2. Criação do Bot no Telegram](#2-criação-do-bot-no-telegram)
  - [3. Obtenção da API Key da OpenWeather](#3-obtenção-da-api-key-da-openweather)
  - [4. Importação do Workflow no N8N](#4-importação-do-workflow-no-n8n)
  - [5. Configuração das Credenciais no N8N](#5-configuração-das-credenciais-no-n8n)
- [Recurso Opcional: Google Gemini com Fallback Determinístico](#-recurso-opcional-google-gemini-com-fallback-determinístico)
- [Guia de Testes e Validação](#-guia-de-testes-e-validação)
- [Checklist de Entrega](#-checklist-de-entrega)

---

## 🎯 Visão Geral

O chatbot recebe mensagens de texto de usuários no Telegram contendo o nome de uma cidade (ex: `Belo Horizonte`, `São Paulo,SP,BR`), formata a entrada, consulta as condições climáticas na API da OpenWeather e responde de forma clara, amigável e concisa com a temperatura arredondada em graus Celsius.

Caso a cidade não seja localizada ou ocorra algum erro de digitação, o chatbot responde com instruções amigáveis para orientar o usuário sobre o formato adequado.

---

## 🧩 Arquitetura do Workflow

O fluxo foi construído respeitando as melhores práticas do N8N:

1. **Telegram Trigger (`Telegram Trigger`)**:
   - Escuta mensagens de texto recebidas pelo bot.
2. **Captura e Formatação (`Captura e Formatação da Entrada`)**:
   - Extrai o texto do usuário e o `chat.id`.
   - Remove espaços excedentes, converte para minúsculas e remove acentos diacríticos (normalização NFD).
   - Atribui o resultado formatado à variável `queue`.
3. **Consulta HTTP (`Consulta OpenWeather`)**:
   - Endpoint: `https://api.openweathermap.org/data/2.5/weather`
   - Parâmetros enviados:
     - `q`: `={{ $json.queue }}`
     - `units`: `metric` (Celsius)
     - `lang`: `pt_br` (Português do Brasil)
     - `appid`: `={{ $env.OPENWEATHER_API_KEY }}`
   - Configurado com `onError: continueRegularOutput` e `neverError: true` para que códigos 404/400 continuem o fluxo até o nó de validação.
4. **Validação e Controle de Fluxo (`Validação da Resposta`)**:
   - Nó IF que verifica se `cod == 200` e se os dados de temperatura existem no payload.
5. **Caminho de Sucesso**:
   - **`Formatação da Resposta (Fallback)`**: Extrai a temperatura, arredonda com `Math.round()` e gera a mensagem padrão:
     `🌤️ A temperatura em {Cidade} é de {Temp}°C.`
   - **`Google Gemini (Opcional)`**: Envia a mensagem para refinamento de linguagem natural caso haja chave configurada.
   - **`Processar Mensagem Final`**: Aplica a resposta aprimorada pelo Gemini ou utiliza o **fallback determinístico** de forma 100% transparente se o Gemini estiver sem credenciais/desativado.
   - **`Telegram: Enviar Sucesso`**: Envia a resposta final para o chat do Telegram.
6. **Caminho de Erro**:
   - **`Telegram: Enviar Erro`**: Caso a cidade não seja encontrada, envia:
     `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).`

---

## 🔑 Requisitos e Variáveis de Ambiente

O workflow utiliza variáveis de ambiente para preservar a segurança e evitar credenciais embutidas no arquivo JSON:

| Variável | Obrigatória | Descrição |
| :--- | :---: | :--- |
| `OPENWEATHER_API_KEY` | **Sim** | Chave de API da OpenWeather para consulta do endpoint `/weather`. |
| `TELEGRAM_BOT_TOKEN` | **Sim** | Token do Bot gerado pelo [@BotFather](https://t.me/botfather). |
| `GEMINI_API_KEY` | *Não* | *(Opcional)* Chave de API do Google AI Studio para refinamento com Gemini. |
| `WEBHOOK_URL` | *Opcional* | URL pública (ex.: via ngrok/cloudflared) caso rode o N8N localmente com Webhooks. |

---

## 🚀 Passo a Passo de Instalação e Execução

### 1. Executando o N8N com Docker (Recomendado)

1. Clone o repositório ou acerte a pasta do projeto:
   ```bash
   git clone <URL_DO_SEU_REPOSITORIO>
   cd chatbot-telegram
   ```

2. Crie seu arquivo de variáveis a partir do modelo:
   ```bash
   cp .env.example .env
   ```

3. Edite o arquivo `.env` inserindo sua chave da OpenWeather e o token do Telegram:
   ```env
   OPENWEATHER_API_KEY=sua_chave_aqui
   TELEGRAM_BOT_TOKEN=seu_token_aqui
   GEMINI_API_KEY=sua_chave_gemini_aqui
   ```

4. Suba o container do N8N:
   ```bash
   docker compose up -d
   ```

5. Acesse o N8N no navegador em [http://localhost:5678](http://localhost:5678).

---

### 2. Criação do Bot no Telegram

1. Abra o Telegram e pesquise por `@BotFather`.
2. Envie o comando `/newbot`.
3. Escolha um nome e um username para o bot (o username deve terminar obrigatoriamente com `bot`, ex.: `ClimaRocketBot`).
4. O BotFather fornecerá um **Token de Acesso HTTP API** (ex.: `7123456789:AAFlkjw9e8...`).
5. Guarde esse token com segurança e configure-o no seu `.env` e nas credenciais do N8N.

---

### 3. Obtenção da API Key da OpenWeather

1. Acesse [https://home.openweathermap.org/users/sign_up](https://home.openweathermap.org/users/sign_up) e crie sua conta.
2. Confirme o e-mail de ativação recebido.
3. Acesse a aba **API Keys** em [https://home.openweathermap.org/api_keys](https://home.openweathermap.org/api_keys).
4. Copie a chave (Default) ou gere uma nova chave de API.
5. Insira a chave na variável `OPENWEATHER_API_KEY` do arquivo `.env`.

> ⚠️ **Nota**: Novas chaves da OpenWeather podem levar entre 10 e 30 minutos para serem ativadas nos servidores da OpenWeather após o cadastro.

---

### 4. Importação do Workflow no N8N

1. Acesse seu painel do N8N (`http://localhost:5678`).
2. No menu lateral esquerdo, clique em **Workflows** e depois no botão **Add Workflow** (ou clique no menu de 3 pontinhos no canto superior direito).
3. Selecione **Import from File...**.
4. Selecione o arquivo [`workflow-chatbot-telegram.json`](./workflow-chatbot-telegram.json) deste repositório.
5. O workflow será carregado na tela com todos os nós e conexões.

---

### 5. Configuração das Credenciais no N8N

1. No N8N, vá em **Credentials** no menu lateral esquerdo > **Add Credential**.
2. Procure por **Telegram API**.
3. No campo **Access Token**, cole o token fornecido pelo BotFather.
4. Salve com o nome de sua preferência.
5. Abra o nó **Telegram Trigger** e os nós **Telegram: Enviar Sucesso** e **Telegram: Enviar Erro** no workflow e selecione a credencial criada.
6. Clique no botão **Save** no canto superior direito do workflow e ative o botão de toggle **Active** (ou clique em **Test step** / **Listen for test event** para testar).

---

## 🤖 Recurso Opcional: Google Gemini com Fallback Determinístico

O workflow conta com o nó **`Google Gemini (Opcional)`**, implementado entre a formatação da resposta e o envio ao Telegram:

- **Localização**: Posicionado imediatamente após o nó `Formatação da Resposta (Fallback)`.
- **Como funciona o Fallback Determinístico**:
  1. O nó `Formatação da Resposta (Fallback)` extrai a temperatura e monta a mensagem padrão:
     `🌤️ A temperatura em {Cidade} é de {Temp}°C.`
  2. O nó do Gemini tenta reescrever a resposta de forma natural com temperatura baixa (`0.1`) e saída estruturada em JSON `{"message": "...", "ok": true}`.
  3. O nó seguinte `Processar Mensagem Final` avalia o resultado:
     - Se o Gemini retornar a resposta com sucesso, a mensagem aprimorada é enviada.
     - Se o Gemini não tiver chave configurada (`GEMINI_API_KEY`), falhar ou estiver desativado, o nó automaticamente repassa a mensagem padrão de fallback.
- **Como Ativar**:
  - Basta definir a variável `GEMINI_API_KEY` no seu `.env` ou nas variáveis do container/sistema.
  - Não é necessário alterar a estrutura do workflow: a avaliação automática funciona 100% via fallback sem custos de API.

---

## 🧪 Guia de Testes e Validação

Para testar o bot, envie mensagens diretamente para o seu bot no aplicativo do Telegram:

### Testes de Sucesso (3 Cidades)

| Mensagem Enviada | Retorno Esperado no Telegram |
| :--- | :--- |
| `Belo Horizonte` | `🌤️ A temperatura em Belo Horizonte é de 25°C.` |
| `São Paulo,SP,BR` | `🌤️ A temperatura em São Paulo é de 22°C.` |
| `Rio de Janeiro` | `🌤️ A temperatura em Rio de Janeiro é de 28°C.` |

*(Os valores numéricos de temperatura variam de acordo com as medições em tempo real da OpenWeather).*

### Teste de Erro (Cidade Inexistente)

| Mensagem Enviada | Retorno Esperado no Telegram |
| :--- | :--- |
| `CidadeQueNaoExiste12345` | `❌ Cidade não encontrada. Use o formato Cidade,UF,BR (ex.: São Paulo,SP,BR).` |

---

## ✅ Checklist de Entrega

- [x] **Trigger inicial**: Telegram Trigger configurado para mensagens de texto.
- [x] **Variável `queue`**: Set node normalizando espaços, acentos e caixa baixa.
- [x] **Chamada HTTP OpenWeather**: Endpoint `/weather` com `q`, `queue`, `units=metric`, `lang=pt_br` e `appid={{ $env.OPENWEATHER_API_KEY }}`.
- [x] **Resiliência a Erro 404**: `onError: continueRegularOutput` e `neverError: true` no nó HTTP.
- [x] **Nó IF de Validação**: Checagem de `cod === 200` e dados válidos.
- [x] **Mensagens formatadas**: Resposta amigável com temperatura arredondada e mensagem clara de erro.
- [x] **Fallback determinístico do Gemini**: Implementado para avaliação sem custos e sem falhas.
- [x] **Arquivos de entrega**:
  - `workflow-chatbot-telegram.json` (e cópia `workflow-telegram-chatbot.json`).
  - `README.md` completo e detalhado.
  - `docker-compose.yml` e `.env.example`.
- [x] **Segurança**: Nenhum token ou segredo sensível embutido no código ou documentação.
