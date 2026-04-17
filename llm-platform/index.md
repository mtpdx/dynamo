# LLM 推理服务平台文档

基于 NVIDIA Dynamo + LiteLLM 构建的企业级 LLM 推理服务平台。

## 文档索引

### 核心文档

| 文档 | 内容 | 说明 |
|------|------|------|
| **architecture-design.md** | 完整架构设计 | 初版 |
| **README.md** | Dynamo 生产最佳实践 | 基于 Dynamo 项目的最佳实践 |
| **litellm-integration.md** | LiteLLM 集成方案 | 评估报告和集成指南 |
| **architecture-review.md** | 架构评审报告 | 资深架构师视角的全面评审 |
| **dynamo-architecture-guide.md** | Dynamo 核心架构理解 | 深入理解 Dynamo 三平面架构 |

### 快速参考

| 文档 | 内容 |
|------|------|
| **quick-reference.md** | 配置模板和命令参考 |
| **aiconfigurator-guide.md** | AIConfigurator 使用指南 |
| **production-best-practices.md** | 生产最佳实践 |

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                     Web UI / SDK / OpenAI SDK                   │
└─────────────────────────────┬───────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────┐
│   Go Backend      │    │  LiteLLM Gateway  │    │   内部服务    │
│   (管理面 :8001)   │    │  (推理面 :4000)   │    │ Dynamo/NATS  │
│                   │    │                   │    │               │
│ 租户/模型/部署/配额│    │ OpenAI 兼容 API   │    │  LLM 推理    │
│ 用户/权限/审计/作业│    │ 成本/限流/缓存    │    │   执行       │
└───────────────────┘    └───────────────────┘    └───────────────┘
      ↕ 同层级服务，并列服务于 Web UI/SDK
```

### 核心设计

| 特性 | 说明 |
|------|------|
| **性能等级** | 简化部署配置，用户只需选择等级（economy/standard/high_performance/enterprise） |
| **异步作业** | 部署、扩缩容等长时操作通过 Job 异步执行 |
| **两层配额控制** | Go Backend 租户级强控制 + LiteLLM 模型/Key 级细粒度限流 |
| **LiteLLM Gateway** | OpenAI 100% 兼容 API，多模型路由，成本日志，熔断器 |

---

## 快速导航

### 场景 1：理解平台架构

1. 阅读 [architecture-design.md](./architecture-design.md) 第 1-2 章
2. 了解 [API 分层](./architecture-design.md#4-api-接口设计)
3. 查看 [技术栈](./architecture-design.md#9-技术栈)

### 场景 2：部署一个模型

1. 使用部署模板或性能等级创建部署
2. 查看 [创建部署 API](./architecture-design.md#51-创建部署简化-api)
3. 查询 [作业状态](./architecture-design.md#52-查询作业状态)

### 场景 3：集成 OpenAI SDK

1. 获取 API Key
2. 调用 [推理 API](./architecture-design.md#55-推理-apiopenai-兼容)
3. 参考 [LiteLLM 文档](https://docs.litellm.ai/)

### 场景 4：使用 Dynamo 高级特性

1. 阅读 [README.md](./README.md) 架构概述
2. 了解 [部署模式](./README.md#2-部署架构模式)
3. 配置 [RDMA](./README.md#5-rdma-与-kv-cache-传输)

---

## 核心概念

### 性能等级 (Performance Tier)

简化部署配置，用户只需选择等级：

| 等级 | 说明 | 适用场景 |
|------|------|----------|
| `economy` | 最小配置 | 测试/开发 |
| `standard` | 推荐配置 | 生产轻负载 |
| `high_performance` | 高性能 | 生产高负载 |
| `enterprise` | 顶级 | 大规模部署 |

### 异步作业 (Job)

部署、扩缩容等长时操作通过 Job 异步执行：

```json
{
    "job_id": "job_abc123",
    "status": "running",
    "progress": {
        "current_step": "creating_workers",
        "steps_completed": 2,
        "total_steps": 4
    }
}
```

### 两层配额控制

```
第一层：Go Backend（租户级强控制）
       - 月度支出限额
       - GPU 小时限额
       - 部署数量限额

第二层：LiteLLM（模型/Key 级细粒度）
       - RPM / TPM 限制
```

---

## API 概览

### 管理面 API (:8001)

| 端点 | 描述 |
|------|------|
| `POST /api/v1/auth/register` | 用户注册 |
| `POST /api/v1/auth/login` | 用户登录 |
| `POST /api/v1/models` | 创建模型 |
| `GET /api/v1/models` | 模型列表 |
| `POST /api/v1/deployments` | 创建部署 |
| `GET /api/v1/deployments/:id` | 部署详情 |
| `POST /api/v1/deployments/:id/scale` | 扩缩容 |
| `GET /api/v1/jobs/:id` | 作业状态 |
| `GET /api/v1/templates` | 部署模板 |

### 推理面 API (:4000)

OpenAI 100% 兼容：

| 端点 | 描述 |
|------|------|
| `POST /v1/chat/completions` | Chat 对话 |
| `POST /v1/completions` | Text Completion |
| `POST /v1/embeddings` | 向量嵌入 |
| `GET /v1/models` | 可用模型 |
