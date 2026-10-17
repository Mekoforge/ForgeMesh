# ForgeMesh

> **面向 AI 算力基础设施的开源自动化控制平台** —— 把 GPU、推理运行时、模型、节点与计算资源组织成可声明、可编排、可治理、可观测的 AI 服务。

ForgeMesh 不是 GPU 自动部署脚本的集合，也不是"仅靠 LLM 执行命令"的自动化 Agent，而是由**声明式资源模型、协调控制器、持久化工作流、类型化决策、策略治理、可插拔执行能力**共同组成的基础设施平台。

- 上游架构基线：[`docs/architecture/architecture.md`](docs/architecture/architecture.md)
- 开发计划（WBS / 里程碑 / 接口契约）：[`docs/architecture/development-plan.md`](docs/architecture/development-plan.md)
- 路线图（1.1.0 之后的阶段与迭代）：[`docs/architecture/roadmap.md`](docs/architecture/roadmap.md)
- 架构决策记录（ADR）：[`docs/design-decisions/`](docs/design-decisions/)
- 变更日志：[`CHANGELOG.md`](CHANGELOG.md)
- v1.0 发布记录：[`docs/releases/v1.0.md`](docs/releases/v1.0.md)

---

## 1. 架构六层概览

ForgeMesh 采用六个逻辑层次。这些是**逻辑边界**，不要求一开始就部署为六组独立服务。

| 层次     | 名称                       | 组成                                                                               | 职责                   |
| ------ | ------------------------ | -------------------------------------------------------------------------------- | -------------------- |
| **L1** | Access Layer             | REST API · CLI · Web Console · SDK · Data Plane Gateway（Go）                     | 统一外部入口、身份认证与 API 兼容性；网关负责推理流量的路由与流式转发 |
| **L2** | Core Control Plane（微内核）  | Resource Registry · Reconciliation · Policy · Scheduler · Plugin Manager · Audit | 权威状态、资源治理、权限与一致性     |
| **L3** | Orchestration & Decision | WorkEngine API · Decision Router · JEV API · Workflow Providers                  | 可靠执行与结构化决策           |
| **L4** | Plugin Capability Layer  | GPU · Runtime · Model · MIG · Quantization · QoS · Metering                      | 可替换的领域能力实现           |
| **L5** | Execution Plane          | Node Agent · 进程与容器启动器 · Kubernetes 推理后端（Kthena / KServe）· GPU · Inference Services | 实际资源操作与状态反馈；Kubernetes 上的推理编排交给 Kthena / KServe（architecture.md 10.4） |
| **L6** | Platform Infrastructure  | SQLite · Outbox/Event Bus · Object Storage · Secret Manager · Observability      | 持久化、消息、制品、密钥与监控      |

### 四条不可违背的架构不变量

1. **核心掌握状态与治理权** —— 插件可以执行操作，但不能绕过核心直接改变其他模块的权威状态。
2. **目标状态与执行过程分离** —— 应当在什么状态由声明式资源与控制器决定；如何执行由 WorkEngine 负责。
3. **决策不等于授权** —— JEV 可以选择或推荐方案，但策略检查与资源可行性检查不能被跳过。
4. **关键状态必须能够恢复和追溯** —— 每次重要操作都必须具有稳定标识、版本与可查询的结果。

> 详细约束见 [`architecture.md` 第 2 章](docs/architecture/architecture.md)，开发落点见 [`development-plan.md` 第 1.2 节](docs/architecture/development-plan.md)。

---

## 2. 目录结构

```
forgemesh/
├── README.md                   # 本文件
├── LICENSE                     # Apache-2.0
├── CHANGELOG.md                # 变更日志（Keep a Changelog 格式）
├── CONTRIBUTING.md             # 贡献指南
├── CODE_OF_CONDUCT.md          # 行为准则
├── SECURITY.md                 # 安全策略与漏洞披露
├── GOVERNANCE.md               # 治理与决策机制
├── pyproject.toml              # 根级工具配置（ruff / pytest / coverage）
├── Makefile                    # install / test / lint / fmt / clean
├── requirements-dev.txt        # 开发依赖
│
├── api/                        # 对外契约（预览版，x-stability: beta）
│   ├── schemas/                # JSON Schema（4 个核心模型）
│   ├── openapi/                # OpenAPI 规范（1.1.0，x-stability: beta）
│   └── protobuf/               # Agent / gRPC 协议
│
├── packages/                   # Python 命名空间包（PEP 420，monorepo）
│   ├── core-contracts/         # 资源基类、Registry / Reconciler / Scheduler Protocol
│   ├── resource-model/         # InferenceService / Node / 分配 / 租户等强类型模型
│   ├── policy-engine/          # 授权、准入、配额、QoS
│   ├── decision-api/           # DecisionProvider / DecisionRouter / 熔断器 / 降级
│   ├── workflow-api/           # WorkflowRuntime 运行契约
│   ├── plugin-sdk/             # 插件 Manifest / 权限词表 / Ed25519 签名 / 入口加载 / SDK / 脚手架
│   ├── control-plane/          # 模块化单体控制平面（Registry / Reconciler / API / Outbox / Audit / Observability / Diagnostics）
│   ├── workflow-runtime/       # 工作流执行引擎（持久化 / DSL / 补偿 / 审批 / GPU onboarding）
│   ├── node-agent/             # 节点侧受控执行组件（注册 / 探测 / 运行时 / 健康 / 恢复）
│   └── cli/                    # 命令行工具
│
├── gateway/                    # Go 高性能网关（REST / 流式代理 / 健康检查）
├── plugins/                    # 可插拔能力实现（10 个插件）
│   ├── decision-rules/         # 确定性规则决策 Provider
│   ├── decision-jev/           # JEV 类型化决策 Provider
│   ├── decision-typesafe-jev/  # TypeSafe AI Jev 决策 Provider
│   ├── workflow-reference/     # 参考工作流执行器
│   ├── workflow-temporal/      # Temporal 工作流 Provider
│   ├── gpu-nvidia/             # NVIDIA GPU Provider（发现 / 预检 / 健康 / MIG / 异构）
│   ├── runtime-vllm/           # vLLM 推理 Runtime Provider
│   ├── runtime-tgi/            # TGI 推理 Runtime Provider
│   ├── qos-basic/              # QoS 优先级与限流插件
│   └── metering-basic/         # Token 计量插件
├── deploy/                     # docker（Dockerfile / compose）/ kubernetes
├── tests/                      # unit / contract / compat / e2e / chaos / security
├── examples/                   # single-node / multi-node / multi-tenant 示例
├── scripts/                    # CI 校验脚本（manifest/OpenAPI/元数据/插件冒烟）
│
└── docs/
    ├── architecture/           # 架构基线、开发计划、模块映射与路线图
    │   ├── architecture.md
    │   ├── development-plan.md
    │   ├── v1.1-development-plan.md
    │   ├── module-mapping.md
    │   └── roadmap.md
    ├── design-decisions/       # ADR 索引与 ADR-001~014
    ├── governance/             # 弃用政策
    ├── operations/             # 备份与灾备
    ├── plugin-development/     # 插件开发指南
    └── releases/               # 发布记录（v1.0.md）
```

> 仓库结构的权威定义见 [`architecture.md` 第 17.1 节](docs/architecture/architecture.md)。

---

## 3. 快速开始

### 环境要求

- Python **3.12+**（CI 覆盖 3.12）
- GNU Make
- Go **1.27+**（仅当需要开发/构建 `gateway/` 时）
- Docker 与 Docker Compose v2（仅容器部署时）
- Git

### 安装与测试

```bash
# 1. 克隆仓库
git clone git@github.com:Mekoforge/ForgeMesh.git
cd ForgeMesh

# 2. 安装全部子包（editable）+ 开发依赖
make install

# 3. 运行全部测试
make test
```

常用命令：

```bash
make test-unit       # 仅单元测试
make test-contract   # 仅契约测试
make coverage        # 全部测试 + 总覆盖率下限（与 CI 相同）
make lint            # ruff 静态检查与格式检查（与 CI 相同）
make fmt             # ruff 自动修复与格式化
make audit           # pip-audit 检查第三方依赖的已知漏洞（与 CI 相同，需要网络）
make clean           # 清理构建与缓存产物
```

> `make install` 会创建 `.venv`，遍历 `packages/*/`，对含 `pyproject.toml` 的子包执行 `pip install --no-deps -e`，因此新增子包无需修改顶层 Makefile。

### 运行控制平面

控制平面与网关默认要求 API key（[ADR-014](docs/design-decisions/adr-014.md)）：每个 key 绑定一个租户和一组 scope（`read` / `write` / `*`），租户只取自 key，不取自请求头。

```bash
# 1. 生成 key：第 2 行是 key（只显示这一次），最后一行是 key 文件条目
.venv/bin/python -m forgemesh.controlplane.keygen --tenant acme --subject ci-bot

# 2. 把条目放进 key 文件的 "keys" 数组（文件只含 SHA-256 摘要，网关与控制平面共用）
#    {"keys": [{"id": "acme-ci-bot", "sha256": "…", "tenantId": "acme", "subject": "ci-bot", "scopes": ["read", "write"]}]}

# 3. 启动并调用
FORGEMESH_API_KEYS_FILE=./api-keys.json FORGEMESH_DB_PATH=./forgemesh.db .venv/bin/forgemesh-control-plane
curl -s -H "Authorization: Bearer fmk_…" http://127.0.0.1:8080/api/v1/inference-services
```

| 环境变量 | 默认值 | 说明 |
| --- | --- | --- |
| `FORGEMESH_AUTH_MODE` | `apikey` | `apikey`：校验 `Authorization: Bearer <key>`；`dev`：直接信任 `X-Tenant-Id` 请求头，只允许监听回环地址，仅供本地开发 |
| `FORGEMESH_API_KEYS_FILE` | — | `apikey` 模式必填，缺失或格式非法时拒绝启动 |
| `FORGEMESH_DB_PATH` | `forgemesh.db` | SQLite 权威存储文件；同一文件只允许一个控制平面实例写入（ADR-013）。以 WAL 模式打开，运行时旁边有 `-wal` / `-shm` 文件，不能直接复制文件做备份，见 [备份与灾备演练](docs/operations/backup-restore.md)。设置了 `FORGEMESH_RUNTIME_LAUNCHER` 时，部署工作流的运行记录在同目录的第二个文件中（`forgemesh.db` → `forgemesh.workflows.db`），两个文件一起备份与恢复；`vllm-process` / `tgi-process` 还在同目录记录实例进程（`forgemesh.instances.json`，不需要备份，见「部署推理服务」） |
| `FORGEMESH_HOST` | `127.0.0.1` | 监听地址 |
| `FORGEMESH_PORT` | `8080` | 监听端口 |
| `FORGEMESH_LOG_FORMAT` | `json` | 日志格式：`json`（每行一个 JSON 对象）或 `text`，写到标准错误，见「可观测性」；不区分大小写，其他取值拒绝启动 |
| `FORGEMESH_LOG_LEVEL` | `INFO` | 日志级别：`DEBUG` / `INFO` / `WARNING` / `ERROR`，不区分大小写，其他取值拒绝启动；资源变更事件的日志不受它影响 |
| `FORGEMESH_INVENTORY_FILE` | — | 静态加速器清单（JSON，格式见下），设置后启用首次适配调度器；文件不可读或格式非法时拒绝启动，修改后需重启。未设置时不启用调度器 |
| `FORGEMESH_INSTANCE_CHECK_INTERVAL_SECONDS` | `10` | 部署推理服务时控制器核对实例是否仍在运行的间隔（秒）：实例退出之后，下一次核对把服务转为 `Degraded` 并重新部署（见「部署推理服务」）。须为有限的正数，不论是否设置了启动器都校验，非法时拒绝启动 |
| `FORGEMESH_RUNTIME_LAUNCHER` | — | 推理实例启动器，设置后部署推理服务（见「部署推理服务」）：`fake`（控制平面进程内的 OpenAI 兼容桩服务，不需要 GPU，供 CI 与本地开发）、`vllm-process`（本机上的 vLLM 子进程）或 `tgi-process`（本机上的 TGI 子进程），不区分大小写。必须同时设置 `FORGEMESH_INVENTORY_FILE`；取值未知、缺少清单或所选启动器的配置非法时拒绝启动。未设置时不部署。下面各启动器的变量只在选中该启动器时读取，空白值视同未设置 |
| `FORGEMESH_FAKE_LAUNCHER_BIND_HOST` | `127.0.0.1` | `fake`：实例监听的 IP 地址，`0.0.0.0` / `::` 监听全部地址 |
| `FORGEMESH_FAKE_LAUNCHER_ADVERTISE_HOST` | 同监听地址 | `fake`：写进 `status.endpoints` 的主机名或 IP 地址，网关用它连接实例；监听全部地址时必须设置 |
| `FORGEMESH_VLLM_COMMAND` | `<控制平面的解释器> -m vllm.entrypoints.openai.api_server` | `vllm-process`：启动 vLLM OpenAI 兼容服务的命令，按 shell 规则拆分（不经过 shell），第一个词须是可执行文件；使用默认命令时控制平面的解释器必须装有 `vllm` |
| `FORGEMESH_VLLM_BIND_HOST` | `127.0.0.1` | `vllm-process`：实例监听的 IP 地址，`0.0.0.0` / `::` 监听全部地址 |
| `FORGEMESH_VLLM_ADVERTISE_HOST` | 同监听地址 | `vllm-process`：写进 `status.endpoints` 的主机名或 IP 地址；监听全部地址时必须设置 |
| `FORGEMESH_VLLM_READY_TIMEOUT_SECONDS` | `1200` | `vllm-process`：进程启动后等 `GET /health` 返回 `200` 的时限（秒），包括下载与加载模型 |
| `FORGEMESH_VLLM_STOP_GRACE_SECONDS` | `10` | `vllm-process`：停止实例时 `SIGTERM` 之后等它退出的秒数，之后 `SIGKILL` |
| `FORGEMESH_TGI_COMMAND` | `text-generation-launcher` | `tgi-process`：启动 TGI 的命令，规则同 `FORGEMESH_VLLM_COMMAND`；使用默认命令时它必须能在 `PATH` 中找到 |
| `FORGEMESH_TGI_BIND_HOST` | `127.0.0.1` | `tgi-process`：实例监听的 IP 地址，`0.0.0.0` / `::` 监听全部地址 |
| `FORGEMESH_TGI_ADVERTISE_HOST` | 同监听地址 | `tgi-process`：写进 `status.endpoints` 的主机名或 IP 地址；监听全部地址时必须设置 |
| `FORGEMESH_TGI_READY_TIMEOUT_SECONDS` | `1200` | `tgi-process`：进程启动后等 `GET /health` 返回 `200` 的时限（秒），包括下载与加载模型 |
| `FORGEMESH_TGI_STOP_GRACE_SECONDS` | `10` | `tgi-process`：停止实例时 `SIGTERM` 之后等它退出的秒数，之后 `SIGKILL` |
| `FORGEMESH_EVENT_WEBHOOK_URL` | — | 资源变更事件的 webhook 地址（`http` / `https`，不得含用户信息与片段）：设置后把事件信封以 JSON `POST` 到这里，2xx 视为送达，不跟随重定向；未设置时事件写日志（见下文）。地址非法时拒绝启动；日志与错误信息只含地址的 scheme、主机与端口 |
| `FORGEMESH_EVENT_WEBHOOK_TOKEN` | — | 可选，以 `Authorization: Bearer <token>` 发给 webhook，只能由可见的 ASCII 字符组成，不写入日志；经明文 `http` 发往非回环地址时启动日志给出警告 |
| `FORGEMESH_EVENT_WEBHOOK_TIMEOUT_SECONDS` | `5` | 每次发送 webhook 的连接与读取超时（秒，大于 0、不超过 60）。这个变量与 token 只能与 `FORGEMESH_EVENT_WEBHOOK_URL` 一起设置，否则拒绝启动 |

> 请求可以不带 `X-Tenant-Id`；带了就必须与 key 绑定的租户一致，否则返回 `403`。缺少或无效的 key 返回 `401`。

加速器清单列出调度器可以分配的节点与设备，例如一个节点上的两张 80 GiB 显卡：

```json
{"nodes": [{"id": "node-a", "devices": [
  {"id": "gpu-0", "type": "nvidia-gpu", "memoryGiB": 80},
  {"id": "gpu-1", "type": "nvidia-gpu", "memoryGiB": 80}
]}]}
```

节点 `id` 在清单内唯一，设备 `id` 在所属节点内唯一，节点与设备都至少一个，不允许其他字段；`type` 与 `spec.resources.accelerator.type` 按字符串精确匹配，`memoryGiB` 是单卡显存（正整数，不接受 `"80"` 或 `80.0`）。调度器按清单顺序取第一个能凑齐 `count` 张卡的节点，在该节点内按顺序取类型匹配、显存不低于 `minMemoryGiB` 且未被占用的设备，不跨节点凑数。分配按资源 `uid` 记录在权威存储中（`allocations` / `allocation_devices`），同一设备同时至多属于一个生效的分配；分配不设租约，在资源删除时释放：持有分配的资源无论处于哪个 phase，`DELETE` 都返回 `202` 并先进入 `Terminating`。部署失败（确认实例已经停止之后）与加速器申请变化时同样释放分配；归属资源已不存在的分配由控制器回收，见「部署推理服务」。

控制平面启动后会自动协调 `InferenceService`：新建、修改或请求删除的资源在 0.5 秒内被处理，协调结果写入 `status.conditions`（例如 `Ready=False` 及其原因）。未设置 `FORGEMESH_INVENTORY_FILE` 时，新建的服务会停在 `Pending`，并带有 `Ready=False`（`SchedulerUnavailable`）条件；只设置清单、未设置 `FORGEMESH_RUNTIME_LAUNCHER` 时，条件为 `Ready=False`（`WorkflowRuntimeUnavailable`），协调器在分配之前停下，不占用设备。两者都设置后，资源每轮推进一个 phase（`Pending` → `Scheduling` → `Preparing` → `Deploying` → `Verifying` → `Ready`）：分配设备，运行部署工作流（准备模型 → 启动实例 → 验证实例），成功后写入 `status.endpoints` 与 `Ready=True`，网关随即可以转发对话；`spec` 变化时经 `Updating` 重新部署（见下文「更新」）；实例丢失时服务转为 `Degraded` 并重新部署；部署失败时资源进入 `Failed`，不自动重试，修改 `spec` 或删除后重新创建即可再次部署；删除时先取消部署、停止实例，再释放分配。每个服务只部署一个实例（`spec.availability.replicas` 不生效）。控制器与 API 运行在同一个进程中，是权威存储的唯一写者（ADR-013）：请以单进程运行，不要使用 `uvicorn --workers` 或多副本。设置了启动器时 `POST /api/v1/workflows` 同样返回 `503`（`WORKFLOW_RUNTIME_UNAVAILABLE`）：部署工作流的运行时只供协调器使用，不对外开放。

每次资源写入（经 API、控制器或启动时的实例核对）都在同一个事务中记下一条资源变更事件（信封见 `api/schemas/event.schema.json`，如 `inference.service.created`、`inference.service.status_updated`），控制平面在后台发布：默认写日志（logger `forgemesh.events`，每个事件一行 JSON；以 `forgemesh-control-plane` 运行时与其他日志一起写到标准错误，见「可观测性」，没有配置 logging 时写标准输出），设置 `FORGEMESH_EVENT_WEBHOOK_URL` 后 `POST` 到 webhook。事件至少投递一次，接收方按 `eventId`（webhook 请求头 `X-Forgemesh-Event-Id`）去重；发送失败时按 1 秒起翻倍、最长 300 秒的间隔重试，失败 10 次后不再发送（`dead`，记一条 ERROR 日志），修复接收方后可以重新投递，见[备份与灾备演练](docs/operations/backup-restore.md)第 5 节。经 API 创建、更新（`PUT` / `PATCH`）与删除资源时，同一事务还写入一条审计记录（租户、主体、key id、correlation id），可用 `GET /api/v1/audit-events` 查询（见下文 REST API）。

### REST API

控制平面的 REST API 以 `/api/v1` 为前缀，契约是 [api/openapi/openapi.yaml](api/openapi/openapi.yaml)。契约测试 `tests/contract/test_openapi_contract.py` 双向核对应用提供的路由、参数、请求体与状态码（探针除外），并逐一调用每个操作、校验响应符合契约：修改路由时须同步修改契约。网关把 `/api/v1/inference-services`、`/api/v1/resources`、`/api/v1/audit-events`、`/api/v1/workflows`、`/api/v1/decisions` 及其子路径的请求（任意方法，连同查询串、请求头与请求体）转发给控制平面。错误响应都是错误信封 `{"error": {"code", "message", "correlationId", "traceId", "details"}}`，`correlationId` 回显请求头 `X-Correlation-Id`（经网关访问时由网关补上），`traceId` 是这个请求的 trace-id（见「可观测性」）。资源不存在的 `404` 错误码为 `NOT_FOUND`；路径不匹配任何端点（如多了一段前缀）的 `404` 错误码为 `ROUTE_NOT_FOUND`，与网关相同，客户端据此区分资源不存在与地址错误。

| 端点 | 说明 |
| --- | --- |
| `POST /api/v1/inference-services` | 创建（`201`）：`uid`、租户与 `resourceVersion` 由服务端设置，`generation` 为 `1`、`observedGeneration` 为 `0`，`status` 被忽略；未知的顶层字段返回 `400`，同名资源已存在返回 `409` |
| `GET /api/v1/inference-services` | 列出推理服务，可按 `namespace` 过滤，分页见下 |
| `GET /api/v1/inference-services/{name}` | 读取一个推理服务，响应带 `ETag` |
| `PUT /api/v1/inference-services/{name}` | 整体替换 `spec`、labels 与 annotations，规则见下 |
| `PATCH /api/v1/inference-services/{name}` | JSON Merge Patch（RFC 7386，`Content-Type: application/merge-patch+json`），规则同 `PUT` |
| `DELETE /api/v1/inference-services/{name}` | 受控删除：直接删除返回 `204`；持有 finalizer 或加速器分配时返回 `202`，资源进入 `Terminating`，由协调器释放后删除 |
| `GET /api/v1/resources?kind=…` | 按 kind 列出资源，分页同上；省略 `kind` 时返回空列表 |
| `GET /api/v1/audit-events` | 列出本租户的审计记录，最新的在前，可按 `correlationId` 过滤；`limit`、`count` 与 `pageToken` 同下文的分页，游标不能与资源列表的游标互换 |
| `POST /api/v1/workflows`、`POST /api/v1/decisions` | 控制平面应用不注入工作流运行时与决策路由器，合法的请求返回 `503`（`WORKFLOW_RUNTIME_UNAVAILABLE` / `DECISION_ROUTER_UNAVAILABLE`）；决策的风险等级由服务端按 `question` 推导，请求中的 `riskLevel` 只能调高 |

单个资源的端点以 `namespace` 查询参数指定命名空间，省略时指不带命名空间的资源。读接口要求 `read` scope，写接口（含 `PUT` / `PATCH`、启动工作流与请求决策）要求 `write` scope。

**分页**：列表按 `(namespace, name)` 升序返回（不带命名空间的资源在最前），每页最多 `limit` 条（`1`–`1000`，默认 `100`），`count` 是本页的条数。还有更多结果时响应带 `nextPageToken`：以相同的过滤条件把它作为 `pageToken` 传回即得下一页，`limit` 可以变化。游标不透明，只对签发它的租户与同一组过滤条件有效，格式不对、被修改或用于其他租户 / 过滤条件时返回 `400`；控制平面重启后仍然有效。

**更新**：`PUT` 的请求体是完整的资源文档（`GET` 的响应修改后可以直接提交），`spec`、`metadata.labels` 与 `metadata.annotations` 整体替换，省略 labels / annotations 即清空；`PATCH` 的补丁合并到当前资源之后按同样的规则处理（`null` 删除字段，对象逐字段合并，数组整体替换）。`kind`、`apiVersion`、`metadata.name`、`metadata.namespace` 与租户不可修改（`400`），`status`、`conditions`、`finalizers`、`generation`、`observedGeneration` 被忽略。只有 `spec` 变化时 `generation` 递增，没有任何变化的更新不写入。乐观并发：把读到的 `metadata.resourceVersion`（及 `metadata.uid`）留在请求体中，或在 `If-Match` 中携带响应的 `ETag`；资源在此期间被修改过（包括控制器写入 `status`）就返回 `409`（`details.reason` 为 `ResourceVersionMismatch`），重新读取后再试。同名资源删除后重建时 `resourceVersion` 从 `0` 重新计数，`uid` 不符（`UidMismatch`）或 `ETag` 不符可以识别这种情况；资源正在删除时返回 `409`（`Terminating`）。

```bash
KEY="Authorization: Bearer fmk_…"
URL="http://127.0.0.1:8081/api/v1/inference-services/llama?namespace=default"
# 带上读到的版本修改模型引用；期间资源被修改过时返回 409
RV=$(curl -s -H "$KEY" "$URL" | jq -r .metadata.resourceVersion)
curl -s -X PATCH -H "$KEY" -H "Content-Type: application/merge-patch+json" \
  -d "{\"metadata\": {\"resourceVersion\": \"$RV\"}, \"spec\": {\"model\": {\"reference\": \"models/llama-3\"}}}" "$URL"
# 翻页：把响应中的 nextPageToken 作为 pageToken 传回，直到响应不再带它
curl -s -H "$KEY" "http://127.0.0.1:8081/api/v1/inference-services?limit=50&pageToken=<nextPageToken>"
```

`spec` 变化（`generation` 递增）之后，控制器按新的 `spec` 重新部署：`Ready` / `Degraded` 的服务经策略放行后进入 `Updating`（`Ready=False`，endpoint 标记为 `ready: false`），然后停止旧实例，取得分配（仍满足新的加速器申请时复用，否则先释放再重新分配；新的申请放不下时停在 `Updating`，`AllocationFailed`，10 秒后重试），以新的 `spec` 运行部署工作流，新实例通过验证后回到 `Ready`；更新失败时进入 `Failed`。先停后起：从进入 `Updating` 到新实例通过验证，服务不可用，网关返回 `503`（`NO_HEALTHY_UPSTREAM`；缓存的几秒内仍连接旧实例，旧实例停止之后为 `502`）。还在部署时提交的变化取消进行中的部署，改为部署最新的 `spec`（phase 不变，不重新评估策略）；`Failed` 的服务在 `spec` 变化之后回到 `Pending`，与新建的服务一样重新部署。被策略拒绝、等待审批或缺少调度器 / 启动器的更新不写 `status`（只记日志），服务保持原状，`generation` 大于 `observedGeneration`。开始更新的一轮就写入新的 `observedGeneration`：`status.phase` 为 `Ready` 且 `observedGeneration` 等于 `generation` 时，服务运行的才是最新的 `spec`。

### Go 网关

```bash
cd gateway
go test ./...       # 运行网关测试
go build ./cmd/gateway  # 构建网关二进制
GATEWAY_API_KEYS_FILE=../api-keys.json make run   # 默认 apikey 模式，监听 :8081
make run-dev        # dev 模式：信任租户请求头，只监听 127.0.0.1:8081
```

> 网关默认不连接回环、链路本地与云元数据地址上的推理实例。实例运行在本机时设置 `GATEWAY_INSTANCE_ALLOWED_CIDRS=127.0.0.0/8,::1/128`（`make run-dev` 已设置），见 [gateway/README.md](gateway/README.md)「实例地址限制」。

### 命令行工具

`forgemesh`（`packages/cli`，`make install` 之后为 `.venv/bin/forgemesh`）经 REST API 管理推理服务，只依赖标准库与 PyYAML。默认经网关访问 `http://127.0.0.1:8081`；直连控制平面时以 `--api-url http://127.0.0.1:8080` 或环境变量 `FORGEMESH_API_URL` 指定地址。地址是网关或控制平面的基地址，不含 `/api/v1`（可以带反向代理的路径前缀），不能带凭证、查询串或片段；CLI 不跟随重定向。

API key 取自 `--api-key-file` 指向的文件（只含一行 key，即 keygen 输出的第 2 行，权限设为 `0600`），其次取自环境变量 `FORGEMESH_API_KEY`（例如由 CI 注入的密钥）。CLI 没有接受 key 本身的选项：命令行参数对本机的其他用户可见，也会留在 shell 历史中。key 只放在 `Authorization` 请求头中，任何输出都不包含它。没有 key 时不发送 `Authorization`，`dev` 模式的控制平面需要 `--tenant`（作为 `X-Tenant-Id` 发送）；带 key 时可以省略 `--tenant`，给出就必须是 key 绑定的租户，否则返回 `403`。

```bash
KEY_FILE=~/.config/forgemesh/acme.key   # 只含 keygen 输出的第 2 行（key），权限 0600
.venv/bin/forgemesh --api-key-file "$KEY_FILE" apply -f - <<'EOF'
kind: InferenceService
metadata: {name: llama, namespace: default}
spec:
  model: {reference: models/llama}
  runtime: {provider: vllm}
  resources: {accelerator: {type: nvidia-gpu, minMemoryGiB: 16}}
EOF
.venv/bin/forgemesh --api-key-file "$KEY_FILE" wait inferenceservice llama -n default --timeout 120
.venv/bin/forgemesh --api-key-file "$KEY_FILE" list inferenceservices
.venv/bin/forgemesh --api-key-file "$KEY_FILE" delete inferenceservice llama -n default
.venv/bin/forgemesh --api-key-file "$KEY_FILE" wait inferenceservice llama -n default --for delete
```

| 命令 | 说明 |
| --- | --- |
| `apply -f FILE` | 资源不存在就创建，存在就整体替换，见下；`-f` 可以重复，`-` 读取标准输入 |
| `get inferenceservice NAME [-o yaml\|json]` | 输出资源，默认 YAML |
| `describe inferenceservice NAME` | 便于阅读的详情：元数据、`spec` 要点、phase、条件与实例地址 |
| `list inferenceservices [-o table\|yaml\|json] [--page-size N]` | 跟随 `nextPageToken` 读取全部页（每页 `N` 条，`1`–`1000`，默认 `100`）；表格列出命名空间、名称、phase、`Ready` 条件及其原因与模型，`-o yaml` 输出以 `---` 分隔的多个文档，`-o json` 输出数组 |
| `delete inferenceservice NAME` | 受控删除：`204` 时报告 `deleted`；`202` 时报告资源进入 `Terminating`，控制器停止实例、释放加速器之后删除它 |
| `wait inferenceservice NAME [--for ready\|delete] [--timeout SECONDS]` | 等服务就绪（默认）或被删除，默认最多等 300 秒，见下 |

CLI 只管理 `InferenceService`（写作 `inferenceservice` 或 `inferenceservices`，不区分大小写）。单个资源的命令以 `-n` / `--namespace` 指定命名空间，省略时指不带命名空间的资源（与 API 相同）；`list` 省略时列出全部命名空间。公共选项（`--api-url`、`--api-key-file`、`--tenant`、`-n`）可以写在子命令之前或之后。

**apply**：对每个文档先 `GET`，资源不存在就 `POST`；存在就 `PUT` 文档，以读到的 `metadata.resourceVersion` 与 `metadata.uid` 作为前置条件（取代文档中的值）。读取之后资源被并发修改、创建或删除时重新读取再试，最多 5 次。每个文档输出一行结果，如 `InferenceService default/llama created`，结果是 `created`、`configured` 或 `unchanged`：没有任何变化的更新不写入，同一个文件再次 `apply` 得到 `unchanged`。`PUT` 整体替换 `spec`、labels 与 annotations（文档中省略的 labels / annotations 被清空）并忽略 `status`，所以 `get` 与 `list -o yaml|json` 的输出修改后可以直接 `apply`。资源正在删除时报告冲突（`409`，原因 `Terminating`），删除完成后再 `apply`。

文件是 YAML（可以有多个以 `---` 分隔的文档）或 JSON（对象或对象的数组）；省略 `kind` 时视为 `InferenceService`；文档没有 `metadata.namespace` 时使用 `-n`，两者都给出时必须相同。全部文档在发出任何请求之前读取并检查，有一个不合法就不发出请求；某个资源失败时报告错误、继续处理其余的资源，以第一个失败的退出码结束。文档中的 `metadata.tenantId` 作为 `X-Tenant-Id` 发送（与 `--tenant` 不同时报错）：与 key 绑定的租户不符时 API 返回 `403`，资源不会被写进 key 的租户。

**wait**：每秒读取一次资源。`--for ready` 在同一次读取中 `status.phase` 为 `Ready`、`Ready` 条件为 `True` 且 `metadata.observedGeneration` 等于 `generation` 时成功，这时服务运行的是当前的 `spec`（见「REST API」）：`spec` 变化之后，服务先以旧的 `observedGeneration` 保持 `Ready`，再经 `Updating` 重新部署，`wait` 等到新的 `spec` 部署完成。资源正在删除，或在当前的 `spec` 下进入 `Failed`（`spec` 变化之前控制器不会再部署它）时立即失败；其他状态都继续等待，如 `Updating`、重新部署中的 `Degraded`，以及 `spec` 变化之后还没重新部署的 `Failed`。超时时报告最后看到的状态：被策略拒绝、等待审批或缺少调度器 / 启动器的更新不写 `status`，服务一直以旧的 `observedGeneration` 停在 `Ready` / `Degraded`，消息说明更新没有开始、原因见控制平面的日志。`--for delete` 在资源不存在时成功（退出码 `0`），输出如 `InferenceService default/llama deleted`：开始等待时资源已经不存在（例如 `delete` 之后控制器在第一次读取之前就删除了它）同样如此，结果不取决于这个先后；因此名称或命名空间写错时也报告成功。读取遇到限流（`429`，按 `Retry-After` 推迟下一次读取）或 `502`–`504` 时继续等待；连接失败只在此前收到过 HTTP 响应时视为暂时故障，第一次请求就连不上时立即失败。

**错误与退出码**：错误消息写到标准错误，以 `Error:` 开头（参数写错时是 argparse 的用法提示）。错误信封显示为 `<message> (HTTP <状态码> <code>[, reason <details.reason>], correlation id <关联 ID>)`，请求校验失败时逐行列出字段与原因，带 `Retry-After` 时（如网关限流的 `429`）附上秒数。不是错误信封的响应报告状态码：纯文本的 `429` 与 `502`（如网关前面的反向代理返回的）报告为被限流与连不上上游，其他附上正文的第一行。一次调用的全部请求带同一个随机的 `X-Correlation-Id`，可以用它在网关与控制平面的日志中找到这次调用。`forgemesh --help` 列出同一张退出码表：

| 退出码 | 含义 |
| --- | --- |
| `0` | 成功 |
| `1` | 其他错误，如地址不提供 ForgeMesh API（错误码不是 `NOT_FOUND` 的 `404`，如网关与控制平面对没有匹配路由的路径返回的 `ROUTE_NOT_FOUND`；重定向）或响应无法解析 |
| `2` | 参数、配置或资源文档不合法，没有发出请求 |
| `3` | 资源不存在（`404 NOT_FOUND`）；`wait --for delete` 此时以 `0` 成功 |
| `4` | 冲突（`409`），如资源正在删除，或 `apply` 的 5 次尝试都遇到并发修改 |
| `5` | 请求被拒绝（`400`、`405`、`413`、`415` 等其他 4xx） |
| `6` | 认证或授权失败（`401`、`403`） |
| `7` | 被限流（`429`） |
| `8` | 服务端错误（`500` 等其他 5xx） |
| `9` | API 暂时不可用（`502`、`503`、`504`） |
| `10` | 没有收到 HTTP 响应：连接被拒绝、主机名无法解析、TLS 失败或超时 |
| `11` | `wait` 超时 |
| `12` | `wait` 等待的状态不会出现：服务正在删除，或在当前的 `spec` 下进入 `Failed` |
| `130` | 被 Ctrl-C 中断 |

| 环境变量 | 默认值 | 说明 |
| --- | --- | --- |
| `FORGEMESH_API_URL` | `http://127.0.0.1:8081` | API 的基地址，默认是网关（控制平面默认监听 `http://127.0.0.1:8080`）；`--api-url` 优先 |
| `FORGEMESH_API_KEY` | — | API key，`--api-key-file` 优先；空白值视同未设置 |

### 部署推理服务

设置 `FORGEMESH_INVENTORY_FILE` 与 `FORGEMESH_RUNTIME_LAUNCHER` 后，控制平面为每个 `InferenceService` 启动一个推理实例，网关经 `/api/v1/namespaces/{ns}/inference/{svc}/chat` 把对话转发给它。启动器有三种：

- `fake`：每个实例是控制平面进程内的 OpenAI 兼容桩服务，监听一个临时端口：`GET /health`；`POST /v1/chat/completions` 返回固定的回复，`"stream": true` 时以 SSE 返回，以 `data: [DONE]` 结束；`POST /fake/exit` 返回 `202`，实例随即像进程退出一样结束，用来演练实例丢失（网关只转发推理协议的路径，经网关调用不到它）。不需要 GPU，也不加载模型，只用于 CI 与本地开发。
- `vllm-process`：每个实例是控制平面所在主机上的一个 vLLM 进程，见下文「vLLM」。
- `tgi-process`：每个实例是控制平面所在主机上的一个 TGI（text-generation-inference）进程，见下文「TGI」。

不需要 GPU 的完整闭环，控制平面与网关都以 `dev` 模式运行在本机：

```bash
# 终端 1：控制平面，使用一块虚构设备的清单与假启动器
FORGEMESH_AUTH_MODE=dev FORGEMESH_DB_PATH=./forgemesh.db \
  FORGEMESH_INVENTORY_FILE=deploy/docker/fake-runtime-inventory.json \
  FORGEMESH_RUNTIME_LAUNCHER=fake .venv/bin/forgemesh-control-plane

# 终端 2：网关（make run-dev 已允许连接本机上的实例）
cd gateway && make run-dev

# 终端 3：创建服务，等 status.phase 变为 Ready（几秒），经网关流式对话，最后删除
curl -s -H "X-Tenant-Id: acme" -H "Content-Type: application/json" \
  -d '{"kind": "InferenceService", "metadata": {"name": "llama", "namespace": "default"},
       "spec": {"model": {"reference": "models/llama"}, "runtime": {"provider": "vllm"},
                "resources": {"accelerator": {"type": "nvidia-gpu", "minMemoryGiB": 16}}}}' \
  http://127.0.0.1:8081/api/v1/inference-services
curl -s -H "X-Tenant-Id: acme" "http://127.0.0.1:8081/api/v1/inference-services/llama?namespace=default" | jq .status
curl -sN -H "X-Tenant-Id: acme" -H "Content-Type: application/json" \
  -d '{"model": "models/llama", "messages": [{"role": "user", "content": "Hello"}], "stream": true}' \
  http://127.0.0.1:8081/api/v1/namespaces/default/inference/llama/chat
curl -s -X DELETE -H "X-Tenant-Id: acme" "http://127.0.0.1:8081/api/v1/inference-services/llama?namespace=default"
```

`Ready` 的服务带有 `Ready=True`（`Reconciled`），`status.endpoints` 形如 `[{"url": "http://127.0.0.1:<端口>", "ready": true, "instanceId": "fake-…", "protocol": "openai-chat"}]`；回复拼起来是 `Hello from the ForgeMesh fake instance fake-….`。删除返回 `202`，实例停止、分配释放之后资源消失（`404`）。网关缓存服务的实例列表 5 秒（`GATEWAY_RESOLVER_TTL_SECONDS`），删除、重新创建或更新之后的几秒内可能仍连接旧实例（`502`）。在容器中运行同样的闭环见「容器部署」。

**实例丢失**：控制器每 `FORGEMESH_INSTANCE_CHECK_INTERVAL_SECONDS` 秒（默认 `10`）向启动器核对 `Ready` / `Degraded` 服务的实例是否仍在运行，只看进程或桩服务是否还在，不探测它能否回答请求。实例已不在运行的 `Ready` 服务转为 `Degraded`：`Ready=False`（`InstanceLost`），`status.endpoints` 中的实例标记为 `ready: false`，网关不再转发（缓存的几秒内除外）。随后控制器以同一个 `spec` 重新部署（不重新评估策略）：停止之前的部署，复用服务持有的分配，清空 `status.endpoints`，新实例通过验证后回到 `Ready`。同一代 `spec` 的第一次重新部署立即开始，之后与上一次开始至少相隔 10 秒，每次翻倍，最多 300 秒，上一次开始已过 600 秒时从头开始；重新部署的运行失败时服务保持 `Degraded`，等待期间 `Ready=False`（`WorkflowFailed`）给出原因。退避只记在内存中，控制平面重启之后从头开始。

**控制平面重启**：三种启动器的实例都活不过控制平面的重启（`fake` 的实例在进程内；进程启动器的实例是子进程，控制平面退出时停止它们，控制平面被杀死时留下的实例在下次启动时结束，见下文「vLLM」），重启之后：

- 重启之前 `Ready` 的服务在控制器启动之前转为 `Degraded`（`InstanceLost`，消息说明实例没能活过控制平面重启），随后像实例丢失一样重新部署，以新实例回到 `Ready`。
- 仍在部署的服务从中断处继续（部署工作流的运行记录在 `forgemesh.workflows.db` 中，启动时继续未结束的运行）：还没有启动实例的照常完成；实例已经启动、还在验证的，运行在验证实例时失败，服务进入 `Failed`（`WorkflowFailed`）；运行已经成功、还没写入 `Ready` 的重新部署。

**回收孤立的分配**：经 `DELETE` 删除的服务在消失之前释放分配。归属资源（同租户、同 `uid`）已不存在的分配只可能来自绕过受控删除的操作或旧版本的数据，它们的设备再也分不出去：控制器在启动时与每次全量同步（每 10 分钟）之后先停止这些分配对应的部署，再把分配标记为 `Expired`，设备随之空闲；停止失败时保留分配，下次再试。资源仍存在的分配不在回收之列，不论它处于哪个 phase。

**vLLM**：`vllm-process` 以 `<FORGEMESH_VLLM_COMMAND> --model=<spec.model.reference> --host <监听地址> --port <空闲端口> --tensor-parallel-size <设备数>` 启动实例，`GET /health` 返回 `200` 后服务进入 `Ready`。它只部署 `spec.runtime.provider` 为 `vllm` 的服务，不检查 `spec.runtime.version` 与 `spec.model.digest`：运行的是命令对应的 vLLM 版本，模型由 vLLM 启动时按引用下载、加载（计入就绪时限）。

- 实例只运行在控制平面所在的主机上：清单必须恰好有一个节点（就是这台主机），设备 id 是 CUDA 设备序号（`0`、`gpu-0`）或 `nvidia-smi -L` 列出的 UUID（`GPU-…`、`MIG-…`），且没有两个 id 指向同一块设备，否则拒绝启动。分配到的设备写进实例的 `CUDA_VISIBLE_DEVICES`；未设置 `CUDA_DEVICE_ORDER` 时设为 `PCI_BUS_ID`，序号与 `nvidia-smi` 一致。
- 默认命令要求控制平面的解释器装有 vLLM（例如 `.venv/bin/pip install vllm`）；也可以用 `FORGEMESH_VLLM_COMMAND` 指向另一个环境中的 vLLM，如 `/opt/vllm/bin/python -m vllm.entrypoints.openai.api_server`。仓库的控制平面镜像不含 vLLM。
- 实例继承控制平面的环境变量（去掉 `FORGEMESH_*`，例如 `HF_TOKEN` 会传给 vLLM），输出写到控制平面的标准输出与标准错误。
- 实例默认监听 `127.0.0.1`：网关在同一台主机上时为它设置 `GATEWAY_INSTANCE_ALLOWED_CIDRS=127.0.0.0/8`；网关在其他主机上时设置 `FORGEMESH_VLLM_BIND_HOST=0.0.0.0` 与网关能够连接的 `FORGEMESH_VLLM_ADVERTISE_HOST`。
- 每个实例在自己的会话中运行：停止时实例进程的进程组收到 `SIGTERM`，实例进程退出或 `FORGEMESH_VLLM_STOP_GRACE_SECONDS` 秒后，会话中的每个进程组收到 `SIGKILL`；在 Linux 上停止要等到会话中的进程都已退出（至多 10 秒，超过即停止失败）。控制平面退出时并行停止全部实例，连同控制器与工作流的收尾，默认设置下最长约 40 秒，在容器或服务管理器中相应调大停止时限。
- 控制平面直接在主机上运行、被 `SIGKILL`（或崩溃、被 OOM 结束）时，实例进程在自己的会话中，不会随之结束、继续占用 GPU。启动器因此把每个实例进程记在数据库旁的 `forgemesh.instances.json` 中（实例停止之后删去），下次启动时在恢复部署之前像停止实例一样结束其中仍在运行的进程。只向 pid 与启动时间都与记录一致的进程及其会话发送信号，另一次开机或另一个 PID 命名空间的记录被忽略。限制：在下次启动之前实例一直占用 GPU；调用 `setsid` 离开了会话的进程、实例进程退出之后会话中剩下的进程找不到；数据库为 `:memory:` 时不记录。以容器或 systemd 服务（默认 `KillMode=control-group`）运行控制平面时，容器或 systemd 还会结束控制平面留下的全部进程。

```bash
# 一台有两块 GPU 的主机：节点 id 任意，设备 id 为 CUDA 设备序号
cat > inventory.json <<'EOF'
{"nodes": [{"id": "gpu-host", "devices": [
  {"id": "gpu-0", "type": "nvidia-gpu", "memoryGiB": 80},
  {"id": "gpu-1", "type": "nvidia-gpu", "memoryGiB": 80}
]}]}
EOF
FORGEMESH_API_KEYS_FILE=./api-keys.json FORGEMESH_DB_PATH=./forgemesh.db \
  FORGEMESH_INVENTORY_FILE=./inventory.json FORGEMESH_RUNTIME_LAUNCHER=vllm-process \
  .venv/bin/forgemesh-control-plane
# 网关：cd gateway && GATEWAY_API_KEYS_FILE=../api-keys.json GATEWAY_INSTANCE_ALLOWED_CIDRS=127.0.0.0/8 make run
# 服务：spec.runtime.provider 为 vllm，spec.model.reference 为 Hugging Face 模型 id 或本地路径，
#       spec.resources.accelerator.count 即 --tensor-parallel-size
```

**TGI**：`tgi-process` 以 `<FORGEMESH_TGI_COMMAND> --model-id <spec.model.reference> --hostname <监听地址> --port <空闲端口> --num-shard <设备数> --master-port <另一个空闲端口> --shard-uds-path <实例私有目录>/shard` 启动实例，只部署 `spec.runtime.provider` 为 `tgi` 的服务。实例的协议为 `tgi-generate`：网关把对话请求转发到实例的 `/generate_stream`，不转换请求体，客户端按 TGI 的格式发送请求。`--master-port` 与私有目录（权限 `0700`，实例停止之后删除）避免同一主机上的实例争用 TGI 默认的端口 `29500` 与 shard 套接字。默认命令 `text-generation-launcher` 须能在 `PATH` 中找到，仓库的镜像不含 TGI。设备 id、`CUDA_VISIBLE_DEVICES`、环境变量、监听地址、`GET /health` 就绪检查、停止（TGI 的 router 与各个 shard 各在自己的进程组中，一并结束）与实例记录同 vLLM，变量为 `FORGEMESH_TGI_*`（见「运行控制平面」）。

### 容器部署

```bash
# 1. 生成 key（同上），把 key 文件写到 deploy/docker/api-keys.json（已在 .gitignore 中），
#    权限设为 0644：两个容器都以非 root 用户读取它
# 2. 构建镜像并启动控制平面与网关（都只发布在 127.0.0.1）
docker compose -f deploy/docker/docker-compose.yaml up --build
curl -s -H "Authorization: Bearer fmk_…" http://127.0.0.1:8081/api/v1/inference-services

# 不需要 GPU 的完整闭环：叠加一块虚构设备的清单与假启动器
docker compose -f deploy/docker/docker-compose.yaml \
  -f deploy/docker/docker-compose.fake-runtime.yaml up --build

scripts/compose_smoke.sh   # 端到端冒烟（CI 同样执行）
```

只用 `docker-compose.yaml` 时控制平面没有清单与启动器，新建的服务停在 `Pending`（`SchedulerUnavailable`）。`docker-compose.fake-runtime.yaml` 为控制平面加上清单 `deploy/docker/fake-runtime-inventory.json`（一个节点、一块设备 `gpu-0`）与 `fake` 启动器：实例在控制平面容器内监听全部地址，写进 `status.endpoints` 的主机是服务名 `control-plane`，网关经 compose 网络连接它（私网地址默认允许），实例端口不发布到宿主机。`scripts/compose_smoke.sh` 用这两个文件经网关走完闭环：无 key 的请求返回 `401`；创建 → `Ready`（一个就绪的 `openai-chat` 端点）→ 流式对话到达实例、以 `[DONE]` 结束 → 不带凭证抓取两边的 `/metrics`，带 `traceparent` 的请求在两边的日志与错误信封中带同一个 trace-id（见「可观测性」）→ 删除后资源消失、实例停止监听 → 同名重新创建（只有一块设备，可见分配已释放）→ 再次对话到达新实例 → 以 `PUT` 修改 `spec` → `Updating` → 以新实例回到 `Ready`（`observedGeneration` 为 `2`，旧实例停止监听）并再次对话 → 让实例退出（在控制平面容器内 `POST /fake/exit`）→ `Degraded`（`InstanceLost`）→ 重新部署，以新实例回到 `Ready` 并再次对话，控制平面的 `/metrics` 计入定期核对发现的一次丢失与一次重新部署 → 重启控制平面：资源仍在，服务转为 `Degraded`（`InstanceLost`）后重新部署，以新实例回到 `Ready` 并再次对话，`/metrics`（随新进程从 0 计数）计入启动时核对发现的一次丢失与一次重新部署 → 删除。叠加文件把核对实例的间隔设为 1 秒；每一步都有时限，只等待状态变化，不依赖网关在某一刻返回的状态码。

控制平面的 SQLite 数据库（部署推理服务时还有工作流存储 `forgemesh.workflows.db`）在 compose 卷 `control-plane-data` 中。容器中的备份与恢复见 [备份与灾备演练](docs/operations/backup-restore.md) 2.5 / 3.4 节。

### Kubernetes 部署

`deploy/kubernetes/` 中的清单把控制平面与网关部署在命名空间 `forgemesh` 中。控制平面的数据库放在 PersistentVolumeClaim 上（应是块存储），副本数固定为 1、更新策略为 `Recreate`，保证任何时刻只有一个控制平面写数据库（ADR-013）；网关无状态，可以增加副本，限流按副本各自计算。两者共用 Secret `forgemesh-api-keys` 中的 key 文件，各以专用的 ServiceAccount 运行、不挂载 token（都不访问 Kubernetes API），以非 root 用户、只读根文件系统、去掉全部 capability 运行，满足 Pod Security 的 `restricted` 级别。

```bash
# 1. 构建镜像并推送到集群能拉取的镜像仓库，相应修改两个 Deployment 的 image
#    （kind 可以直接 kind load docker-image forgemesh/control-plane:0.0.1 forgemesh/gateway:0.0.1）
docker build -f deploy/docker/Dockerfile.control-plane -t forgemesh/control-plane:0.0.1 .
docker build -f deploy/docker/Dockerfile.gateway -t forgemesh/gateway:0.0.1 .
# 2. 命名空间与 key 文件（生成方法见「运行控制平面」）
kubectl apply -f deploy/kubernetes/namespace.yaml
kubectl -n forgemesh create secret generic forgemesh-api-keys --from-file=api-keys.json
# 3. 控制平面、网关与网络策略；网关的 /readyz 在控制平面就绪之后才通过
kubectl apply -f deploy/kubernetes/
kubectl -n forgemesh rollout status deployment/forgemesh-gateway
# 4. 经网关调用：集群外的调用方经 Ingress 或 LoadBalancer 访问 Service forgemesh-gateway（8081），这里用端口转发
kubectl -n forgemesh port-forward service/forgemesh-gateway 8081:8081 &
curl -s -H "Authorization: Bearer fmk_…" http://127.0.0.1:8081/api/v1/inference-services
```

**网络策略**（`network-policies.yaml`）：默认拒绝命名空间内全部 Pod 的入站与出站流量，只放行全部 Pod 访问集群 DNS（`kube-system` 中 `k8s-app: kube-dns` 的 Pod）、网关访问控制平面的 8080 端口、任何来源访问网关的 8081 端口。集群内的其他工作负载因此不能直接访问控制平面，改为访问 `forgemesh-gateway:8081`，API 路径不变。kubelet 的探针与 `kubectl exec` / `cp` / `port-forward` 不受网络策略限制，容器中的备份与恢复照常进行。NetworkPolicy 只在执行它的 CNI（如 Calico、Cilium）上生效，其他 CNI 静默忽略。命名空间中新增的工作负载同样默认拒绝，需要为它另写策略；集群 DNS 的标签或端口不同（例如使用 NodeLocal DNSCache）时修改 `forgemesh-allow-dns`；调用方只经 Ingress 控制器访问时，可以把网关的入站来源收窄到 Ingress 控制器所在的命名空间。`kubectl apply -f deploy/kubernetes/` 不包括子目录，下文的 `launcher/`、`monitoring/` 与 `event-webhook/` 按需另行应用。

**指标抓取**：控制平面（8080）与网关（8081）的 `/metrics` 不要求认证，只输出跨租户聚合的指标（见「可观测性」），但仍能反映整体负载。默认的策略下只有网关能访问控制平面，而网关不转发 `/metrics`，从 Pod 网络抓取不到控制平面；网关的 `/metrics` 与探针和 API 共用对任何来源开放的 8081 端口，经 Ingress 暴露网关时只路由 `/api/v1/`，不要把整个端口暴露到公网。可选的 `monitoring/network-policy.yaml` 允许命名空间 `monitoring` 中带有 `app.kubernetes.io/name: prometheus` 标签的 Pod 访问这两个端口：应用之前把两条策略的 `namespaceSelector` 与 `podSelector` 改成自己的 Prometheus 所在的命名空间与它的 Pod 的标签。网关的那条在收窄了网关的入站来源之后才起作用。NetworkPolicy 按端口而不是路径放行，这些 Pod 也能直接调用控制平面的 API（仍须 API key），不经过网关的限流与请求体上限。Prometheus 应按 Pod 抓取，不经 Service：网关有多个副本时，经 Service 每次抓到的是不同副本的计数。例如（Prometheus 需要读取命名空间 `forgemesh` 中 EndpointSlice、Service 与 Pod 的 RBAC 权限）：

```yaml
scrape_configs:
  - job_name: forgemesh
    kubernetes_sd_configs:
      - role: endpointslice
        namespaces:
          names: [forgemesh]
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        regex: forgemesh-control-plane|forgemesh-gateway
        action: keep
```

**资源变更事件的 webhook**：资源变更事件默认写进控制平面的日志（见「运行控制平面」）。`event-webhook/patch.yaml` 为控制平面设置 `FORGEMESH_EVENT_WEBHOOK_URL`（应用之前改成接收方的地址）、`FORGEMESH_EVENT_WEBHOOK_TOKEN`（取自 Secret `forgemesh-event-webhook` 的键 `token`，Secret 不存在时不发送 token）与 `FORGEMESH_EVENT_WEBHOOK_TIMEOUT_SECONDS`。默认的策略只允许控制平面出站访问集群 DNS，还要应用 `event-webhook/network-policy.yaml`。NetworkPolicy 不能按域名放行：接收方在集群内时把规则改成它的命名空间、Pod 的标签与容器端口，在集群外时改成它的 IP 地址段与端口；地址不固定的接收方改经出站代理发送（控制平面遵循 `HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY`），规则改为允许访问代理。缺少出站规则时每次发送都超时，事件失败 10 次之后不再自动发送（`dead`）；放行之后按[备份与灾备演练](docs/operations/backup-restore.md)第 5 节重新投递，命令以 `kubectl -n forgemesh exec deployment/forgemesh-control-plane -- python -c "…"` 执行，数据库路径为 `/var/lib/forgemesh/forgemesh.db`。

```bash
kubectl -n forgemesh create secret generic forgemesh-event-webhook --from-file=token=./webhook-token
# 先按接收方修改 event-webhook/network-policy.yaml 的规则与 patch.yaml 中的地址
kubectl apply -f deploy/kubernetes/event-webhook/network-policy.yaml
kubectl -n forgemesh patch deployment forgemesh-control-plane \
  --patch-file deploy/kubernetes/event-webhook/patch.yaml
```

**加速器清单与启动器**：控制平面的 Deployment 没有设置 `FORGEMESH_INVENTORY_FILE` 与 `FORGEMESH_RUNTIME_LAUNCHER`，新建的服务停在 `Pending`（`SchedulerUnavailable`）。在 Kubernetes 上启用它们（变量见「运行控制平面」与「部署推理服务」）：

- 加速器清单放在 ConfigMap 中，挂载到 `/etc/forgemesh` 以外的目录（`/etc/forgemesh` 是 key 文件 Secret 的只读挂载点），`FORGEMESH_INVENTORY_FILE` 指向挂载的文件。控制平面只在启动时读取清单，修改 ConfigMap 之后执行 `kubectl -n forgemesh rollout restart deployment/forgemesh-control-plane`。
- 推理实例运行在控制平面的 Pod 内，监听内核挑选的临时端口（Linux 默认 32768–60999），网关直接连接 Pod IP：启动器的监听地址设为 `0.0.0.0`，写进 `status.endpoints` 的主机用 Downward API 取 Pod IP（Service 只转发 8080，不能写 Service 名）。同时应用 `launcher/network-policy.yaml`，允许网关访问这些端口；修改了 Pod 的 `net.ipv4.ip_local_port_range` 时相应修改端口范围。
- 实例随 Pod 结束：更新 Deployment、Pod 被驱逐或节点故障之后，新的 Pod 启动时 `Ready` 的服务转为 `Degraded`（`InstanceLost`），随后以新实例重新部署（见「部署推理服务」）。

不需要 GPU 的试用，与 `docker-compose.fake-runtime.yaml` 对应（一块虚构设备的清单与 `fake` 启动器，每秒核对一次实例）：

```bash
kubectl -n forgemesh create configmap forgemesh-inventory \
  --from-file=inventory.json=deploy/docker/fake-runtime-inventory.json
kubectl apply -f deploy/kubernetes/launcher/network-policy.yaml
kubectl -n forgemesh patch deployment forgemesh-control-plane \
  --patch-file deploy/kubernetes/launcher/fake-runtime-patch.yaml
# 之后按「部署推理服务」经网关创建服务、对话、删除，以 Authorization 头代替 X-Tenant-Id
# 演练实例丢失：在控制平面的容器内向实例的端口（status.endpoints 中地址的端口）发出 POST /fake/exit，
# 服务在下一次核对时转为 Degraded（InstanceLost），随后以新实例重新部署、回到 Ready
kubectl -n forgemesh exec deployment/forgemesh-control-plane -- python -c \
  "import urllib.request as r; r.urlopen(r.Request('http://127.0.0.1:<端口>/fake/exit', method='POST'))"
```

`vllm-process` 与 `tgi-process` 启动器在控制平面的容器内以子进程运行 vLLM / TGI。仓库没有为它们提供清单，也没有在 Kubernetes 上验证过；使用它们需要：

- 装有 vLLM 或 TGI（`text-generation-launcher`）的控制平面镜像，或用 `FORGEMESH_VLLM_COMMAND` / `FORGEMESH_TGI_COMMAND` 指向镜像中另一个装有它们的环境；
- 为控制平面的容器申请 GPU（例如 NVIDIA device plugin 的 `nvidia.com/gpu`）；加速器清单恰好一个节点，设备 id 是容器内可见的 CUDA 设备序号或 UUID；
- 根文件系统只读：模型缓存目录（如 `HF_HOME`）挂载可写的卷；一个实例使用多块 GPU 时在 `/dev/shm` 挂载 `medium: Memory` 的 emptyDir（容器默认的 `/dev/shm` 只有 64 MiB，vLLM 与 TGI 的多卡通信需要更多共享内存，用量计入内存限制）；按模型调大内存限制；`terminationGracePeriodSeconds` 调到 45 秒以上（默认 30 秒，停止全部实例连同收尾最长约 40 秒）；
- 按上文把所选启动器的监听地址（`FORGEMESH_VLLM_BIND_HOST` / `FORGEMESH_TGI_BIND_HOST`）设为 `0.0.0.0`、以 Pod IP 作为写进 `status.endpoints` 的主机（`FORGEMESH_VLLM_ADVERTISE_HOST` / `FORGEMESH_TGI_ADVERTISE_HOST`），并应用 `launcher/network-policy.yaml`；
- 网络策略不允许控制平面访问外网：vLLM 与 TGI 启动时下载模型，需要为控制平面另加出站策略，或者预先把模型放进挂载的卷、以本地路径作为 `spec.model.reference`。

两个启动器把实例进程记在数据库旁的 `forgemesh.instances.json` 中（见「部署推理服务」的「vLLM」），在清单中就是数据卷上的 `/var/lib/forgemesh/forgemesh.instances.json`。写不进这个文件时实例启动失败，`FORGEMESH_DB_PATH` 因此要留在可写的卷上。清单的 Pod 中控制平面是容器的 1 号进程（镜像以 exec 形式启动它），即使它被杀死，内核也会随之结束容器中的实例进程；容器重启之后启动器找不到记录中的进程（只认 pid 与启动时间都一致的进程），不向任何进程发送信号，直接丢弃这些记录。TGI 实例的私有目录建在临时目录下（默认 `/tmp`，清单中是 emptyDir）。

控制平面只有一个副本，所有实例都运行在这一个 Pod 里：只能使用分配给这个 Pod 的 GPU，也就是单个节点上的 GPU，不能把实例调度到其他节点。

### 可观测性

控制平面与网关各自在 `/metrics` 以 Prometheus 文本格式（0.0.4）输出指标，不依赖第三方库：

```bash
curl -s http://127.0.0.1:8080/metrics   # 控制平面
curl -s http://127.0.0.1:8081/metrics   # 网关
```

两个 `/metrics` 与探针一样不要求认证，网关也不对它限流。输出只有跨租户聚合的计数与耗时：标签只取路由模板（网关为路由类别）、方法、状态码与固定的枚举值（加速器类型取自运维配置的清单），不含租户、服务名、命名空间、uid、路径或实例地址；每个指标最多 200 个标签组合，超出的计入标签全为 `overflow` 的一个序列。控制平面的 `/metrics` 不在 `/api/v1` 之下，网关不转发它。指标仍能反映整体负载，不要经公网入口暴露这两个端点，只让 Prometheus 所在的网络访问（compose 只发布在 `127.0.0.1`，Kubernetes 见「Kubernetes 部署」的「指标抓取」）。

控制平面的指标（网关的指标见 [gateway/README.md](gateway/README.md)「可观测性」）：

| 指标 | 类型 | 说明 |
| --- | --- | --- |
| `forgemesh_http_requests_total{method,route,status}` | counter | HTTP 请求数；`route` 为路由模板（如 `/api/v1/inference-services/{name}`），未匹配路由的请求为 `unmatched` |
| `forgemesh_http_request_duration_seconds{method,route}` | histogram | 处理 HTTP 请求的耗时 |
| `forgemesh_reconcile_total{operation,outcome}` | counter | 协调次数；`operation` 为 `reconcile` / `recover`（`Degraded`）/ `reclaim`（`Failed`）/ `terminate`（`Terminating`），`outcome` 为 `done` / `requeue` / `wait` / `conflict` / `error` |
| `forgemesh_reconcile_duration_seconds{operation}` | histogram | 一次协调的耗时，含 status 回写 |
| `forgemesh_controller_retries_total` | counter | 协调抛出异常、按退避重新排队的次数 |
| `forgemesh_controller_queue_depth{state}` | gauge | 工作队列中的键数：`ready`（可立即处理）/ `delayed`（等待延迟到期）/ `processing` |
| `forgemesh_deployments_total{outcome,reason}` | counter | 推理服务进入 `Ready`（`outcome="ready"`）或 `Failed`（`outcome="failed"`）的次数，`reason` 为 `Ready` 条件的原因 |
| `forgemesh_instance_checks_total{outcome}` | counter | 控制器定期核对实例的次数：`completed` / `failed`（下一个周期重试）；启动时的核对不计入（它失败时控制平面不会启动） |
| `forgemesh_instance_check_results_total{check,result}` | counter | 核对实例得出结论的服务数：`check` 为 `startup`（启动时）/ `periodic`（定期）；`result` 为 `lost`（实例已不在运行，服务转为 `Degraded`）/ `back`（`Degraded` 服务的实例全部又在运行，交给协调器转回 `Ready`，转回之前每次核对都计入）/ `conflict`（服务在核对期间被修改或删除，没有报告） |
| `forgemesh_lost_instance_redeploys_total` | counter | 为实例已不在运行的服务启动重新部署运行的次数；退避期间推迟的协调不计入 |
| `forgemesh_orphaned_allocation_reclaims_total{outcome}` | counter | 回收孤立分配的次数：`reclaimed`（实例已停止，分配已过期）/ `failed`（没能停止实例，分配保留，下一次全量同步之后重试） |
| `forgemesh_inference_services{phase}` | gauge | 各阶段的推理服务数 |
| `forgemesh_allocations_active{accelerator_type}` | gauge | Active 分配数，按其设备在清单中的类型 |
| `forgemesh_accelerator_devices{accelerator_type,state}` | gauge | 清单中的设备数，`state` 为 `free` / `allocated`；未配置清单时没有样本 |
| `forgemesh_workflow_runs{status}` | gauge | 各状态的部署工作流运行数；未设置启动器时没有样本 |
| `forgemesh_outbox_events{state}` | gauge | 未发布的资源变更事件数：`pending`（尚未发布失败过）/ `retrying`（发布失败过，等待重试）/ `dead`（失败次数达到上限，不再自动重试，重新投递见[备份与恢复](docs/operations/backup-restore.md)第 5 节） |
| `forgemesh_outbox_publish_attempts_total{outcome}` | counter | 发布事件的尝试次数，在结果写入数据库之后计数：`published` / `failed`（将重试）/ `dead`（最后一次失败）；三个取值都从 0 开始输出 |
| `forgemesh_database_lock_wait_seconds` | histogram | 控制平面数据库的一次操作等待连接锁的时间：注册中心、Outbox（含事件发布）与审计记录共用这个连接与锁（不含 SQLite 内部的忙等待） |

核对实例、重新部署与回收孤立分配的四个计数器的每个标签组合都从 0 开始输出，未设置启动器时保持为 0。

**追踪上下文与日志**：网关沿用请求中合法的 W3C `traceparent`，否则生成一个，随请求转发给控制平面与推理实例。控制平面按同样的规则读取 `traceparent`（直连控制平面的请求同样适用），并沿用 `X-Correlation-Id`（缺失时生成，审计记录与资源变更事件使用同一个关联 ID），这个请求的日志与错误信封（`{"error": {"code", "message", "correlationId", "traceId", …}}`）都带上 trace-id 与关联 ID，一次请求在网关与控制平面的日志里可以按 trace-id 对上。控制平面的日志写到标准错误，默认每行一个 JSON 对象，uvicorn 的访问日志也是如此：

```json
{"time": "2026-10-13T03:53:36.002Z", "level": "INFO", "logger": "uvicorn.access", "message": "127.0.0.1:38956 - \"GET /api/v1/inference-services/llama HTTP/1.1\" 200", "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736", "correlation_id": "ad1cd525116940f682ea882c0c7a6b67"}
```

`FORGEMESH_LOG_FORMAT=text` 时每行为 `时间 级别 logger trace=<trace-id> corr=<关联 ID> 消息`（UTC 时间，缺失的 ID 记为 `-`）。资源变更事件（logger `forgemesh.events`）也以这个格式输出，消息是事件信封的 JSON；它固定为 `INFO`，不受 `FORGEMESH_LOG_LEVEL` 影响。网关的访问日志见 [gateway/README.md](gateway/README.md)「可观测性」。两边都不记录 span，也不导出追踪（OTLP 导出尚未实现）；`forgemesh.controlplane.observability` 中的内存记录器（`MetricsRecorder` / `TraceRecorder`）不经 `/metrics` 输出。

---

## 4. 功能概览（v1.0.0）

ForgeMesh 目前处于预览阶段，API 稳定性标注为 `beta`：minor 版本可能包含破坏性变更，均在 [CHANGELOG](CHANGELOG.md) 中标注（规则见[弃用政策](docs/governance/deprecation-policy.md) 1.4 节）。v1.0.0 发布时曾标注为稳定版本，这一标注已经更正；改回 `stable` 之前，应当满足[架构验收标准](docs/architecture/architecture.md)（第 20 节）。

以下是 v1.0.0 发布时列出的核心能力，与实现不符或尚未完成的部分在条目后以「现状」注明：

### 核心控制平面（M2）
- **资源注册中心**：SQLite 持久化，乐观并发 CAS，租户隔离，分页游标（现状：自 1.1.0 起列表接口返回签名的分页游标 `nextPageToken`）
- **协调控制器**：9 步声明式闭环（获取→查询→比较→生成候选→策略校验→调度分配→执行→重观察→更新状态）（现状：自 1.1.0 起由控制器自动驱动；同时设置 `FORGEMESH_INVENTORY_FILE` 与 `FORGEMESH_RUNTIME_LAUNCHER` 后，调度、部署、写入 `status.endpoints`、`spec` 变化之后的重新部署、实例丢失（包括控制平面重启之后）时的重新部署与受控删除形成完整闭环，见[部署推理服务](#部署推理服务)；未设置启动器时，新建的推理服务停在 `Pending`，并报告 `Ready=False`（`SchedulerUnavailable` 或 `WorkflowRuntimeUnavailable`）。每个服务只部署一个实例；核对实例只看实例是否还在运行，不探测它能否回答请求）
- **策略引擎**：allow/deny/require_approval 三态决策，配额规则，高风险审批触发
  - 说明：Decision Router 的 `OK/FALLBACK_REQUIRED/FAILED` 是决策路由层的另一组状态，与 Policy 三态不混用
- **插件管理器**：Manifest 解析校验，生命周期状态机，能力发现，权限授予（现状：自 1.1.0 起注册时验证 Ed25519 签名，权限按管理员的授予判定；控制平面应用尚未加载插件）
- **审计与 Outbox**：事务性 Outbox，异步事件发布，幂等消费，correlation_id 全链路（现状：自 1.1.0 起接入控制平面：每次资源写入在同一事务中记下资源变更事件，经 API 的写入还记下审计记录，两者带同一个关联 ID（请求的 `X-Correlation-Id`）；事件在后台发布到日志或 webhook（`FORGEMESH_EVENT_WEBHOOK_URL`），至少投递一次，接收方按 `eventId` 去重；审计记录经 `GET /api/v1/audit-events` 查询）
- **REST API 与 CLI**：资源 CRUD，状态查询，任务管理，租户上下文注入（现状：自 1.1.0 起提供 `PUT` / `PATCH` 更新接口（以 `resourceVersion` 或 `ETag` 做乐观并发），CLI 经 REST API 管理推理服务（`apply` / `get` / `list` / `describe` / `delete` / `wait`），只管理 `InferenceService`；控制平面应用不对外提供工作流运行时，`POST /api/v1/workflows` 返回 `503`）

### 工作流与决策（M3）
- **工作流引擎**：持久化执行，幂等去重，三级超时，重试策略
- **参考执行器**：DSL 解析，补偿步骤（反向执行），审批等待与重校验
- **熔断器与降级**：closed/open/half-open 状态机，按风险决定是否尝试备用
- **决策插件**：decision-rules（确定性规则）、decision-jev（JEV 类型化决策）

### 执行平面与 GPU 生命周期（M4）
- **Node Agent**：注册认证，设备/驱动探测，本地运行时管理，断线重连，任务恢复（现状：与控制平面之间只有进程内调用，没有网络传输）
- **gRPC 任务协议**：6 类任务枚举，心跳与离线检测，幂等键去重（现状：协议定义在 `api/protobuf/agent.proto`，尚未生成代码）
- **gpu-nvidia 插件**：GPU 枚举，能力采集，驱动/CUDA 预检，周期性健康检查
- **runtime-vllm 插件**：模型拉取，实例部署，冒烟验证，租约释放（现状：插件中的默认 launcher 仍抛出 `NotImplementedError`；控制平面部署推理服务时不使用这个插件，而是使用自带的 vLLM 进程启动器 `FORGEMESH_RUNTIME_LAUNCHER=vllm-process`）
- **GPU 上线 E2E**：发现→验证→准备→部署→验收→提交 完整闭环（现状：工作流级测试，各步骤为模拟实现）
- **故障恢复**：节点离线不误判、控制平面重启恢复、重复事件不盲目重试

### 生产级治理（M5）
- **配额与 QoS**：多租户配额策略，服务优先级排队，限流
- **Token 计量**：追加式 UsageRecord，计量修正事件
- **可观测性**：OpenTelemetry 接入，8 类关键指标，敏感数据过滤（现状：自 1.1.0 起控制平面与网关在 `/metrics` 以 Prometheus 文本格式输出跨租户聚合的指标，按 W3C `traceparent` 传递 trace-id 并写进日志与错误信封；没有接入 OpenTelemetry SDK，不记录 span，也没有 OTLP 导出）
- **诊断包**：统一诊断聚合（资源/工作流/插件/决策/日志），按租户权限受限
- **插件安全**：制品签名验证，隔离宿主（受限 RPC），升级兼容性（现状：自 1.1.0 起制品使用 Ed25519 签名，注册时验签；受限 RPC 宿主尚未实现，插件都在宿主进程内运行，清单只能声明 `in-process`）
- **安全测试**：租户隔离，插件权限，密钥脱敏，灾备文档

### 扩展与生态（M6）
- **MIG 支持**：NVIDIA MIG 实例管理（创建/销毁/发现/健康检查/能力报告），并发安全，资源不超分
- **异构 GPU**：非 NVIDIA 能力差异显式报告（差异不隐藏）
- **Temporal Provider**：WorkflowRuntime 契约适配，补偿注册，信号投递（现状：只有内存客户端，没有接入 temporalio）
- **TGI Runtime**：HuggingFace TGI 推理 Runtime Provider（deploy→verify→release 闭环）（现状：插件中的默认 launcher 仍抛出 `NotImplementedError`；控制平面部署推理服务时不使用这个插件，而是使用自带的 TGI 进程启动器 `FORGEMESH_RUNTIME_LAUNCHER=tgi-process`）
- **插件 SDK**：脚手架生成，Manifest 校验，打包发布，开放契约测试套件
- **治理文档**：GOVERNANCE / SECURITY / CONTRIBUTING / CODE_OF_CONDUCT 完善，弃用政策
- **API 版本管理**：OpenAPI 版本号与稳定性标注（当前为 1.1.0、`beta`），CHANGELOG，发布记录，CI 架构文档检查

### 测试统计

| 类型 | 数量 |
|------|------|
| Python 测试 | **2698** |
| Python 覆盖率 | 97.2%（`packages/` 的语句与分支，CI 要求不低于 97%） |
| Go 测试 | 188 个测试函数，全部通过 |
| ruff | `ruff check` 与 `ruff format --check` 均通过 |

---

## 5. 插件体系

ForgeMesh 通过可插拔架构支持 6 种插件类型：

| 类型 | 已实现插件 | 说明 |
|------|-----------|------|
| decision | `decision-rules`, `decision-jev` | 决策 Provider（规则 / JEV） |
| workflow | `workflow-reference`, `workflow-temporal` | 工作流执行后端 |
| runtime | `runtime-vllm`, `runtime-tgi` | 推理运行时 Provider |
| gpu | `gpu-nvidia` | GPU Provider（含 MIG / 异构） |
| metering | `metering-basic` | Token 计量 |
| qos | `qos-basic` | QoS 优先级与限流 |

插件在宿主进程内运行，目前没有进程隔离（清单 `isolation` 只能是 `in-process`），入口点形如 `<模块>:<类>`（如 `provider:RulesProvider`）。插件管理器注册插件时要求随制品分发的 Ed25519 签名（覆盖制品内容与清单），只接受插件策略文件 `trustedKeys` 中的公钥；清单的 `permissions` 只是申请，插件实际拥有的权限是申请与策略文件 `grants` 授予的交集。密钥生成、签名与验签：`python -m forgemesh.plugin.signtool {keygen,sign,verify}`。控制平面应用目前不加载插件，也不构造插件管理器，因此没有对应的环境变量。

插件开发指南见 [`docs/plugin-development/guide.md`](docs/plugin-development/guide.md)。

---

## 6. 里程碑

| 里程碑 | 阶段 | 状态 | 描述 |
|--------|------|------|------|
| M0-M1 | P0 | ✅ 完成 | 工程骨架 + 领域契约冻结 |
| M2 | P1 | ✅ 完成 | 核心控制平面 |
| M3 | P2 | ✅ 完成 | 工作流与决策 |
| M4 | P3 | ✅ 完成 | 执行平面与 GPU 生命周期 |
| M5 | P4 | ✅ 完成 | 生产级治理 |
| M6 | P5 | ✅ 完成 | 扩展与生态（v1.0.0 发布） |
| v1.1 | — | ✅ 已交付 | 可信基线（阶段 A–E），见 [`v1.1-development-plan.md`](docs/architecture/v1.1-development-plan.md) |
| 1.1.0 | 阶段 1–3 | ✅ 已合入 | #5–#23：数据正确性、认证与控制器；推理服务的最小闭环；资源的生命周期、可观测性与工程化（尚未发布） |
| 1.1.0 之后 | 阶段 4–5 | 计划中 | 执行面与 MaaS 基础；生产规模与正式版。见 [`roadmap.md`](docs/architecture/roadmap.md) |

> 里程碑定义与准入/准出条件（DoD）见 [`development-plan.md`](docs/architecture/development-plan.md)；1.1.0 之后的阶段、迭代与验收见 [`roadmap.md`](docs/architecture/roadmap.md)。

---

## 7. 参与贡献

请先阅读 [`CONTRIBUTING.md`](CONTRIBUTING.md) 与 [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md)；安全漏洞请遵循 [`SECURITY.md`](SECURITY.md) 的**私有披露**流程；项目治理与决策机制见 [`GOVERNANCE.md`](GOVERNANCE.md)。重大架构变更需通过 **ADR** 提案，模板见 [`docs/design-decisions/README.md`](docs/design-decisions/README.md)。

---

## 8. 许可证

本项目采用 **Apache License 2.0** 许可，完整文本见 [`LICENSE`](LICENSE)。

```
Copyright 2026 Mekoforge

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
