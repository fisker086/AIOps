# Steward — the agent workbench for production ops

[中文](README.md)

---

## English

### Introduction

Steward — the agent workbench for production ops — is an open-source AI Agent platform with Agent management, Skill system, MCP protocol, knowledge-base RAG, workflows, IM bots, chat, and RBAC.

- **Backend**: Go (Hertz) + PostgreSQL
- **Frontend**: Optional embedded Vue3 Web UI
- **Desktop**: Optional Tauri desktop client (see `aiops-desktop/`)

### Features

| Feature | Description |
|---------|-------------|
| Agent Management | Create, configure, bind skills / MCP / knowledge bases, execution modes (default, ReAct, plan-execute) |
| Skill System | Skill encapsulation and dynamic loading |
| MCP Protocol | Model Context Protocol tool binding per agent |
| Knowledge Base | Document upload & HTTPS URL import (OpenViking vector store), preview, semantic search, agent RAG |
| Workflows | Visual graph workflows and sequential multi-agent chat flows |
| Chat | Multi-session, streaming, long-term memory, attachments |
| IM Bots | Lark / Feishu, DingTalk, Telegram — agents reply in IM channels |
| SSO | OAuth login via Feishu / Lark (`feishu`=CN Feishu, `lark`=intl), plus password login |
| Schedules | Cron-style scheduled agent runs |
| RBAC | Users, roles, permissions |
| Long-term Memory | OpenViking Session extraction and recall; PostgreSQL transcripts |
| Sandbox Execution | Sandboxed code execution: remote CubeSandbox micro-VMs by default (`SANDBOX_TYPE=cube`, official Go SDK, KVM/PVM isolation, egress denied by default); custom providers via `RegisterSandbox` |
| Observability | Langfuse tracing (optional) |

### Knowledge Base (optional)

Requires PostgreSQL + OpenViking.

1. Enable in `.env`:
   ```bash
   OPENVIKING_ENABLED=true
   OPENVIKING_URL=http://127.0.0.1:1933
   OPENVIKING_API_KEY=<matches deploy/openviking/ov.conf>
   ```
2. Start OpenViking with Compose profile:
   ```bash
   docker compose --profile openviking up -d
   ```
   Or point `OPENVIKING_URL` at an external OpenViking instance.

**What you get:**
- Upload PDF, Markdown, Office, HTML, etc.; async indexing via OpenViking
- Import one or many HTTPS document URLs from the UI dialog
- Preview originals from local storage; semantic search in the KB detail page
- Bind knowledge bases to agents — RAG runs on web chat, IM bots, and scheduled jobs

**Storage:** Original files live under `UPLOAD_DIR` (default `./uploads`, Docker `/data/uploads`). Use `UPLOAD_HOST_PATH` in `.env` to bind-mount host disk in Compose. Vector data is in the OpenViking volume — back up Postgres, `UPLOAD_DIR`, and OpenViking data together.

### Requirements

| Component | Minimum Version |
|-----------|-----------------|
| Docker | 20.10+ |
| Docker Compose | 2.0+ |
| PostgreSQL | 15+ |
| Node.js | 22+ (frontend dev) |
| Go | 1.24+ (backend dev) |
| OpenViking | optional, for knowledge base |

### Quick Start

#### One command (password login, PostgreSQL / Redis / MinIO only)

```bash
docker compose up -d
# postgres + redis + minio + minio-init + app (one image serves both the SPA and the API).
# On first boot the app generates random JWT / admin password / Vault key into the
# aiops-config volume and prints the admin password once:
docker compose logs app | grep admin_password
open http://localhost:8080
```

No `.env` is needed: internal service addresses and the default mode
(`AUTH_TYPE=password`, `LLM_GATEWAY_TYPE=none`) are fixed in the Compose file, so
a local development `.env` cannot leak into the containers. Log in with `admin`
and the one-time password from the logs; restarts neither reprint it nor rotate
the secrets. For overrides (upstream API key, external DB/S3, SSO), edit
`/data/config/.env` in the config volume and run
`docker compose up -d --force-recreate app` — that file wins. LiteLLM is an
optional profile (`docker compose --profile litellm up -d`).
`docker compose down` keeps data; add `-v` to delete volumes.

#### Custom Docker deployment

The steps below build the Zenith image with `Dockerfile`, start
PostgreSQL/Redis/MinIO/LiteLLM through Compose, and run the application with
`docker run -d`. Use them only when you need a layout different from the
one-command deployment above.

```bash
cp .env.example .env
# Edit .env — required: JWT_SECRET_KEY, ADMIN_DEFAULT_PASSWORD,
# LLM_GATEWAY_ADMIN_KEY

# 1. Build the Zenith image
docker build -t zenith:latest .

# 2. Prepare persistent upload storage
sudo mkdir -p /data/zenith/uploads
sudo chown -R 10001:10001 /data/zenith/uploads

# 3. Start dependencies
docker compose up -d postgres redis litellm minio minio-init

# 4. Run the Zenith application
docker run -d \
  --name zenith-app \
  --restart unless-stopped \
  --network aiops-net \
  --env-file .env \
  -e SERVER_HOST=0.0.0.0 \
  -e SERVER_PORT=8080 \
  -e DATABASE_URL='postgres://postgres:postgres@postgres:5432/aiops?sslmode=disable' \
  -e REDIS_URL='redis://redis:6379/0' \
  -e OBJECT_STORE_ENDPOINT='http://minio:9000' \
  -e UPLOAD_DIR=/data/uploads \
  -e AIOPS_ENV_FILE=/data/config/.env \
  -e SANDBOX_TYPE=local \
  -p 8080:8080 \
  -v zenith-config:/data/config \
  -v /data/zenith/uploads:/data/uploads \
  zenith:latest

# 5. Check logs and health
docker logs -f zenith-app
curl http://localhost:8080/healthz
```

LiteLLM admin key configuration (Zenith and LiteLLM must use the same key):

```dotenv
LLM_GATEWAY_TYPE=litellm
LLM_GATEWAY_BASE_URL=http://litellm:4000
LLM_GATEWAY_ADMIN_KEY=sk-zenith-admin-change-me
LLM_GATEWAY_VIRTUAL_KEY_ENABLED=true
LITELLM_PORT=4000
```

`LLM_GATEWAY_ADMIN_KEY` is used by Zenith to call LiteLLM management APIs
(for example, `/key/generate` to create one virtual key per user), and is also
passed to LiteLLM as `LITELLM_MASTER_KEY`. Zenith does not call the real model
directly.

If the gateway is published behind Cloudflare Access, also set a service token
(Access → Service Auth → Service Tokens). Both values are required together;
Zenith sends them as `CF-Access-Client-Id` / `CF-Access-Client-Secret` on admin
calls. Prefer reaching the origin directly when you can:

```dotenv
LLM_GATEWAY_CF_ACCESS_CLIENT_ID=xxxxx.access
LLM_GATEWAY_CF_ACCESS_CLIENT_SECRET=yyyyy
```

LiteLLM proxies the real model. Provider API keys, base URLs, and model
configuration belong on the LiteLLM side. The repository's
`deploy/litellm/config.yaml` is only an OpenAI-compatible upstream example;
that example needs `OPENAI_API_KEY` in the LiteLLM container. If you use an
already configured external LiteLLM, Zenith only needs the gateway URL and
admin key.

### Running without LiteLLM (password login + user-supplied model keys)

Leave `LLM_GATEWAY_TYPE` unset and it follows `AUTH_TYPE`:

| AUTH_TYPE | Default gateway |
|-----------|-----------------|
| `password` | `none` (direct; user fills Base URL + key) |
| `lark` / `feishu` / other SSO | `litellm` |

Test or single-user deployments only need:

```dotenv
AUTH_TYPE=password
# Do not set LLM_GATEWAY_TYPE=litellm unless you want a gateway override
```

What changes:

- Password login never mints a LiteLLM virtual key;
- Users register an OpenAI-compatible provider (base URL, API key, model) on the Models
  page; chat calls go straight to that upstream with no proxy in between;
- The model catalog accepts a typed upstream model id — no gateway sync needed;
- **Degraded features**: the spend/quota chip is hidden (upstreams do not expose LiteLLM's
  activity API), and daily budget caps are no longer enforced by the proxy.

Knowledge base + bundled OpenViking:

```bash
# Set OPENVIKING_ENABLED=true and OPENVIKING_API_KEY in .env first
docker compose --profile openviking up -d postgres redis litellm minio minio-init openviking
```

Stop and remove the application container without deleting data volumes:

```bash
docker stop zenith-app
docker rm zenith-app
```

Alternatively, use bare `docker compose up -d` to build and manage the whole
stack, including the application.

To use an external PostgreSQL instead, set `DATABASE_URL` in `.env` (the stack still starts the bundled `postgres` service unless you adjust compose with an override).

**Access:**
- Web: http://localhost:8080
- API: http://localhost:8080/api/v1
- Swagger: http://localhost:8080/swagger

#### Local Development

```bash
# Backend
cp .env.example .env
go run ./cmd/server

# Frontend (separate terminal)
cd ui && npm install && npm run dev
```

### Sandbox Execution (CubeSandbox)

> Full deploy + integration guide: [docs/cube-sandbox-部署与对接.md](docs/cube-sandbox-部署与对接.md) (PVM / bare-metal install, template creation, `.env` config, verification & troubleshooting).

Code execution goes through **CubeSandbox** (Tencent Cloud Container micro-VMs, E2B-compatible, KVM/PVM isolation) by default: every run starts a fresh micro-VM on the Cube node (`<60ms` cold start, snapshot/resume supported), the Steward server submits code and reads back stdout/stderr/exit code over the official Go SDK, then destroys the sandbox. The Steward host needs no Docker / bwrap / KVM — any platform with outbound access to the Cube control plane works (macOS included).

```bash
# ——— Steward side (.env; requires Cube env + template id) ———
SANDBOX_TYPE=cube
CUBE_API_URL=http://<cube-api-host>:3000      # or E2B_API_URL
CUBE_API_KEY=<cube api key>                   # or E2B_API_KEY
CUBE_TEMPLATE_ID=<cube template id>
# Optional: CUBE_SANDBOX_DOMAIN=cube.app  CUBE_PROXY_NODE_IP/...  CUBE_TIMEOUT / CUBE_REQUEST_TIMEOUT
```

**Choosing a provider**: only `SANDBOX_TYPE=cube` is bundled (custom providers implement `workflow.Sandbox` and call `RegisterSandbox`). Each run: create sandbox → inject `Env` (`os.environ` / `process.env`) and `input.mapping` globals → run via envd process API (`/bin/bash -l -c`) with python3 / node / bash and a real exit code → return result + duration → kill sandbox. Egress is **denied by default**; set `Resource.network=true` to allow it (`allowInternetAccess`).

> Note: a remote micro-VM cannot bind-mount the Steward host's `workspace_root`; `WorkspaceHost` is passed to the Cube node as metadata (`hostdir-mount`) and an env var, so workspace files must reach the node through Cube's host-dir mount / volume mechanism — it is not the same tree as the local `file_workspace`.

- Verify isolation: `GET /api/v1/sandbox/status` shows `type=cube`, `ready=true`; `POST /api/v1/sandbox/probe` returns `host_marker_seen=false` (the micro-VM cannot see Steward host files).
- Missing config: if `CUBE_TEMPLATE_ID` is unset the server fails fast at boot and disables code execution (no silent fallback).

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_PORT` | `8080` | HTTP port |
| `JWT_SECRET_KEY` | - | **Required** JWT secret |
| `MODEL_TYPE` | `openai` | Default chat model: `openai` / `ark` / `anthropic` (`claude`) / `gemini` / `deepseek` / `ollama` |
| `OPENAI_API_KEY` | - | Direct OpenAI mode or the bundled LiteLLM example's upstream credential |
| `ANTHROPIC_API_KEY` | - | Anthropic Claude key (when `MODEL_TYPE=anthropic`) |
| `GEMINI_API_KEY` | - | Google Gemini key (when `MODEL_TYPE=gemini`) |
| `DEEPSEEK_API_KEY` | - | DeepSeek key (when `MODEL_TYPE=deepseek`) |
| `OLLAMA_MODEL` | - | Local Ollama model (when `MODEL_TYPE=ollama`) |
| `APPROVAL_PROVIDER_TYPE` | - | External approval: `lark` (empty = in-app approvals) |
| `DATABASE_URL` | - | PostgreSQL connection string |
| `OPENVIKING_ENABLED` | `false` | `true` → OV memory/skills/KB; `false` → session_search |
| `UPLOAD_DIR` | `uploads` | Chat attachments & KB originals (persist this path) |
| `UPLOAD_HOST_PATH` | - | Docker Compose: host path bind-mounted to `/data/uploads` |
| `OPENVIKING_URL` | `http://127.0.0.1:1933` | OpenViking HTTP base URL |
| `OPENVIKING_API_KEY` | - | OpenViking API key |
| `AUTH_TYPE` | `password` | `password` / SSO: `feishu`, `lark` (`feishu`=CN Feishu, `lark`=intl) |
| `LANGFUSE_*` | - | Optional LLM observability |

See `.env.example` for the full variable list.

### Usage Guide

| Task | Where in UI |
|------|----------------|
| Create knowledge base | **Knowledge** → New KB → upload files or **Import from URL** (one HTTPS link per line) |
| Bind KB to agent | **Agents** → edit agent → select knowledge bases (`kb_ids`) |
| Test retrieval | **Knowledge** → open KB → **Search** tab |
| Preview document | Click filename in document list (side drawer) |
| Graph workflow | **Workflows** → visual editor (agent / knowledge / tool nodes) |
| IM bot | **Agents** → enable IM (Lark / DingTalk / Telegram) and fill app credentials |
| Scheduled runs | **Schedules** → cron + target agent |

**Agent RAG flow:** User message → bound KBs searched via OpenViking → top chunks injected into context → LLM reply. Works on web chat, IM bots, and scheduled jobs (not only HTTP `/chat`).

**OpenViking config:** When using the bundled service, set `root_api_key` in `deploy/openviking/ov.conf` to match `OPENVIKING_API_KEY`, and configure `providers.embedding` (and optionally `vlm`) API keys — indexing fails without a working embedding provider.

### Project Layout

```
cmd/server/          # Main API entry
internal/
  agent/             # Runtime, ReAct, tools (MCP + builtin skills)
  service/           # Chat, knowledge base, workflows
  openviking/        # OpenViking HTTP client
  controller/        # REST handlers
  dingtalkbot/       # DingTalk bot
  larkbot/           # Lark / Feishu bot
  telegrambot/       # Telegram bot
ui/                  # Vue 3 + Quasar SPA
skills/              # SKILL.md skill definitions
deploy/openviking/   # OpenViking ov.conf for Compose profile
docker-compose.yml
```

### Troubleshooting

| Symptom | Check |
|---------|--------|
| KB stuck on **indexing** | OpenViking reachable; embedding API key in `ov.conf`; server logs `kb: indexing started` / `document indexed` |
| Agent ignores KB | Agent has `kb_ids` saved; `OPENVIKING_ENABLED=true`; logs show `knowledge bases bound for agent` and `knowledge base retrieval started` |
| IM bot no RAG | Same as above — IM uses Runtime RAG (look for retrieval logs on each message) |
| Upload lost after restart | Persist `UPLOAD_DIR` / set `UPLOAD_HOST_PATH` in Compose |
| `kb_retrieval_enabled=false` in logs | KB service not initialized (needs Postgres + `OPENVIKING_ENABLED`) |

Useful log lines: `knowledge base feature enabled`, `knowledge bases bound for agent`, `knowledge base retrieval completed` (with `hit_count`).

### Tech Stack

- **Backend**: Go, Hertz, GORM, OpenViking (optional)
- **Frontend**: Vue 3, Quasar, TypeScript
- **Desktop**: Tauri, React
- **Database**: PostgreSQL 15+
- **LLM**: OpenAI-compatible API, Volcengine Ark

### License

Apache License 2.0
