# CODEBUDDY.md This file provides guidance to CodeBuddy when working with code in this repository.

## 仓库现状

仓库处于**设计阶段**，尚无任何源码、依赖清单或构建脚本，仅有以下两份文档：

- `架构设计.md` — 开发阶段唯一契约（当前 v1.1）：模块划分、数据模型、API 契约均以此为准。改任何模块前先读它。
- `项目需求清单-协同知识库.md` — 功能范围与 P0~P3 优先级、验收底线。

计划中的三个服务目录 `api/`、`web/`、`collab/` 尚未创建。下文命令按设计文档约定的技术栈书写，**尚未落地**；目录或脚本名与实际情况不一致时，以实际配置为准，并同步修正本文件。

## 常用命令（计划中）

### 基础设施

```bash
docker compose up -d postgres redis minio   # 仅拉起中间件，后端/前端在宿主机调试
docker compose up --build                   # 全栈一键构建启动（P3 验收用）
```

数据库需 PostgreSQL 16 + `pgvector` 扩展；全文检索为**应用层 jieba 分词 + DB `simple` 配置**（免编译 zhparser 自建镜像）。

### 后端 `api/`

```bash
uvicorn app.main:app --reload --port 8000   # 工作目录为 api/；Swagger 在 /docs，ReDoc 在 /redoc
alembic revision --autogenerate -m "描述"    # 由 SQLAlchemy 模型生成迁移，生成后必须人工复核
alembic upgrade head                        # 应用迁移
arq app.workers.main.WorkerSettings         # 启动 ARQ worker（摘要/Embedding/清理等异步任务）
pytest                                      # 全量测试
pytest tests/test_permission_service.py::test_xxx -q   # 单个用例
```

设计文档未指定 lint/format 工具，落地时若引入 ruff 等，需在本节补齐命令，不要在无配置的情况下假定其存在。

### 前端 `web/`

```bash
pnpm install
pnpm dev      # Next.js 开发服务器
pnpm build    # 生产构建
pnpm lint     # 依赖 package.json 中定义的 script
```

### 协同 `collab/`

Hocuspocus（Node）独立进程，对外提供 WebSocket，由 Nginx 以 `/collab` 路径转发并处理 WS 升级。启动命令取决于该目录的 `package.json`，尚未创建。

## 架构总览

### 拓扑与职责边界

Nginx 单入口分流：`/` → web，`/api` → api，`/collab` → collab（WebSocket 升级）。三个服务**必须**保持以下边界，跨界即架构退化：

| 服务 | 负责 | 明确不负责 |
|---|---|---|
| `web`：Next.js App Router + TS + Tailwind + shadcn/ui + Tiptap | 页面渲染、编辑器 UI、REST/SSE 调用、连接协同 WS | 业务规则 |
| `api`：FastAPI + SQLAlchemy 2.0 async + Pydantic v2 | 认证鉴权、业务 CRUD、权限判定、签发协同 Token、AI/RAG、全文检索、文件上传 | 实时内容同步 |
| `worker`：arq + Redis 队列 | 异步任务：摘要、Embedding、版本清理、审计落库 | HTTP 请求处理 |
| `collab`：Hocuspocus + Yjs CRDT | 实时同步、冲突合并、在线状态、断线恢复 | 业务数据校验（只验 Token）；**不直连 PostgreSQL**（经 /internal 由 api 落库） |

`api/` 内部分层固定为：`api/`（路由，只做参数校验 + 调 service）→ `services/`（业务逻辑）→ `models/`、`schemas/`、`db/`；`core/` 放配置、JWT、依赖注入（`get_db`、`get_current_user`、`require_doc_access`）。业务逻辑不要写进路由层。

### 实时协同鉴权链路

协同服务无状态、不含业务数据，鉴权全部回调主服务：

1. 前端调 `POST /documents/{docId}/collab-token`，API 用 `permission_service` 校验文档访问权，签发 **5 分钟**、`aud=collab` 的 Token 并返回 `{collabToken, wsUrl}`（普通 access token 不得连 WS）。
2. 前端带 Token 连 `/collab`；Hocuspocus 收到连接后回调 API 验证 Token。
3. Yjs 状态经 `/internal/collab/{docId}/state|snapshot`（`X-Internal-Token` 鉴权）由 api **单点落库**：协同会话中 Yjs 为唯一真源，回写时同步 `documents.content`/`content_text`/`version++`；房间非空时 REST 保存返回 `40902` 互斥。在线成员存 Redis（30s TTL 心跳）。

协同服务不持有业务状态、鉴权全部外置，故可水平扩容；新增 collab 节点时保持连接路由一致即可。

### API 全局约定

- Base URL `/api/v1`；**HTTP 状态码为权威**：成功 2xx 返回 `{"code":0,...}`，失败 4xx/5xx 返回 `{"code":<业务码>,"message","data":null}`。
- 统一响应体 `{"code": 0, "message": "ok", "data": {...}}`；分页用 `?page=1&page_size=20`，响应含 `total`。
- 错误码（定死）：`40001` 参数错、`40101` 未登录、`40102` token 失效、`40301` 无权限、`40401` 不存在、`40901` 版本冲突、`40902` 协同会话中禁止 REST 保存、`42901` 超额度、`42902` 触发限流、`41301` 内容超限。
- SSE 端点（`/ai/actions`、`/workspaces/{wsId}/ask`）用 `text/event-stream`，事件固定为 `delta`（增量文本）、`citations`（引用，仅问答，先于 delta）、`done`、`error`；Nginx 该路径 `proxy_buffering off`。
- 认证：`Authorization: Bearer <accessToken>`；access（15min）存内存、refresh（7d）存 httpOnly cookie；refresh rotation + 复用检测；改密走 `token_version` 全端下线（PyJWT + argon2-cffi）。

### 数据模型要点

- **文件夹与文档同表**：`documents.type` 区分 1=folder / 2=document，`parent_id` 自关联成树，`sort_order` 持久化同级拖拽排序；移动即改 `parent_id`，需递归 CTE 防环 + `pg_advisory_xact_lock` 防并发成环；删除为软删除（`deleted_at`），子树一并进回收站。
- **文档级权限**：`document_permissions(document_id, user_id, permission)`；判定顺序固定为文档显式授权 → 祖先目录继承 → workspace 角色；缓存失效用版本号 bump。
- **内容双写**：`content` 存 Tiptap JSON（JSONB），`content_text` 存 **jieba 分词后空格拼接**的文本，`search_vector` 为生成列（`simple` 配置 + GIN），检索用 `plainto_tsquery('simple', q)`。写入 `content` 时必须同步刷新 `content_text`，否则检索静默失效。
- **乐观锁**：`documents.version`，`PUT /documents/{docId}/content` 带 `baseVersion`，不一致返回 **409 + 最新内容**；不做自动合并，前端提供「采用服务端版本 / 保留本地副本」。
- **RAG 切片**：`doc_chunks.embedding` 为 `VECTOR(1024)`（HNSW/cosine），冗余 `workspace_id`，检索范围必须限定工作区；带 `embedding_model` 版本字段与 `char_start/char_end` 原文偏移（引用跳转定位依据）。
- **其余表**：`workspaces.invite_code` 邀请入组；`workspace_members.role` 1=owner 2=admin 3=editor 4=viewer；`document_versions.kind` 区分 auto（5min 窗口聚合，保留 50）与 manual（永不自动清理）；`document_shares.token_hash` 只存 sha256（明文仅返回一次）；`comments.anchor` 存 Yjs RelativePosition 序列化 + `quote` 原文兜底；`ai_usage_logs` 含 `workspace_id/status/latency_ms`；`audit_logs` 记录删除/改权限等敏感操作。

### 不可违反的约定

1. 所有文档访问必须经 `permission_service`（文档显式授权 → 目录继承 → workspace 角色；Redis 缓存带版本号失效，Redis 故障降级查库），**禁止**在其他模块自行查库判断权限。
2. 保存冲突一律 409 并回传最新内容，不做服务端静默覆盖；房间非空时 REST 保存返回 `40902`（Yjs 为唯一真源，两路径互斥）。
3. RAG 召回必须带 `workspace_id` 过滤，避免跨知识库串数据；`embedding` 换模型时按 `embedding_model` 字段分批重建。
4. 协同 Token 短时效（5min, `aud=collab`），collab 不落业务数据、不做权限判断、不直连 PostgreSQL（经 /internal）。
5. 保存文档后需异步刷新切片与摘要/标签（**ARQ 队列**，幂等 + 重试），不要让 AI 调用阻塞保存请求。
6. AI 调用统一走 `ai_service`，模型与供应商通过环境变量切换（OpenAI 兼容 SDK）；额度用 Redis `INCR` 原子预扣，计量写入 `ai_usage_logs`，超日额度返回 `42901`。
7. 写接口必须支持 `X-Request-Id` 幂等（5 分钟内重复请求返回首次结果快照）与滑动窗口限流；创建/删除/分享/改权限等敏感操作必须写 `audit_logs`（只增不改，BIGSERIAL + 异步落库）。
