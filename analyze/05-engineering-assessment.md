# 工程化、测试与风险评估

**结论摘要**：harness 与后端测试较强；前端测试与 CI 偏弱；部署可用；文档/脚本偶有漂移。下文分主题展开。

## 1. 依赖与构建

### 后端

后端使用：

- `uv` 作为 Python 依赖管理工具
- workspace 管理 `deerflow-harness`
- Python `3.12+`

特点：

- `backend/pyproject.toml` 管理应用层依赖
- `backend/packages/harness/pyproject.toml` 管理核心运行时包依赖
- `uv.lock` 锁定后端依赖版本

### 前端

前端使用：

- `pnpm`
- `Next.js 16`
- `React 19`
- `TypeScript`

前端依赖现代化程度较高，具备中大型 Web 应用的基础设施。

## 2. 启动与运行方式

### 本地开发

推荐入口：

- `make config`
- `make install`
- `make dev`

由根目录 `scripts/serve.sh` 编排：

1. LangGraph
2. Gateway
3. Frontend
4. Nginx

### Docker

项目同时提供：

- 开发用 `docker-compose-dev.yaml`
- 生产用 `docker-compose.yaml`

整体说明该项目并非只面向本地实验，也考虑了容器化部署。

## 3. 测试现状

### 后端测试

后端测试目录 `backend/tests/` 覆盖面较广，约 40 个测试文件。

覆盖主题包括：

- Gateway router
- MCP
- Memory
- Sandbox
- Subagent
- 工具核心逻辑
- harness/app 边界约束
- embedded client

这说明后端并不是“纯 demo 代码”，已经具备相对系统化的测试意识。

### 前端测试

前端测试明显偏弱。

当前只发现很少量测试文件，且 `package.json` 中没有标准 `test` 脚本。前端自动化测试体系明显落后于后端。

## 4. CI 现状

当前 GitHub Actions 主要覆盖后端：

- 安装后端依赖
- 运行 `ruff`
- 运行 `pytest`

优点：

- 至少保证了后端基础回归

不足：

- 前端 `lint`
- 前端 `typecheck`
- 前端 `build`
- 端到端测试

都没有进入现有 CI 主链。

## 5. 代码质量工具

### Python

后端当前主要依赖：

- `ruff`

优点：

- Lint 接入简单
- CI 已覆盖

不足：

- 没有看到系统级 `mypy` 或 `pyright` 配置
- 类型约束强度有限

### TypeScript

前端具备：

- `eslint`
- `tsc --noEmit`
- `prettier`

但问题在于：

- 并未完整进入 CI
- 部分严格规则关闭
- 实际约束强度低于表面配置

## 6. 工程一致性问题

这是本次分析中最明确的风险之一。

### 文档与实现漂移

发现的问题包括：

- `CONTRIBUTING.md` 中的项目结构描述与当前实现不完全一致
- 文档中提到前端可运行 `pnpm test`，但脚本并不存在
- 前端 `Makefile` 里引用了 `pnpm format:write`，但 `package.json` 里没有对应 script
- 部分架构文档示例仍带旧结构痕迹

影响：

- 新贡献者容易踩坑
- 文档可信度下降
- 工程入口成本变高

## 7. 部署与运行风险

### `langgraph dev` 的使用

从当前说明看，项目在运行 Agent 服务时仍大量依赖 `langgraph dev`。

这对于开源项目和开发阶段是可接受的，但对严格生产化来说意味着：

- 服务形态还偏过渡
- 部署策略未来可能继续演进

### 沙箱与 Docker Socket

沙箱能力是 DeerFlow 的核心优势，但如果使用 Docker 模式并挂载宿主机 Docker socket，也会带来更高的部署面风险。K8s + Provisioner、hostPath 工作区、多副本时的存储与网络，见 [06-sandbox-provisioner-memory.md](06-sandbox-provisioner-memory.md)。

这要求：

- 更谨慎的主机隔离策略
- 更严格的镜像来源控制
- 更明确的生产安全说明

## 8. 优势总结

从工程角度，项目的主要优点是：

- `harness` 与 `app` 职责边界清晰
- Agent 运行时设计集中且可扩展
- 后端测试体系具备一定规模
- 容器化与本地运行路径都比较完整
- 技能、MCP、子代理、记忆体系已经形成平台化基础

## 9. 风险总结

当前最主要的风险点包括：

- 前端测试与 CI 覆盖不足
- 文档与脚本存在漂移
- 静态类型约束不够强
- 生产部署形态仍在演进
- 沙箱部署需要更强的安全治理

## 10. 建议的改进优先级

建议按下面顺序推进：

1. 先修文档与脚本不一致问题
2. 为前端补齐 `test`、`format` 等脚本
3. 将前端 `lint`、`typecheck`、`build` 纳入 CI
4. 为后端补 coverage 报告
5. 逐步增强 Python 和 TypeScript 的静态类型约束
6. 梳理更正式的 LangGraph 生产部署方案

## 结论

与篇首摘要一致：运行时与扩展面强，工程一致性与前端保障是主要短板；已具备可二次开发的平台形态。
