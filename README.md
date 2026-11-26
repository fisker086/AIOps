# Steward · 会自己干活的运维工作台

[English](README-en.md)

---

## 中文

### 简介

Steward · 会自己干活的运维工作台 是一个开源的 AI Agent 平台，支持 Agent 管理、技能系统、MCP 协议、知识库 RAG、工作流、IM 机器人、聊天对话与 RBAC 权限控制。

- **后端**：Go (Hertz) + PostgreSQL
- **前端**：可选嵌入的 Vue3 Web UI
- **桌面端**：可选 Tauri 客户端（见 `aiops-desktop/`）

### 功能特性

| 功能 | 说明 |
|------|------|
| Agent 管理 | 创建、配置，绑定 Skill / MCP / 知识库，支持多种执行模式（默认、ReAct、计划执行） |
| 技能系统 | Skill 封装与动态加载 |
| MCP 协议 | 按 Agent 绑定 MCP 工具 |
| 知识库 | 文档上传与 HTTPS 链接批量导入（OpenViking 向量库）、预览、语义检索、Agent RAG |
| 工作流 | 可视化图工作流、顺序多 Agent 对话流 |
| 聊天对话 | 多会话、流式输出、长期记忆、附件 |
| IM 机器人 | 飞书 / Lark、钉钉、Telegram — Agent 在 IM 渠道自动回复 |
| SSO 登录 | 飞书 / Lark OAuth（`feishu`=国内飞书，`lark`=国际版），以及密码登录 |
| 定时任务 | Cron 式调度 Agent 执行 |
| RBAC | 用户、角色、权限控制 |
| 长期记忆 | OpenViking Session 自动提取与语义召回；或自托管 PostgreSQL 记忆（`MEMORY_PROVIDER=postgres`，pg_trgm 相似度，无需外部向量服务）；PostgreSQL 保留会话原文 |
| 沙箱执行 | 代码沙箱执行：默认走 CubeSandbox 远程微 VM（官方 Go SDK，KVM/PVM 隔离，`SANDBOX_TYPE=cube`），网络默认拒绝外联；也可通过 `RegisterSandbox` 二开自定义 provider |
| 凭证库 | 自研 Credential Vault：AES-256-GCM 加密存储 API Key / token / kubeconfig / DSN 等，技能与脚本参数中用 `$secret.<name>` 引用，执行前注入、输出自动脱敏、全程审计；可选沙箱环境变量侧信道 |
| 可观测性 | Langfuse 链路追踪（可选） |

### 知识库（可选）

依赖 PostgreSQL + OpenViking。

1. 在 `.env` 中启用：
   ```bash
   OPENVIKING_ENABLED=true
   OPENVIKING_URL=http://127.0.0.1:1933
   OPENVIKING_API_KEY=<与 deploy/openviking/ov.conf 中一致>
   ```
2. 使用 Compose profile 启动 OpenViking：
   ```bash
   docker compose --profile openviking up -d
   ```
   也可将 `OPENVIKING_URL` 指向外部 OpenViking 服务。

**能力概览：**
- 支持 PDF、Markdown、Office、HTML 等格式上传；OpenViking 异步建立向量索引
- UI 弹窗支持单个或多个 HTTPS 链接批量导入
- 原文预览（本地存储）+ 知识库详情页语义检索
- Agent 绑定知识库后，Web 聊天、IM 机器人、定时任务均会自动 RAG 检索

**存储说明：** 原件保存在 `UPLOAD_DIR`（本地默认 `./uploads`，Docker 内 `/data/uploads`）。生产环境可在 `.env` 设置 `UPLOAD_HOST_PATH` 将宿主机目录挂进容器。向量数据在 OpenViking 卷中 — 需同时备份 Postgres、`UPLOAD_DIR` 与 OpenViking 数据。

### 环境要求

| 组件 | 最低版本 |
|------|----------|
| Docker | 20.10+ |
| Docker Compose | 2.0+ |
| PostgreSQL | 15+ |
| Node.js | 22+ (前端开发) |
| Go | 1.24+ (后端开发) |
| OpenViking | 可选，知识库功能需要 |

### 快速开始

#### 一键部署（密码登录，只依赖 PostgreSQL / Redis / MinIO）

```bash
docker compose up -d
# 拉起 postgres + redis + minio + minio-init + app（一个容器同时提供前端和后端）。
# 首次启动 app 会生成随机 JWT / 管理员密码 / Vault 密钥，写入 aiops-config 卷，
# 并在日志里打印一次管理员密码：
docker compose logs app | grep admin_password
open http://localhost:8080
```

无需预先创建 `.env`：容器内的服务地址与运行模式（`AUTH_TYPE=password`、
`LLM_GATEWAY_TYPE=none`）在 compose 里固定，仓库开发用 `.env` 不会带进容器。
首次登录用日志里的随机密码（用户名 `admin`）；之后重启不会重复打印，也不会轮换密钥。
要改配置（例如填 `OPENAI_API_KEY`、切外部数据库/S3、开 SSO），改配置卷里的
`/data/config/.env` 后 `docker compose up -d --force-recreate app` 即可 —— 应用以该文件为准。
模型在「模型」页手填上游 Base URL / API Key。SSO / 虚拟 Key 场景在那里设
`AUTH_TYPE=lark|feishu`、`LLM_GATEWAY_TYPE=litellm`，并加 `--profile litellm` 启动可选网关。
`docker compose down` 只停容器不删数据（加 `-v` 才删）。

#### Docker 部署

下面是使用 `Dockerfile` 构建 Zenith 镜像、再用 `docker run -d` 启动应用的完整步骤。
PostgreSQL、Redis、MinIO、LiteLLM 由 Compose 负责启动，应用本身单独用 `docker run` 运行。

```bash
cp .env.example .env
# 编辑 .env，至少填写：
# JWT_SECRET_KEY、ADMIN_DEFAULT_PASSWORD、LLM_GATEWAY_ADMIN_KEY
# 本地 MinIO 使用 OBJECT_STORE_* 配置，不需要 MINIO_ROOT_* 配置。

# 1. 构建 Zenith 镜像
docker build -t zenith:latest .

# 2. 准备 /data 持久化目录
sudo mkdir -p /data/zenith/uploads
sudo chown -R 10001:10001 /data/zenith/uploads

# 3. 启动 PostgreSQL、Redis、MinIO、LiteLLM 等依赖
# MinIO 控制台：http://localhost:19001
# MinIO API：http://localhost:19000
# LiteLLM API：http://localhost:4000
docker compose up -d postgres redis litellm minio minio-init

# 4. 启动 Zenith 应用
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

# 5. 查看日志和健康状态
docker logs -f zenith-app
curl http://localhost:8080/healthz
```

`.env` 中的本地对象存储配置示例：

```dotenv
OBJECT_STORE_TYPE=aws
OBJECT_STORE_BUCKET=lark
OBJECT_STORE_REGION=us-east-1
OBJECT_STORE_ENDPOINT=http://minio:9000
OBJECT_STORE_ACCESS_KEY=admin
OBJECT_STORE_SECRET_KEY=minio-change-me
OBJECT_STORE_API_PORT=19000
OBJECT_STORE_CONSOLE_PORT=19001
```

LiteLLM 管理密钥配置（必须填写，且 Zenith 与 LiteLLM 使用同一个值）：

```dotenv
LLM_GATEWAY_TYPE=litellm
LLM_GATEWAY_BASE_URL=http://litellm:4000
LLM_GATEWAY_ADMIN_KEY=sk-zenith-admin-change-me
LLM_GATEWAY_VIRTUAL_KEY_ENABLED=true
LITELLM_PORT=4000
```

`LLM_GATEWAY_ADMIN_KEY` 只用于 Zenith 调用 LiteLLM 管理接口
（例如通过 `/key/generate` 为每个用户创建 virtual key），同时会作为
LiteLLM 的 `LITELLM_MASTER_KEY`。Zenith 不直接连接真实模型。

网关若发布在 Cloudflare Access 之后，额外设置服务令牌（Access → Service Auth →
Service Tokens），两个值必须同时提供；Zenith 会在管理请求上带
`CF-Access-Client-Id` / `CF-Access-Client-Secret`。能内网直连 origin 时优先直连，
不要走边缘：

```dotenv
LLM_GATEWAY_CF_ACCESS_CLIENT_ID=xxxxx.access
LLM_GATEWAY_CF_ACCESS_CLIENT_SECRET=yyyyy
```

LiteLLM 再负责代理真实模型。真实模型的 API Key、Base URL 和模型配置应放在
LiteLLM 侧；当前仓库的 `deploy/litellm/config.yaml` 只是一个 OpenAI 兼容上游示例。
如果使用这个示例，需要给 LiteLLM 容器提供 `OPENAI_API_KEY`；如果使用已经配置好的
外部 LiteLLM，则 Zenith 只需要上面的网关地址和 admin key。

### 不使用 LiteLLM（账号密码登录 + 自带模型 Key）

`LLM_GATEWAY_TYPE` 可不写：未设置时按 `AUTH_TYPE` 自动推断——

| AUTH_TYPE | 默认网关 |
|-----------|----------|
| `password` | `none`（直连，手填 Base URL / Key） |
| `lark` / `feishu` / 其他 SSO | `litellm` |

测试或个人部署只需：

```dotenv
AUTH_TYPE=password
# 不要写 LLM_GATEWAY_TYPE=litellm；需要强制走网关时再显式写
```

此时的行为：

- 密码登录不会创建 LiteLLM virtual key；
- 在「模型」页由用户自己填写 provider（openai 兼容）、Base URL、API Key、模型名，
  聊天请求直接发往该上游，不经过任何代理；
- 「模型目录」里可以直接手输上游模型名，不需要从网关同步；
- **功能降级**：用量/额度面板会隐藏（上游不提供 LiteLLM 的 activity 接口），
  每日预算上限不再由代理强制执行。

如果修改了 PostgreSQL 用户名或密码，需要同步修改 `docker run` 命令里的
`DATABASE_URL`。`/data/zenith/uploads` 和 Docker 卷 `zenith-config` 需要纳入备份。

停止和删除应用容器（不会删除数据卷）：

```bash
docker stop zenith-app
docker rm zenith-app
```

也可以直接用裸 Compose 命令完成全量启动（应用也由 Compose 管理）：

```bash
docker compose up -d
```

启用知识库 + 内置 OpenViking：

```bash
# 先在 .env 中设置 OPENVIKING_ENABLED=true 和 OPENVIKING_API_KEY
docker compose --profile openviking up -d postgres redis litellm minio minio-init openviking
```

若改用外部 PostgreSQL，在 `.env` 中设置 `DATABASE_URL`（默认仍会启动 compose 内的 `postgres` 服务，高级场景可自行用 override 禁用）。

**访问：**
- Web: http://localhost:8080
- API: http://localhost:8080/api/v1
- Swagger: http://localhost:8080/swagger

#### 本地开发

```bash
# 后端
cp .env.example .env
go run ./cmd/server

# 前端（另开终端）
cd ui && npm install && npm run dev
```

### 沙箱执行（Cube / OpenKruise / local）

> Cube 部署与对接见 [docs/cube-sandbox-部署与对接.md](docs/cube-sandbox-部署与对接.md)。OpenKruise Agents（SandboxSet 预热池）见 [docs/openkruise-agents-sandbox.md](docs/openkruise-agents-sandbox.md) 与 `deploy/openkruise/`。

代码执行默认走 **CubeSandbox**（E2B 兼容 MicroVM）。也可切到 **OpenKruise Agents**（`SANDBOX_TYPE=openkruise`，从 SandboxSet 暖池领取）或本地子进程（`local`）。两侧远程 provider 共用 E2B Create→Files→Commands→Kill 路径；自定义 provider 可 `RegisterSandbox`。

```bash
# —— Cube（默认）——
SANDBOX_TYPE=cube
CUBE_API_URL=http://<cube-api-host>:3000
CUBE_API_KEY=<cube api key>
CUBE_TEMPLATE_ID=<cube template id>

# —— 或 OpenKruise Agents（K8s 统一后的第二 provider）——
# SANDBOX_TYPE=openkruise
# OPENKRUISE_API_URL=http://sandbox-manager.<ns>.svc:<port>
# OPENKRUISE_API_KEY=<token>
# OPENKRUISE_TEMPLATE_ID=code-interpreter   # 必须等于 SandboxSet.metadata.name
# OPENKRUISE_USER_VOLUME=true               # 按 (user, agent) 长期 PVC，Kill 不删数据
# OPENKRUISE_VOLUME_NAMESPACE=agents-sandbox
# OPENKRUISE_VOLUME_STORAGE_CLASS=<sc>
```

每次执行：领取/新建沙箱 → 注入 `Env` 与 `input.mapping` → envd 跑 python3/node/bash → 销毁。网络默认拒绝出网（`Resource.network=true` 才允许）。远程工作区仍走 Files API 同步（非 bind-mount）。

- 确认：`GET /api/v1/sandbox/status` 的 `type`/`ready`；`POST /api/v1/sandbox/probe` 应 `host_marker_seen=false`。
- 配置缺失：远程 template id 未设置时启动失败并回退 `local`。

### 凭证库（Credential Vault）

不依赖外部 vault 产品，自研加密存储。敏感值以 AES-256-GCM 落库（带随机 nonce），界面上永不回显明文；任何技能/HTTP 头/脚本参数可直接写 `$secret.<name>` 占位符，工具执行**前一刻**才解密注入，执行后从输出中打码（长度 ≥ `VAULT_MIN_SECRET_LEN` 才替换）。

```bash
# .env —— 主密钥（base64 的 32 字节）
# 生成：openssl rand -base64 32
VAULT_MASTER_KEY=<...>
# 留空时自动从 JWT_SECRET_KEY 派生（弱，仅限小规模部署）
```

- **使用**：管理后台 → **凭证库**（管理员）→ 新建凭证，得到名字如 `github-read` → 在技能参数里写 `"Authorization": "Bearer $secret.github-read"` → 提交后运行前自动替换、输出自动打码。
- **授权范围（scope）**：`agent:<id>` / `env=prod` 等标签，逗号分隔；留空 = 全局。agent 运行的工具以 `agent:<id>` 作为请求 scope，越权凭证解析会被拒绝并记录。
- **审计**：每次解析写入 `vault_credential_use_logs`（agent / 用户 / 会话 / 工具 / 状态）。界面可查看“使用记录”。
- **沙箱侧信道（可选）**：`VAULT_SANDBOX_ENV=true` 时，凡“带 `EnvName` 且 scope 为空”的凭证会作为环境变量注入每次沙箱代码运行，代码可经 `os.environ["<EnvName>"]` / `process.env` 读取，无需占位符。

### 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `SERVER_PORT` | `8080` | HTTP 端口 |
| `JWT_SECRET_KEY` | - | **必填** JWT 密钥 |
| `MODEL_TYPE` | `openai` | 默认聊天模型：`openai` / `ark` / `anthropic` (`claude`) / `gemini` / `deepseek` / `ollama` |
| `OPENAI_API_KEY` | - | 仅用于直接调用 OpenAI，或仓库内 LiteLLM 示例配置的上游凭证 |
| `ANTHROPIC_API_KEY` | - | Anthropic Claude 密钥（当 `MODEL_TYPE=anthropic`） |
| `GEMINI_API_KEY` | - | Google Gemini 密钥（当 `MODEL_TYPE=gemini`） |
| `DEEPSEEK_API_KEY` | - | DeepSeek 密钥（当 `MODEL_TYPE=deepseek`） |
| `OLLAMA_MODEL` | - | 本地 Ollama 模型（当 `MODEL_TYPE=ollama`） |
| `APPROVAL_PROVIDER_TYPE` | - | 外部审批：`lark`（空 = 站内审批） |
| `DATABASE_URL` | - | PostgreSQL 连接串 |
| `OPENVIKING_ENABLED` | `false` | `true` → OV 记忆/技能/知识库；`false` → session_search |
| `MEMORY_PROVIDER` | - | 长期记忆后端显式选择：`openviking` / `postgres`（PG pg_trgm，无外部依赖）/ `none`；留空沿用 `OPENVIKING_ENABLED` |
| `UPLOAD_DIR` | `uploads` | 聊天附件与知识库原件目录（需持久化） |
| `UPLOAD_HOST_PATH` | - | Docker Compose：宿主机路径挂载到 `/data/uploads` |
| `OPENVIKING_URL` | `http://127.0.0.1:1933` | OpenViking HTTP 地址 |
| `OPENVIKING_API_KEY` | - | OpenViking API Key |
| `AUTH_TYPE` | `password` | `password` / SSO：`feishu`、`lark`（`feishu`=国内飞书，`lark`=国际版） |
| `VAULT_MASTER_KEY` | - | 凭证库主密钥（base64 32 字节，`openssl rand -base64 32`）；`/setup` 首次初始化会自动生成，留空则从 `JWT_SECRET_KEY` 派生 |
| `VAULT_MIN_SECRET_LEN` | `8` | 输出脱敏的最小密钥长度 |
| `VAULT_SANDBOX_ENV` | `false` | `true` → 全局 `EnvName` 凭证作为环境变量注入沙箱 |
| `LANGFUSE_*` | - | 可选，LLM 可观测性 |

完整变量列表见 `.env.example`。

### 使用指南

| 操作 | UI 入口 |
|------|---------|
| 创建知识库 | **知识库** → 新建 → 上传文件或 **链接导入**（每行一个 HTTPS 链接） |
| Agent 绑定知识库 | **智能体** → 编辑 → 选择知识库（`kb_ids`） |
| 测试检索 | **知识库** → 进入详情 → **检索** 标签页 |
| 预览文档 | 文档列表中点击文件名（侧栏抽屉） |
| 图工作流 | **工作流** → 可视化编辑器（Agent / 知识库 / 工具节点） |
| IM 机器人 | **智能体** → 启用 IM（飞书 / 钉钉 / Telegram）并填写应用凭证 |
| 定时任务 | **定时任务** → Cron + 目标 Agent |
| 凭证库 | **凭证库**（管理员）→ 新建凭证 → 技能/脚本中以 `$secret.<name>` 引用 |

**Agent RAG 流程：** 用户消息 → OpenViking 检索已绑定知识库和当前用户/Agent 的长期记忆 → 注入相关片段 → LLM 生成回复。会话原文仍保存在 PostgreSQL，供历史记录、附件和审计 API 使用。

**OpenViking 配置：** 使用 Compose 内置服务时，`deploy/openviking/ov.conf` 中的 `root_api_key` 需与 `.env` 的 `OPENVIKING_API_KEY` 一致；`providers.embedding` 和 `providers.vlm` 需配置有效 API Key。Embedding 用于知识库和记忆检索，VLM/LLM 用于 Session 提交后的记忆提取；配置无效会导致索引或记忆提取失败。

### 目录结构

```
cmd/server/          # 主程序入口
internal/
  agent/             # Runtime、ReAct、工具（MCP + 内置 Skill）
  service/           # 聊天、知识库、工作流
  knowledge/openviking/ # OpenViking HTTP 客户端
  memory/openviking/    # OpenViking Session 长期记忆 Provider
  controller/        # REST API
  dingtalkbot/       # 钉钉机器人
  larkbot/           # 飞书机器人
  telegrambot/       # Telegram 机器人
ui/                  # Vue 3 + Quasar 前端
skills/              # SKILL.md 技能定义
deploy/openviking/   # Compose profile 用的 ov.conf
docker-compose.yml
```

### 常见问题

| 现象 | 排查 |
|------|------|
| 文档一直 **索引中** | OpenViking 是否可达；`ov.conf` 中 embedding API Key；日志是否有 `kb: document indexed` |
| Agent 不用知识库 | 是否保存了 `kb_ids`；`OPENVIKING_ENABLED=true`；日志是否有 `knowledge bases bound for agent`、`knowledge base retrieval started` |
| IM 机器人无 RAG | 同上 — IM 走 Runtime 检索，每条消息应出现 retrieval 相关日志 |
| 重启后上传文件丢失 | 持久化 `UPLOAD_DIR`，或 Compose 中配置 `UPLOAD_HOST_PATH` |
| 日志 `kb_retrieval_enabled=false` | 知识库未初始化（需 Postgres + `OPENVIKING_ENABLED`） |

关键日志：`knowledge base feature enabled`、`knowledge bases bound for agent`、`knowledge base retrieval completed`（含 `hit_count`）。

### 技术栈

- **后端**: Go, Hertz, GORM, OpenViking（可选）
- **前端**: Vue 3, Quasar, TypeScript
- **桌面端**: Tauri, React
- **数据库**: PostgreSQL 15+
- **LLM**: OpenAI 兼容 API, 火山方舟

### 许可证

Apache License 2.0
