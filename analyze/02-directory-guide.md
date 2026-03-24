# 按目录逐层展开的源码导览图

## 顶层结构

### `Makefile`
- 统一入口：配置、安装、开发启动、Docker 启停。

### `scripts/`
- 运行与运维脚本：
  - `serve.sh`：本地/生产模式启动
  - `check.py`：环境检查
  - `configure.py`：配置生成
  - `deploy.sh`：生产部署
  - `docker.sh`：开发容器辅助

### `docker/`
- `docker-compose.yaml` / `docker-compose-dev.yaml`
- `nginx/` 反向代理配置
- `provisioner/` K8s 沙箱供给服务（与后端 `RemoteSandboxBackend` 配合；行为见 [06-sandbox-provisioner-memory.md](06-sandbox-provisioner-memory.md)）

### `skills/`
- 公共与自定义技能目录，供 Agent 动态注入 workflow 指导。

## `backend/` 结构

后端核心分层：
- `app/`：应用壳层（Gateway + channels）
- `packages/harness/deerflow/`：核心运行时（Agent、tools、sandbox、memory、MCP、skills）

### `backend/app/`

#### `backend/app/gateway/`
- FastAPI Gateway。
- 管理面 API：模型、技能、MCP、记忆、上传、产物、建议等。
- 关键文件：`backend/app/gateway/app.py`，`backend/app/gateway/routers/*`

#### `backend/app/channels/`
- IM 渠道接入层（Feishu/Slack/Telegram）。
- 将外部消息映射到 LangGraph 会话线程并回传结果。

## `backend/packages/harness/deerflow/` 结构

### `agents/`
- Agent 编排中心。
- `lead_agent/agent.py`：`make_lead_agent()` 主入口。
- `lead_agent/prompt.py`：系统提示词拼装。
- `thread_state.py`：会话状态结构。
- `middlewares/`：横切能力链（上传、沙箱、摘要、标题、记忆、澄清等）。
- `memory/`：长期记忆抽取与注入。
- `checkpointer/`：状态持久化。

### `tools/`
- 工具聚合入口 `tools.py`。
- `builtins/` 内置工具集合。
- 负责组合配置工具 + 内置工具 + MCP 工具 + 可选子代理工具。

### `sandbox/`
- 沙箱抽象、Local 实现、工具与中间件；Docker/K8s 实现在 `community/aio_sandbox/`。
- 生命周期（`release` / warm 池 / 销毁）与记忆关系见 [06-sandbox-provisioner-memory.md](06-sandbox-provisioner-memory.md)。

### `subagents/`
- 子代理执行体系：
  - `executor.py`：调度/并发/超时
  - `registry.py`：注册表
  - `builtins/`：内建类型

### `models/`
- 模型工厂，按配置构建 LangChain 模型实例。

### `config/`
- 全局配置系统：
  - `app_config.py`
  - 各类 `*_config.py`
- 统一解析 `config.yaml` 和环境变量。

### `mcp/`
- MCP 集成层。
- 负责读取 `extensions_config.json`、工具缓存与失效重建。

### `skills/`
- 技能加载和启用状态管理。
- 扫描 `skills/public` + `skills/custom`，解析 `SKILL.md`。

### `reflection/`
- 反射加载工具与模型定义，支撑配置驱动装配。

### `community/`
- 社区扩展能力（搜索、抓取、图片等可选组件）。

### `client.py`
- 嵌入式 Python 客户端，支持不经 HTTP 的进程内调用。

## `frontend/` 结构

建议优先阅读：
- `frontend/package.json`
- `frontend/src/core/`
- `frontend/src/app/workspace/`
- `frontend/src/components/workspace/`

### `frontend/src/core/`
- 前端基础设施层：`api`、`config`、`threads`、`uploads`、`models`、`skills`、`mcp` 等。

### `frontend/src/app/workspace/`
- 聊天页面入口和路由层。

### `frontend/src/components/workspace/`
- 工作区 UI 组件：输入框、消息列表、标题、Todo、产物面板。

## `backend/tests/`

后端测试覆盖较全面，按主题可分：
- router
- middleware
- sandbox
- subagent
- memory
- client
- boundary

建议把测试作为理解真实设计意图的第二阅读入口。
