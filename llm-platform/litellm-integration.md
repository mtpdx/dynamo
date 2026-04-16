# LiteLLM 集成评估报告

## 一、LiteLLM 概述

### 1.1 什么是 LiteLLM

LiteLLM 是一个**LLM 网关/代理**，通过统一接口暴露 100+ LLM 提供商的 API，同时提供 OpenAI 兼容的 API 格式。

```
┌─────────────────────────────────────────────────────────────────┐
│                        LiteLLM Gateway                          │
│                                                                  │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │  OpenAI   │ │ Anthropic  │ │  VLLM      │ │  SGLang    │  │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘  │
│         │              │              │              │           │
│         └──────────────┴──────────────┴──────────────┘           │
│                                │                                  │
│                    ┌───────────▼───────────┐                     │
│                    │   Unified API Layer    │                     │
│                    │  /v1/chat/completions │                     │
│                    │  /v1/completions      │                     │
│                    │  /v1/embeddings       │                     │
│                    └───────────┬───────────┘                     │
│                                │                                 │
│                    ┌───────────▼───────────┐                     │
│                    │   OpenAI Compatible   │                     │
│                    │   API Response       │                     │
│                    └───────────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心特性

| 特性 | 说明 |
|------|------|
| **统一接口** | 一个 API 调用任何 LLM（OpenAI、Anthropic、VLLM、SGLang 等） |
| **OpenAI 兼容** | 100% 兼容 OpenAI API 格式，零代码修改迁移 |
| **成本跟踪** | 自动记录每次调用的 token 消耗和成本 |
| **限流** | 按用户/组织/API Key 设置 RPM/TPM 限制 |
| **熔断器** | 后端故障时自动熔断，防止雪崩 |
| **重试机制** | 自动重试失败请求 |
| **Cache** | Redis 缓存响应，减少重复调用 |
| **多模型路由** | 基于成本/延迟/可用性的智能路由 |
| **流式输出** | 完整的 Server-Sent Events 支持 |

---

## 二、LiteLLM 能补全什么

### 2.1 功能映射分析

根据当前设计缺失项，评估 LiteLLM 的补全能力：

| 缺失功能 | 当前设计状态 | LiteLLM 补全 | 补全程度 |
|----------|-------------|--------------|----------|
| **推理请求 API** | ❌ 缺失 | ✅ 提供完整 `/v1/chat/completions` 等接口 | **90%** |
| **OpenAI 兼容** | ❌ 无 | ✅ 原生支持 | **100%** |
| **成本跟踪** | ❌ 缺失 | ✅ 内置成本日志 | **80%** |
| **请求限流** | ❌ 缺失 | ✅ RPM/TPM 控制 | **90%** |
| **多模型编排** | ❌ 缺失 | ✅ 模型路由 | **85%** |
| **断路器/重试** | ❌ 缺失 | ✅ 内置 | **100%** |
| **API 响应缓存** | ❌ 缺失 | ✅ Redis 集成 | **100%** |
| **Prometheus 指标** | ⚠️ 部分 | ✅ 完整暴露 | **100%** |
| **对话历史** | ❌ 缺失 | ⚠️ 仅基础会话 | **30%** |
| **多租户隔离** | ⚠️ 需配置 | ⚠️ 支持但需定制 | **50%** |
| **配额精确计量** | ⚠️ 基础 | ⚠️ 基础日志 | **60%** |

### 2.2 补全优先级矩阵

```
                    ┌─────────────────┬─────────────────┐
                    │   高业务价值     │   低业务价值     │
        ┌───────────┼─────────────────┼─────────────────┤
        │   高补全度  │  ★★★★★ 优先使用  │  ★★★ 可选使用   │
        │           │  推理API/限流   │  缓存/重试      │
        ├───────────┼─────────────────┼─────────────────┤
        │   低补全度  │  ★★★★ 配合自研  │  ★★ 自行实现   │
        │           │  多租户/配额    │  对话历史      │
        └───────────┴─────────────────┴─────────────────┘
```

---

## 三、LiteLLM 架构集成方案

### 3.1 推荐架构：Dynamo + LiteLLM 双层架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           LLM 推理服务平台                                   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        平台管理层 (Go Backend)                        │   │
│  │                                                                       │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐           │   │
│  │  │ 模型管理  │  │ 部署管理  │  │ 租户管理  │  │ 配额计费  │           │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                    ┌─────────────────┴─────────────────┐                    │
│                    │         LiteLLM Gateway            │                    │
│                    │                                    │                    │
│                    │  ┌──────────────────────────────┐│                    │
│                    │  │    /v1/chat/completions     ││                    │
│                    │  │    /v1/completions         ││                    │
│                    │  │    /v1/embeddings          ││                    │
│                    │  └──────────────────────────────┘│                    │
│                    │                                    │                    │
│                    │  ┌────────┐  ┌────────┐  ┌──────┐│                    │
│                    │  │ Cost   │  │ Rate   │  │Cache ││                    │
│                    │  │ Track  │  │ Limit  │  │      ││                    │
│                    │  └────────┘  └────────┘  └──────┘│                    │
│                    └─────────────────┬─────────────────┘                    │
│                                        │                                     │
│    ┌───────────────────────────────────┼───────────────────────────────┐    │
│    │                      推理引擎层 (Dynamo + Backend)                  │    │
│    │                                                               │    │
│    │  ┌─────────────────┐        ┌─────────────────┐              │    │
│    │  │  Dynamo Frontend │        │  Dynamo Frontend │              │    │
│    │  │  (Model A)       │        │  (Model B)       │              │    │
│    │  └────────┬────────┘        └────────┬────────┘              │    │
│    │           │                            │                      │    │
│    │  ┌────────┴────────┐        ┌────────┴────────┐              │    │
│    │  │  Prefill Worker │        │  Prefill Worker │              │    │
│    │  │  Decode Worker  │        │  Decode Worker  │              │    │
│    │  └─────────────────┘        └─────────────────┘              │    │
│    │                                                               │    │
│    └───────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 职责边界

```
┌────────────────────────────────────────────────────────────────────────┐
│                          LiteLLM 负责                                  │
│                                                                        │
│  ✅ LLM API 统一暴露（OpenAI 兼容）                                    │
│  ✅ 请求路由（模型 A → 模型 B）                                        │
│  ✅ 成本跟踪（Token 统计）                                            │
│  ✅ 限流（RPM/TPM 控制）                                              │
│  ✅ 熔断器（后端故障保护）                                            │
│  ✅ 重试机制（失败自动重试）                                          │
│  ✅ 响应缓存（相同请求加速）                                          │
│  ✅ 基础指标（Prometheus 格式）                                       │
│  ✅ Key 认证（API Key 验证）                                          │
│                                                                        │
├────────────────────────────────────────────────────────────────────────┤
│                          Go Backend 负责                               │
│                                                                        │
│  ✅ 模型/部署生命周期管理                                              │
│  ✅ DynamoGraphDeployment CRD 管理                                     │
│  ✅ 多租户隔离与配额强控制                                            │
│  ✅ 精确计量与计费                                                     │
│  ✅ 用户/角色/权限管理                                                │
│  ✅ 审计日志                                                          │
│  ✅ 告警管理                                                          │
│  ✅ 控制台 API（管理面）                                              │
│                                                                        │
├────────────────────────────────────────────────────────────────────────┤
│                          Dynamo 负责                                   │
│                                                                        │
│  ✅ 实际 LLM 推理执行                                                  │
│  ✅ disaggregated prefill/decode 架构                                 │
│  ✅ KV cache 传输                                                     │
│  ✅ GPU 资源调度                                                       │
│  ✅ 请求处理优化                                                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 四、LiteLLM 配置设计

### 4.1 LiteLLM 配置文件

```yaml
# litellm/config.yaml

model_list:
  # vLLM Backend (Dynamo)
  - model_name: qwen-32b
    litellm_params:
      model: vllm/qwen-32b
      api_base: http://dynamo-frontend.default.svc.cluster.local:8000
      rpm: 100
      tpm: 100000
      timeout: 600
      max_retries: 2
    
  # SGLang Backend (Dynamo)
  - model_name: llama-70b
    litellm_params:
      model: sglang/llama-70b
      api_base: http://dynamo-sglang.default.svc.cluster.local:8000
      rpm: 50
      tpm: 50000

  # 代理到 OpenAI (如有需要)
  - model_name: gpt-4
    litellm_params:
      model: gpt-4
      api_key: os.environ/OPENAI_API_KEY

# 通用限流配置
rate_limit:
  default_rpm: 60
  default_tpm: 90000

# Litellm Server 配置
litellm_settings:
  drop_params: true
  set_verbose: false
  json_logs: false
  
# Database for cost tracking
database:
  db_path: /var/lib/litellm/litellm.db
  
# Redis for caching
redis:
  host: redis.default.svc.cluster.local
  port: 6379
  db: 0
```

### 4.2 LiteLLM Docker Compose 部署

```yaml
# litellm/docker-compose.yaml
version: '3.8'

services:
  litellm:
    image: ghcr.io/berriai/litellm:main
    container_name: litellm-gateway
    ports:
      - "4000:4000"  # LiteLLM API
    volumes:
      - ./config.yaml:/app/config.yaml
      - ./litellm_data:/var/lib/litellm
    environment:
      - DATABASE_URL=sqlite:////var/lib/litellm/litellm.db
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - OS_PATH=./litellm_data
      - LITELLM_MASTER_KEY=your-master-key
      - STORE_MODEL_IN_DB=True
    depends_on:
      - redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

volumes:
  redis_data:
  litellm_data:
```

### 4.3 Kubernetes 部署

```yaml
# litellm/kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: litellm-gateway
  namespace: llm-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: litellm-gateway
  template:
    metadata:
      labels:
        app: litellm-gateway
    spec:
      containers:
        - name: litellm
          image: ghcr.io/berriai/litellm:main
          ports:
            - containerPort: 4000
          env:
            - name: LITELLM_MASTER_KEY
              valueFrom:
                secretKeyRef:
                  name: litellm-secrets
                  key: master-key
            - name: DATABASE_URL
              value: "postgresql://postgres:password@postgres:5432/litellm"
            - name: REDIS_HOST
              value: "redis.llm-platform.svc.cluster.local"
            - name: REDIS_PORT
              value: "6379"
            - name: LITELLM_SETTINGS
              value: '{"drop_params": true, "set_verbose": false}'
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 4Gi
          livenessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 4000
            initialDelaySeconds: 10
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: litellm-gateway
  namespace: llm-platform
spec:
  selector:
    app: litellm-gateway
  ports:
    - port: 4000
      targetPort: 4000
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: litellm-ingress
  namespace: llm-platform
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: nginx
  rules:
    - host: litellm.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: litellm-gateway
                port:
                  number: 4000
```

---

## 五、API 设计整合

### 5.1 统一 API 结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           平台 API 分层                                    │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      管理面 API (Go Backend)                          │   │
│  │                         :8001 /api/v1                                │   │
│  │                                                                       │   │
│  │  POST   /deployments              # 创建部署                         │   │
│  │  GET    /deployments/:id           # 获取部署详情                     │   │
│  │  PUT    /deployments/:id/scale     # 扩缩容                          │   │
│  │  GET    /models                    # 模型列表                        │   │
│  │  POST   /tenants                   # 租户管理                        │   │
│  │  GET    /usage                     # 配额使用                         │   │
│  │  ...                                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      推理面 API (LiteLLM)                           │   │
│  │                         :4000 /v1                                   │   │
│  │                                                                       │   │
│  │  POST   /chat/completions         # Chat 对话 (OpenAI 兼容)         │   │
│  │  POST   /completions              # Text Completion                │   │
│  │  POST   /embeddings               # 向量嵌入                        │   │
│  │  GET    /models                   # 可用模型列表                    │   │
│  │  POST   /router/chat/completions  # 智能路由                        │   │
│  │  ...                                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      兼容层 API (可选)                                │   │
│  │                         :4000 /openai                                │   │
│  │                                                                       │   │
│  │  # 100% OpenAI 兼容，现有 SDK 可直接使用                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 API Key 设计

```go
// LiteLLM API Key 结构
type APIKey struct {
    KeyID        string    `json:"key_id"`         // litellm 对外暴露的 key
    KeyHash      string    `json:"-"`              // 内部存储的 hash
    TenantID     string    `json:"tenant_id"`
    UserID       string    `json:"user_id"`
    AllowedModels []string `json:"allowed_models"` // ["qwen-32b", "llama-70b"]
    RPM          int       `json:"rpm"`             // 每分钟请求限制
    TPM          int       `json:"tpm"`             // 每分钟 Token 限制
    CreatedAt    time.Time `json:"created_at"`
    ExpiresAt    *time.Time `json:"expires_at,omitempty"`
    IsActive     bool      `json:"is_active"`
}

// LiteLLM Key 注册格式
// LiteLLM 支持通过数据库存储 key，支持自定义 metadata
// key: sk-12345678-xxxx-xxxx
// team_id: tenant_abc123  (映射到我们的 tenant)
```

### 5.3 整合后的推理 API 调用流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant GoAPI as Go Backend<br/>(管理面)
    participant LiteLLM as LiteLLM Gateway
    participant Dynamo as Dynamo Frontend

    Note over Client,LiteLLM: 用户请求推理 API

    Client->>+LiteLLM: POST /v1/chat/completions<br/>Authorization: Bearer sk-xxx

    Note over LiteLLM: 1. 验证 API Key
    LiteLLM->>LiteLLM: 查询 Key 对应 tenant_id
    LiteLLM->>LiteLLM: 检查 RPM/TPM 限制

    Note over LiteLLM: 2. 路由到后端
    LiteLLM->>+Dynamo: 转发请求到<br/>/v1/chat/completions

    Note over Dynamo: 3. Dynamo 执行推理
    Dynamo-->>-LiteLLM: 返回生成内容

    Note over LiteLLM: 4. 成本记录
    LiteLLM->>LiteLLM: 记录 token 消耗<br/>更新成本日志

    LiteLLM-->>-Client: 返回 OpenAI 格式响应

    Note over Client,GoAPI: 后台异步：计量同步
    GoAPI->>LiteLLM: 定期同步成本数据
    GoAPI->>GoAPI: 更新租户配额使用
```

---

## 六、关键问题与解决方案

### 6.1 多租户隔离

**问题**：LiteLLM 原生支持多团队，但隔离粒度需要配置

**解决方案**：

```yaml
# LiteLLM Virtual Keys + Team 映射
# 每个租户对应 LiteLLM 的一个 team

# 数据库中 team 表结构
# team_id: tenant_123
# team_alias: "租户A"
# metadata: {"tenant_id": "xxx", "quota_rpm": 1000}
```

**Go Backend 集成**：

```go
// 租户创建时，自动创建 LiteLLM Team
func (s *TenantService) CreateLiteLLMTeam(tenant *Tenant) error {
    // 1. 调用 LiteLLM Admin API 创建 team
    resp, err := s.litellmClient.CreateTeam(&litellm.CreateTeamRequest{
        TeamAlias: tenant.Name,
        Metadata: map[string]string{
            "tenant_id": tenant.ID,
            "max_budget": strconv.FormatFloat(tenant.Quota.MaxMonthlySpend, 'f', 2),
        },
    })
    
    // 2. 更新租户记录
    tenant.LiteLLMTeamID = resp.TeamID
    return s.db.Save(tenant)
}

// API Key 创建时，绑定到对应 team
func (s *APIKeyService) CreateKey(req *CreateKeyRequest) (*APIKey, error) {
    // 1. 获取租户的 LiteLLM Team ID
    tenant, _ := s.GetTenant(req.TenantID)
    
    // 2. 在 LiteLLM 创建 key
    keyResp, err := s.litellmClient.CreateKey(&litellm.CreateKeyRequest{
        TeamID: tenant.LiteLLMTeamID,
        KeyAlias: req.Name,
        Duration: req.Duration,
    })
    
    // 3. 返回给用户
    return &APIKey{
        KeyID: keyResp.Key,
        TenantID: tenant.ID,
        // ...
    }, nil
}
```

### 6.2 配额精确计量

**问题**：LiteLLM 成本跟踪是准实时的，但需要聚合到租户维度

**解决方案**：

```go
// 定时任务：同步 LiteLLM 成本数据
type CostSyncJob struct {
    litellmClient *litellm.Client
    db            *gorm.DB
}

func (j *CostSyncJob) Run() error {
    // 1. 从 LiteLLM 获取所有团队的当前周期成本
    teams, err := j.litellmClient.GetTeamSpending()
    if err != nil {
        return err
    }
    
    // 2. 聚合到租户
    for _, team := range teams {
        tenantID := team.Metadata["tenant_id"]
        if tenantID == "" {
            continue
        }
        
        // 3. 更新租户配额使用
        usage := &QuotaUsage{
            TenantID:      tenantID,
            Period:        getCurrentPeriod(),
            TotalSpend:    team.TotalSpend,
            TotalTokens:   team.TotalTokens,
            RequestCount:  team.RequestCount,
            UpdatedAt:     time.Now(),
        }
        j.db.Save(usage)
    }
    
    // 4. 检查配额超限
    j.checkQuotaExceeded()
    return nil
}
```

### 6.3 模型动态注册

**问题**：部署新模型后，需要同步到 LiteLLM

**解决方案**：

```go
// 部署创建成功后，自动注册到 LiteLLM
func (s *DeploymentService) OnDeploymentReady(deployment *Deployment) error {
    model, _ := s.GetModel(deployment.ModelID)
    
    // 1. 获取 Dynamo Frontend URL
    frontendURL := fmt.Sprintf(
        "http://%s.%s.svc.cluster.local:%d",
        deployment.KubernetesName,
        deployment.Namespace,
        8000,
    )
    
    // 2. 注册到 LiteLLM
    return s.litellmClient.RegisterModel(&litellm.ModelConfig{
        ModelName: model.Name,  // 对外暴露的模型名
        LitellmParams: litellm.LitellmParams{
            Model:       fmt.Sprintf("vllm/%s", model.Name),
            ApiBase:     frontendURL,
            MaxRetries: 2,
            Timeout:     600,
        },
        ModelInfo: litellm.ModelInfo{
            Description: model.Description,
            Mode:        "chat",
        },
    })
}

// 部署删除时，从 LiteLLM 注销
func (s *DeploymentService) OnDeploymentDeleted(deployment *Deployment) error {
    model, _ := s.GetModel(deployment.ModelID)
    return s.litellmClient.DeleteModel(model.Name)
}
```

### 6.4 请求限流增强

**问题**：LiteLLM 支持基础限流，但租户级配额需要 Go Backend 控制

**两层限流架构**：

```
┌─────────────────────────────────────────────────────────────┐
│                    请求流程                                 │
│                                                              │
│  Client                                                     │
│    │                                                        │
│    ▼                                                        │
│  ┌──────────────────┐                                       │
│  │   Go Backend     │  ← 租户配额检查（第一层）              │
│  │   配额预检       │    - 检查剩余配额                     │
│  │   (同步拦截)     │    - 拒绝超配额请求                   │
│  └────────┬─────────┘                                       │
│           │  通过                                           │
│           ▼                                                 │
│  ┌──────────────────┐                                       │
│  │   LiteLLM        │  ← RPM/TPM 限流（第二层）            │
│  │   细粒度限流     │    - 按模型/Key 限流                  │
│  └────────┬─────────┘                                       │
│           │                                                  │
│           ▼                                                  │
│  ┌──────────────────┐                                       │
│  │   Dynamo          │                                       │
│  │   Backend         │                                       │
│  └──────────────────┘                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Go Backend 限流中间件**：

```go
func QuotaCheckMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. 从 JWT 提取 tenant_id
        claims, _ := c.Get("claims")
        tenantID := claims.TenantID
        
        // 2. 获取租户当前使用量
        usage, err := quotaService.GetCurrentUsage(tenantID)
        if err != nil {
            c.AbortWithStatusJSON(500, Error{"quota check failed"})
            return
        }
        
        // 3. 检查配额
        quota, _ := quotaService.GetQuota(tenantID)
        if usage.TotalSpend >= quota.MaxMonthlySpend {
            c.AbortWithStatusJSON(429, Error{
                Code:    "QUOTA_EXCEEDED",
                Message: "Monthly quota exceeded",
            })
            return
        }
        
        // 4. 估算本次请求成本
        reqCost := estimateRequestCost(c)
        if usage.RemainingBudget() < reqCost {
            c.AbortWithStatusJSON(429, Error{
                Code:    "INSUFFICIENT_BUDGET",
                Message: "Insufficient budget for this request",
            })
            return
        }
        
        c.Next()
    }
}
```

---

## 七、Litellm 补充的指标

### 7.1 LiteLLM 暴露的 Prometheus 指标

```promql
# LiteLLM 自带指标
litellm_requests_total{model, api_base, status_code}
litellm_request_duration_seconds{model, api_base}
litellm_tokens_used{model, api_base, token_type}
litellm_cost_seconds{model, api_base}
litellm_failed_requests_total{model, api_base, error_type}
litellm_retries_total{model, api_base}
litellm_queue_size{model, api_base}
litellm_cache_hits_total{model}
litellm_cache_misses_total{model}
```

### 7.2 平台自定义指标

```go
// Go Backend 自定义指标
var (
    // 租户维度
    TenantBudgetRemaining = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "llm_platform_tenant_budget_remaining",
            Help: "Remaining budget per tenant",
        },
        []string{"tenant_id"},
    )
    
    // 部署维度
    DeploymentReplicaCount = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "llm_platform_deployment_replicas",
            Help: "Current replica count per deployment",
        },
        []string{"deployment_id", "replica_type"},
    )
    
    // API 维度
    APIRequestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "llm_platform_api_request_duration",
            Help:    "API request duration",
            Buckets: []float64{0.01, 0.05, 0.1, 0.5, 1, 5},
        },
        []string{"endpoint", "method", "status"},
    )
)
```

### 7.3 整合后的 Grafana Dashboard

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      LLM 平台统一监控 Dashboard                           │
│                                                                          │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐       │
│  │      平台概览               │  │      租户配额               │       │
│  │                             │  │                             │       │
│  │  总请求: 1.2M               │  │  tenant-A: $450 / $1000    │       │
│  │  总成本: $12,345            │  │  tenant-B: $200 / $500    │       │
│  │  平均延迟: 245ms            │  │  tenant-C: $800 / $800 ⚠️ │       │
│  │  GPU 利用率: 78%            │  │                             │       │
│  └─────────────────────────────┘  └─────────────────────────────┘       │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      请求量趋势                                    │    │
│  │                                                                   │    │
│  │  10k ────────────────────────────                              │    │
│  │       │ ╲                                                        │    │
│  │   5k  │  ╲ ╲                                                     │    │
│  │       │   ╲  ╲                                                    │    │
│  │    0  ──────╲─────────────────────                              │    │
│  │           00:00  06:00  12:00  18:00  24:00                      │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐       │
│  │      模型调用分布            │  │      Token 消耗             │       │
│  │                             │  │                             │       │
│  │  qwen-32b:  45%  ████████  │  │  input:   500M tokens      │       │
│  │  llama-70b: 35%  ███████   │  │  output:  120M tokens      │       │
│  │  gpt-4:      20%  ████     │  │  cache:   80M tokens (40%) │       │
│  └─────────────────────────────┘  └─────────────────────────────┘       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 八、实施路线图

### 8.1 集成 Phase 1：MVP（4 周）

**目标**：快速上线推理 API，补全核心功能

```
Week 1-2：LiteLLM 部署
─────────────────────────────────────────────────
- [ ] LiteLLM 单实例部署
- [ ] 连接 Dynamo Backend
- [ ] 基础 API Key 认证
- [ ] 成本日志基本可用

Week 3-4：Go Backend 整合
─────────────────────────────────────────────────
- [ ] Go Backend 集成 LiteLLM SDK
- [ ] 租户 → LiteLLM Team 映射
- [ ] 配额检查中间件
- [ ] 成本同步任务
```

### 8.2 集成 Phase 2：生产化（4 周）

**目标**：高可用生产部署，完善可观测性

```
Week 5-6：高可用
─────────────────────────────────────────────────
- [ ] LiteLLM 多副本部署
- [ ] Redis 集群
- [ ] 限流精细化配置
- [ ] 熔断器配置

Week 7-8：可观测性
─────────────────────────────────────────────────
- [ ] 统一 Grafana Dashboard
- [ ] 告警规则完善
- [ ] 日志聚合
- [ ] SLA 报表
```

### 8.3 集成 Phase 3：高级功能（4 周）

**目标**：差异化能力建设

```
Week 9-10：智能路由
─────────────────────────────────────────────────
- [ ] 多模型路由配置
- [ ] 成本优化路由
- [ ] 故障转移路由

Week 11-12：商业化能力
─────────────────────────────────────────────────
- [ ] 详细成本报表
- [ ] 发票生成
- [ ] API 使用分析
- [ ] 渠道伙伴支持
```

---

## 九、总结与建议

### 9.1 LiteLLM 适合补全的场景

```
✅ 推理 API 统一暴露（核心价值）
✅ OpenAI 兼容性
✅ 成本跟踪基础能力
✅ 限流/熔断/重试
✅ 快速上线（vs 自研）
```

### 9.2 需要自研的场景

```
❌ 租户级配额强控制（Go Backend 层拦截）
❌ 精确计费与发票
❌ 复杂的成本分摊逻辑
❌ 平台特有的产品逻辑（如模型市场）
❌ 深度 Kubernetes 集成
```

### 9.3 最终推荐架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          推荐的 LLM 平台架构                                 │
│                                                                             │
│    ┌───────────────────────────────────────────────────────────────────┐   │
│    │                     Go Backend (管理面)                            │   │
│    │                                                                    │   │
│    │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐              │   │
│    │  │ 租户    │  │ 模型    │  │ 部署    │  │ 配额    │              │   │
│    │  │ 管理    │  │ 管理    │  │ 管理    │  │ 计费    │              │   │
│    │  └─────────┘  └─────────┘  └─────────┘  └─────────┘              │   │
│    │                                                                    │   │
│    │  ┌─────────────────────────────────────────────────────┐           │   │
│    │  │              配额检查中间件                          │           │   │
│    │  │    (租户级强控制，防止超配额)                        │           │   │
│    │  └─────────────────────────────────────────────────────┘           │   │
│    └───────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│    ┌───────────────────────────────────────────────────────────────────┐   │
│    │                      LiteLLM Gateway                               │   │
│    │                                                                    │   │
│    │  ┌─────────────────────────────────────────────────────────┐       │   │
│    │  │              OpenAI 兼容 API (/v1/*)                     │       │   │
│    │  └─────────────────────────────────────────────────────────┘       │   │
│    │                                                                    │   │
│    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐          │   │
│    │  │ 成本日志 │  │ 限流    │  │ 熔断器  │  │ 缓存    │          │   │
│    │  │ (DB)    │  │ RPM/TPM │  │         │  │ Redis   │          │   │
│    │  └──────────┘  └──────────┘  └──────────┘  └──────────┘          │   │
│    └───────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│    ┌───────────────────────────────────────────────────────────────────┐   │
│    │                     Dynamo Backend                                │   │
│    │                                                                    │   │
│    │  ┌─────────────────┐        ┌─────────────────┐                 │   │
│    │  │  Dynamo Frontend │        │  Dynamo Frontend │                 │   │
│    │  │  (Model A)       │        │  (Model B)       │                 │   │
│    │  └─────────────────┘        └─────────────────┘                 │   │
│    │                                                                    │   │
│    └───────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.4 关键决策点

| 决策 | 选项 | 建议 |
|------|------|------|
| **推理 API** | 自研 vs LiteLLM | ✅ LiteLLM（快速上线） |
| **成本跟踪** | LiteLLM 内置 vs 自研 | ⚠️ 两者结合（LiteLLM 记录 + Go 聚合） |
| **限流** | LiteLLM vs 自研 | ⚠️ 两者结合（LiteLLM 细粒度 + Go 租户级） |
| **配额控制** | 自研 | ✅ Go Backend（必须自研） |
| **模型路由** | LiteLLM 内置 vs 自研 | LiteLLM（可选高级功能） |

### 9.5 风险与缓解

| 风险 | 影响 | 概率 | 缓解措施 |
|------|------|------|----------|
| LiteLLM 社区活跃度下降 | 中 | 低 | 已有生产验证，有能力自研替换 |
| 单点故障 | 高 | 中 | 多副本部署 + 熔断降级 |
| 功能定制受限 | 中 | 中 | 开源可 fork，需提前评估 |
| 版本升级兼容性 | 中 | 低 | 版本锁定 + 充分测试 |

---

## 附录

### A. LiteLLM 文档链接

- [LiteLLM 官方文档](https://docs.litellm.ai/)
- [LiteLLM GitHub](https://github.com/BerriAI/litellm)
- [LiteLLM Proxy](https://docs.litellm.ai/docs/proxy)
- [LiteLLM Database Schema](https://docs.litellm.ai/docs/db)

### B. LiteLLM API 参考

```bash
# 健康检查
curl http://localhost:4000/health

# 模型列表
curl http://localhost:4000/model/info \
  -H "Authorization: Bearer sk-xxx"

# Chat Completions
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen-32b",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'

# 成本查询 (LiteLLM Enterprise)
curl http://localhost:4000/spend/total \
  -H "Authorization: Bearer sk-xxx"
```

### C. 成本对比

| 方案 | 开发成本 | 运维成本 | 灵活性 | 推荐度 |
|------|----------|----------|--------|--------|
| 纯自研 | 6 个月 | 高 | 高 | ⭐⭐ |
| LiteLLM | 2 个月 | 中 | 中 | ⭐⭐⭐⭐⭐ |
| 混合方案 | 3-4 个月 | 中 | 中高 | ⭐⭐⭐⭐ |
