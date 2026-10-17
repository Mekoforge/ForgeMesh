# Changelog

本文件记录 ForgeMesh 所有显著变更，遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/) 格式，
版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/) 规范。

## [1.1.0] - Unreleased

### Added

- SQLite 单实例部署约束文档：`deploy/docker/docker-compose.yaml` 顶部注释（ADR-013：仅允许单控制平面实例写同一权威库，不得扩展 replicas）与 [docs/operations/backup-restore.md](docs/operations/backup-restore.md) 新增"单实例写入约束"小节（含升级流程：停写 → 备份 → 升级 → 恢复）。
- 新增 [docs/architecture/v1.1-development-plan.md](docs/architecture/v1.1-development-plan.md)（阶段 A–E 可信基线计划）与 [docs/architecture/module-mapping.md](docs/architecture/module-mapping.md)（逻辑模块 ↔ 真实代码路径映射，含依赖边界 import 循证）。
- 新增 [docs/architecture/roadmap.md](docs/architecture/roadmap.md)：1.1.0 之后的路线图。说明与 KServe、Kthena 的分工：Kubernetes 上把二者作为执行后端，ForgeMesh 负责不依赖 Kubernetes 的执行面与平台治理层。阶段 4（执行面与 MaaS 基础）与阶段 5（生产规模与正式版）按迭代列出主题、依赖与验收，附 architecture.md 第 20 节的正式版验收表，以及 #8–#22 的 PR 描述、ADR-014 与 1.1.0 文档中遗留问题的归属。
- [docs/architecture/architecture.md](docs/architecture/architecture.md) 修订为 v1.1：去掉复制时带入的对话文字与界面标签，三张示意图改为 mermaid 源码（原图是本机文件路径与 GitHub 不渲染的 data URI）；与 ADR-013、ADR-014 和 1.1.0 的实现对齐（ADR 总表、权威存储、单写者部署、API 与认证、事件、可观测性、仓库结构）；新增执行后端（10.4，写明与 KServe、Kthena 的分工）与模型服务的接入（11.5）。仓库中的本文件改为唯一事实来源，修订内容记在文末的修订记录中；README 的六层概览同步。
- 新增 CI 校验脚本：`scripts/validate_manifests.py`（JSON Schema metaschema 自校验 + 10 插件 manifest 实例校验）、`scripts/openapi_semantic_check.py`（operationId 唯一 / $ref 可解析 / 响应完整性）、`scripts/check_metadata_consistency.py`（requires-python>=3.12、描述禁 skeleton/placeholder）、`scripts/plugin_entrypoint_smoke.py`（插件入口进程内冒烟）。
- 新增契约 / 混沌 / 兼容测试 122 项（1115 vs 993）：Registry CAS 并发 / 租户过滤 / 重启恢复（19）、Workflow 恢复版本固定 / 取消补偿 + Outbox 事务 / 幂等 / 死信（12）、10 插件发布链路 conformance——manifest/isolation 一致性 + 四类 Provider 契约 + 卸载安全（91）。
- 控制平面可运行入口：`control-plane` 包新增控制台脚本 `forgemesh-control-plane`（`forgemesh.controlplane.app:main`）；权威存储路径取自 `FORGEMESH_DB_PATH`（默认 `forgemesh.db`），监听地址取自 `FORGEMESH_HOST` / `FORGEMESH_PORT`（默认 `127.0.0.1:8080`），启动方式见 README「运行控制平面」。
- 新增 `tests/contract/test_registry_hardening.py`：并发创建全部持久化、空 namespace 身份唯一、删除 / 释放 CAS、标签过滤与分页等 Registry 回归测试。
- 新增 [ADR-014](docs/design-decisions/adr-014.md)（Accepted）：对外 API 采用绑定租户的 API Key 认证。
- API Key 认证（ADR-014）：网关（`GATEWAY_AUTH_MODE` / `GATEWAY_API_KEYS_FILE`）与控制平面（`FORGEMESH_AUTH_MODE` / `FORGEMESH_API_KEYS_FILE`）各自校验 `Authorization: Bearer <key>`；key 文件只保存 SHA-256 摘要，每个 key 绑定租户、主体与 scope（`read` / `write` / `*`）；`python -m forgemesh.controlplane.keygen --tenant <租户> --subject <主体>` 生成 key 与对应的 key 文件条目。
- 网关 `make run-dev`：以 `dev` 认证模式在 `127.0.0.1:8081` 启动，仅供本地开发。
- OpenAPI：新增 `apiKey`（HTTP bearer）安全方案并设为全局 `security`（探针豁免），全部 `/api/v1` 端点声明 `401` / `403`。
- 控制器：控制平面在应用启动（lifespan）时运行 `ControllerManager`，自动协调 `InferenceService` 并回写 `status`。Registry 每次写入时在同一事务中追加一条 `resource_changes` 变更记录，控制器按序号增量读取并清理已读记录；启动时全量重建工作队列，另每 10 分钟全量同步一次。工作队列按资源去重、支持延迟重试，失败时指数退避；`status` 以 `resource_version` 做 CAS 写入，去重键在写入成功之后才登记。可选的 `instance_check` 每 `instance_check_interval` 秒（默认 `10`）调用一次，报告的资源加入队列；可选的 `reclaim` 在启动时与每次全量同步之后调用；二者失败时记日志，下一个周期重试，应用在部署推理服务时注入它们（核对实例与回收孤立的分配，见下文）。控制器是单写者（ADR-013），应用只能以单进程运行。
- Registry status 子资源写入 `update_status`：只修改 `conditions` / `phase` / `observedGeneration`，不递增 `generation`；条件按 `type` 合并，`status` 不变时保留 `lastTransitionTime`；支持以 `resource_version` 做 CAS，未建模的 payload 字段原样保留。另新增 `get_with_phase`、`list_resource_keys`、`changes_since`、`latest_change_seq`、`prune_changes`。
- `ReconciliationController` 新增 `record_dedup_keys` 参数与 `mark_processed()`：调用方可以在持久化 `status` 之后再登记去重键。
- `InferenceServiceStatus.endpoints`：每个实例包含 `url`（http/https 基地址，不含凭证、查询串与片段）、`ready`（默认 `true`）、可选的 `instanceId`，以及必填的 `protocol`（`openai-chat` / `tgi-generate`），资源模型、JSON Schema 与 OpenAPI 一致；`update_status` 可以整体替换 `endpoints`，写入前逐项校验。
- 网关新增带命名空间的推理路由 `/api/v1/namespaces/{ns}/inference/{svc}/chat`：Resolver 以 `?namespace=` 查询控制平面，缓存按租户与命名空间隔离；`/api/v1/inference/{svc}/chat` 继续解析不带命名空间的服务。
- 网关请求体上限 `GATEWAY_MAX_REQUEST_BODY_BYTES`（默认 10 MiB）：声明的 `Content-Length` 超限时直接返回 `413 REQUEST_TOO_LARGE`，不访问上游；分块传输的请求体读到上限时同样返回 `413`，不计实例失败。
- 容器镜像：`deploy/docker/Dockerfile.control-plane`（控制平面，以 uid 10001 运行）与 `deploy/docker/Dockerfile.gateway`（静态编译的网关，distroless `nonroot` 用户）；根目录 `.dockerignore` 只把 `packages/` 与 `gateway/` 送入构建上下文。
- `scripts/compose_smoke.sh` 端到端冒烟：构建镜像，以 `deploy/docker/docker-compose.yaml` 叠加 `deploy/docker/docker-compose.fake-runtime.yaml`（一块设备的加速器清单 + 假启动器，见下）启动，经网关验证无 key 返回 `401`，并走完推理服务的闭环：创建 → `Ready`（`status.endpoints` 指向假实例）→ 经网关流式对话（以 `data: [DONE]` 结束）→ 删除（资源消失、实例端口关闭）→ 同名重新创建（只有一块设备，可见分配已释放）并再次对话 → 以 `PUT` 修改 `spec` → `Updating`（没有就绪的端点）→ 以新实例回到 `Ready`（`observedGeneration` 为 `2`，旧实例停止监听）并再次对话 → 让实例退出（`POST /fake/exit`）→ `Degraded`（`InstanceLost`）→ 重新部署，以新实例回到 `Ready` 并再次对话 → 重启控制平面：资源仍在，服务转为 `Degraded`（`InstanceLost`）后重新部署，以新实例回到 `Ready` 并再次对话 → 删除。每一步都有时限，只等待状态变化，不依赖网关在某一刻返回的状态码。CI 新增 `compose-smoke` job 执行该脚本（依赖 runner 自带的 `curl` 与 `jq`）。
- CI `go` job 新增 `gofmt` 检查与 `go vet`，网关测试改以 `-race` 运行。
- [docs/operations/backup-restore.md](docs/operations/backup-restore.md) 新增容器部署中的备份与恢复（2.5 / 3.4 节）：镜像不含 `sqlite3` 命令行、根文件系统只读，改在容器内用备份 API 生成快照并写回；Kubernetes 上 PVC 应使用块存储。
- 调度器 `forgemesh.controlplane.scheduler.FirstFitScheduler`：按静态加速器清单首次适配分配。清单是 JSON 文件，路径取自 `FORGEMESH_INVENTORY_FILE`（未设置时不启用调度器），列出节点及其设备（`id` / `type` / `memoryGiB`），严格校验（未知字段、隐式类型转换、空列表、重复 id 都是错误），文件不可读或非法时控制平面拒绝启动，格式见 README「运行控制平面」。调度器按清单顺序取第一个能凑齐数量的节点（类型精确匹配、单卡显存不低于 `minMemoryGiB`，不跨节点）；同一幂等键返回同一分配（重启后亦然）；资源已持有仍满足需求的分配时直接复用，不满足时报冲突，须先释放；`feasible()` 只读，`release()` 幂等。分配不设租约：协调器在删除资源、部署失败（实例停止之后）与加速器申请变化时释放分配，归属资源已不存在的分配由控制器回收（见下文）。未设置 `FORGEMESH_RUNTIME_LAUNCHER` 时应用不注入工作流运行时，协调器在分配之前即报告 `Ready=False`（`WorkflowRuntimeUnavailable`）。
- 设备级独占：新表 `allocation_devices` 记录每个分配占用的 `(node_id, device_id)`，部分唯一索引 `idx_allocation_devices_one_active` 保证一个设备至多属于一个 Active 分配，触发器 `trg_allocation_devices_follow_status` 让设备随分配状态一起释放。`SQLiteRegistry.create_allocation` 在同一个 `BEGIN IMMEDIATE` 事务中检查归属资源（存在、uid 一致、不在 `Terminating`）、幂等键（重放返回原分配，用过的键不再复用）、资源与设备是否已被占用，再一起写入分配与设备行；另新增 `get_allocation`、`get_allocation_by_idempotency_key`、`allocated_devices`、`release_active_allocation`，分配记录包含 `devices`。
- 调度契约：`ScheduleRequest` 新增 `resource_ref`（`ResourceRef`：`kind` / `namespace` / `name` / `uid`）与 `idempotency_key`，`AllocationRef` 新增 `devices`（`AllocatedDevice`：`node_id` / `device_id`）。协调器以 `{uid}:{generation}`（`allocation_idempotency_key`）为幂等键，`requirements` 取自 `spec.resources.accelerator`（`{"accelerator": {"type", "count", "minMemoryGiB"}}`）；没有 `metadata.uid` 的资源报告 `Ready=False`（`MissingUid`），不分配。
- 工作流契约：运行快照 `RunInfo`（`status`、按定义顺序的步骤状态 `steps`、已成功步骤的输出 `outputs`（步骤之后被补偿也保留）、失败原因 `error`、属性 `terminal`）；`WorkflowRuntime` 新增三个非抽象方法：`get_run(run_id)`（默认只填 `status`）、`find_run(idempotency_key)`（按幂等键查找运行、不创建）与 `find_runs(key_prefix)`（查找幂等键以 `key_prefix` 开头的全部运行、不创建，返回按创建先后排列的 `{幂等键: 快照}`），后两者默认抛 `NotImplementedError`，既有实现（如 `workflow-temporal`）无需修改。
- 工作流运行时 `forgemesh.workflowruntime.runtime.LocalWorkflowRuntime`：`start()` 创建（或按幂等键复用）运行后立即返回，步骤由运行时持有的后台任务按定义顺序执行（重试、步骤超时与工作流超时沿用 `WorkflowExecutor`），每一步的状态与结果都落盘。全部成功 → `Succeeded`；某一步用尽重试或工作流超时 → `Compensating`，反向补偿已成功且注册了补偿的步骤，未执行的步骤置为 `Skipped`，然后 `Failed`（`error` 记录失败的步骤与错误）。`cancel()` 中断正在执行的步骤并等待运行进入 `Cancelled`（`cleanup=True` 时先补偿）；`recover()`（进程启动时调用一次）让未结束的运行从第一个未成功的步骤继续，或完成中断的补偿；`wait()` 等待后台执行结束；`aclose()` 取消或等待全部后台任务，之后才能关闭存储。审批结果只保存在内存中，重启后需要重新审批。`workflow-reference` 插件的 `ReferenceWorkflowRuntime` 即是它的子类。`WorkflowStore` 记录运行失败原因（打开旧库时自动补上 `error` 列）、补偿后保留步骤结果、记录步骤尝试次数，新增 `get_run_by_idempotency_key` 与按幂等键前缀查询的 `get_runs_by_idempotency_key_prefix`（`GLOB`，区分大小写，`find_runs` 使用）；连接以锁串行化，文件库以 WAL 模式打开。
- 推理服务部署 `forgemesh.controlplane.deployment`：启动器契约 `RuntimeLauncher`（`prepare`（可选）/ `launch` / `verify` / `stop`，均须幂等，`stop` 须在有限时间内返回；`is_running` 只查看启动器自己的记录，报告实例是否仍在运行；`start`（可选）在应用启动、恢复部署之前调用，实例可能比控制平面进程活得更久的启动器在这里结束上次运行留下的实例；`aclose`（可选）在应用关闭时调用，实例随控制平面进程结束的启动器在这里停止全部实例）及其请求 `LaunchRequest`（实例幂等键、租户、资源身份、`spec`、分配 id 与设备）与结果 `LaunchedInstance`（实例 id、地址、协议，按 `status.endpoints` 的规则校验）；部署工作流 `inference-service-deployment`：`prepare-model`（单次时限 3600 秒）→ `launch-instance`（600 秒，补偿为 `stop`）→ `verify-instance`（1800 秒），步骤 handler 为 `DeploymentSteps`；资源 uid 的每一代 `spec` 一次部署运行，幂等键 `deployment:{uid}:{generation}`，实例丢失后重新部署时同一代 `spec` 再启动一次（第 n 次为 `deployment:{uid}:{generation}:{n}`，`deployment_run_key(uid, generation, attempt=1)`），幂等键同时是实例幂等键；`find_deployment_runs(runtime, uid, generation=None)` 经 `find_runs` 按创建先后找回资源的全部（或某一代 `spec` 的）部署运行。`DeploymentReleaser`（协调器的 `ReleaseProvider`）在资源删除时、启动新的部署运行之前与释放 `Failed` 服务的分配之前，取消资源每一个进行中的部署运行（补偿停止已启动的实例）并停止可能仍在运行的实例；`release_uid(uid)` 只凭 uid 做同样的清理（回收孤立的分配时使用）。启动器有测试替身 `InMemoryLauncher`、假启动器 `FakeLauncher` 与 vLLM / TGI 进程启动器 `VLLMProcessLauncher` / `TGIProcessLauncher`（见下），应用按 `FORGEMESH_RUNTIME_LAUNCHER` 选择。`forgemesh.controlplane.sqlite.workflow_db_path()` 给出工作流存储的文件：控制平面数据库旁的独立文件（`forgemesh.db` → `forgemesh.workflows.db`），步骤写入不必等待 Registry 的写锁。
- 假启动器 `forgemesh.controlplane.deployment.FakeLauncher`（`FORGEMESH_RUNTIME_LAUNCHER=fake`）：每个实例是控制平面进程内的一个 OpenAI 兼容桩服务（只用标准库，运行在控制平面的事件循环中），监听在内核挑选的临时端口上，提供 `GET /health`、`POST /v1/chat/completions`（`"stream": true` 时以 SSE 返回若干 `chat.completion.chunk` 事件、以 `data: [DONE]` 结束，否则返回一个 `chat.completion` 对象；回复为含实例 id 的固定文本）与 `POST /fake/exit`（返回 `202`，实例随即像进程退出一样结束：启动器立即不再认为它在运行，随后停止监听、中断其余请求；用来演练实例丢失，网关只转发推理协议的路径，经网关调用不到它）。不需要 GPU、不加载模型，供 CI 与本地开发使用；实例随控制平面进程结束。
- vLLM 进程启动器 `forgemesh.controlplane.deployment.VLLMProcessLauncher`（`FORGEMESH_RUNTIME_LAUNCHER=vllm-process`）：每个实例是控制平面所在主机上的一个 vLLM OpenAI 兼容服务子进程，命令为 `<FORGEMESH_VLLM_COMMAND> --model=<spec.model.reference> --host <监听地址> --port <空闲端口> --tensor-parallel-size <设备数>`；`CUDA_VISIBLE_DEVICES` 取自分配的设备（清单中的设备 id 须是 CUDA 设备序号 `0`、`gpu-0`，或以 `GPU-` / `MIG-` 开头的 UUID），未设置 `CUDA_DEVICE_ORDER` 时设为 `PCI_BUS_ID`；实例继承控制平面的环境变量（去掉 `FORGEMESH_*`），输出即控制平面的输出。只服务 `spec.runtime.provider` 为 `vllm` 的资源，`spec.runtime.version` 与 `spec.model.digest` 不检查；加速器清单必须恰好有一个节点（控制平面所在的主机）。`verify` 等到 `GET /health` 返回 `200`，进程退出或启动后超过就绪时限即失败；每个实例在自己的会话中运行，`stop` 向实例进程的进程组发送 `SIGTERM`，实例进程退出或宽限期过后向会话中的每个进程组发送 `SIGKILL`，在 Linux 上（读 `/proc`）等到会话中的进程都已退出（至多 10 秒，否则 `stop` 失败）；`aclose` 并行停止全部实例。启动、就绪检查与停止由两个进程启动器共用的 `forgemesh.controlplane.deployment.process.ProcessLauncher` 实现。
- TGI 进程启动器 `forgemesh.controlplane.deployment.TGIProcessLauncher`（`FORGEMESH_RUNTIME_LAUNCHER=tgi-process`）：每个实例是控制平面所在主机上的一个 TGI（text-generation-inference）子进程，命令为 `<FORGEMESH_TGI_COMMAND> --model-id <spec.model.reference> --hostname <监听地址> --port <空闲端口> --num-shard <设备数> --master-port <另一个空闲端口> --shard-uds-path <实例私有目录>/shard`；只服务 `spec.runtime.provider` 为 `tgi` 的资源，实例的协议为 `tgi-generate`（网关转发到 `/generate_stream`，不转换请求体）。`--master-port` 与私有目录（权限 `0700`，实例停止后删除）避免同一主机上的实例争用 TGI 默认的 torch distributed 端口 `29500` 与 shard 套接字 `/tmp/text-generation-server-<rank>`。`CUDA_VISIBLE_DEVICES`、环境变量、`GET /health` 就绪检查、停止（TGI 把 router 与每个 shard 各放在自己的进程组中，它们一并结束）与对加速器清单的要求同 vLLM 进程启动器。
- 进程启动器的实例记录（`forgemesh.controlplane.deployment.process_records`）：控制平面直接在主机上运行、被 `SIGKILL`（或因崩溃、OOM 而结束）时，实例进程在自己的会话中，不会随之结束、继续占用 GPU。`vllm-process` / `tgi-process` 因此把每个实例（实例 id、实例键、pid、`/proc/<pid>/stat` 中的启动时间、私有目录，连同本机的 boot id 与 PID 命名空间）记在数据库旁的 JSON 文件中（`record_file_path`：`forgemesh.db` → `forgemesh.instances.json`），实例停止之后删去记录；下次启动时 `start()` 在恢复部署之前像 `stop` 一样结束其中仍在运行的实例，再删除它们的私有目录与记录。只向 pid 与启动时间都与记录一致的进程（及其会话中的进程）发送信号；另一次开机或另一个 PID 命名空间的记录被忽略；实例进程本身已经退出时不再寻找它会话中剩下的进程，调用 `setsid` 离开了会话的进程也找不到；结束不了的实例保留记录，下次启动时再试。只在 Linux 上记录，数据库为 `:memory:` 时不记录；无法写入记录时启动实例失败（进程随即停止）。没有使用 `PR_SET_PDEATHSIG`：它只作用于实例进程本身，不作用于 TGI 派生的 router 与 shard，而且只能在 `preexec_fn` 中设置，控制平面有线程时这并不安全。
- 启动器配置 `forgemesh.controlplane.deployment.launcher_from_env`：`FORGEMESH_RUNTIME_LAUNCHER`（`fake` / `vllm-process` / `tgi-process`，不区分大小写；未设置或为空时不部署）必须与 `FORGEMESH_INVENTORY_FILE` 一起设置。`fake` 读取 `FORGEMESH_FAKE_LAUNCHER_BIND_HOST`（实例监听的 IP 地址，默认 `127.0.0.1`）与 `FORGEMESH_FAKE_LAUNCHER_ADVERTISE_HOST`（写进实例地址的主机，默认同监听地址，监听全部地址时必须设置）；`vllm-process` 读取 `FORGEMESH_VLLM_COMMAND`（按 shell 规则拆分、不经过 shell，默认 `<控制平面的解释器> -m vllm.entrypoints.openai.api_server`，此时该解释器须装有 `vllm`）、`FORGEMESH_VLLM_BIND_HOST` / `FORGEMESH_VLLM_ADVERTISE_HOST`（同上）、`FORGEMESH_VLLM_READY_TIMEOUT_SECONDS`（默认 `1200`）与 `FORGEMESH_VLLM_STOP_GRACE_SECONDS`（默认 `10`）；`tgi-process` 读取 `FORGEMESH_TGI_COMMAND`（拆分规则同上，默认 `text-generation-launcher`，须能在 `PATH` 中找到）、`FORGEMESH_TGI_BIND_HOST` / `FORGEMESH_TGI_ADVERTISE_HOST`、`FORGEMESH_TGI_READY_TIMEOUT_SECONDS`（默认 `1200`）与 `FORGEMESH_TGI_STOP_GRACE_SECONDS`（默认 `10`），含义同 `vllm-process`；未选中的启动器的变量不读取。参数 `record_file` 为进程启动器的实例记录文件（应用传入数据库旁的 `forgemesh.instances.json`，见上）。`instance_check_interval_from_env` 读取控制器核对实例的间隔 `FORGEMESH_INSTANCE_CHECK_INTERVAL_SECONDS`（默认 `10` 秒，不论是否选择了启动器都校验）。取值未知、缺少清单或配置非法（如监听地址不是 IP 地址、命令找不到、清单不止一个节点、秒数不是有限的正数）时抛出 `LauncherConfigError`，控制平面在打开数据库之前拒绝启动。变量见 README「运行控制平面」，用法见 README「部署推理服务」。
- 控制平面部署推理服务：设置了 `FORGEMESH_RUNTIME_LAUNCHER` 时，应用以 `LocalWorkflowRuntime` 与所选启动器组装部署工作流，连同 `DeploymentReleaser` 注入协调器：服务从 `Pending` 推进到 `Ready` 并写入 `status.endpoints`，网关随即可以转发；`spec` 变化时重新部署（见 Changed）；删除时取消部署、停止实例、释放分配。工作流运行存放在数据库旁的独立文件中（`workflow_db_path`，如 `forgemesh.workflows.db`）。启动时先调用启动器的 `start()`（进程启动器在这里结束控制平面被杀死时留下的实例进程，见上），再以 `recover()` 让未结束的部署运行继续，然后核对已部署的实例（见下一条），最后才启动控制器；控制器运行中定期核对实例，并回收孤立的分配（见下文）。关闭时依次停止控制器、工作流运行时、启动器（其实例随之停止）。`POST /api/v1/workflows` 不使用这个运行时，仍返回 `503`。
- 核对已部署的实例 `check_instances`（`forgemesh.controlplane.deployment.recovery`）：经启动器的 `is_running` 核对 `Ready` / `Degraded` 服务的实例，只问实例是否在运行，不探测它能否服务请求。实例已不在运行（endpoint 没有 `instanceId` 时无从确认，同样算作不在运行）的 `Ready` 服务转为 `Degraded`：这些实例的 endpoint 标记为 `ready: false`，`Ready=False`（`InstanceLost`），网关不再向它转发（缓存解析结果的几秒内除外）；实例全部又在运行的 `Degraded` 服务交给协调器转回 `Ready`。写入以读到的版本做 CAS，不修改 `observedGeneration`。应用在启动控制器之前调用一次：三种启动器在重启之后都不认识重启之前的实例，重启之前 `Ready` 的服务都转为 `Degraded`，消息说明实例没能活过控制平面重启；之后控制器每 `FORGEMESH_INSTANCE_CHECK_INTERVAL_SECONDS` 秒（默认 `10`）调用一次，实例退出之后的下一次核对发现它。
- 实例丢失后重新部署：`spec` 没有变化的 `Degraded` 服务由协调器恢复，phase 不变，不重新评估策略（这一代 `spec` 的部署已经授权）。这一代 `spec` 最近一次部署的运行成功、实例在运行时回到 `Ready`；否则停止之前的部署，取得分配（通常复用资源持有的分配），以同一代 `spec` 再启动一次部署运行，清空 `status.endpoints`，`Ready=False`（`InstanceLost`，消息给出运行与次数），运行成功后以新实例回到 `Ready`。重新部署按资源退避：同一代 `spec` 的第一次立即开始，之后与上一次开始至少相隔 10 秒，每次翻倍，最多 300 秒，上一次开始已过 600 秒时从头开始（`REDEPLOY_BACKOFF_INITIAL_SECONDS` / `REDEPLOY_BACKOFF_MAX_SECONDS` / `REDEPLOY_BACKOFF_RESET_SECONDS`）；等待期间上一次的运行失败时 `Ready=False`（`WorkflowFailed`）说明原因；退避只记在内存中，控制平面重启之后从头开始。运行成功之后、写入 `Ready` 之前实例就已不在运行时（如控制平面恰好在这时重启）同样重新部署。重启时仍在部署的服务继续部署：实例尚未启动的照常完成；实例已经启动、还在验证的，运行在验证实例时失败，服务进入 `Failed`（`WorkflowFailed`）。
- 回收孤立的分配 `reclaim_orphaned_allocations`（`forgemesh.controlplane.deployment.recovery`）：经 `request_delete` 删除的资源在物理删除之前释放分配；绕过受控删除的 `delete()` 或旧版本可能留下归属资源（同租户、同 uid）已不存在的 Active 分配，它的设备再也分不出去。回收逐个先经 `DeploymentReleaser.release_uid` 取消这个 uid 的部署运行、停止实例，再把分配标记为 `Expired`，设备随之空闲；停止失败时保留分配，下次回收时重试。旧库中以 `{kind}/{name}` 记录的分配不对应任何资源的 uid，同名资源存在时也被回收（这类分配不持有设备，也不会被现在的部署使用）。部署推理服务时控制器在启动时与每次全量同步之后回收。回收不用租约（`lease_until`）：控制器是唯一的写者（ADR-013），没有需要别的进程接管的租约，续约只会增加写入，而控制平面停机较久之后租约到期的却是仍在服务的部署。资源仍存在的分配不在回收之列（`Failed` 服务的分配由协调器在停止实例之后释放，见 Changed）。Registry 新增 `list_orphaned_allocations()`（跨租户，含 `devices`）与 `expire_orphaned_allocation(allocation_id)`（再次确认归属资源不存在与标记在同一条 `UPDATE` 中）。
- `deploy/docker/docker-compose.fake-runtime.yaml`：叠加在 `docker-compose.yaml` 之上，以一块虚构设备的加速器清单 `deploy/docker/fake-runtime-inventory.json` 与假启动器打开推理服务的完整闭环，不需要 GPU；实例在控制平面容器内监听全部地址，写进实例地址的主机为服务名 `control-plane`，实例端口不发布到宿主机；控制器每秒核对一次实例（`FORGEMESH_INSTANCE_CHECK_INTERVAL_SECONDS: "1"`），冒烟据此演练实例丢失。
- 新增 `tests/e2e/test_closed_loop.py`：以控制平面应用、一块设备的加速器清单与假启动器覆盖创建 → `Ready` → 向实例发出流式对话 → 删除（实例停止、分配释放）→ 重新创建 → 重启后经 `Degraded`（`InstanceLost`）以新实例重新部署、回到 `Ready` 并再次对话 → 删除的闭环。
- [docs/operations/backup-restore.md](docs/operations/backup-restore.md) 覆盖工作流存储：部署推理服务时控制平面有两个数据库文件，备份与恢复脚本、Python 示例与容器内的命令都按同一组处理两个文件（先备份注册表、再备份工作流存储，同一组恢复，见 2.6 节）；新增 3.5 节说明恢复之后的推理服务（`Ready` 的经 `Degraded` 重新部署；进程启动器的实例记录文件不需要备份）。
- 插件签名工具 `python -m forgemesh.plugin.signtool`：`keygen --private-key <路径>` 生成 Ed25519 密钥对（私钥文件权限 `0600`，已有文件不覆盖），输出 key id 与可以直接追加到插件策略文件 `trustedKeys` 的公钥条目；`sign --private-key <私钥> --manifest <清单> --artifact <制品>` 写出签名文档（默认 `<制品>.sig`）；`verify --public-key <公钥> --manifest <清单> --artifact <制品>` 校验失败时以退出码 `1` 结束。签名文档是随制品分发的 JSON（`algorithm` / `keyId` / `artifactDigest` / `manifestDigest` / `signature`），私钥只由插件作者保管。
- 插件策略文件（JSON，由管理员维护）：`trustedKeys` 列出受信任的 Ed25519 签名公钥（PEM），`grants` 按插件的 `metadata.id` 授予权限，对该插件的所有版本生效。`load_plugin_policy` 严格校验：文件不可读、未知字段、非法或重复的公钥、带版本号的 `pluginId`、同一插件的多条授予、词表之外或重复的权限都抛出 `PluginPolicyError`；`PluginManager.from_policy_file` 按策略文件构造插件管理器，`PluginManager.effective_permissions` 返回插件实际拥有的权限。控制平面应用目前不构造插件管理器，因此没有对应的环境变量。
- 插件权限词表 `forgemesh.plugin.permissions.Permission`（15 项，形如 `<域>.<动作>`，如 `resource.read`、`network.egress`、`platform.admin`）：清单的 `spec.permissions`、JSON Schema 的权限枚举、管理员的授予与 `forgemesh.plugin.isolation` 共用这一份词表，契约测试保证 Schema 枚举与词表逐项一致。
- `forgemesh.plugin.loader.load_entrypoint(plugin_dir, entrypoint)`：按清单 `spec.entrypoint`（`<模块>:<类>`）把插件目录作为 Python 包加载并返回入口类；插件目录必须含 `__init__.py`，入口类必须定义在插件包内。加载在当前进程内执行插件代码，不提供任何隔离。
- 推理服务更新接口：`PUT /api/v1/inference-services/{name}` 整体替换 `spec`、`metadata.labels` 与 `metadata.annotations`，请求体是完整的资源文档（`GET` 的响应修改后可以直接提交，省略 labels / annotations 即清空）；`PATCH` 同一路径接受 JSON Merge Patch（RFC 7386，`Content-Type: application/merge-patch+json`，其他媒体类型返回 `415 UNSUPPORTED_MEDIA_TYPE` 并带 `Accept-Patch`），补丁合并到当前资源的 JSON 表示之后按与 `PUT` 相同的规则处理。两者要求 `write` scope，与 `GET` / `DELETE` 一样以 `namespace` 查询参数指定命名空间。`kind`、`apiVersion`、`metadata.name`、`metadata.namespace` 与租户不可修改（给出时必须与当前资源一致，否则 `400`），`status`、`conditions`、`finalizers`、`generation`、`observedGeneration` 被忽略；只有 `spec` 变化时 `generation` 递增，没有任何变化的更新不写入。乐观并发：`metadata.resourceVersion`、`metadata.uid` 与 `If-Match` 是前置条件，不满足时返回 `409 CONFLICT`，`details.reason` 为 `ResourceVersionMismatch` 或 `UidMismatch`（`If-Match` 不匹配同样是 `409`，而不是 `412`）；处于 `Terminating` 的资源拒绝更新（`409`，`details.reason` 为 `Terminating`）。读取与写入之间资源被并发修改（如控制器写入 `status`）时，服务端重新读取、重新检查前置条件后再写，最多尝试 5 次，不会覆盖并发写入的 `status`。单个资源的响应（`GET` / `POST` / `PUT` / `PATCH`）带强 `ETag`，随 `resourceVersion` 变化，同名资源删除后重建也不会得到旧值。
- Registry 期望状态写入 `SQLiteRegistry.update_spec(kind, namespace, name, tenant_id, spec, *, labels, annotations, expected_version=None, expected_uid=None)`：整体替换 `spec`、labels 与 annotations，只支持登记了强类型模型的 kind（目前只有 `InferenceService`），结果须满足该模型；status、条件、finalizer、uid 与 `observedGeneration` 原样保留。`spec` 按模型规范化后比较，变化时 `generation` 加 1；有变化时 `resource_version` 加 1 并追加 `updated` 变更记录（控制器据此重新协调），没有变化时不写库。检查与写入在同一个 `BEGIN IMMEDIATE` 事务中：资源处于 `Terminating`（先于其他检查）、uid 或版本与预期不同时抛出 `ConflictError`，`details["reason"]` 为 `ConflictReason` 的取值（`Terminating` / `UidMismatch` / `ResourceVersionMismatch`）。
- 列表分页：`GET /api/v1/inference-services` 与 `GET /api/v1/resources` 接受 `pageToken`，还有更多结果时响应带 `nextPageToken`（最后一页不带），`count` 为本页条数；结果按 `(namespace, name)` 升序，不带命名空间的资源在最前。游标不透明，绑定签发它的租户与过滤条件（`namespace`，`/resources` 还有 `kind`），`limit` 可以逐页变化；格式不对、被修改或用于其他租户 / 过滤条件的游标返回 `400`，空的 `pageToken` 等同于省略。Registry 新增 `list_page(kind, tenant_id, namespace=None, labels=None, page_token=None, limit=100)`，返回 `ResourcePage`（`items` / `next_page_token`），无效的游标抛出 `InvalidPageTokenError`（`ValueError` 的子类）。游标形如 `<载荷>.<签名>`，签名是覆盖载荷与租户、kind、`namespace`、标签选择器的 HMAC-SHA256，密钥在首次打开数据库时随机生成、保存在新表 `registry_meta` 中，控制平面重启后游标仍然有效。游标记录位置而不是快照：翻页期间新建或删除的资源可能出现也可能缺席，全程存在的资源恰好出现一次。
- 决策风险等级（改进计划 5.10）：`POST /api/v1/decisions` 的风险等级由服务端按 `question` 推导（`forgemesh.controlplane.api.risk`）。`question` 是形如 `<域>.<动作>` 的操作标识，只看第一个 `.` 之前的域（忽略大小写与首尾空白）：`driver`、`node`、`mig`、`gpu`、`device`、`plugin`、`model`、`quota`、`tenant`、`policy`、`system`、`platform` 为 `high`（architecture.md 14.2 节列出的高风险操作所在的域），`inference` 为 `low`，其他（含自由文本）为 `medium`；请求中的 `riskLevel` 只能调高，低于推导值时被忽略。实际采用的等级在响应的 `riskLevel` 中返回（`503 DECISION_NO_CANDIDATE` / `DECISION_FALLBACK_REQUIRED` 的错误信封在 `details.riskLevel` 中）。高风险请求的候选来自备用 Provider 时返回 `503 DECISION_FALLBACK_REQUIRED`，候选在 `details.candidates` 中供人工审批，不自动采用。
- 新增 `tests/contract/test_openapi_contract.py`（随 pytest 在 CI 中运行）：双向核对应用提供的路由与 `api/openapi/openapi.yaml`（探针除外），路径与方法、每个操作的参数（位置、名称、是否必填、是否弃用，查询与路径参数还有类型）与请求体的媒体类型都必须一致；再逐一调用每个操作，响应的状态码必须列在契约中，响应体与响应头符合契约，错误信封回显关联 ID，契约列出的每个状态码都至少被调用到一次；不对应任何操作的路径返回契约中的 `RouteNotFound`。只使用已有的开发依赖 `jsonschema` 与 PyYAML。
- 命令行工具 `forgemesh` 改为 REST API 的客户端（改进计划 5.11，用法见 README「命令行工具」）：`apply -f`（资源不存在就创建，存在就以读到的 `resourceVersion` 与 `uid` 为前置条件整体替换，遇到并发修改时重新读取再试，最多 5 次；同一份文档再次提交不产生写入，输出 `created` / `configured` / `unchanged`；接受多文档 YAML、JSON 对象或数组与标准输入 `-`，`-f` 可以重复）、`get`（`-o yaml|json`）、`describe`、`list`（跟随 `nextPageToken` 读取全部页，`-o table|yaml|json`，`--page-size`）、`delete`（受控删除，区分 `204` 已删除与 `202` 进入 `Terminating`），以及新命令 `wait`（`--for ready` 等到服务以当前的 `spec` 就绪，即 `Ready`、`Ready=True` 且 `observedGeneration` 等于 `generation`，经过 `Updating` 与重新部署中的 `Degraded` 继续等待，资源正在删除或在当前的 `spec` 下进入 `Failed` 时立即失败，被拦下的更新在超时消息中指明更新没有开始；`--for delete` 等到资源消失；`--timeout` 默认 300 秒）。默认经网关访问 `http://127.0.0.1:8081`，`--api-url` / `FORGEMESH_API_URL` 指向其他地址（如控制平面 `http://127.0.0.1:8080`）；API key 取自 `--api-key-file` 指向的文件或 `FORGEMESH_API_KEY`，只放在 `Authorization` 请求头中，不出现在任何输出中；`--tenant` 作为 `X-Tenant-Id` 发送。错误信封（含网关自己的错误）与不是错误信封的错误响应都转换为带关联 ID 的错误消息，退出码按原因区分（`0`–`12` 与 `130`，`forgemesh --help` 列出）。HTTP 只用标准库 `urllib`，不跟随重定向。
- 新增 `tests/e2e/test_cli.py`：在后台线程中以 uvicorn 运行控制平面应用（API key 认证、一块设备的加速器清单与假启动器），CLI 经真实的 HTTP 连接走完 `apply` → `wait` 就绪 → 再次 `apply`（`unchanged` / `configured`）→ 修改 `spec` 后再次 `apply` 与 `wait`（以新的实例部署第 2 代 `spec`）→ `get` → 逐页 `list` → `delete`（`204` / `202`）→ `wait --for delete`，并核对 `401` / `403` / `400` / `404` 的退出码与消息，key 不出现在任何输出中。
- 资源变更事件（事务性 Outbox，ADR-008）：`SQLiteRegistry` 对资源对象的每次写入（创建、更新（含 `PUT` / `PATCH` 所用的 `update_spec`）、status 写入、请求删除、删除）在写入资源的同一个事务中，经共用连接的 Outbox 写入一条领域事件，任何一步失败都一起回滚；分配记录（`allocations`）的写入（含回收孤立的分配）不产生事件。信封符合 `api/schemas/event.schema.json`（`forgemesh.controlplane.events`）：`eventId` 即 Outbox 事件 id；`eventType` 为 `<kind>.<change>`，kind 按单词拆成小写并以点连接（`inference.service.created` / `.updated` / `.status_updated` / `.terminating` / `.deleted`）；`data` 含资源身份、`phase` 与写入之后的资源表示（`deleted` 为删除前最后的状态）；同一资源（`resourceUid`）的事件以 `resourceVersion` 判断新旧。API 写入使用请求的 `X-Correlation-Id`（缺省时生成），控制器同一轮协调的 status 写入与移除 finalizer 共用一个 correlation id；核对实例时把服务转为 `Degraded` 的 status 写入同样产生事件，它不属于哪一轮协调，correlation id 是生成的。
- 控制平面在应用启动（lifespan）时运行 Outbox 的 `Dispatcher`（每 0.5 秒查询一次到期事件）发布资源变更事件：至少投递一次，接收方按 `eventId` 去重，顺序只是尽力按写入顺序。发布器由 `forgemesh.controlplane.publishers.publisher_from_env` 按环境变量选择：默认 `LogPublisher`，每个事件写一行 JSON 日志（logger `forgemesh.events`；进程没有配置 logging 时写标准输出，以 `forgemesh-control-plane` 运行时随控制平面日志写到标准错误，见 Changed「控制平面日志」）；设置 `FORGEMESH_EVENT_WEBHOOK_URL`（`http` / `https`）后为 `WebhookPublisher`，把信封以 JSON `POST` 到该地址，请求头带 `X-Forgemesh-Event-Id`，设置了 `FORGEMESH_EVENT_WEBHOOK_TOKEN` 时带 `Authorization: Bearer <token>`；2xx 视为送达，其余状态码（不跟随重定向）、超时与连接错误视为失败，每次发送的超时为 `FORGEMESH_EVENT_WEBHOOK_TIMEOUT_SECONDS`（默认 `5`，大于 0、不超过 `60`），遵循 `HTTPS_PROXY` 等代理环境变量。取值非法（如地址含用户信息或片段、token 含空白或非 ASCII 字符）或没有设置地址却设置了 token / 超时时抛出 `EventConfigError`，控制平面在打开数据库之前拒绝启动。Dispatcher 在恢复部署与启动时核对实例之前启动，关闭时在控制器（含定期核对实例与回收孤立的分配）、工作流运行时与启动器之后、关闭数据库之前停止，仍待发布的事件在下次启动后发布。`create_app` 新增参数 `event_publisher`，供嵌入方注入发布器。变量见 README「运行控制平面」。
- Outbox 失败重试与 `dead`：发布失败按指数退避重试（`RetryPolicy`：1 秒起翻倍、最长 300 秒，默认最多 10 次，前后约 8.5 分钟），失败次数 `attempts`、最近一次错误 `last_error`（异常类型与消息）与下次发布时间 `next_attempt_at` 记在数据库中，重启后继续；达到上限后事件进入 `dead` 并记一条 ERROR 日志，`list_dead()` 列出，`redrive()` 放回待发布队列（命令见 [docs/operations/backup-restore.md](docs/operations/backup-restore.md) 第 5 节）。`Dispatcher` 只按数据库状态取到期的事件，一轮中遇到失败即结束本轮（目标不可用时不会对每个事件各等一次超时），已发布的事件保留 7 天后删除；新增 `start()` / `shutdown()` 以后台任务运行，循环中的数据库错误不会使它退出。另新增 `Outbox.get()`（按 id 查询任意状态的事件）、`due()` 与 `purge_published()`。
- 审计记录：经 API 创建、更新与删除资源（`POST /api/v1/inference-services`，`PUT` / `PATCH` / `DELETE /api/v1/inference-services/{name}`）时，在写入资源与事件的同一个事务中写入一条审计记录，含租户、主体（`actor`）、API key id（`key_id`，`dev` 模式为空）、correlation id（与事件相同）、命名空间、`action`（`create` / `update` / `delete`）与 `outcome`（`success`），`details` 记录变更类型、资源 uid、`resourceVersion` 与对应事件的 `eventId`。被拒绝或失败的请求、没有任何变化的更新、对已在 `Terminating` 的资源重复的删除请求以及控制器与协调器的写入（含核对实例时的 status 写入）不产生审计记录。`SQLiteRegistry.put` / `delete` / `request_delete` 新增仅限关键字参数 `principal`（凭证的租户与资源不一致时在写入任何数据之前拒绝），`update_status` 新增 `correlation_id`，`update_spec`（见上）同样接受 `correlation_id` 与 `principal`，`audit_log` 属性返回共用连接的审计日志；`Principal` 新增 `key_id`；`AuditEvent` 新增 `key_id` / `namespace`（打开旧库时自动补列）；`AuditLog.put()` 在调用方的事务中写入，不自行提交。
- 审计查询 `GET /api/v1/audit-events`：返回凭证所属租户的审计记录，最新的在前，需要 `read` scope；参数 `limit`（1–1000，默认 100，与列表接口一致）、`pageToken` 与可选的 `correlationId`，响应为 `{items, count, nextPageToken}`，还有下一页时才有 `nextPageToken`。按位置（时间戳、记录 id）分页（`AuditLog.list_page`），翻页期间写入的新记录不会使后面的页重复或遗漏。游标与资源列表的游标一样签名，使用同一个 `registry_meta` 密钥（签发与校验移到 `forgemesh.controlplane.pagetokens`，资源列表的游标格式不变），绑定租户与 `correlationId`：格式不对、被修改、为其他租户或条件签发或取自资源列表的游标返回 `400`，控制平面重启后仍然有效。OpenAPI 新增该操作与 `AuditEvent` / `AuditEventList`，契约测试覆盖该操作。Go 网关转发这个路径，与其他资源路由相同：校验 API key、按已验证租户限流、透传关联 ID 与查询串，`read` scope 由控制平面检查。
- `forgemesh.controlplane.sqlite.Database`：一个 SQLite 连接加一把可重入锁。Registry、Outbox 与审计日志共用一个实例，资源写入与它的事件、审计记录在同一个事务中提交或回滚（ADR-008 的工作单元）：`transaction()` 以 `BEGIN IMMEDIATE` 开始，`call()` / `run()` 在函数抛出异常时回滚它未提交的写入，并拒绝留下未结束的事务。`TaskStore`、`AuditLog` 与 `IdempotentConsumer` 都可以经 `database=` 共用连接。
- 控制平面 `GET /metrics`：以 Prometheus 文本格式 0.0.4 输出聚合指标，不引入新依赖（`forgemesh.controlplane.metrics`：计数器、仪表盘与直方图，每个指标最多 200 个标签组合，超出的计入标签全为 `overflow` 的一个序列）。指标定义在 `forgemesh.controlplane.telemetry`：`forgemesh_http_requests_total{method,route,status}` 与 `forgemesh_http_request_duration_seconds{method,route}`（`route` 为路由模板，未匹配路由的请求记为 `unmatched`）、`forgemesh_reconcile_total{operation,outcome}` 与 `forgemesh_reconcile_duration_seconds{operation}`、`forgemesh_controller_retries_total`、`forgemesh_controller_queue_depth{state}`、`forgemesh_deployments_total{outcome,reason}`（推理服务进入 `Ready` / `Failed` 的次数）、`forgemesh_inference_services{phase}`、`forgemesh_allocations_active{accelerator_type}`、`forgemesh_accelerator_devices{accelerator_type,state}`、`forgemesh_workflow_runs{status}` 与 `forgemesh_database_lock_wait_seconds`（控制平面数据库的操作等待连接锁的时间，注册中心、Outbox 与审计记录共用这个连接与锁），状态类指标在抓取时读取。`/metrics` 与探针一样不要求认证，只输出跨租户聚合的数据，标签不含租户、名称、命名空间、uid 或地址；它不在 `/api/v1` 之下，网关不转发。OpenAPI 新增 `/metrics`（`security: []`）。为此 `ControllerManager` 新增可选参数 `metrics` 与 `queue_depth()`（工作队列新增 `depth()`），`SQLiteRegistry` 新增 `count_by_phase`、`active_allocation_devices` 与 `database` 属性，`Database` 新增 `set_lock_wait_observer`，`WorkflowStore` 新增 `count_runs_by_status`。指标说明见 README「可观测性」。
- 控制平面的 Outbox 指标：`forgemesh_outbox_events{state}`（抓取时读取的未发布事件数，`state` 为 `pending`（尚未发布失败过）/ `retrying`（发布失败过、等待重试）/ `dead`）与 `forgemesh_outbox_publish_attempts_total{outcome}`（Dispatcher 发布事件的尝试次数，在结果写入数据库之后计数，`outcome` 为 `published` / `failed`（将重试）/ `dead`（最后一次失败），三个取值都从 0 开始输出）。为此 `Outbox` 新增 `backlog()`（返回 `OutboxBacklog`，只读取 `pending` 与 `dead` 的行），`Dispatcher` 新增仅限关键字参数 `on_attempt`（每次发布尝试的结果写入数据库之后以 `PublishOutcome` 调用，未能写入时不调用）。
- 控制平面核对实例、重新部署与回收孤立分配的指标：`forgemesh_instance_checks_total{outcome}`（控制器定期核对实例的次数，`completed` / `failed`；启动时的核对不计入，它失败时控制平面不会启动）、`forgemesh_instance_check_results_total{check,result}`（核对实例得出结论的服务数，`check` 为 `startup` / `periodic`，`result` 为 `lost`（服务转为 `Degraded`）/ `back`（`Degraded` 服务的实例全部又在运行）/ `conflict`（服务在核对期间被修改或删除，没有报告））、`forgemesh_lost_instance_redeploys_total`（为实例已不在运行的服务启动重新部署运行的次数，退避期间推迟的协调不计入）与 `forgemesh_orphaned_allocation_reclaims_total{outcome}`（`reclaimed` / `failed`：没能停止实例，分配保留，下一次全量同步之后重试）；每个标签组合都从 0 开始输出，未设置启动器时保持为 0。为此 `check_instances` 与 `reclaim_orphaned_allocations` 新增仅限关键字参数 `on_result`（以每个服务的 `InstanceCheckResult` / 每个分配的 `ReclaimOutcome` 调用），`ReconciliationController` 新增参数 `on_redeploy`（启动重新部署运行之后调用）。
- 网关 `GET /metrics`（`internal/metrics`，只用标准库，文本格式与标签上限同上）：`forgemesh_gateway_http_requests_total{route,status}` 与 `forgemesh_gateway_http_request_duration_seconds{route}`（`route` 为按路由表确定的路由类别 `probe` / `metrics` / `control_plane` / `inference` / `unmatched`，不取请求路径；含 Recovery 兜底的 `500`）、`forgemesh_gateway_stream_first_byte_seconds`（流式代理从开始处理请求到写出实例的第一个响应字节，只统计实例返回 2xx 的流）、`forgemesh_gateway_upstream_errors_total{upstream,kind}`（`control_plane` / `instance`；`connect` / `status`（5xx）/ `body` / `address_not_allowed` / `invalid_endpoint`，客户端已断开的请求不计）、`forgemesh_gateway_rate_limited_total`、`forgemesh_gateway_resolver_lookups_total{result}`（`hit` / `miss` / `coalesced`）与 `forgemesh_gateway_instances{state}`（Resolver 缓存中 `healthy` / `unhealthy` 的实例数）。与探针一样不经过认证与限流，只输出聚合数据，见网关 README「可观测性」。
- W3C Trace Context：网关的 `TraceContext` 中间件沿用请求中恰好一个合法的 `traceparent`（连同 `tracestate`），否则（缺失、不合法或多个）生成新的 `traceparent`（版本 `00`，flags `00`）并丢弃 `tracestate`；确定的 `traceparent` 随请求转发给控制平面（资源反代与 Resolver 的查询）与推理实例，响应不回写。控制平面的 `RequestContextMiddleware` 按同样的规则确定 trace-id，沿用 `X-Correlation-Id`（缺失时生成一次并写回请求头，路由回显的、审计记录与资源变更事件中的关联 ID 因此与日志一致），在请求处理期间绑定两者，这个请求的日志与错误信封都带上它们。两边都不记录 span、不导出追踪，OTLP 导出留待后续。错误信封新增可选字段 `traceId`（OpenAPI `Error.traceId`，32 个小写十六进制字符），网关与控制平面一致。
- 网关访问日志：每个 `/api/v1` 请求与没有匹配路由的请求输出一行 `access method=… path="…" status=… duration=… corr=<关联 ID> trace=<trace-id>`（路径加引号转义、不含查询串；流式请求的 `duration` 是整个流的时长；处理器在写出响应头之前 panic 时记 `500`），位于认证与限流之前，被拒绝的请求同样记录；探针与 `/metrics` 不记录。反代与流式代理的错误日志、Recovery 的 panic 日志同样带 `trace=`。
- `scripts/compose_smoke.sh` 在第一次对话之后不带凭证抓取一次控制平面与网关的 `/metrics`，核对 Content-Type、若干指标及其此时确定的取值，以及输出不含租户、服务名、key 与实例 id；并带 `traceparent` 经网关读取不存在的服务，核对控制平面的 `404` 信封与日志、网关的访问日志都带同一个 trace-id。实例丢失后重新部署之后与重启控制平面之后再各抓取一次控制平面的 `/metrics`：定期核对与启动时的核对各发现一次丢失，各重新部署一次（重启之后计数从 0 开始），定期核对没有失败，没有回收分配。
- CI 新增 `package-imports` job 与 `scripts/check_package_imports.py`：为 `packages/` 下的每个包新建一个虚拟环境，只安装这个包（含它的全部 extras）与它声明的依赖（`forgemesh-*` 依赖用本仓库的源码构建），以隔离模式导入根模块，再用 `pkgutil.walk_packages` 导入全部子模块，任一模块导入失败（例如漏声明依赖）即失败。此前 `python` job 把全部包装进同一个环境，漏声明的依赖会被兄弟包或开发依赖掩盖。本地运行 `.venv/bin/python scripts/check_package_imports.py [包目录名...]`（需要网络）。
- CI 新增 `dependency-audit` job：`pip-audit` 2.10.1 审计 `make install` 装进 `.venv` 的全部第三方包（以 editable 方式安装的本仓库 `forgemesh-*` 包不审计），`govulncheck` v1.9.0 检查网关代码实际调用到的已知漏洞（含标准库）；本地为 `make audit` 与 `make -C gateway vulncheck`。尚无修复或经评估不适用的 Python 漏洞以 `--ignore-vuln` 记入根 `Makefile` 的 `PIP_AUDIT_IGNORE`（目前为空），govulncheck 没有忽略机制；处理规则见 `SECURITY.md` 3.1（同步改写，删去「依赖版本在 CI 中固定」这一并不属实的说法）。
- pytest 总覆盖率下限：根 `pyproject.toml` 的 `[tool.coverage.report]` 设 `fail_under = 97`（引入时实测 97.20%）与 `precision = 2`（默认取整到整数，96.5% 会被当作 97% 而通过）；CI `python` job 改以 `pytest --cov` 运行，本地为 `make coverage`。下限只升不降。
- CI `python` job 新增格式检查 `ruff format --check .`，`make lint` 同样执行：全仓库以 ruff 0.17 格式化一次（只改格式；Markdown 中的代码块不格式化，见根 `pyproject.toml`），`requirements-dev.txt` 把 ruff 限定在 0.17 系列，因为格式风格与规则集随 ruff 的 minor 版本变化。提交之前运行 `make fmt`。
- Kubernetes 网关清单 `deploy/kubernetes/gateway-deployment.yaml`（ServiceAccount、Deployment 与 Service `forgemesh-gateway`，8081 端口）：与控制平面共用 Secret `forgemesh-api-keys`，经 Service `forgemesh-control-plane` 访问控制平面；探针为 `/readyz`（控制平面就绪之后才通过）与 `/healthz`；安全上下文与控制平面一致（非 root、只读根文件系统、去掉全部 capability、`RuntimeDefault` seccomp）；宽限期 35 秒，长于网关排空在途请求的 30 秒。启用启动器时使用 `deploy/kubernetes/launcher/`：`network-policy.yaml` 允许网关访问控制平面 Pod 上推理实例的临时端口（32768–60999），`fake-runtime-patch.yaml` 为控制平面加上来自 ConfigMap `forgemesh-inventory` 的加速器清单与 `fake` 启动器（实例监听全部地址，写进 `status.endpoints` 的主机是 Pod IP；与 compose 的叠加文件一样每秒核对一次实例）。用法见 README「Kubernetes 部署」，包括 `vllm-process` / `tgi-process` 在 Kubernetes 上的前提（含多卡时的 `/dev/shm`）、实例记录文件在数据卷上，以及单节点限制。
- Kubernetes 可选的网络策略 `deploy/kubernetes/monitoring/network-policy.yaml`：允许 Prometheus 的 Pod（默认为命名空间 `monitoring` 中带有 `app.kubernetes.io/name: prometheus` 标签的 Pod，按部署修改）访问控制平面的 8080 与网关的 8081 端口，抓取两边的 `/metrics`。默认的策略下从 Pod 网络抓取不到控制平面的 `/metrics`；它与网关的 `/metrics` 一样不要求认证，只输出跨租户聚合的指标。说明与抓取配置示例见 README「Kubernetes 部署」的「指标抓取」。
- Kubernetes 上把资源变更事件发往 webhook：`deploy/kubernetes/event-webhook/patch.yaml` 为控制平面设置 `FORGEMESH_EVENT_WEBHOOK_URL`、`FORGEMESH_EVENT_WEBHOOK_TOKEN`（取自 Secret `forgemesh-event-webhook` 的键 `token`，Secret 不存在时不设置）与 `FORGEMESH_EVENT_WEBHOOK_TIMEOUT_SECONDS`；`network-policy.yaml` 允许控制平面出站访问接收方（集群内按命名空间与 Pod 标签，集群外按 IP 地址段，按部署修改）。默认的策略只允许控制平面访问集群 DNS，没有这条出站规则时每次发送都超时。用法见 README「Kubernetes 部署」。

### Changed

- README 策略引擎描述修正：`allow/deny/degraded` 三态改为 `allow/deny/require_approval` 三态（对齐 v1.0.1 已交付语义），并补充说明 Decision Router 的 `OK/FALLBACK_REQUIRED/FAILED` 为决策路由层状态、与 Policy 三态不混用。
- [docs/releases/v1.0.md](docs/releases/v1.0.md) 插入"勘误与现状（2026-10-11）"节：澄清 v1.0.0 发布记录中 PostgreSQL 相关表述与 ADR-013 不一致，当前权威存储为 SQLite、PostgreSQL 为目标态，文档中 PostgreSQL 备份/迁移命令不可用。
- API 稳定性标注由 `stable` 更正为 `beta`：OpenAPI `info.x-stability` 改为 `beta`，`info.version` 改为 `1.1.0`；README、SECURITY.md 与 v1.0 发布记录同步更正。v1.0.0 宣称的「首个稳定版本、major 版本内保证兼容」与实际不符：1.0.x 可运行入口的业务端点仍返回 `501`、没有认证，1.1.0 也包含多项破坏性变更。[弃用政策](docs/governance/deprecation-policy.md)新增 1.4 节：beta 阶段的 minor 版本可以包含破坏性变更，但必须以「**破坏性**」标注；改回 `stable` 之前应满足架构验收标准（architecture.md 第 20 节）。
- 包元数据事实统一：全部 10 个子包 `requires-python` 从 `>=3.9` 校准为 `>=3.12`（对齐 README/CI/Ruff 的 Python 3.12 基线）；`control-plane` 与 `workflow-runtime` 包描述去除过时的 skeleton/placeholder 表述，`cli` 与 `node-agent` 包描述同步移除 skeleton 字样。
- [.github/workflows/ci.yml](.github/workflows/ci.yml)：ADR 检查由硬编码 001–012 改为索引驱动动态校验；architecture-docs job 新增 manifest / OpenAPI 语义 / 元数据一致 / 插件冒烟四道步骤。
- 修正 `plugins/runtime-tgi`（移除 schema 未定义的 spec.capabilities）与 `plugins/workflow-temporal`（category 改为 provider 并移除 capabilities）manifest 至 schema 合规。
- **破坏性**：网关与控制平面默认使用 `apikey` 认证模式，未配置 key 文件时拒绝启动；`dev` 模式（直接信任租户请求头）只允许监听回环地址。
- **破坏性**：缺少凭证的请求由 `400`（缺少 `X-Tenant-Id`）改为 `401 UNAUTHENTICATED`，并携带 `WWW-Authenticate: Bearer realm="forgemesh"`；`X-Tenant-Id` 与凭证绑定的租户不一致返回 `403 FORBIDDEN`；读接口要求 `read` scope，写接口（含启动工作流与请求决策）要求 `write` scope。
- **破坏性**：网关的 `401` 响应体由纯文本改为统一错误信封 `{"error": {"code", "message", "correlationId"}}`（新增的 `403` 同样使用信封）；推理实例不再收到调用方的 `Authorization` / `Cookie`。
- **破坏性**：控制平面全部错误响应统一为错误信封：框架层 404 / 405 不再返回 `{"detail": ...}`，请求校验失败由 `422` 改为 `400`，未捕获异常返回不含内部细节的 `500 INTERNAL_ERROR`（细节写入日志）。客户端改为读取 `error.code` 与 `error.message`：请求校验失败的错误码为 `BAD_REQUEST`，字段与原因在 `error.details.errors` 中（每项含 `loc`、`msg` 与 `type`）。路径不匹配任何端点的 `404` 错误码为 `ROUTE_NOT_FOUND`（与网关相同，OpenAPI 的 `RouteNotFound`），资源不存在的 `404` 为 `NOT_FOUND`，`405` 为 `METHOD_NOT_ALLOWED`：命令行工具 `forgemesh` 直连控制平面而 `--api-url` 多了一段路径时以 `1` 退出并提示检查 `--api-url`，不会报告资源不存在，`wait --for delete` 也不会因此成功。
- 网关接受任意大小写的 `Bearer` scheme（如 `bearer`）。
- **破坏性**：网关消费的 `status.endpoints` 由字符串数组改为对象数组（见 Added），只使用 `ready` 且协议已知的实例；JSON Schema 中资源模型从未实现的单数 `status.endpoint` 一并移除。
- **破坏性**：网关默认不连接回环、链路本地、未指定、组播地址与已知云元数据服务地址上的推理实例（见 Security）；实例运行在本机时需设置 `GATEWAY_INSTANCE_ALLOWED_CIDRS`（`make run-dev` 已允许 `127.0.0.0/8,::1/128`）。
- 网关连接池：流式代理与控制平面客户端每个主机最多保留 64 个空闲连接（此前为 `net/http` 默认的 2 个，一波并发请求之后大部分连接被关闭，下一波重新建连）；资源反代等待控制平面响应头最多 30 秒。
- 网关 Resolver 合并同一服务的并发未命中，只查询一次控制平面，查询不随发起它的请求取消；"服务不存在 / 未就绪 / 无可用实例"缓存 1 秒（不超过 `GATEWAY_RESOLVER_TTL_SECONDS`），`403`、其他控制平面错误与网络错误仍不缓存。
- 控制平面的 SQLite 存储（Registry、去重键、任务、审计、Outbox）统一以 WAL 模式打开并启用外键约束（`forgemesh.controlplane.sqlite.connect`）。此前为回滚日志模式，而 Registry 与去重键存储共用同一个文件：一个连接上的读事务未结束时，另一个连接的提交只能等待，超过 5 秒以 `database is locked` 失败。运行期间数据库文件旁会有 `-wal` / `-shm` 文件，不能直接复制文件做备份；`synchronous` 保持 `FULL`。
- 去重键存储 `ProcessedKeyStore` 不再在每次 `seen` / `mark` 时删除过期键并提交：`seen` 不报告超过 TTL 的键，过期键最多每 `min(TTL, 60 秒)` 删除一次，启动后的第一次调用即清理上次运行留下的过期键。
- `deploy/docker/docker-compose.yaml` 改为可运行的本地拓扑：control-plane 与 gateway 从本仓库的 Dockerfile 构建，共用 compose secret 中的 key 文件 `deploy/docker/api-keys.json`（需事先生成，已加入 `.gitignore`），只发布在 `127.0.0.1`，以只读根文件系统、去掉全部 capability 运行；gateway 在控制平面 `/readyz` 通过后才启动；控制平面数据库在卷 `control-plane-data` 中；postgres 移入 `postgres` profile，默认不启动；移除过时的 `version` 键。
- Kubernetes 控制平面清单：数据库放在 PVC `forgemesh-control-plane-data`（ReadWriteOnce）上，更新策略为 `Recreate`；key 文件来自 Secret `forgemesh-api-keys`（键名 `api-keys.json`，部署前需创建）；Pod 以 uid 10001、只读根文件系统运行，不挂载 ServiceAccount token。
- pytest 把 `RuntimeWarning` 与 `PytestUnraisableExceptionWarning` 视为错误：协程未被 await 时被调用的代码实际没有执行，测试现在会失败，而不是只打印一条警告。
- README 与 CONTRIBUTING 的安装命令改为 `make install`：在仓库根目录执行 `uv sync` 只会创建一个空的虚拟环境，随后 `make test` 找不到 pytest。README 目录树列出全部 10 个插件，并补充容器部署说明。
- [docs/architecture/module-mapping.md](docs/architecture/module-mapping.md)：更正网关文件与测试数量（33 / 17）和上游变量名（`GATEWAY_UPSTREAM_URL`），部署小节与 CI 小节按当前工件更新，偏差 D2 处置后移除。
- 删除源码与测试 docstring 中「兼容 Python 3.9+」/「兼容 3.9–3.12」的说法（各子包 `requires-python` 已是 `>=3.12`，代码使用 `X | None` 与 `enum.StrEnum`，在 3.9 上无法导入）；CONTRIBUTING 的类型注解约定改为与 ruff `UP` 规则一致的 `list[X]` / `X | None`。
- **破坏性**：`ScheduleRequest` 必须提供 `resource_ref` 与 `idempotency_key`（见 Added）；`resource_kind` 改为只读属性（等同 `resource_ref.kind`），构造时传入会被拒绝。`AllocationRef.resource_id` 改为归属资源的 `uid`。
- **破坏性**：`allocations.resource_id` 改为归属资源的 `uid`（此前协调器按 `{kind}/{name}` 查找），同一租户内同一 uid 至多一个 Active 分配：部分唯一索引 `idx_allocations_one_active_per_tenant_resource`（`ON allocations(tenant_id, resource_id) WHERE status = 'Active'`）取代 `idx_allocations_one_active_per_resource`，打开旧库时自动替换，已有行保留。旧库中以 `{kind}/{name}` 记录的分配不做迁移，也不再随资源删除释放（此前仓库没有调度器实现，正常部署不会产生这类分配）；部署推理服务时控制器把它们当作孤立的分配回收（标记为 `Expired`，见 Added）。
- **破坏性**：持有 Active 分配的资源无论处于哪个 phase，`DELETE` 都返回 `202` 并先进入 `Terminating`（此前 `Pending` / `Scheduling` / `Preparing` / `Failed` / `Unknown` 直接物理删除并返回 `204`），由协调器释放分配后再物理删除；检查之后才提交的分配使这次删除失败（`409`），而不是遗留一个没有归属的分配。OpenAPI 已同步。
- **破坏性**：协调器按 phase 逐轮部署 `InferenceService`，每一轮只做一个合法的 phase 转移并写回 `status.phase`：`Pending` → `Scheduling`（策略放行；策略只在离开 `Pending` 与开始更新时评估）→ `Preparing`（分配加速器）→ `Deploying`（以同一幂等键取回分配，停止资源之前的部署，启动当前 generation 的部署工作流，输入携带分配的设备）→ `Verifying`（`launch-instance` 已成功）→ `Ready`（运行成功：写入 `status.endpoints` 与 `Ready=True`，原因 `Reconciled`）。部署过程中 `Ready=False`，原因即当前 phase；运行未结束时按幂等键前缀 `find_runs()` 查询当前 generation 的运行，每 `workflow_poll_interval` 秒一次。只有进入 `Ready` 或 `Failed` 的一轮登记去重键（实例丢失后回到 `Ready` 的一轮除外：同一代 `spec` 可能再次丢失实例）。
- **破坏性**：部署失败时资源进入 `Failed`（`Ready=False`）：运行失败、被取消或超时 → `WorkflowFailed`（消息为运行记录的失败原因，如 `step 'verify-instance' failed: …`），释放分配；补偿没能停止实例时先保留分配（消息说明实例可能仍在运行），随后 `spec` 没有变化的一轮经 `DeploymentReleaser` 停止资源的实例，再释放分配（停止失败时退避重试）；找不到运行（`WorkflowRunNotFound`，如工作流数据库被重置）或运行成功却没有报告实例时同样进入 `Failed`，无从确认、无从停止实例，分配保留到删除资源或重新部署。运行成功、实例却已不在运行时不进入 `Failed`，而是重新部署（见 Added）。协调器不自动重试：`spec` 变化（`generation` 大于 `observedGeneration`）时 `Failed` 回到 `Pending`（`Ready=False`，原因 `Pending`，这一轮没有副作用），之后与新建的资源一样经过策略校验、分配与部署，还没停止的实例在启动新的部署运行之前停止，仍持有的分配满足新的申请时复用；`spec` 没有变化时删除后重新创建资源即可再次部署。资源状态机为此新增 `Failed → Pending`（`forgemesh.resource.states`；此前 `Failed` 只能转为 `Terminating`），architecture.md 附录 A.1 同步。
- **破坏性**：修改已部署服务的 `spec` 时重新部署：`generation` 大于 `observedGeneration` 的 `Ready` / `Degraded` 服务经策略放行后进入 `Updating`（`Ready=False`，原因 `Updating`，endpoint 标记为 `ready: false`，这一轮没有副作用）；下一轮停止之前的部署，按新的 generation 取得分配（资源持有的分配满足新的加速器申请时复用；不满足时先停止实例、释放这个分配，再重新分配，新的申请放不下时停在 `Updating`，`AllocationFailed`，10 秒后重试），启动这一代 `spec` 的部署运行并清空 `status.endpoints`，运行成功后回到 `Ready`、写入新实例的地址；更新的运行失败时进入 `Failed`。先停后起：从进入 `Updating` 到新实例通过验证，服务不可用。部署或更新过程中 `spec` 再次变化时，取消进行中的运行，改为部署最新的 `spec`，phase 不变，不重新评估策略。被策略拦下或缺少调度器 / 工作流运行时的更新不写 `status`（任何写入都会把 `observedGeneration` 推进到新的 generation，这次更新便不会再被处理），只记日志，策略拒绝或等待审批时 30 秒后重新评估。`status.phase` 为 `Ready` 且 `observedGeneration` 等于 `generation` 时，服务运行的是最新的 `spec`（开始更新的一轮即写入新的 `observedGeneration`）。资源状态机新增 `Degraded → Updating`。`spec` 没有变化的 `Ready` 与 `Unknown` 的资源不被协调（不评估策略、不写 `status`）。此前协调器对 `Updating` / `Failed` / `Unknown` 的资源（`Ready` / `Degraded` 在 generation 变化时）重新评估策略后写入 `Ready=True`，不重新部署。
- **破坏性**：`ReconciliationController` 注入 `workflow_runtime` 时必须同时注入 `release_provider`（否则抛 `ValueError`：删除资源会留下运行中的实例），运行时须实现 `find_runs` 并在快照中返回步骤输出；新增参数 `workflow_poll_interval`（默认 `2.0` 秒，须为正数）、可选的 `launcher`（注入时写入 `Ready` 之前与恢复 `Degraded` 服务时以 `is_running` 确认实例在运行，不在运行时重新部署）与 `clock`（重新部署退避所用的单调时钟，默认 `time.monotonic`）。不是 `InferenceService` 的资源报告 `Ready=False`（`NotDeployable`），不分配、不启动工作流。
- **破坏性**：`ReferenceWorkflowRuntime` 在后台执行运行（见 Added）：`start()` 返回时运行可能仍在执行，步骤失败时补偿并以 `Failed` 结束，`cancel()` 等待运行进入终态；使用方须在启动时调用 `recover()`、关闭时调用 `aclose()`。
- `control-plane` 包新增依赖 `forgemesh-workflow-runtime`（应用以其中的 `LocalWorkflowRuntime` 组装部署工作流），`deploy/docker/Dockerfile.control-plane` 随之安装该包。
- **破坏性**：插件制品签名由 HMAC-SHA256 改为 Ed25519（`forgemesh.plugin.signing`，`plugin-sdk` 新增依赖 `cryptography>=42`）。`PluginSigner.sign(content, manifest, private_key_pem)` 返回签名文档 `PluginSignature`（`algorithm` / `keyId` / `artifactDigest` / `manifestDigest` / `signature`），签名覆盖算法标识、制品内容摘要与清单摘要；`PluginSigner.verify(content, manifest, signature, trusted_keys)` 从制品内容与清单重新计算两个摘要，通过时返回签名公钥的 key id（此前返回 `True`），未签名、算法不是 `ed25519`、公钥不受信任、摘要不符、签名不符都抛出 `SigningError`。签名模块的 `PluginArtifact` 以及 `PluginSignature` 原有的 `signer` / `signed_at` 字段移除，`signature` 由原始字节改为 base64 字符串。
- **破坏性**：`PluginManager.register(manifest, *, artifact=None, signature=None)` 注册前按 `PluginManifest` 重新校验清单（记录中保存校验后的副本），再用策略中受信任的公钥校验签名；清单非法、未签名、有签名但没有制品内容、签名校验失败都抛出 `PluginRejectedError`（`ForbiddenError` 的子类），插件不会进入注册表。`PluginManager()` 不传策略时不信任任何公钥；未签名的插件只能在显式开启的开发模式（`dev_mode_allow_unsigned=True`，默认关闭）下注册，开启开发模式和每次注册未签名插件都会记录告警。注册记录新增 `signing_key_id` / `artifact_digest`。
- **破坏性**：`PluginManager.check_permissions` 改按管理员授予判定：插件实际拥有的权限是清单申请与策略 `grants` 授予的交集，清单申请了、管理员没有授予的权限不生效（此前直接把清单的 `spec.permissions` 当作已授予）；`required` 中有词表之外的权限名时抛出 `ValueError`。
- **破坏性**：清单 `spec.permissions` 只能取权限词表中的名字（模型与 Schema 枚举一致），模型同样拒绝重复的权限；此前可以是任意字符串。`IsolationGuard.check_access` / `validate_manifest` 改用同一份词表（此前为 `database` / `filesystem` / `network` / `admin` 等自由字符串，`check_access` 的参数 `resource` 改名为 `permission`），未知的权限名抛出 `IsolationError`。
- **破坏性**：清单 `spec.isolation` 只能是 `in-process`（模型与 Schema 一致），声明 `restricted-rpc` / `sandbox` 校验失败：插件都在宿主进程内运行，ADR-006 的受限 RPC 宿主尚未实现（见 Fixed）。脚手架生成的清单改为 `in-process`。
- **破坏性**：清单 `spec.entrypoint` 必须形如 `<模块>:<类>`（模块路径相对于插件包，模型与 Schema 使用同一正则），插件目录须含 `__init__.py`。全部 10 个插件与示例清单 `examples/single-node/plugin-vllm.yaml` 改为真实入口（如 `provider:RulesProvider`、`discovery:GPUDiscoverer`、`deploy:VLLMDeployer`），`plugins/workflow-reference` 补上 `__init__.py`；脚手架生成 `"provider:<类名>"`。
- `scripts/plugin_entrypoint_smoke.py` 按清单 `spec.entrypoint` 经 `load_entrypoint` 加载入口类，缺少 `entrypoint` 即失败（此前跳过），并做基本健康检查：构造器没有必填参数时实例化一次（目前 6 个插件）；需要宿主注入依赖的只报告所需参数（`decision-typesafe-jev` 的客户端、`metering-basic` 的存储、`qos-basic` 的配置、`workflow-reference` 的存储与执行器等）。
- `PluginSdk.validate_manifest` 的告警与实际行为一致：未设置 `isolation` 时说明插件在进程内运行（此前称安装器默认使用 `restricted-rpc`），未设置 `entrypoint` 时说明插件无法加载；清单仍写 `artifactDigest` / `signature` 时给出弃用告警（见 Deprecated）。
- [docs/plugin-development/guide.md](docs/plugin-development/guide.md) 按当前实现更新运行方式与隔离、权限词表、入口点、签名与注册流程；`SECURITY.md` 3.2 / 3.3 节、README 与 [ADR-006](docs/design-decisions/adr-006.md)（新增实施现状）同步更新；[docs/architecture/module-mapping.md](docs/architecture/module-mapping.md) 更新插件的加载方式与运行形态，更正 `runtime-vllm` 的文件列表（没有 `provider.py`），新增偏差 D3（受限 RPC 宿主未实现）。
- **破坏性**：`metadata.generation` 由服务端维护：创建时一律为 `1`，`observedGeneration` 为 `0`，请求中的 `generation` / `observedGeneration` 被忽略（此前原样保存请求中的值，未给出时为 `0`）；之后只有 `spec` 变化时递增，labels / annotations 变化不递增。1.1.0 之前创建的资源不做迁移：保留已保存的 `generation`（通常为 `0`），下一次 `spec` 变化时从原值递增。OpenAPI 新增 `ObjectMeta` schema，服务端维护的字段标为 `readOnly`；JSON Schema 的说明同步更新。
- **破坏性**：`POST /api/v1/inference-services` 拒绝未知的顶层字段（`400`，此前静默忽略）；顶层字段与 `PUT` 相同：`apiVersion`、`kind`、`metadata`、`spec`，以及被忽略的 `status`、`conditions`、`finalizers`。
- **破坏性**：`SQLiteRegistry.list()` 的 `continue_token` 必须是 `list_page` 以同一租户、kind 与过滤条件返回的 `next_page_token`（见 Added），此前格式（未签名的 base64 JSON，任何人都能构造，也不绑定租户与过滤条件）的游标抛出 `InvalidPageTokenError`；私有方法 `_encode_token` / `_decode_token` 移除。
- **破坏性**：`POST /api/v1/decisions` 的风险等级改由服务端推导（见 Added）。此前请求一律以 `low` 交给决策路由器（请求中的 `riskLevel` 被忽略），默认配置下高风险保护经 API 从不生效；现在推导为 `high` 的操作在路由器没有备用 Provider 时返回 `503 DECISION_NO_CANDIDATE`，候选来自备用 Provider 时返回 `503 DECISION_FALLBACK_REQUIRED`（此前自动采用该候选并返回 `200`），`FALLBACK_REQUIRED` 也不再与全部失败一样返回 `DECISION_NO_CANDIDATE`。控制平面应用目前不注入决策路由器，经 `forgemesh-control-plane` 访问时这个端点仍返回 `503 DECISION_ROUTER_UNAVAILABLE`。
- **破坏性**：命令行工具 `forgemesh` 的全部命令改经 REST API 执行（见 Added），需要运行中的网关或控制平面。此前每条命令新建一个内存 SQLite 数据库（`SQLiteRegistry(":memory:")`），从不连接控制平面：`apply` 写入的资源（`spec` 还被丢弃）在命令结束时即丢失，`get` / `describe` / `list` 永远找不到资源，`delete` 调用物理删除、绕过受控删除，CLI 实际不可用，因此没有需要迁移的数据。影响范围是依赖旧选项、输出与退出码的脚本：`--api-url` 的默认值由从未使用的 `http://localhost:8080` 改为网关 `http://127.0.0.1:8081`；`--tenant` 不再默认为 `default`（省略时租户由 key 确定），也不再覆盖文档中的 `metadata.tenantId`，两者不同时报错；`KIND` 只接受 `inferenceservice` / `inferenceservices`；输出格式改变（如 `apply` 由 `Created: InferenceService/llama (version=…)` 改为 `InferenceService default/llama created`，`get` 默认输出 YAML、`-o json` 输出 JSON，`list` 输出表格，`describe` 不再等同于 `get`）；命令失败时的退出码由一律为 `1` 改为按原因区分（见 README「命令行工具」）。迁移：以 `--api-url` / `FORGEMESH_API_URL` 指向网关或控制平面，以 `--api-key-file` 或 `FORGEMESH_API_KEY` 提供 key（`dev` 模式的控制平面改用 `--tenant`），解析输出的脚本改用 `-o json`，按新的退出码判断结果。
- **破坏性**：`Outbox.put` 不再自行提交，只能在调用方已开启的事务中调用（如 `Database.transaction()` 之内），事务之外调用抛出 `RuntimeError`，事件不会写入；`Outbox` 的 `conn` 参数与属性移除，改为 `database=` 参数与 `database` 属性（`Database`）。
- **破坏性**：Outbox 事件的终态 `failed` 改为 `dead`（打开旧库时已有的 `failed` 事件改为 `dead`，可以 `redrive()`），`get_stats()` 的键随之改变；`mark_failed()` 记录一次失败并按重试策略推迟，达到上限才进入 `dead`，返回更新后的事件（此前直接置为 `failed`）。
- 控制平面日志：`forgemesh-control-plane` 启动时按 `FORGEMESH_LOG_FORMAT`（`json`，默认；或 `text`）与 `FORGEMESH_LOG_LEVEL`（`DEBUG` / `INFO`（默认）/ `WARNING` / `ERROR`）配置根 logger，写到标准错误，取值非法时拒绝启动（`forgemesh.controlplane.logs`）。JSON 格式每行一个对象（`time` / `level` / `logger` / `message`，请求中的日志另有 `trace_id` 与 `correlation_id`，带异常时另有 `exception`）；uvicorn 不再另行配置日志，它的访问日志与错误日志经同一个处理器输出。资源变更事件（`LogPublisher`，logger `forgemesh.events`）固定以 `INFO` 输出，不受 `FORGEMESH_LOG_LEVEL` 影响，与其他日志格式相同，`message` 为事件信封的 JSON。此前根 logger 没有配置：控制平面自己的 `INFO` 日志（如部署就绪）不输出，`WARNING` 以上只输出消息文本。
- **破坏性**：网关自己返回的错误响应全部使用 JSON 错误信封 `{"error": {"code", "message", "correlationId", "traceId"}}`（`Content-Type: application/json`），状态码与 `Retry-After` 不变：限流的 `429`（`RATE_LIMITED`）、Recovery 兜底的 `500`（`INTERNAL_ERROR`）、连接控制平面（资源接口反代）或推理实例失败的 `502`（`UPSTREAM_UNAVAILABLE`）与没有匹配路由的 `404`（`ROUTE_NOT_FOUND`，有别于控制平面表示资源不存在的 `NOT_FOUND`；此前为 `net/http` 默认的 `404 page not found`）此前是纯文本；流式代理的错误（如 `NO_HEALTHY_UPSTREAM`、`INVALID_UPSTREAM`、`INSTANCE_ADDRESS_NOT_ALLOWED`、`REQUEST_TOO_LARGE`）此前只有 `code` 与 `message`。已有的错误码不变。按纯文本读取这些响应体的客户端改为读取 `error.code`，或只看状态码。命令行工具 `forgemesh` 的退出码仍按状态码确定，错误消息改为信封中的消息与错误码；`ROUTE_NOT_FOUND` 的 `404` 以 `1` 退出并提示检查 `--api-url`，不是错误信封的 `429` / `502`（如网关前面的反向代理返回的）不再说成来自网关。`/api/v1/` 之外没有匹配路由的请求现在经过 Correlation、TraceContext 与访问日志（仍不经过认证与限流），响应带关联 ID。控制平面与推理实例的响应原样透传；探针的状态文档与 `net/http` 在路由之前拒绝的请求（如请求头格式错误，纯文本）不变；错误码见网关 README「错误响应」。
- **破坏性**：Kubernetes 清单新增网络策略 `deploy/kubernetes/network-policies.yaml`：默认拒绝命名空间 `forgemesh` 内全部 Pod 的入站与出站流量，只放行全部 Pod 访问集群 DNS、网关访问控制平面的 8080 端口、任何来源访问网关的 8081 端口。`kubectl apply -f deploy/kubernetes/` 之后，集群内直接访问 `forgemesh-control-plane:8080` 的调用方会被拒绝，须改为访问网关 `forgemesh-gateway:8081`（API 路径不变）；命名空间中的其他工作负载需要各自的策略。网络策略只在执行 NetworkPolicy 的 CNI 上生效；启用启动器时还要应用 `deploy/kubernetes/launcher/network-policy.yaml`。
- Kubernetes 控制平面清单：Pod 改以专用的 ServiceAccount `forgemesh-control-plane` 运行（此前为命名空间的 `default`，同样不挂载 token），并设置 `enableServiceLinks: false`：同命名空间中名为 `forgemesh` 的 Service 会注入 `FORGEMESH_PORT=tcp://…`，覆盖镜像中的 `FORGEMESH_PORT`，控制平面随即无法启动。网关清单同样设置。
- 网关改用 Go 1.27 构建与测试：`deploy/docker/Dockerfile.gateway` 的构建镜像由 `golang:1.22` 改为 `golang:1.27`，CI 的 `go` 与 `dependency-audit` job 使用 workflow 中的 `GO_VERSION`（`1.27`，取最新补丁版本；此前固定为 `1.22`），`gateway/go.mod` 的 go 行由 `go 1.22` 改为 `go 1.27.0`，开发网关需要 Go 1.27+（Go 1.21 起的 go 命令默认会自动下载）。GODEBUG 的默认值随 go 行而定，网关因此采用 Go 1.27 的默认行为：查询参数超过 10000 个时，转发到控制平面的请求不带查询串（此前原样转发）；设置了 CPU 限制的容器中（如 Kubernetes 清单的 `limits.cpu: "1"`），GOMAXPROCS 按限制向上取整、至少为 2，此前等于节点的 CPU 数；监听端口在内核启用 MPTCP 时也接受 Multipath TCP 连接。

### Deprecated

- `GET /api/v1/inference-services` 的 `tenantId` 查询参数：从未生效，租户取自凭证（ADR-014）；计划在 1.3.0 移除。
- 插件清单的 `spec.artifactDigest` / `spec.signature`：从未被任何代码读取或校验，也无法起作用（清单打包在它要描述的制品之内，签名无法覆盖包含它自身的文档）。Schema 标为 `deprecated`，`validate_manifest` 给出告警，插件管理器注册时既不把它们当作签名，也不拿它们与制品比对；改用 `python -m forgemesh.plugin.signtool sign` 生成随制品分发的签名文档（见插件开发指南第 6 节）。计划在 2.0.0 移除。

### Fixed

- 勘误：1.0.0 变更日志中以下条目与 v1.0.x 的实际交付不符（原文保留作历史记录；README「功能概览」已逐条注明现状）：
  - 「gRPC 通信层：控制平面 ↔ Node Agent 双向流」：`api/protobuf/agent.proto` 没有生成代码，Node Agent 与控制平面之间只有进程内调用。
  - 「执行平面 E2E 测试：控制平面 → Node Agent → GPU → 推理全链路」：没有这样的测试；`tests/e2e/` 中只有各步骤为模拟实现的 GPU 上线工作流测试。
  - 「混沌测试 Chaos：网络分区、节点宕机、GPU 故障注入」：混沌测试以 mock 与内存存储模拟节点离线、控制平面重启、重复事件与工作流恢复，没有网络分区与 GPU 故障注入。
  - 「可观测性 Observability：metrics（Prometheus）、tracing（OpenTelemetry）」：指标与追踪只记录在内存中，没有 Prometheus / OTLP 导出。
  - 「插件安全沙箱（ADR-006）：按信任级别隔离执行」与 Security 中的「插件 manifest 签名验证」：隔离只是信任级别到隔离模式的映射，没有进程级隔离；签名为 HMAC 对称签名，注册插件时不验签。
  - 「Temporal 工作流插件 `workflow-temporal`：生产级工作流编排引擎集成」：只有抽象客户端与内存实现，没有接入 temporalio。
  - Changed 中的「所有端点从 501 Not Implemented 占位升级为正式实现」，以及 Security 中的「所有 API 端点强制认证与租户隔离」「SSRF 防护：外部 URL 请求经白名单校验」：在 v1.0.x 中均不成立，分别自 1.1.0 起提供（可运行入口挂载真实 API、API Key 认证、网关实例地址策略，见本节与 Security）。
- 控制平面唯一的可运行应用仍注册阶段 0 占位路由，全部业务端点返回 `501`，已实现的 API 路由只在测试中挂载：现挂载真实路由；SQLite 在应用启动（lifespan）时打开，导入模块不再产生文件副作用；`/readyz` 探测权威存储可用性。
- `control-plane` 包补齐 `forgemesh-policy-engine` 与 `forgemesh-plugin-sdk` 依赖声明（此前独立安装后导入即失败）。
- `POST /api/v1/inference-services` 丢弃请求体中的 `spec`、接受任意 `kind`、不分配 `uid` 且返回 `200`：现按 `InferenceService` 强类型校验（`uid`、租户与 `status` 由服务端维护），成功返回 `201`，请求体不合法返回 `400` 错误信封，同名冲突返回 `409`。
- 列表接口的 `limit` 原样传给 SQLite，`limit=-1` 返回全部行：API 层限定为 `[1, 1000]`，越界返回 `400`；Registry 对 `limit < 1` 抛出 `ValueError`。
- Registry 标签选择器在 `LIMIT` 之后才在 Python 中过滤，首页之外的匹配项永远不会返回，翻页会静默跳过数据：现以 `json_extract` 在 SQL 中先过滤、再排序分页。
- Registry 全部方法在 `asyncio.to_thread` 工作线程上共用一个 sqlite3 连接且无锁，交错事务中失败写入者的回滚会丢弃其他写入者尚未提交的插入（创建报告成功但未持久化，R-1）：现以实例级 `RLock` 串行化全部数据库调用。
- `namespace` 为空（NULL）的资源可以同名重复创建（SQLite 主键视 NULL 互不相等），一次 DELETE 会删除全部副本：新增唯一表达式索引 `(tenant_id, kind, COALESCE(namespace, ''), name)`；打开已含重复数据的旧库时抛出 `PreconditionFailedError`，不静默丢行。
- `request_delete` 与 `release_allocation` 读后写不校验版本，期间提交的写入会丢失、期间追加的 finalizer 会被绕过：现以读取到的 `resource_version` / 状态为条件做 CAS，竞争失败抛出 `ConflictError`（DELETE 端点返回 `409`）。
- 调和器在 deny / require_approval / 分配失败等无副作用路径上也记录去重键，重新入队后命中幂等短路，被阻塞的资源在 spec 变更或 24h 去重 TTL 过期前不会被重新评估：这些路径不再记录去重键。
- 生产代码中没有任何地方驱动调和器，创建的资源永远停在 `Pending`、`status` 从不更新：现由控制器驱动（见 Added）。
- 调和器在没有注入调度器或工作流运行时的情况下跳过分配与部署、登记去重键并报告 `Ready=True`（`Reconciled`），实际什么都没做：现在任何副作用之前检查，返回 `Ready=False`，原因为 `SchedulerUnavailable` 或 `WorkflowRuntimeUnavailable`，且不登记去重键。未设置 `FORGEMESH_INVENTORY_FILE` 时（默认部署），新建的服务会停在 `Pending` 并带有 `Ready=False`（`SchedulerUnavailable`）。
- 网关把入站路径原样拼接到实例地址，请求以 `/api/v1/inference/{svc}/chat` 到达推理运行时，而 vLLM 与 TGI 都不提供这个路径：现按实例的 `protocol` 转发到 `/v1/chat/completions`（`openai-chat`）或 `/generate_stream`（`tgi-generate`），查询串保留。
- 网关解析推理服务时不带命名空间，控制平面因此只查找不带命名空间的服务，带命名空间的服务无法经网关调用：现由带命名空间的路由解析（见 Added）。
- OpenAPI 没有描述 `GET` / `DELETE /api/v1/inference-services/{name}` 的 `namespace` 查询参数，而实现一直支持它，网关解析带命名空间的服务时也依赖它：现已补上，并写明省略时指不带命名空间的资源（列表接口省略时则不按命名空间过滤）。
- 网关 `/readyz` 只对上游做 TCP 拨号，控制平面的权威存储不可用（其 `/readyz` 返回 `503`）或端口被其他进程占用时仍报告就绪：现请求控制平面的 `/readyz`，返回 `200` 才就绪，否则返回 `503`（`upstream not ready` / `upstream unreachable`）。
- 推理实例返回错误（如 vLLM 对非法参数返回 `400` JSON）时，网关把响应的 `Content-Type` 改成 `text/event-stream`：现只对 2xx 响应这样设置，错误响应保留实例的 `Content-Type`。
- [docs/operations/backup-restore.md](docs/operations/backup-restore.md) 的热备份方法「`PRAGMA wal_checkpoint(TRUNCATE)` 后 `cp` 主库文件」在控制平面写入期间不安全：复制途中的自动 checkpoint 会改写主库文件，备份可能不一致甚至损坏；脚本只看 `PRAGMA integrity_check` 的退出码，而 `sqlite3` 检查出损坏时同样返回 `0`。现改为 `VACUUM INTO`（脚本）/ `Connection.backup`（Python）在线快照，校验输出必须为 `ok`，数据库路径不存在时报错，而不是备份一个新建的空库。
- 备份文档列出的 `data/registry.db` / `audit.db` / `decisions.db` 并不存在，控制平面的权威状态只在 `FORGEMESH_DB_PATH` 一个文件中（部署推理服务时另有同目录的工作流存储，两个文件一起备份，见 Added）。恢复时旧库的 `-wal` / `-shm` 随旧库一起改名保留（`-wal` 中可能有尚未合并回主库的已提交事务），全量恢复示例改为先停止、再启动控制平面。
- compose 拓扑无法运行（module-mapping 偏差 D2）：引用的 `Dockerfile.control-plane` 不存在，gateway 没有构建配置；compose 设置的 `FORGEMESH_UPSTREAM` 网关并不读取（网关读取 `GATEWAY_UPSTREAM_URL`），端口写成 `8000`（网关监听 `8081`）；控制平面默认只监听容器内的回环地址，网关访问不到；数据库写在容器文件系统中，重建容器即丢失；控制平面依赖它并不使用的 postgres，postgres 以默认口令发布在所有网卡上。现改为见 Changed。
- Kubernetes 控制平面清单把数据库写在容器文件系统中，Pod 重建即丢失全部资源；默认的 `RollingUpdate` 在更新期间同时运行新旧两个 Pod，违反 ADR-013 的单写者约束；控制平面默认只监听回环地址，kubelet 探针访问不到。现改为 PVC + `Recreate`，并设置 `FORGEMESH_HOST=0.0.0.0`（见 Changed）。
- `tests/unit/controlplane/test_registry.py` 有两处 `reg.close()` 未被 await，关闭逻辑从未执行。
- 分配不是幂等的（改进计划 4.10）：调度请求不携带资源身份与幂等键，同一 generation 的每次重放（去重键丢失、进程在分配之后 status 写入之前退出、重启后重试）都会再分配一次；数据库也不阻止两个分配占用同一设备：现以 `{uid}:{generation}` 为幂等键，同一键返回同一分配，设备独占由数据库约束保证（见 Added）；同名资源删除后重建（新 uid）得到新的分配。
- 协调器传给调度器的 `requirements` 恒为空，调度器无从得知资源申请了几张什么卡：现取自 `spec.resources.accelerator`。
- 删除处于 `Pending` / `Scheduling` 等无副作用 phase、但已持有分配的资源时直接物理删除，分配及其设备永远不会被释放：现先进入 `Terminating`（见 Changed）。
- `Terminating` 协调按 `{kind}/{name}` 释放分配，忽略命名空间，删除一个服务可能释放另一个命名空间中同名服务的分配：现按资源 uid（限本租户）释放。资源已被删除、或同名资源已以新 uid 重建时，只释放原资源的分配，不再移除新资源的 finalizer 并把它删除。
- 协调器启动一个没有步骤的工作流后立即登记去重键并写入 `Ready=True`（`Reconciled`），启动失败时只记一条警告、同样报告 `Ready=True`；`phase` 停在 `Pending`，`status.endpoints` 从不写入（改进计划 4.10 遗留问题、5.3）：现按 phase 逐轮部署，运行成功之后才 `Ready=True`（见 Changed）。部署工作流没能启动时停在当前 phase（首次部署为 `Preparing`）、`Ready=False`（`WorkflowStartFailed`），保留分配，不登记去重键，退避重试（从 `workflow_poll_interval` 起每次翻倍，最多 60 秒）；查询运行失败（`WorkflowLookupFailed`）与停止之前的部署失败（`StopFailed`）同样处理。
- 参考工作流运行时的 `start()` 只创建运行记录并推进到 `Running`，从不执行步骤，运行永远停在 `Running`，`cancel(cleanup=True)` 补偿的是从未执行过的步骤：现由 `LocalWorkflowRuntime` 在后台执行（见 Added）。
- 全部 10 个插件清单与示例清单的 `entrypoint` 写成 `python -m forgemesh_<名称>`，而这些模块并不存在，没有一个插件能按清单加载；入口冒烟只用 `entrypoint` 判断是否跳过，随后绕开它按文件路径加载 `provider.py`（或包的 `__init__.py`），只断言其中有公有类。现改为真实的 `<模块>:<类>` 入口，冒烟按入口加载（见 Changed）。
- 全部 10 个插件与示例清单声明 `isolation: restricted-rpc`，而受限 RPC 宿主并不存在，插件实际都在宿主进程内运行；其中 `decision-typesafe-jev` 在进程内经 `urllib` 直接请求 TypeSafe API。现全部改为 `in-process`（`decision-typesafe-jev` 已申请 `network.egress`）。
- `plugin-sdk` 包补齐 `pyyaml` 依赖声明：`forgemesh.plugin.sdk` 与签名工具都用它读取清单，此前独立安装（如控制平面镜像）后导入 `forgemesh.plugin.sdk` 即失败。
- `get_registry()` 在 `init_registry()` 之前被调用时新建一个内存数据库（改进计划 4.6 遗留问题）：组装出错或请求与关闭竞争时，资源只写进进程内存、重启即丢失，API 却报告成功，`/readyz` 也返回 `200`。现在抛出 `ComponentNotReadyError`，资源端点返回 `503 REGISTRY_UNAVAILABLE`（仍先校验凭证），`/readyz` 返回 `503`；应用关闭时清空全局 Registry，关闭之后的请求不再拿到已关闭的连接。
- 列表接口最多返回 `limit` 条，没有办法取得其余的资源，OpenAPI 声明的 `nextPageToken` 从未返回：现以 `pageToken` / `nextPageToken` 分页（见 Added）。
- 两个列表接口的 `limit` 越界（`400`）与删除时的并发冲突（`409`）在错误信封中放的是新生成的关联 ID，与请求的 `X-Correlation-Id`（经网关访问时总会带上）对不上：现与其他错误一样回显请求头。OpenAPI 在全部业务端点上声明 `X-Correlation-Id` 请求头。
- OpenAPI 与实现不一致：网关一直转发的 `GET /api/v1/resources` 没有描述；`POST /api/v1/workflows` 的 `404` / `500` / `503` 与 `POST /api/v1/decisions` 的 `500` 没有列出；列表响应的 `count` 没有描述；`DecisionRequest.context` 标为必填（服务端默认为空对象）。现已补齐，`info.description` 说明任何端点遇到未预期的内部错误都返回 `500 INTERNAL_ERROR`，契约测试 `tests/contract/test_openapi_contract.py` 防止两者再次偏离（见 Added）。
- `cli` 包的依赖声明与实际导入不一致：此前导入 `yaml` 与 `forgemesh.controlplane`，却只声明了 `forgemesh-core`，独立安装后命令一运行即因导入失败而退出；包描述列出的 `version` 命令并不存在。改为 HTTP 客户端之后只导入标准库与 PyYAML，依赖声明为 `pyyaml>=6.0.1`，不再依赖 `forgemesh-core`，也不需要控制平面的包；`tests/unit/cli/test_packaging.py` 核对 CLI 导入的第三方发行包与声明的依赖完全一致，并且不导入其他 ForgeMesh 包。
- `TaskStore` 与 `Outbox` 各自使用不加锁的 sqlite3 连接，而把 Outbox 写入放进业务事务的唯一办法是把别人的连接交给它：sqlite3 的事务属于连接，线程交错时一方的提交或回滚会连带另一方尚未完成的写入（与 Registry 内已修复的 R-1 相同）。现经 `Database` 串行化（见 Added）。
- Registry 的写入路径抛出 `IntegrityError` 以外的异常时，未提交的部分写入留在连接上，被同一连接上的下一次提交一并提交：现回滚。
- `Outbox.put` 自行提交，事件无法与它描述的业务写入处于同一个事务：在工作单元中途调用会提交此前的全部写入，`put` 之后的业务写入失败会留下孤立的事件（改进计划 5.4）。现在调用方的事务中写入（见 Changed）。
- Outbox 发布失败一次即标记为 `failed`，此后再不发布，一次瞬时错误就让事件永远送不到；`Dispatcher` 在内存中记录已处理的事件 id，集合无限增长，重启后清空。现按数据库状态退避重试，达到上限进入 `dead`（见 Added）。
- `IdempotentConsumer` 已处理的事件 id 只保存在内存集合中：无限增长，重启后清空，重新投递的事件会再次执行副作用（投递本就是至少一次）。现记在 SQLite 表 `consumed_events`（按消费者名称与事件 id，可经 `database=` 与其他存储共用数据库）；handler 正常返回后才记录，handler 失败时再次投递会重新处理；同一实例的 `consume()` 串行执行，并发投递同一事件只处理一次。
- 网关流式代理的错误日志从固定的 `X-Correlation-ID` 请求头读取关联 ID，设置了 `GATEWAY_CORRELATION_HEADER` 时记录的不是网关实际使用的关联 ID（通常为空）：现取自 Correlation 中间件注入的 context。
- 网关 Recovery 的 panic 日志中 `corr=` 总是为空，也没有 trace-id：Recovery 位于 Correlation 与 TraceContext 之外，读不到它们写入下游 context 的 ID。现由这两个中间件把 ID 记录给 Recovery，panic 日志与兜底的 `500` 错误信封都带上这个请求的关联 ID 与 trace-id。
- 响应头写出之后发生的 panic 被网关 Recovery 当作普通 panic：记录堆栈后仍调用 `http.Error`，把 `Internal Server Error` 写在已写出的响应体（来自控制平面或推理实例）之后；`httputil.ReverseProxy` 复制控制平面的响应体失败时以 `http.ErrAbortHandler` 中止响应，同样被记录为 panic 并追加这段文本。现在 `http.ErrAbortHandler` 原样交给 `net/http`，不记录为 panic；其他 panic 记录日志后同样中止响应，客户端读到的是不完整的响应。

### Security

- Go 网关多租户安全验证：跨租户拒绝、伪造 `X-Tenant-Id` 不得提权、SSE 客户端取消 / 上游断开连接释放、中间件链序（Recovery→Correlation→Authenticate→RateLimit→LimitRequestBody→Route）测试；网关 README 新增 header 转发 / 剥离策略表。
- 租户身份改由凭证绑定（ADR-014）：此前网关只检查 `X-Tenant-ID` 是否存在，控制平面直接信任该请求头且可被直连，任何调用方都能读写任意租户的资源。控制平面现自行校验凭证，不依赖请求必经网关；网关对推理路由检查 `read` scope，命中 Resolver 缓存的请求同样受约束。
- 网关不再把调用方的 `Authorization` / `Cookie` 转发给推理实例（此前原样透传，凭证会泄露给实例）；资源反代与 Resolver 查询仍向控制平面转发 `Authorization`，由控制平面复核。
- 网关限流改按已验证的租户计数，不再读取客户端可控的租户头或 `X-Forwarded-For`（此前更换头值即可获得新的令牌桶）。
- 网关只连接允许网段内的推理实例：此前流式代理会连接 `status.endpoints` 中的任意地址，实例地址被写成 `169.254.169.254` 或内网服务时，网关就成了访问它们的跳板。现在拨号时逐个检查解析出的 IP（主机名之后解析到别处同样生效）；`GATEWAY_INSTANCE_ALLOWED_CIDRS` 列出允许的网段，未设置时拒绝回环、链路本地、未指定、组播地址与已知云元数据服务地址；被拒绝的连接返回 `502 INSTANCE_ADDRESS_NOT_ALLOWED`，并计一次实例失败。
- 网关请求体此前不设上限，原样转发给控制平面与推理实例：现限制为 `GATEWAY_MAX_REQUEST_BODY_BYTES`（见 Added）。
- 插件签名此前使用 HMAC-SHA256 对称密钥，能验签的一方同样能伪造签名；`verify` 只在调用方提供制品内容时重新计算摘要，否则直接信任调用方给出的摘要；签名中的算法标识从不校验；清单不在签名范围内，签名之后改动清单（例如追加申请的权限）不会被发现。现改为 Ed25519（见 Changed）：控制平面只保存公钥，摘要总是从制品内容与清单重新计算。
- `PluginManager.register` 此前不做任何校验，签名校验没有接入注册，任何清单都能注册；`check_permissions` 把清单自己声明的权限当作已授予，插件可以给自己授权。现注册时校验清单与签名，权限以管理员授予为准（见 Changed）。
- 控制平面此前在生产路径中不写审计记录（`AuditLog` 默认使用内存库，应用没有接入）：现经 API 的创建、更新（`PUT` / `PATCH`）与删除在写入资源的同一事务中记录发起者的租户、主体、API key id 与 correlation id；`GET /api/v1/audit-events` 只返回凭证所属租户的记录，分页游标不能用于其他租户（见 Added）。
- webhook 发布器不泄露凭证：日志、错误信息与 Outbox 的 `last_error` 只含 webhook 的 scheme、主机与端口（路径与查询串可能带有签名），token 从不输出；不跟随重定向，token 不会被带到另一个地址；地址中的用户信息被拒绝（改用 `FORGEMESH_EVENT_WEBHOOK_TOKEN`），token 经明文 `http` 发往非回环地址时启动日志给出警告。
- 网关镜像此前以已停止维护的 Go 1.22 构建：govulncheck 报告网关代码实际调用到的 41 个标准库漏洞（`net/http`、`crypto/tls`、`crypto/x509`、`net/url` 等，以 Go 1.22.12 检查），Go 1.27 上没有。现以 Go 1.27 构建（见 Changed），并由 `dependency-audit` job 持续检查。`gateway/go.mod` 的 go 行同为 1.27：标准库中由 GODEBUG 控制的防护（如 URL 查询参数个数的上限）对较旧的 go 行默认关闭，只换构建所用的 Go 版本不会启用它们。
- Kubernetes 清单此前不限制网络访问：集群内任何 Pod 都能直接访问控制平面，绕过网关的限流与请求体上限，控制平面也能访问任意地址。现在只有网关可以访问控制平面，网关出站只能访问控制平面，控制平面出站只能访问集群 DNS（见 Changed 的网络策略）。

## [1.0.1] - 2026-10-11

### Added

- 新增 [ADR-013](docs/design-decisions/adr-013.md)：权威存储采用 SQLite 先行、PostgreSQL 为目标态的过渡决策。
- finalizer 受控删除闭环：`request_delete` → `Terminating` → `ReleaseProvider` → 清空 finalizer（默认 `forgemesh.io/resource-release`）→ 物理删除；失败重入保持 `Terminating`，全流程幂等。
- Allocation 持久化表：部分唯一索引 `idx_allocations_one_active_per_resource`（`ON allocations(resource_id) WHERE status = 'Active'`）保证同一资源至多一个 Active 分配。
- `POST /api/v1/workflows` 与 `POST /api/v1/decisions` 端点正式实现（工作流启动返回 202、结构化决策请求）。
- Go 网关推理服务路由 Resolver：`{svc}` → 实例 endpoint 解析（消费控制平面 `status.endpoints`）、TTL 缓存 5s、健康摘流（连续 3 次失败冷却 10s）、轮询负载均衡与 `X-Tenant-Id` 透传。
- `tests/performance` 性能基准层（4 项：API P99、协调吞吐、重启恢复、冲突正确性；网关流式首字节基准属 Go 网关 CI 范围）。
- 调和去重键 SQLite 持久化（`ProcessedKeyStore`）：控制平面重启后已处理事件不重复调和。

### Changed

- `PolicyDecisionType` 恢复 `allow` / `deny` / `require_approval` 三态：移除 `degraded` 取值，条件放行语义转入 obligations 表达。
- `DecisionRouter` 路由结果新增 `status` 三态（`OK` / `FALLBACK_REQUIRED` / `FAILED`）：高风险请求的候选来自备用 Provider 时返回 `FALLBACK_REQUIRED` 且不自动采用，交由调用方决定；全部失败返回 `FAILED`，绝不解释为"允许执行默认操作"。
- `DecisionRequest.risk_level` 恢复 `Literal["low", "medium", "high"]` 强类型（默认 `low`）、`deadline_ms` 恢复 `int` 强类型与默认值。
- 推理服务状态转移表开放"任意非吸收态（含 `Failed` / `Unknown`）→ `Terminating`"，删除入口对所有现实状态可达。
- `store.put` 接线状态机校验：非法状态转移在持久层被拒绝（`InvalidStateTransition`）。
- DELETE 端点对存在副作用（finalizer 非空）的资源返回 `202 Terminating`，不再直接物理删除。

### Fixed

- 修复决策路由风险等级字典序比较 bug：`"low" >= "high"` 在字符串比较下为 `True`，导致低风险请求被误判为高风险；改为显式等级序（rank）比较。
- 清理 OpenAPI 中已实现端点（workflows/decisions 等）的 `501 Not Implemented` 残留标记。
- 勘误：1.0.0 变更日志 P1 段中"PostgreSQL 权威存储（ADR-007）：Repository 模式、迁移脚本"的表述与实际不符——v1.0.0 实际交付为 SQLite 权威存储先行（`SQLiteRegistry` 实现 `ResourceRegistry` 抽象、CAS 乐观并发 `resource_version`），未引入 PostgreSQL 部署与迁移脚本。PostgreSQL 仍为权威存储目标态（ADR-007），过渡决策与迁移判据详见 [ADR-013](docs/design-decisions/adr-013.md)。

### Security

- 决策高风险降级保护：高风险请求的候选来自备用 Provider 时不再自动采用（`FALLBACK_REQUIRED`），防止降级路径绕过审批与授权语义（对齐 ADR-004 / ADR-005）。

## [1.0.0] - 2026-10-11

首个稳定版本。API 兼容性冻结，major 版本内保证向后兼容。
完整发布记录见 [docs/releases/v1.0.md](docs/releases/v1.0.md)。

### Added

#### P0 — 工程骨架与领域契约（125 测试）
- 初始化 monorepo 工程骨架（Python 3.12 + Go 1.22 + ruff + pytest）
- 建立领域契约包 `core-contracts`：实体、值对象、领域事件不变量
- 建立资源模型包 `resource-model`：声明式资源（apiVersion/kind/metadata/spec/status）
- 定义 OpenAPI 3.1 规范骨架（`api/openapi/openapi.yaml`）
- 定义 JSON Schema 权威契约（`api/schemas/`：inference-service、device-profile、event、plugin-manifest）
- 引入 Go 网关骨架（`gateway/`）：/api/v1/ 版本前缀、统一错误信封
- 搭建 CI 流水线（`.github/workflows/ci.yml`）：Python lint + test、Go build + test
- 编写架构基线文档 `docs/architecture/architecture.md`（11.7 万字）
- 编写开发计划文档 `docs/architecture/development-plan.md`（13.9 万字）
- 采纳 ADR-001 ~ ADR-012 架构决策记录
- 制定 CONTRIBUTING、GOVERNANCE、SECURITY、CODE_OF_CONDUCT 治理文档
- 共 140 文件、125 测试

#### P1 — 核心控制平面（215 测试，累计）
- 控制平面包 `control-plane`：Registry（资源注册表）、Controller（调和循环）
- 策略引擎包 `policy-engine`：Scheduler/Policy 最终治理权（ADR-005 不变量）
- 插件管理器 PluginManager：发现、加载、生命周期管理
- 审计日志 Audit：所有治理操作可查询、可追溯
- 事务性 Outbox（ADR-008）：事件与状态原子写入、幂等消费者
- 对外 REST API 实现（FastAPI）：推理服务 CRUD、工作流启动、决策请求
- CLI 工具包 `cli`：声明式资源管理命令行
- SQLite 权威存储先行（ADR-013 过渡决策）：ResourceRegistry 抽象、CAS 乐观并发（resource_version）；PostgreSQL 迁移判据见 ADR-013
- 统一响应信封与错误码体系
- 累计 215 测试

#### P2 — 工作流与决策（341 测试，累计）
- 工作流运行时包 `workflow-runtime`：WorkEngine 可替换 Provider 架构（ADR-003）
- 参考执行器插件 `workflow-reference`：顺序/并行/补偿编排
- 工作流 API 包 `workflow-api`：定义版本化、运行状态机
- 决策 API 包 `decision-api`：结构化决策请求与追踪
- JEV 决策插件 `decision-jev`：JSON 表达式求值
- 规则决策插件 `decision-rules`：规则引擎 Provider
- 熔断器 CircuitBreaker：Provider 故障隔离与恢复
- 降级路径 Degradation：决策降级标记与可观测追踪
- 决策插件接口标准化（ADR-004：JEV 非唯一决策系统）
- 累计 341 测试

#### P3 — 执行平面与 GPU 生命周期（503 测试，累计）
- Node Agent 包 `node-agent`：受认证任务协议（ADR-009，禁止默认远程 Shell）
- gRPC 通信层：控制平面 ↔ Node Agent 双向流
- GPU NVIDIA 插件 `gpu-nvidia`：GPU 发现、拓扑、显存管理
- vLLM 运行时插件 `runtime-vllm`：模型加载、推理服务生命周期
- 执行平面 E2E 测试：控制平面 → Node Agent → GPU → 推理全链路
- 混沌测试 Chaos：网络分区、节点宕机、GPU 故障注入
- 能力驱动异构资源模型（ADR-010）：厂商差异通过 Provider 显式表达
- 累计 503 测试

#### P4 — 生产级治理（712 测试，累计）
- 配额管理 `qos-basic` 插件：租户级资源配额与限流
- QoS 调度优先级：高/中/低优先级队列与抢占
- 计量 `metering-basic` 插件：GPU 时、推理请求数、令牌数计量
- 可观测性 Observability：metrics（Prometheus）、tracing（OpenTelemetry）、structured logging
- 诊断 Diagnostics：健康检查、就绪探针、运行时自检
- 插件安全沙箱（ADR-006）：按信任级别隔离执行，高风险插件受限
- 安全测试：认证/授权/注入/越权/SSRF 覆盖
- 插件与工作流版本固定（ADR-011）：升级不破坏运行中任务
- 累计 712 测试

#### P5 — 扩展与生态（898 测试，累计）
- 异构资源 MIG（Multi-Instance GPU）支持：GPU 分区与共享
- Temporal 工作流插件 `workflow-temporal`：生产级工作流编排引擎集成
- TGI 运行时插件 `runtime-tgi`：HuggingFace TGI 推理后端集成
- 插件 SDK 包 `plugin-sdk`：脚手架生成、开发工具链、契约测试套件
- 治理文档 `docs/governance/`：弃用政策（deprecation-policy.md）
- 插件开发指南 `docs/plugin-development/guide.md`
- 运维文档 `docs/operations/`
- 兼容性测试套件 `tests/compat/`：API 契约回归保护
- 插件契约测试套件 `tests/contract/`
- OpenAPI v1.0 稳定标记（`x-stability: stable`）与弃用扩展字段（`x-deprecated`）
- 架构文档纳入 CI 检查
- 累计 898 测试

### Changed

- OpenAPI `info.version` 从 `0.0.1` 升级至 `1.0.0`
- OpenAPI 添加 `x-stability: stable` 扩展字段标记稳定 API
- OpenAPI 添加 `x-deprecation-policy` 扩展字段定义弃用标记规则
- OpenAPI tags 添加描述信息
- CI 流水线新增架构文档检查 job（docs/architecture、docs/design-decisions、api/openapi、api/schemas）
- 所有端点从 501 Not Implemented 占位升级为正式实现
- ruff target-version 固定为 py312
- README 更新为 v1.0 稳定版描述

### Fixed

- 修复插件加载时信任级别校验遗漏问题
- 修复 Outbox 消费者在重复消息时的幂等性问题
- 修复 Node Agent 在 GPU 热拔插时的资源泄漏
- 修复工作流补偿在并发场景下的状态竞争
- 修复决策追踪在降级路径下的上下文丢失

### Security

- 插件安全沙箱按信任级别隔离执行（ADR-006）
- Node Agent 采用受认证任务协议，禁止默认执行任意远程 Shell（ADR-009）
- 所有 API 端点强制认证与租户隔离
- SSRF 防护：外部 URL 请求经白名单校验
- 插件 manifest 签名验证

### Deprecated

- 无。v1.0 为首个稳定版本，无弃用项。
- 弃用政策：未来版本中弃用的端点或字段将使用 `x-deprecated: true` 标记，
  至少保留 2 个 minor 版本后移除。详见 [docs/governance/deprecation-policy.md](docs/governance/deprecation-policy.md)。

---

## 版本对比链接

[1.0.1]: https://github.com/Mekoforge/ForgeMesh/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/Mekoforge/ForgeMesh/releases/tag/v1.0.0