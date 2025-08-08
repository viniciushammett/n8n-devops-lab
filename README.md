## 🔧 Pré-requisitos

* Docker e Docker Compose
* `curl` para testes
* Porta `5678` livre

---

## 🚀 Subir o ambiente local

1. Copie o arquivo `.env.example` para `.env` e ajuste se necessário:

   ```bash
   cp .env.example .env
   ```
2. Suba os serviços:

   ```bash
   docker compose up -d
   ```
3. Acesse o editor do n8n: **[http://localhost:5678](http://localhost:5678)**

   > Se `N8N_BASIC_AUTH` estiver ativo, use as credenciais definidas no `.env`. No primeiro acesso o n8n pedirá para criar o **usuário admin**.

---

## 🧪 Workflow 0 — Hello Webhook (aquecimento)

**Objetivo:** receber um JSON, transformar e responder algo útil.

**Passo a passo (UI):**

1. **Create → New Workflow** → renomeie para `hello-webhook`.
2. **Webhook (Trigger)**

   * *HTTP Method*: `POST`
   * *Path*: `hello-lab`
   * *Response*: **Using Respond to Webhook Node**
   * Clique **Execute Workflow** (modo Test).
3. **Function / Code** → cole:

   ```js
   // items[0].json contém o corpo do POST
   const name = (items[0].json.name || "Dev");
   return [{
     json: {
       message: `Olá, ${name}! Workflow no ar.`,
       timestamp: new Date().toISOString()
     }
   }];
   ```
4. **Respond to Webhook**

   * *Respond With*: `JSON`
   * *Response Body*: `{{$json}}`
5. **Testar (URL de Test)**:

   ```bash
   curl -X POST http://localhost:5678/webhook-test/hello-lab \
     -H "Content-Type: application/json" \
     -d '{"name":"Seu Nome"}'
   ```
6. **Activate** o workflow e use a URL de produção:

   ```bash
   curl -X POST http://localhost:5678/webhook/hello-lab \
     -H "Content-Type: application/json" \
     -d '{"name":"Seu Nome"}'
   ```

---

## 🛠️ Workflow 1 — Ingestão de Incidente (DevOps)

**Objetivo:** simular um alerta que vira *notificação* e *ticket* (mocks).

**Passo a passo (UI):**

1. **New Workflow** → `incident-ingest-lab`.
2. **Webhook (Trigger)**

   * *Method*: `POST`
   * *Path*: `incident-lab`
   * Espera: `service`, `severity`, `message`, `runbookUrl`
3. **Function / Code** (normalização):

   ```js
   const { service, severity, message, runbookUrl } = items[0].json;
   const sev = (severity || "info").toLowerCase();
   const sevOrder = { critical: 4, high: 3, medium: 2, low: 1, info: 0 };
   const priority = sevOrder[sev] >= 3 ? "P1" : (sevOrder[sev] === 2 ? "P2" : "P3");
   return [{
     json: {
       title: `[${priority}] ${service || "unknown"} - ${message || "no message"}`,
       severity: sev, priority, service: service || "unknown",
       runbookUrl: runbookUrl || null,
       createdAt: new Date().toISOString()
     }
   }];
   ```
4. **HTTP Request** (Mock Slack)

   * *Method*: `POST`
   * *URL*: `https://httpbin.org/post`
   * *JSON Body*:

     ```json
     { "channel": "#devops", "text": "{{$json.title}}" }
     ```
5. **HTTP Request** (Mock Jira)

   * *Method*: `POST`
   * *URL*: `https://httpbin.org/post`
   * *JSON Body*:

     ```json
     {
       "projectKey": "OPS",
       "summary": "{{$json.title}}",
       "labels": ["n8n","incident"],
       "custom": {
         "severity": "{{$json.severity}}",
         "service": "{{$json.service}}",
         "runbookUrl": "{{$json.runbookUrl}}"
       }
     }
     ```
6. **Respond to Webhook**

   * *Response Body*:

     ```json
     {
       "status": "queued",
       "title": "{{$json.title}}",
       "severity": "{{$json.severity}}",
       "service": "{{$json.service}}"
     }
     ```
7. **Ativar e testar**:

   ```bash
   curl -X POST http://localhost:5678/webhook/incident-lab \
     -H "Content-Type: application/json" \
     -d '{"service":"checkout","severity":"high","message":"Taxa de erro > 5%","runbookUrl":"https://runbooks/checkout"}'
   ```

> Depois troque os mocks por **Slack Incoming Webhook** real e **Jira/ServiceNow** (com *Credentials* no n8n).

---

## ⏱️ Workflow 2 — Digest diário de SLO

**Objetivo:** todo dia às 9h, consolidar métricas e postar um resumo (mock).

**Passo a passo (UI):**

1. **New Workflow** → `slo-digest-lab`.
2. **Schedule (Cron)**: *Minutes* `0`, *Hours* `9` (ajuste o fuso).
3. **HTTP Request** (mock métricas):

   * *URL*: `https://httpbin.org/anything`
4. **Function / Code** (error budget simulado):

   ```js
   const target = 0.999, actual = 0.997;
   const errorBudget = (1 - target);
   const consumed = (1 - actual);
   const remaining = Math.max(errorBudget - consumed, 0);
   return [{ json: { target, actual, errorBudget, consumed, remaining } }];
   ```
5. **HTTP Request** (Mock Slack):

   * *URL*: `https://httpbin.org/post`
   * *JSON Body*:

     ```json
     {
       "channel":"#sre-digest",
       "text":"SLO: alvo={{$json.target}}, atual={{$json.actual}}, restante={{$json.remaining}}"
     }
     ```

---

## 🧪 Testes rápidos (`docs/test-requests.http`)

```http
### Hello webhook
POST http://localhost:5678/webhook/hello-lab
Content-Type: application/json

{"name":"Vinicius"}

### Incident ingest
POST http://localhost:5678/webhook/incident-lab
Content-Type: application/json

{"service":"checkout","severity":"high","message":"Taxa de erro > 5%","runbookUrl":"https://runbooks/checkout"}
```

> Dica: na UI, use **Test URL** enquanto constrói e só então **Activate** para trocar para `/webhook/...`.

---

## 📦 Versionando workflows (para o GitHub)

* No editor: **Workflow → Export** e salve o `.json` em `./workflows/` (ex.: `incident-ingest-lab.json`).
* Comite `docker-compose.yml`, `.env.example`, `README.md`, `docs/` e `workflows/*.json`.
* **Nunca** commite `.env` real ou segredos.

---

## ⚙️ Executando com arquivos de ambiente dedicados (`--env-file`)

Se preferir organizar seus ambientes em arquivos separados (ex.: `env/n8n.local.env`):

```bash
mkdir -p env
cp .env.example env/n8n.local.env
docker compose --env-file env/n8n.local.env up -d
```

> Mantenha variantes como `env/n8n.dev.env`, `env/n8n.prod.env` e escolha com `--env-file`.

---

## 🔐 Boas práticas (mesmo no lab)

* Defina `N8N_ENCRYPTION_KEY` e use volume persistente para `/home/node/.n8n`.
* Troque mocks por integrações reais via **Credentials**.
* Configure **retries/backoff** e ramos de **Error** para robustez.
* Separe ambientes e parametrize por **ENV**.

---

## 🗂️ `.gitignore` sugerido

```gitignore
# Env files
.env
.env.*
env/*.env
.env/*.env

# n8n local data (se mapear volume)
n8n_data/
home/node/.n8n/

# OS/Editor
.DS_Store
Thumbs.db
.idea/
.vscode/
```

---

## 📄 Licença

Este projeto usa a licença **MIT**. Veja [LICENSE](./LICENSE).

---

Feito com ❤️ para prática de DevOps/SRE com n8n.
