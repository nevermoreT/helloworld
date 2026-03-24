# 沙箱、Provisioner 与记忆持久化

本文档归纳 **Sandbox 如何执行**、**Docker / K8s（Provisioner）如何供给**、**release / warm 池 / 销毁**、以及 **工作区数据与 `memory.json`、checkpointer 的边界**。代码路径以仓库当前实现为准。

---

## 1. 三种 Sandbox 模式（执行与调度）

| 模式 | Provider / 实现 | 命令如何执行 | 调度方式 |
|------|-----------------|-------------|----------|
| **Local** | `LocalSandboxProvider` → `LocalSandbox` | 宿主 **`subprocess.run(..., shell=True)`**，shell 为 `/bin/zsh`→`bash`→`sh` 或 PATH 的 `sh` | 无容器；每命令一个子进程 |
| **Docker（本机）** | `AioSandboxProvider` + `LocalContainerBackend` | **`docker run` / `container run`（subprocess）** 起守护容器；后端通过 **`AioSandbox` + HTTP**（`agent_sandbox`）调 `shell.exec_command` | 本机容器运行时；端口映射 `host:N → 8080` |
| **K8s（Provisioner）** | `AioSandboxProvider` + `RemoteSandboxBackend` | 与 Docker 相同：**HTTP 进沙箱镜像**；差别在 **谁起 Pod** | **Provisioner** 服务调 K8s API 创建 **Pod + NodePort Service** |

核心抽象：`Sandbox.execute_command` / 文件 API；**Local 是子进程，Aio 是 HTTP**。

**关键路径**

- 抽象与工具：`backend/packages/harness/deerflow/sandbox/sandbox.py`、`sandbox/tools.py`
- 本地实现：`sandbox/local/local_sandbox.py`
- HTTP 实现：`community/aio_sandbox/aio_sandbox.py`
- 供给与生命周期：`community/aio_sandbox/aio_sandbox_provider.py`
- 本机容器：`community/aio_sandbox/local_backend.py`
- 远端 K8s：`community/aio_sandbox/remote_backend.py`
- Provisioner 服务：`docker/provisioner/app.py`

---

## 2. Provisioner 是什么？

**本仓库中的 Provisioner** = 独立 **FastAPI 小服务**（默认 `:8002`），**只做一件事**：按 `sandbox_id` / `thread_id` 在 Kubernetes 里 **创建或复用** 沙箱 **Pod + Service**，并返回 **`sandbox_url`**（如 `http://NODE_HOST:NodePort`）。

- **不做**：跑 LangGraph、写 `memory.json`、执行用户业务逻辑。
- **后端角色**：`RemoteSandboxBackend` 对其发 `POST/DELETE/GET /api/sandboxes...`，拿到 URL 后与普通 Docker 沙箱一样用 `AioSandbox` 访问。

启用条件：`config.yaml` 中 `sandbox.provisioner_url` 指向该服务；Compose 中可通过 `profiles: provisioner` 启动（见 `docker/docker-compose.yaml`）。

---

## 3. K8s 模式下的卷：不是 PVC

当前 Provisioner 生成的 Pod 使用 **`hostPath`**，**未使用 PV/PVC**：

- **skills**：`SKILLS_HOST_PATH` → 容器内只读挂载（如 `/mnt/skills`）。
- **user-data**：`{THREADS_HOST_PATH}/{thread_id}/user-data` → 容器内 `/mnt/user-data`（`DirectoryOrCreate`）。

含义：

- 工作区在 **节点本地路径**；多节点调度时，若无共享存储，**Pod 漂移到其他节点会看不到原目录**。
- 生产若要多节点、可漂移，需 **自改** `_build_pod` 等，改为 **RWX PVC** 等（仓库无现成清单）。

Compose 示例：`THREADS_HOST_PATH=${DEER_FLOW_HOME}/threads`，与 harness 里 `Paths` 的 `threads/{thread_id}/user-data` 布局对齐。

---

## 4. `release()`、warm 池、Pod 何时销毁

### 调用入口

- **`SandboxMiddleware.after_agent`**（`sandbox/middleware.py`）在每轮 agent 结束后调用 **`get_sandbox_provider().release(sandbox_id)`**。

### `AioSandboxProvider.release()`（`aio_sandbox_provider.py`）

- 从内存中的「活跃」映射里摘掉该 sandbox，**不**调用 `_backend.destroy`。
- 将 **`SandboxInfo` 放入 `_warm_pool`**，**容器 / Pod 继续运行**，便于同线程下一轮快速复用。

### 何时真正 `destroy`（会删 Docker 容器或 K8s Pod）

- **`idle_timeout`**（默认 600s，可配置）：后台线程对 **warm 池** 中超时的项调用 `_backend.destroy`；远端即 **Provisioner DELETE**，删 Service + Pod。
- **`destroy(sandbox_id)`** 显式路径：shutdown、部分驱逐逻辑等。
- **`replicas` 软上限**：仅 **驱逐 warm 池最老项**，不会强行停正在服务某 thread 的活跃沙箱。

### `LocalSandboxProvider.release`

- **`pass` 空实现**；本地单例沙箱无需释放。

---

## 5. Pod 删掉后，工作区如何「接回去」

与沙箱进程无关，由 **路径约定 + 稳定 id** 保证：

1. **`sandbox_id = sha256(thread_id)[:8]`**（同 `thread_id` 始终同一 id）。
2. Provisioner 挂载路径 **`.../{thread_id}/user-data`**，数据在 **宿主机（或 hostPath 所指盘）**，**不在 Pod 生命周期内**。
3. 再次 **`POST /api/sandboxes`**：若资源已不存在则新建 Pod，**仍挂同一 hostPath**；若 Service 仍在则 **幂等返回** 已有 URL。

前提：`THREADS_HOST_PATH` 在集群侧 **可复现同一路径内容**（单节点 hostPath 或后续改为共享卷）。

---

## 6. 「记忆」与沙箱：两条独立链路

避免混淆 **沙箱磁盘** 与 **产品记忆**：

| 数据 | 写入方 | 典型路径 / 存储 | 是否进沙箱容器 |
|------|--------|-----------------|----------------|
| **对话状态 / messages** | LangGraph + **checkpointer** | 未配置时常为进程内 `InMemorySaver`；可配 **sqlite / postgres**（`config.yaml` `checkpointer`） | 否 |
| **长期 Memory（画像/事实）** | `MemoryUpdater` 写 **`memory.json`** | `{base_dir}/memory.json` 或 `memory.storage_path`（相对则相对 `base_dir`） | 否；注入靠 **system prompt** 中的 `<memory>` 文本（`lead_agent/prompt.py`） |
| **线程工作区 / 上传 / 产物** | 工具写 **`/mnt/user-data/...`** | 宿主 `threads/{thread_id}/user-data/...` | 是（挂载） |

**Docker 部署后端时**：将 **`DEER_FLOW_HOME`** 挂到持久目录，则 **`memory.json` 与 threads 树** 在宿主持久；与「沙箱是否 Docker/K8s」正交。

**Memory 的「淘汰」**（`agents/memory/updater.py`）：LLM 可输出 **`factsToRemove`**；事实数超过 **`max_facts`** 时按 **confidence** 截断；新事实受 **`fact_confidence_threshold`** 过滤；**无**按时间的整库 TTL。

**异步落盘**：`MemoryMiddleware` → 队列 **`debounce_seconds`** 后再写盘；极端硬杀进程可能丢失队列中未刷新的更新。

---

## 7. 工具输出如何回到模型

- **Local**：`subprocess` 捕获 stdout/stderr，拼成字符串返回。
- **Aio（Docker/K8s）**：HTTP API 返回的 **output 字符串**；异常时工具侧返回 `"Error: ..."`。

结果作为 **ToolMessage** 进入 LangGraph；是否跨会话可见取决于 **checkpointer**，不是沙箱镜像。

---

## 8. 扩展为「云沙箱底座」时的缺口（简表）

| 能力 | 现状 |
|------|------|
| 全局沙箱配额 / 队列 | 无；仅有每进程 **`replicas` 软上限 + warm 驱逐** |
| 多副本 LangGraph 共享状态 | 需 **外部 checkpointer（如 Postgres）+ 共享 `DEER_FLOW_HOME` 或等价存储** |
| 多节点沙箱工作区 | **hostPath 不足**；需 PVC / 共享文件系统 |
| Provisioner | 单控制面服务；NodePort **每沙箱一个**，大规模需网络与端口规划 |

---

## 相关文档

- 总览与架构：`01-project-overview.md`
- 中间件与全链路：`03-backend-call-chain.md`
- Docker 环境变量与卷：`docker/docker-compose.yaml`、`backend/docs/CONFIGURATION.md`（Sandbox / Provisioner 小节）
