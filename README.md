# n8n DevOps Lab

Automação e orquestração com **n8n** para um contexto **DevOps/SRE** — pronto para rodar localmente via Docker e servir de **projeto-portfólio** no GitHub.

> Conteúdo: setup local, 3 workflows “MVP” (webhook hello, ingestão de incidente e digest SLO), testes via `curl`, versionamento e próximos passos.

## 🧱 Arquitetura do lab

```
flowchart LR
  A[Cliente/CLI/cURL] -->|HTTP POST| B[Webhook Trigger]
  B --> C[Function/Code (JS)]
  C --> D[HTTP Request (Mock Slack/Jira)]
  subgraph n8n (queue mode)
    B
    C
    D
  end
  n8n <--> Redis[(Redis Queue)]
  n8n <--> Postgres[(Postgres)]
```

## 🔧 Pré-requisitos
- Docker e Docker Compose
- `curl` para testes
- Porta `5678` livre

## 🚀 Subir o ambiente local

1. Copie o arquivo `.env.example` para `.env` e ajuste se necessário:
   ```bash
   cp .env.example .env
   ```
2. Suba os serviços:
   ```bash
   docker compose up -d
   ```
3. Acesse o editor do n8n: **http://localhost:5678**  
   > Se ativou `N8N_BASIC_AUTH`, use `admin / change-me` (ou os valores que você definiu). Na primeira entrada, o n8n pedirá para criar o **usuário admin** da aplicação.

## 🧪 Workflow 0 — Hello Webhook (aquecimento)
Objetivo: receber um JSON, transformá-lo e responder algo útil.

**Passo a passo (UI):**
1. **Create** → **New Workflow** → renomeie para `hello-webhook`.
2. Adicione **Webhook** (Trigger):  
   - *HTTP Method*: `POST`  
   - *Path*: `hello-lab`  
   - Clique em **Execute Workflow** (fica “escutando” a URL de **Test**).
3. Adicione **Function** (ou *Code*): cole:
   ```js
   // items[0].json contém o corpo do POST
   const name = (items[0].json.name || "Dev");
   return [{ json: { message: `Olá, ${name}! Workflow no ar.`, timestamp: new Date().toISOString() } }];
   ```
4. Adicione **Respond to Webhook**:  
   - *Response Body*: selecione `{{$json}}` (o JSON do nó anterior).
5. **Save** → **Execute Workflow** (deixe em modo Test).  
6. Em outro terminal, teste:
   ```bash
   curl -X POST http://localhost:5678/webhook-test/hello-lab \
     -H "Content-Type: application/json" \
     -d '{"name":"Seu Nome"}'
   ```
7. Quando estiver ok, **Activate** o workflow e use a URL “Production”:
   ```bash
   curl -X POST http://localhost:5678/webhook/hello-lab \
     -H "Content-Type: application/json" \
     -d '{"name":"Seu Nome"}'
   ```

## 🛠️ Workflow 1 — Ingestão de Incidente (DevOps)
Objetivo: simular um alerta que vira *ticket* e *notificação* (usando mocks).

**Passo a passo (UI):**
1. **New Workflow** → `incident-ingest-lab`.
2. **Webhook** (Trigger):
   - *Method*: `POST`
   - *Path*: `incident-lab`
   - Campos esperados (exemplo): `service`, `severity`, `message`, `runbookUrl`
3. **Function/Code** (normalização):
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
4. **HTTP Request** (Mock Slack):  
   - *Method*: `POST`  
   - *URL*: `https://httpbin.org/post`  
   - *JSON/Body*: 
     ```json
     {
       "channel": "#devops",
       "text": "{{$json.title}}"
     }
     ```
5. **HTTP Request** (Mock Jira):  
   - *Method*: `POST`  
   - *URL*: `https://httpbin.org/post`  
   - *JSON/Body*:
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
6. **Merge** (caso use) ou ligue ambos ao **Respond to Webhook** com um pequeno resumo.
7. **Activate** e teste:
   ```bash
   curl -X POST http://localhost:5678/webhook/incident-lab \
     -H "Content-Type: application/json" \
     -d '{"service":"checkout","severity":"high","message":"Taxa de erro > 5%","runbookUrl":"https://runbooks/checkout"}'
   ```

> Dica: depois troque os mocks por **Slack Incoming Webhook** real e a API do **Jira/ServiceNow** com credenciais seguras.

## ⏱️ Workflow 2 — Digest diário de SLO/SLA
Objetivo: todo dia às 9h, consolidar métricas e postar um resumo (mock).

**Passo a passo (UI):**
1. **New Workflow** → `slo-digest-lab`.
2. **Schedule** (Trigger):
   - *Mode*: `Cron`
   - *Minutes*: `0`, *Hours*: `9`, *Days*: `*` (ajuste ao seu fuso)
3. **HTTP Request** (Mock de métricas):  
   - *URL*: `https://httpbin.org/anything`
   - Interprete como se fosse sua query ao Prometheus/Azure Monitor/CloudWatch.
4. **Function/Code** (cálculo de error budget simulado):
   ```js
   // Simula 99.9% target e 99.7% real
   const target = 0.999, actual = 0.997;
   const errorBudget = (1 - target);
   const consumed = (1 - actual);
   const remaining = Math.max(errorBudget - consumed, 0);
   return [{ json: { target, actual, errorBudget, consumed, remaining } }];
   ```
5. **HTTP Request** (Mock Slack):
   - *URL*: `https://httpbin.org/post`
   - *JSON/Body*:
     ```json
     {
       "channel":"#sre-digest",
       "text":"SLO: alvo={{$json.target}}, atual={{$json.actual}}, restante={{$json.remaining}}"
     }
     ```
6. **Activate** e valide a execução manual pelo botão **Execute**.

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

> Dica: na UI do n8n, use **Test URL** enquanto constrói e só então **Activate** para trocar para a URL de produção (`/webhook/...`).

## 📦 Versionando workflows (para o GitHub)
- No editor: **Workflow → Export** e salve o `.json` em `./workflows/` (ex.: `incident-ingest-lab.json`).
- Comite `docker-compose.yml`, `.env.example`, `README.md`, `docs/` e `workflows/*.json`.  
- **Nunca** commite `.env` real ou segredos.

## 🔐 Boas práticas (mesmo no lab)
- Use `N8N_ENCRYPTION_KEY` e volume persistente para `/home/node/.n8n`.
- Troque mocks por integrações reais via *Credentials* do n8n.
- Configure **retries/backoff** e ramos de **Error** para robustez.
- Separe ambientes: exporte/import workflows e parametrize por **ENV**.

## ➕ Próximos passos (idéias p/ portfolio)
- Adicionar **pipeline S3 ⇄ SFTP com PGP + comprovante no Teams**.
- Enriquecimento de alerta com **logs** (CloudWatch/Azure Monitor) antes do ticket.
- Publicar um **diagrama** e screenshots no README (sem segredos).

---

Feito com ❤️ para prática de DevOps/SRE com n8n.
