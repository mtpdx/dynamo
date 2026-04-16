# LLM 推理服务平台 - 架构设计文档

**文档版本**：V2.0  
**编制日期**：2026-01-30  
**文档状态**：已根据需求评审和 LiteLLM 集成评估优化

---

## 变更记录

| 版本 | 日期 | 变更内容 |
|------|------|----------|
| V1.0 | 2026-01-30 | 初始版本 |
| V2.0 | 2026-01-30 | 根据评审意见和 LiteLLM 集成方案优化架构 |

### V2.0 主要变更

```
1. 【API 设计】部署 API 简化为「性能等级」概念
2. 【API 设计】增加推理请求 API（/v1/chat/completions）
3. 【架构】引入 LiteLLM Gateway 双层架构
4. 【数据】引入异步 Job 机制
5. 【产品】增加部署模板概念
6. 【技术】简化 API Gateway 选型（apisix）
7. 【范围】明确 MVP 范围，延后高级功能
```

---

## 1. 项目概述

### 1.1 项目背景

基于 NVIDIA Dynamo 分布式 LLM 推理运行时，结合 LiteLLM 网关，构建企业级 LLM 推理服务平台。

**核心价值**：
- 简化 LLM 部署复杂度（ Dynamo CRD → 托管服务）
- 统一 API 暴露（OpenAI 兼容）
- 多租户隔离与配额管理
- 自服务运维能力

### 1.2 设计原则

| 原则 | 说明 |
|------|------|
| **API First** | 所有功能通过 RESTful API 暴露 |
| **分层解耦** | 管理面 / 推理面 / 执行面分离 |
| **云原生** | 基于 Kubernetes，实现声明式部署 |
| **开放兼容** | OpenAI 兼容 API，降低用户迁移成本 |
| **自研 + 开源** | 核心差异能力自研，通用能力复用 LiteLLM |

### 1.3 项目目标

| 目标 | 描述 | 成功指标 |
|------|------|----------|
| **简化部署** | 屏蔽 Kubernetes/Dynamo 复杂度 | 部署从 30 分钟缩短到 5 分钟 |
| **开放兼容** | OpenAI 兼容 API | 100% OpenAI SDK 兼容 |
| **多租户** | 租户隔离、配额控制 | 支持 100+ 租户 |
| **弹性伸缩** | 基于 SLA 的自动扩缩容 | TTFT p99 < 500ms |
| **可观测** | 统一指标、日志、告警 | 单一界面查看所有服务状态 |

---

## 2. 系统架构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              LLM 推理服务平台                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                     │
│  │   Web UI    │    │   SDK/CLI   │    │ OpenAI SDK  │                     │
│  │  (前端应用)  │    │  (客户端)   │    │ (官方 SDK)  │                     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                     │
│         │                   │                   │                            │
│         └───────────────────┴───────────────────┘                            │
│                                     │                                         │
│                          ┌──────────▼──────────┐                             │
│                          │   API Gateway        │                             │
│                          │   (apisix)          │                             │
│                          │  认证/鉴权/限流     │                             │
│                          └──────────┬──────────┘                             │
│                                     │                                        │
│  ┌──────────────────────────────────┴───────────────────────────────────┐   │
│  │                        Go Backend Service                             │   │
│  │                         (管理面 API :8001)                            │   │
│  │                                                                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │   │
│  │  │ 模型管理   │  │ 部署管理   │  │ 租户管理   │  │ 配额计费   │    │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘    │   │
│  │                                                                      │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐    │   │
│  │  │ 用户权限   │  │ 审计日志   │  │ 作业管理   │  │ 模板市场   │    │   │
│  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘    │   │
│  │                                                                      │   │
│  │  ┌────────────────────────────────────────────────────────────┐    │   │
│  │  │              Quota Check Middleware (配额拦截)              │    │   │
│  │  └────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                         │
│                                     ▼                                         │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      LiteLLM Gateway (:4000)                          │  │
│  │                         (推理面 API)                                   │  │
│  │                                                                        │  │
│  │  ┌────────────────────────────────────────────────────────────────┐   │  │
│  │  │              OpenAI 兼容 API (/v1/*)                           │   │  │
│  │  │   POST /chat/completions  POST /completions  POST /embeddings│   │  │
│  │  └────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                        │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │  │
│  │  │ 成本日志  │  │ RPM/TPM  │  │ 熔断器   │  │ 响应缓存 │             │  │
│  │  │ (DB)     │  │ 限流     │  │          │  │ (Redis)  │             │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                     │                                         │
└─────────────────────────────────────┼───────────────────────────────────────┘
                                      │
┌─────────────────────────────────────┼───────────────────────────────────────┐
│                        Kubernetes Cluster                                     │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    Dynamo Platform (dynamo-system)                    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │ Dynamo       │  │  etcd        │  │  NATS        │              │   │
│  │  │ Operator     │  │              │  │              │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    User Namespace (tenant-xxx)                         │   │
│  │                                                                        │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │   │
│  │  │ DynamoGraph  │  │ Frontend    │  │ Worker Pods  │              │   │
│  │  │ Deployment   │  │ Pod         │  │ Prefill/    │              │   │
│  │  │ (CRD)       │  │             │  │ Decode      │              │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘              │   │
│  │                                                                        │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 架构分层

| 层级 | 组件 | 职责 | 技术选型 |
|------|------|------|----------|
| **接入层** | API Gateway | 认证、鉴权、限流、日志 | apisix |
| **管理面** | Go Backend | 租户管理、部署管理、配额控制、审计 | Go + Gin |
| **推理面** | LiteLLM | 统一 API、成本日志、限流、缓存 | LiteLLM |
| **执行层** | Dynamo | 实际 LLM 推理执行 | Dynamo + vLLM/SGLang |
| **数据层** | PostgreSQL + Redis | 持久化、缓存、消息队列 | PostgreSQL + Redis |

### 2.3 职责边界

```
┌────────────────────────────────────────────────────────────────────────┐
│                           Go Backend 负责                              │
│                                                                        │
│  ✅ 模型/部署生命周期管理（CRUD）                                       │
│  ✅ DynamoGraphDeployment CRD 管理                                      │
│  ✅ 租户隔离与配额强控制（第一层拦截）                                   │
│  ✅ 精确计量与计费                                                      │
│  ✅ 用户/角色/权限管理                                                   │
│  ✅ 审计日志                                                            │
│  ✅ 作业管理（异步任务）                                                │
│  ✅ 部署模板管理                                                        │
│                                                                        │
├────────────────────────────────────────────────────────────────────────┤
│                         LiteLLM Gateway 负责                           │
│                                                                        │
│  ✅ 统一 LLM API 暴露（OpenAI 兼容）                                   │
│  ✅ 多模型路由                                                          │
│  ✅ 成本跟踪（Token 统计）                                              │
│  ✅ 细粒度限流（RPM/TPM，按模型/Key）                                  │
│  ✅ 熔断器/重试机制                                                     │
│  ✅ 响应缓存                                                            │
│  ✅ Prometheus 指标暴露                                                  │
│                                                                        │
├────────────────────────────────────────────────────────────────────────┤
│                          Dynamo 负责                                    │
│                                                                        │
│  ✅ disaggregated prefill/decode 架构                                 │
│  ✅ KV cache 传输（RDMA）                                              │
│  ✅ GPU 资源调度                                                        │
│  ✅ 实际 LLM 推理执行                                                  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 功能模块规划

### 3.1 功能模块总览

```
LLM 推理服务平台
│
├── 1. 模型管理 (Model Management)
│   ├── 1.1 模型注册/编辑/删除
│   ├── 1.2 模型版本管理
│   └── 1.3 模型配置
│
├── 2. 部署管理 (Deployment Management)
│   ├── 2.1 创建部署（简化 API）
│   ├── 2.2 更新部署
│   ├── 2.3 扩缩容（手动/自动）
│   ├── 2.4 部署状态监控
│   ├── 2.5 部署模板市场
│   └── 2.6 异步作业管理
│
├── 3. 推理 API (Inference API)
│   ├── 3.1 Chat Completions (OpenAI 兼容)
│   ├── 3.2 Text Completions (OpenAI 兼容)
│   ├── 3.3 Embeddings
│   ├── 3.4 请求查询与取消
│   └── 3.5 多模型路由
│
├── 4. 租户管理 (Tenant Management)
│   ├── 4.1 租户 CRUD
│   ├── 4.2 资源配额配置
│   └── 4.3 配额使用监控
│
├── 5. 用户与权限 (User & Access Control)
│   ├── 5.1 用户注册/登录
│   ├── 5.2 角色定义
│   ├── 5.3 API Key 管理
│   └── 5.4 权限控制
│
├── 6. 可观测性 (Observability)
│   ├── 6.1 指标看板
│   ├── 6.2 日志查询
│   └── 6.3 告警配置
│
└── 7. 运营管理 (Operations)
    ├── 7.1 审计日志
    ├── 7.2 系统健康检查
    └── 7.3 作业历史
```

### 3.2 MVP 范围（Phase 1-2）

**本版本实现**：

```
✅ 用户认证（登录/注册/JWT）
✅ 模型管理（CRUD）
✅ 部署管理（创建/查看/删除/扩缩容）
✅ 推理 API（/v1/chat/completions）
✅ 租户配额基础控制
✅ 基础指标监控
✅ 审计日志
✅ 部署模板
```

**延后到后续版本**：

```
❌ 模型版本管理与回滚
❌ 自动告警处理
❌ 精细化计费报表
❌ 多集群管理
❌ 多模型智能路由
```

---

## 4. API 接口设计

### 4.1 API 分层

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        管理面 API (Go Backend)                           │
│                           :8001 /api/v1                                  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 认证                                                         │   │
│  │   POST   /auth/register                                       │   │
│  │   POST   /auth/login                                          │   │
│  │   POST   /auth/logout                                         │   │
│  │   GET    /auth/me                                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 模型                                                         │   │
│  │   POST   /models                                             │   │
│  │   GET    /models                                             │   │
│  │   GET    /models/:id                                         │   │
│  │   PUT    /models/:id                                         │   │
│  │   DELETE /models/:id                                         │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 部署                                                         │   │
│  │   POST   /deployments                                         │   │
│  │   GET    /deployments                                         │   │
│  │   GET    /deployments/:id                                     │   │
│  │   PUT    /deployments/:id                                     │   │
│  │   DELETE /deployments/:id                                     │   │
│  │   POST   /deployments/:id/scale                               │   │
│  │   POST   /deployments/:id/restart                             │   │
│  │   GET    /deployments/:id/logs                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 作业                                                         │   │
│  │   GET    /jobs                                                │   │
│  │   GET    /jobs/:id                                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 租户（管理员）                                                 │   │
│  │   POST   /tenants                                             │   │
│  │   GET    /tenants                                             │   │
│  │   GET    /tenants/:id                                         │   │
│  │   PUT    /tenants/:id                                         │   │
│  │   GET    /tenants/:id/quota                                   │   │
│  │   PUT    /tenants/:id/quota                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 模板                                                         │   │
│  │   GET    /templates                                           │   │
│  │   GET    /templates/:id                                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 可观测性                                                      │   │
│  │   GET    /metrics                                             │   │
│  │   GET    /logs                                                │   │
│  │   GET    /alerts                                              │   │
│  │   POST   /alerts                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                        推理面 API (LiteLLM)                             │
│                           :4000 /v1                                     │
│                                                                         │
│  OpenAI 100% 兼容                                                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 推理                                                         │   │
│  │   POST   /chat/completions                                    │   │
│  │   POST   /completions                                         │   │
│  │   POST   /embeddings                                          │   │
│  │   GET    /models                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 请求管理                                                      │   │
│  │   (LiteLLM 内置，仅透传)                                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 统一响应格式

**成功响应**：

```json
{
    "success": true,
    "data": { ... },
    "metadata": {
        "request_id": "req_abc123",
        "timestamp": "2026-01-30T10:00:00Z"
    }
}
```

**错误响应**：

```json
{
    "success": false,
    "error": {
        "code": "DEPLOYMENT_NOT_FOUND",
        "message": "Deployment 'dep-123' not found"
    },
    "metadata": {
        "request_id": "req_abc123",
        "timestamp": "2026-01-30T10:00:00Z"
    }
}
```

### 4.3 认证方式

| 方式 | 用途 | Header |
|------|------|--------|
| **JWT Token** | 用户访问管理 API | `Authorization: Bearer <jwt_token>` |
| **API Key** | 服务间调用/推理 API | `Authorization: Bearer <api_key>` |

---

## 5. 核心 API 详细设计

### 5.1 创建部署（简化 API）

**设计理念**：封装 Dynamo 复杂配置为「性能等级」，降低用户门槛。

**请求**：

```http
POST /api/v1/deployments
Authorization: Bearer <token>
Content-Type: application/json

{
    "name": "qwen-chat-32b",
    "display_name": "Qwen 32B Chat 部署",
    "description": "生产环境对话模型",
    "model_id": "model_abc123",
    
    // 简化配置：性能等级
    "performance_tier": "standard",
    
    // 或高级配置（可选）
    "advanced_config": {
        "serving_mode": "disaggregated",
        "prefill_replicas": 2,
        "decode_replicas": 1,
        "gpu_per_worker": 2
    },
    
    // 扩缩容配置
    "scaling": {
        "auto_scaling_enabled": true,
        "min_replicas": 1,
        "max_replicas": 5,
        "metrics": [
            {"name": "ttft", "threshold": 1000, "direction": "up", "adjustment": 1}
        ]
    }
}
```

**性能等级定义**：

| 等级 | 说明 | 适用场景 | 配置 |
|------|------|----------|------|
| `economy` | 最小配置，成本优先 | 测试/开发 | 单卡，聚合部署 |
| `standard` | 推荐配置 | 生产轻负载 | 2 GPU，聚合部署，自动扩缩容 |
| `high_performance` | 高性能配置 | 生产高负载 | 解耦部署，RDMA，自动扩缩容 |
| `enterprise` | 顶级配置 | 大规模部署 | 多节点，顶级 GPU，自动扩缩容 |

**响应**：

```json
{
    "success": true,
    "data": {
        "id": "dep_xyz789",
        "name": "qwen-chat-32b",
        "status": "pending",
        "job_id": "job_abc123",
        "message": "Deployment creation initiated",
        "created_at": "2026-01-30T10:00:00Z"
    },
    "metadata": {
        "request_id": "req_abc123"
    }
}
```

### 5.2 查询作业状态

由于部署创建是异步操作，提供作业查询接口：

```http
GET /api/v1/jobs/job_abc123
Authorization: Bearer <token>
```

**响应**：

```json
{
    "success": true,
    "data": {
        "id": "job_abc123",
        "type": "deployment_create",
        "target_id": "dep_xyz789",
        "status": "running",
        "progress": {
            "current_step": "creating_workers",
            "total_steps": 4,
            "steps_completed": 2,
            "message": "Creating prefill workers..."
        },
        "result": null,
        "error": null,
        "created_at": "2026-01-30T10:00:00Z",
        "updated_at": "2026-01-30T10:02:00Z",
        "completed_at": null
    }
}
```

**作业状态机**：

```
pending → running → completed
              ↓
           failed
              
作业类型：
- deployment_create
- deployment_update
- deployment_scale
- deployment_delete
- model_import
```

### 5.3 获取部署详情

```http
GET /api/v1/deployments/dep_xyz789
Authorization: Bearer <token>
```

**响应**：

```json
{
    "success": true,
    "data": {
        "id": "dep_xyz789",
        "name": "qwen-chat-32b",
        "display_name": "Qwen 32B Chat 部署",
        "model": {
            "id": "model_abc123",
            "name": "Qwen/Qwen3-32B-FP8",
            "framework": "vllm"
        },
        "status": "running",
        "tier": "high_performance",
        
        "endpoint": {
            "internal_url": "http://dynamo-frontend.svc:8000",
            "external_url": "http://qwen-chat-32b.example.com",
            "api_format": "openai"
        },
        
        "replicas": {
            "prefill": 2,
            "decode": 1
        },
        
        "resource_usage": {
            "gpu_utilization": 78.5,
            "memory_usage": 65.2,
            "active_requests": 8,
            "queue_depth": 3
        },
        
        "conditions": [
            {"type": "Ready", "status": "True", "message": ""},
            {"type": "PrefillWorkersReady", "status": "True"},
            {"type": "DecodeWorkersReady", "status": "True"}
        ],
        
        "created_at": "2026-01-30T10:00:00Z",
        "deployed_at": "2026-01-30T10:05:00Z"
    }
}
```

### 5.4 扩缩容部署

```http
POST /api/v1/deployments/dep_xyz789/scale
Authorization: Bearer <token>
Content-Type: application/json

{
    "prefill_replicas": 3,
    "decode_replicas": 2,
    "reason": "Traffic increase"
}
```

### 5.5 推理 API（OpenAI 兼容）

**直接透传到 LiteLLM**，完全兼容 OpenAI API：

```http
POST /v1/chat/completions
Authorization: Bearer <api_key>
Content-Type: application/json

{
    "model": "qwen-chat-32b",     // 部署名称作为模型名
    "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello!"}
    ],
    "temperature": 0.7,
    "max_tokens": 1000,
    "stream": false
}
```

**响应（OpenAI 格式）**：

```json
{
    "id": "chatcmpl_abc123",
    "object": "chat.completion",
    "created": 1706600000,
    "model": "qwen-chat-32b",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": "Hello! How can I help you today?"
            },
            "finish_reason": "stop"
        }
    ],
    "usage": {
        "prompt_tokens": 20,
        "completion_tokens": 15,
        "total_tokens": 35
    }
}
```

---

## 6. 数据模型设计

### 6.1 数据库架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        PostgreSQL                                 │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      认证授权层                            │   │
│  │                                                              │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │   │
│  │  │ tenants  │  │  users  │  │api_keys │  │ roles   │   │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      应用数据层                            │   │
│  │                                                              │   │
│  │  ┌─────────┐  ┌────────────┐  ┌──────────────────┐      │   │
│  │  │ models  │  │deployments │  │ deployment_templates│    │   │
│  │  └─────────┘  └─────┬──────┘  └──────────────────┘      │   │
│  │                     │                                    │   │
│  │                     ▼                                    │   │
│  │              ┌────────────┐                            │   │
│  │              │    jobs    │                            │   │
│  │              └────────────┘                            │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      运营数据层                            │   │
│  │                                                              │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐         │   │
│  │  │audit_logs │  │quota_usage │  │ alert_rules │         │   │
│  │  └────────────┘  └────────────┘  └────────────┘         │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 核心表结构

#### 6.2.1 租户表 (tenants)

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(255),
    description TEXT,
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    
    -- 配额配置
    quota JSONB NOT NULL DEFAULT '{
        "max_deployments": 10,
        "max_gpus": 32,
        "max_storage_gb": 500,
        "max_monthly_spend": 10000.00
    }',
    
    -- LiteLLM Team 映射
    litellm_team_id VARCHAR(255),
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE
);
```

#### 6.2.2 模型表 (models)

```sql
CREATE TABLE models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    
    name VARCHAR(255) NOT NULL,
    display_name VARCHAR(255),
    description TEXT,
    
    -- 模型配置
    model_type VARCHAR(50) NOT NULL,      -- llm, embedding
    framework VARCHAR(50) NOT NULL,       -- vllm, sglang, trtllm
    model_path VARCHAR(500) NOT NULL,     -- HuggingFace 路径
    
    -- 运行时配置
    model_config JSONB NOT NULL DEFAULT '{}',  -- tensor_parallel_size, max_model_len 等
    
    -- 对外暴露名称
    public_name VARCHAR(255) NOT NULL,
    
    -- 状态
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    UNIQUE(tenant_id, name)
);
```

#### 6.2.3 部署表 (deployments)

```sql
CREATE TABLE deployments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    model_id UUID NOT NULL REFERENCES models(id),
    
    name VARCHAR(255) NOT NULL,
    display_name VARCHAR(255),
    description TEXT,
    
    -- 性能等级
    performance_tier VARCHAR(50) NOT NULL DEFAULT 'standard',
    
    -- 运行时配置（实际应用的配置）
    deployment_config JSONB NOT NULL DEFAULT '{}',
    scaling_config JSONB NOT NULL DEFAULT '{}',
    
    -- 扩缩容配置
    replicas JSONB NOT NULL DEFAULT '{"prefill": 1, "decode": 1}',
    
    -- 状态
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    status_message TEXT,
    
    -- Kubernetes 资源
    kubernetes_namespace VARCHAR(255),
    kubernetes_dgd_name VARCHAR(255),
    
    -- 端点信息
    endpoint JSONB DEFAULT '{}',
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deployed_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE,
    
    UNIQUE(tenant_id, name)
);

CREATE INDEX idx_deployments_tenant_id ON deployments(tenant_id);
CREATE INDEX idx_deployments_status ON deployments(status);
CREATE INDEX idx_deployments_public_name ON deployments(public_name);
```

#### 6.2.4 作业表 (jobs)

```sql
CREATE TABLE jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    user_id UUID NOT NULL REFERENCES users(id),
    
    -- 作业类型
    type VARCHAR(50) NOT NULL,  -- deployment_create, deployment_scale, etc.
    target_type VARCHAR(50) NOT NULL,  -- deployment, model
    target_id VARCHAR(255) NOT NULL,
    
    -- 作业状态
    status VARCHAR(50) NOT NULL DEFAULT 'pending',  -- pending, running, completed, failed
    progress JSONB DEFAULT '{
        "current_step": "",
        "total_steps": 0,
        "steps_completed": 0,
        "message": ""
    }',
    
    -- 输入/输出
    input JSONB NOT NULL DEFAULT '{}',
    result JSONB,
    error JSONB,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_jobs_tenant_id ON jobs(tenant_id);
CREATE INDEX idx_jobs_status ON jobs(status);
CREATE INDEX idx_jobs_target ON jobs(target_type, target_id);
```

#### 6.2.5 部署模板表 (deployment_templates)

```sql
CREATE TABLE deployment_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    name VARCHAR(255) NOT NULL,
    description TEXT,
    
    -- 模板类型
    template_type VARCHAR(50) NOT NULL,  -- quick_chat, production, code_assistant
    model_type VARCHAR(50) NOT NULL,      -- llm, embedding
    
    -- 预设配置
    base_config JSONB NOT NULL DEFAULT '{}',
    recommended_scaling JSONB DEFAULT '{}',
    alert_thresholds JSONB DEFAULT '{}',
    
    -- 官方模板
    is_official BOOLEAN DEFAULT false,
    
    -- 使用统计
    usage_count INT DEFAULT 0,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 预置模板
INSERT INTO deployment_templates (name, template_type, is_official, base_config) VALUES
('Quick Chat', 'quick_chat', true, '{"tier": "economy", "serving_mode": "aggregated"}'),
('Production Chat', 'production', true, '{"tier": "standard", "serving_mode": "aggregated", "auto_scaling": true}'),
('High Performance', 'high_performance', true, '{"tier": "high_performance", "serving_mode": "disaggregated", "rdma": true}');
```

#### 6.2.6 配额使用表 (quota_usage)

```sql
CREATE TABLE quota_usage (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    
    -- 统计周期
    period VARCHAR(20) NOT NULL,  -- monthly, daily
    period_start TIMESTAMP WITH TIME ZONE NOT NULL,
    period_end TIMESTAMP WITH TIME ZONE NOT NULL,
    
    -- 使用量
    total_spend DECIMAL(12, 4) DEFAULT 0,
    total_tokens INT DEFAULT 0,
    input_tokens INT DEFAULT 0,
    output_tokens INT DEFAULT 0,
    request_count INT DEFAULT 0,
    
    -- GPU 使用
    gpu_hours DECIMAL(10, 2) DEFAULT 0,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(tenant_id, period, period_start)
);

CREATE INDEX idx_quota_usage_tenant_period ON quota_usage(tenant_id, period_start);
```

### 6.3 Redis 缓存设计

```
# Session 管理
session:{session_id} -> {user_id, tenant_id, expires_at}  TTL: 24h

# API Key 缓存
apikey:{key_hash} -> {user_id, tenant_id, allowed_models, rpm, tpm}  TTL: 5min

# 部署状态缓存
deployment:status:{deployment_id} -> {status, replicas, resource_usage}  TTL: 30s

# 限流计数器
ratelimit:{tenant_id}:{endpoint} -> count  TTL: 60s

# 作业队列
job:queue -> [job_id1, job_id2, ...]  LIST

# 作业状态
job:status:{job_id} -> {status, progress}  TTL: 24h
```

---

## 7. 异步作业机制

### 7.1 为什么需要异步作业

| 操作 | 同步等待问题 | 异步方案优势 |
|------|-------------|-------------|
| 创建部署 | 5+ 分钟超时 | 返回 job_id，立即响应 |
| 扩缩容 | 2+ 分钟超时 | 后台执行，可查询进度 |
| 模型导入 | 分钟级 | 后台执行，不阻塞 UI |

### 7.2 作业流程

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│   API    │────>│  Redis   │────>│  Worker  │────>│   K8s    │
│          │     │  Queue    │     │          │     │   API    │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
     │                │                │                │
     │ 1. 提交作业   │ 2. 入队        │ 3. 取出执行     │
     │               │                │                 │
     │<──────────────│                │                 │
     │ 4. 返回 job_id │                │                 │
     │               │                │                 │
     │               │                │ 5. 更新状态      │
     │               │<───────────────│                 │
     │               │ 6. 状态更新     │                 │
```

### 7.3 Worker 实现

```go
type JobWorker struct {
    redis     *redis.Client
    db        *gorm.DB
    k8sClient *kubernetes.Clientset
}

func (w *JobWorker) Start() {
    // 启动多个 worker
    for i := 0; i < w.cfg.WorkerCount; i++ {
        go w.runWorker(i)
    }
}

func (w *JobWorker) runWorker(id int) {
    for {
        // 从队列获取作业
        jobID, err := w.redis.BLPop(context.Background(), 0, "job:queue").Result()
        if err != nil {
            continue
        }
        
        // 处理作业
        w.processJob(jobID)
    }
}

func (w *JobWorker) processJob(jobID string) {
    // 1. 获取作业信息
    job, err := w.getJob(jobID)
    if err != nil {
        return
    }
    
    // 2. 更新状态为 running
    w.updateJobStatus(job, "running")
    
    // 3. 根据类型执行
    switch job.Type {
    case "deployment_create":
        w.createDeployment(job)
    case "deployment_scale":
        w.scaleDeployment(job)
    case "deployment_delete":
        w.deleteDeployment(job)
    }
}

func (w *JobWorker) createDeployment(job *Job) {
    // 1. 更新进度
    w.updateProgress(job, "creating_crd", 1, 4, "Creating CRD...")
    
    // 2. 创建 DynamoGraphDeployment CRD
    dgd, err := w.createDGD(job)
    if err != nil {
        w.failJob(job, err)
        return
    }
    
    // 3. 等待 workers 创建
    w.updateProgress(job, "creating_workers", 2, 4, "Creating workers...")
    w.waitForWorkers(dgd)
    
    // 4. 注册到 LiteLLM
    w.updateProgress(job, "registering_model", 3, 4, "Registering to gateway...")
    w.registerToLiteLLM(job)
    
    // 5. 完成
    w.completeJob(job, map[string]interface{}{
        "deployment_id": dgd.Name,
        "endpoint":      dgd.Status.Endpoint,
    })
}
```

### 7.4 作业状态查询

```http
GET /api/v1/jobs/{job_id}
```

```json
{
    "success": true,
    "data": {
        "id": "job_abc123",
        "type": "deployment_create",
        "target_id": "dep_xyz789",
        "status": "running",
        "progress": {
            "current_step": "creating_workers",
            "total_steps": 4,
            "steps_completed": 2,
            "message": "Creating prefill workers (2/2)..."
        },
        "created_at": "2026-01-30T10:00:00Z"
    }
}
```

---

## 8. 配额控制机制

### 8.1 两层配额控制

```
┌─────────────────────────────────────────────────────────────────┐
│                         第一层：Go Backend 配额拦截               │
│                         （租户级强控制）                          │
│                                                                  │
│   检查逻辑：                                                     │
│   1. 从 JWT/Key 获取 tenant_id                                  │
│   2. 查询 quota_usage 当前周期使用量                             │
│   3. 估算本次请求成本                                            │
│   4. 如果 remaining_budget < estimated_cost → 拒绝 (429)         │
│                                                                  │
│   拒绝条件：                                                     │
│   - 月度支出超限                                                │
│   - GPU 小时超限                                                │
│   - 部署数量超限                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 通过
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         第二层：LiteLLM 细粒度限流               │
│                         （模型/Key 级控制）                       │
│                                                                  │
│   配置：                                                        │
│   - RPM: 每分钟请求数                                           │
│   - TPM: 每分钟 Token 数                                        │
│                                                                  │
│   拒绝条件：                                                     │
│   - 单个 Key 请求超限                                           │
│   - 单个模型请求超限                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 配额检查中间件

```go
func QuotaCheckMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        claims, exists := c.Get("claims")
        if !exists {
            c.Next()
            return
        }
        
        tenantID := claims.TenantID
        
        // 1. 检查月度支出配额
        usage, err := quotaService.GetCurrentUsage(tenantID)
        if err != nil {
            c.AbortWithError(500, err)
            return
        }
        
        quota, err := quotaService.GetQuota(tenantID)
        if err != nil {
            c.AbortWithError(500, err)
            return
        }
        
        // 2. 检查是否超限
        if usage.TotalSpend >= quota.MaxMonthlySpend {
            c.AbortWithStatusJSON(429, Error{
                Code:    "QUOTA_EXCEEDED",
                Message: "Monthly spend quota exceeded",
                Details: map[string]interface{}{
                    "used":    usage.TotalSpend,
                    "limit":   quota.MaxMonthlySpend,
                    "reset_at": usage.PeriodEnd,
                },
            })
            return
        }
        
        // 3. 估算本次请求成本
        reqCost := estimateRequestCost(c.Request, claims)
        if usage.RemainingBudget() < reqCost {
            c.AbortWithStatusJSON(429, Error{
                Code:    "INSUFFICIENT_BUDGET",
                Message: "Estimated cost exceeds remaining budget",
            })
            return
        }
        
        c.Next()
    }
}
```

### 8.3 成本估算

```go
func estimateRequestCost(req *http.Request, claims *Claims) float64 {
    // 解析请求体获取 token 数量
    var body ChatRequest
    if err := json.NewDecoder(req.Body).Decode(&body); err != nil {
        return 0
    }
    
    // 估算 token 数
    inputTokens := estimateTokens(body.Messages)
    maxTokens := intVal(body.MaxTokens, 1000)
    outputTokens := maxTokens // 预估
    
    // 获取模型单价
    price := getModelPrice(body.Model)
    
    // 计算成本
    cost := float64(inputTokens)*price.InputPerToken + 
            float64(outputTokens)*price.OutputPerToken
    
    return cost
}
```

---

## 9. 技术栈

### 9.1 技术选型

| 组件 | 技术选型 | 版本 | 说明 |
|------|----------|------|------|
| **语言** | Go | 1.21+ | 高性能、与 Dynamo 生态一致 |
| **框架** | Gin | v1.9+ | 轻量级、高性能 |
| **ORM** | GORM | v1.25+ | 功能完整 |
| **数据库** | PostgreSQL | 15+ | 可靠、功能强 |
| **缓存** | Redis | 7.0+ | 会话、限流、缓存 |
| **消息队列** | Redis Streams | - | 作业队列 |
| **API Gateway** | apisix | 3.4+ | 轻量、功能完善 |
| **LLM 网关** | LiteLLM | main | 统一 API |
| **指标** | Prometheus + Grafana | - | 可观测性 |
| **日志** | Loki | 2.9+ | 日志聚合 |
| **容器** | Docker + Kubernetes | 24+ / 1.28+ | 基础设施 |

### 9.2 项目结构

```
llm-platform/
├── cmd/
│   ├── api/                    # API 服务入口
│   │   └── main.go
│   └── worker/                 # Worker 服务入口
│       └── main.go
│
├── internal/
│   ├── config/                 # 配置
│   │   └── config.go
│   │
│   ├── handler/                # HTTP handlers
│   │   ├── auth.go
│   │   ├── model.go
│   │   ├── deployment.go
│   │   ├── job.go
│   │   ├── tenant.go
│   │   └── template.go
│   │
│   ├── middleware/             # 中间件
│   │   ├── auth.go
│   │   ├── quota.go
│   │   ├── logging.go
│   │   └── ratelimit.go
│   │
│   ├── service/                # 业务逻辑
│   │   ├── auth.go
│   │   ├── model.go
│   │   ├── deployment.go
│   │   ├── job.go
│   │   ├── quota.go
│   │   ├── lite_llm.go       # LiteLLM 集成
│   │   └── dynamo.go         # Dynamo CRD 管理
│   │
│   ├── repository/            # 数据访问
│   │   ├── user.go
│   │   ├── tenant.go
│   │   ├── model.go
│   │   ├── deployment.go
│   │   ├── job.go
│   │   └── audit.go
│   │
│   ├── worker/                # 后台作业
│   │   ├── worker.go
│   │   └── deployment.go
│   │
│   └── pkg/                   # 公共包
│       ├── errors/
│       ├── response/
│       └── utils/
│
├── migrations/                # 数据库迁移
│   └── 001_init.sql
│
├── configs/
│   ├── api.yaml
│   └── worker.yaml
│
├── deploy/
│   ├── kubernetes/
│   │   ├── api/
│   │   ├── worker/
│   │   ├── apisix/
│   │   └── litellm/
│   └── docker-compose.yaml
│
└── go.mod
```

---

## 10. 部署架构

### 10.1 开发环境

```yaml
# docker-compose.yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: llm_platform
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  api:
    build: ./cmd/api
    ports:
      - "8001:8001"
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/llm_platform
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  worker:
    build: ./cmd/worker
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/llm_platform
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis

  apisix:
    image: apache/apisix:3.4.0-debian
    ports:
      - "8000:9080"
      - "8443:9443"
    environment:
      APISIX_STAND_ALONE: "true"
    volumes:
      - ./deploy/apisix/config.yaml:/usr/local/apisix/conf/config.yaml

  litellm:
    image: ghcr.io/berriai/litellm:main
    ports:
      - "4000:4000"
    environment:
      DATABASE_URL: postgresql://postgres:password@postgres:5432/litellm
      REDIS_HOST: redis
      REDIS_PORT: 6379
      LITELLM_MASTER_KEY: sk-test
    depends_on:
      - redis

volumes:
  postgres_data:
  redis_data:
```

### 10.2 生产 Kubernetes 部署

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         生产环境架构                                     │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      Ingress Layer                                 │  │
│  │                      (apisix Ingress)                              │  │
│  └───────────────────────────┬───────────────────────────────────────┘  │
│                              │                                           │
│  ┌───────────────────────────▼───────────────────────────────────────┐  │
│  │                    LLM Platform Namespace                           │  │
│  │                                                                      │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │  │
│  │  │  API Service   │  │  Worker        │  │ LiteLLM       │        │  │
│  │  │  (3 replicas)  │  │  (2 replicas)  │  │ (3 replicas)  │        │  │
│  │  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘        │  │
│  │          │                    │                    │                │  │
│  │          └────────────────────┼────────────────────┘                │  │
│  │                               │                                     │  │
│  │  ┌────────────────┐  ┌───────▼────────┐  ┌────────────────┐      │  │
│  │  │  PostgreSQL    │  │    Redis       │  │   apisix       │      │  │
│  │  │  (HA)          │  │    Cluster     │  │   (3 nodes)    │      │  │
│  │  └────────────────┘  └───────────────┘  └────────────────┘      │  │
│  │                                                                      │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                              │                                           │
│  ┌───────────────────────────▼───────────────────────────────────────┐  │
│  │                   Dynamo Namespace (dynamo-system)                  │  │
│  │                                                                      │  │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐       │  │
│  │  │ Dynamo         │  │  etcd          │  │  NATS          │       │  │
│  │  │ Operator       │  │                │  │                │       │  │
│  │  └────────────────┘  └────────────────┘  └────────────────┘       │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                              │                                           │
│  ┌───────────────────────────▼───────────────────────────────────────┐  │
│  │                    User Namespaces                                  │  │
│  │                                                                      │  │
│  │  ┌────────────────────────────────────────────────────────────┐    │  │
│  │  │  Tenant: tenant-xxx                                        │    │  │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │  │
│  │  │  │ DGD (CRD)   │  │ Frontend Pod │  │ Worker Pods │   │    │  │
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │  │
│  │  └────────────────────────────────────────────────────────────┘    │  │
│  │                                                                      │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 11. 实施计划

### 11.1 MVP 实施计划（12 周）

| 周 | 阶段 | 交付内容 | 里程碑 |
|----|------|----------|--------|
| 1-2 | 基础设施 | Go 项目框架、DB 迁移、配置管理、部署脚本 | M1 |
| 3-4 | 认证 + 模型 | 用户认证、模型 CRUD、API Key | M1 |
| 5-6 | 部署管理 | 创建/删除/状态、作业机制、LiteLLM 集成 | M2 |
| 7-8 | 扩缩容 + 配额 | 手动扩缩容、配额检查中间件、成本估算 | M2 |
| 9-10 | 可观测性 | 指标暴露、Grafana Dashboard、日志集成 | M3 |
| 11-12 | 完善 + 测试 | 单元测试、集成测试、部署模板、文档 | M3 |

### 11.2 里程碑定义

| 里程碑 | 时间 | 验收标准 |
|--------|------|----------|
| **M1: 基础可用** | Week 4 | 用户登录、模型注册、API Key 可用 |
| **M2: 核心功能** | Week 8 | 部署创建/删除、扩缩容、配额控制 |
| **M3: 生产可用** | Week 12 | 监控告警、部署模板、文档完善 |

### 11.3 团队规模

| 角色 | 数量 | 职责 |
|------|------|------|
| 后端开发 | 2-3 | Go Backend + Worker |
| K8s 运维 | 1 | Kubernetes 部署、运维 |
| 测试 | 1 | 测试、CI/CD |
| PM | 0.5 | 项目管理 |

---

## 12. 风险与缓解

### 12.1 技术风险

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| LiteLLM 功能受限 | 中 | 低 | 提前评估，开源自 fork 能力 |
| Dynamo CRD 变更 | 高 | 中 | 版本隔离，定期同步 |
| 性能目标不达标 | 中 | 低 | 早期 POC，识别瓶颈 |
| 多租户隔离不完善 | 高 | 低 | 安全审计，权限最小化 |

### 12.2 运营风险

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| 需求变更 | 中 | 高 | 敏捷迭代，MVP 聚焦 |
| 上游依赖变更 | 中 | 中 | 版本锁定，回归测试 |

---

## 13. 附录

### 13.1 API 错误码

| 错误码 | HTTP 状态 | 说明 |
|--------|----------|------|
| `UNAUTHORIZED` | 401 | 未认证 |
| `FORBIDDEN` | 403 | 无权限 |
| `NOT_FOUND` | 404 | 资源不存在 |
| `QUOTA_EXCEEDED` | 429 | 配额超限 |
| `RATE_LIMITED` | 429 | 请求过于频繁 |
| `VALIDATION_ERROR` | 400 | 请求参数错误 |
| `INTERNAL_ERROR` | 500 | 内部错误 |

### 13.2 术语表

| 术语 | 说明 |
|------|------|
| DGD | DynamoGraphDeployment，Kubernetes CRD |
| Performance Tier | 性能等级，简化部署配置 |
| Job | 异步作业，用于长时操作 |
| Quota | 租户资源配额 |
| LiteLLM | LLM 网关，提供统一 API |

### 13.3 参考资料

- [Dynamo 文档](https://github.com/ai-dynamo/dynamo)
- [LiteLLM 文档](https://docs.litellm.ai/)
- [apisix 文档](https://apisix.apache.org/)

---

**文档结束**
