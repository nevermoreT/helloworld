# DeerFlow 分析文档索引

本目录沉淀对仓库的**结构化阅读路径**：先总览与调用链，再按需深入目录、前端或沙箱/记忆专题。

## 文档列表

| 文档 | 内容 |
|------|------|
| [01-project-overview.md](01-project-overview.md) | 定位、分层架构、核心入口（模型/工具/中间件/状态） |
| [02-directory-guide.md](02-directory-guide.md) | 根目录与 `backend` / `frontend` 目录导览 |
| [03-backend-call-chain.md](03-backend-call-chain.md) | 启动与 Web 对话链、`make_lead_agent`、中间件与工具链 |
| [04-frontend-api-langgraph.md](04-frontend-api-langgraph.md) | 前端 → `/api/langgraph` / Gateway 的双通道 |
| [05-engineering-assessment.md](05-engineering-assessment.md) | 工程化、测试、CI、风险与改进优先级 |
| [06-sandbox-provisioner-memory.md](06-sandbox-provisioner-memory.md) | **沙箱三种模式、Provisioner、release/warm/销毁、卷与记忆边界** |

## 推荐阅读顺序

1. **01** → **03**（建立「谁在执行、数据从哪过」）
2. 做前端或联调：**04**
3. 查目录结构：**02**
4. 部署、质量、风险：**05**
5. 沙箱 / K8s / `memory.json` 与线程盘关系：**06**

## 一句话

DeerFlow 是 **LangGraph 上的 Agent 运行时**：把模型、工具、**沙箱**、子 Agent、MCP、技能与**长期记忆**串成可配置流水线；执行核心在 **harness**，Gateway 与前端负责管理与接入。
