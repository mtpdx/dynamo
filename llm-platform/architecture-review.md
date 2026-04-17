# LLM 推理服务平台架构评审报告

**评审日期**：2026-01-30  
**评审人**：LLM 平台架构师 & 产品经理  
**评审范围**：Dynamo 核心架构 + llm-platform 架构设计

---

## 1. Dynamo 核心架构理解总结

### 1.1 三大平面架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     Request Plane (关键路径)                     │
│  Client → Frontend → Router → Prefill Worker → Decode Worker    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                     Control Plane (自适应路径)                   │
│  Planner → Dynamo Operator → DynamoGraph → Grove/KAI → K8s     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  Storage & Events Plane (状态路径)               │
│  KV Events → KVBM (Block Manager) → NIXL → Remote Storage       │
└─────────────────────────────────────────────────────────────────┘
```

**架构亮点**：
- **三平面分离设计**：关键路径（Request）、控制路径（Control）、状态路径（Storage）解耦
- **Dynamo Operator**：基于 Kubebuilder 的 K8s Operator，声明式管理 CRD
- **KVBM 分层存储**：G1(GPU HBM) → G2(Pinned DRAM) → G3(NVMe) → G4(S3) 的 tiered storage
- **NIXL 传输层**：跨 worker GPU-to-GPU KV 传输，支持 RDMA/UCX

### 1.2 核心 CRD 设计

| CRD | 职责 | 设计评价 |
|-----|------|----------|
| DynamoGraphDeployment | 完整推理管线部署 | ✅ 声明式、组合式 |
| DynamoComponentDeployment | 单组件部署 | ✅ 细粒度控制 |
| DynamoModel | LoRA/Adapter 管理 | ✅ 解耦模型与服务 |
| DynamoGraphDeploymentRequest | SLA 驱动部署 | ✅ 用户友好 |
| DynamoGraphDeploymentScalingAdapter | 扩缩容适配器 | ✅ HPA/KEDA 集成 |

### 1.3 部署模式

| 模式 | 适用场景 | 复杂度 | 推荐度 |
|------|----------|--------|--------|
| Aggregated (agg) | 开发测试 | ⭐ | ✅ 测试环境 |
| Aggregated + Router | 生产轻负载 | ⭐⭐ | ✅ 简单生产 |
| Disaggregated | 生产高吞吐 | ⭐⭐⭐ | ✅ 高性能场景 |
| Disaggregated + Router | 大规模部署 | ⭐⭐⭐⭐ | ✅ 复杂生产 |
| Disaggregated + Planner | SLA 驱动 | ⭐⭐⭐⭐⭐ | ✅ 企业级 |

---

## 2. llm-platform 架构评审

### 2.1 整体架构评价

**优点**：

1. **分层清晰**：接入层 → 管理面 → 推理面 → 执行层 → 数据层，职责边界明确
2. **API First 设计**：管理面 8001 + 推理面 4000，OpenAI 兼容
3. **LiteLLM 集成**：利用开源方案减少重复开发，多模型路由能力
4. **性能等级抽象**：封装 Dynamo 复杂配置为「性能等级」概念，降低用户门槛
5. **同层级服务设计**：Go Backend 和 LiteLLM Gateway 并列服务，简化架构

**需关注之处**：

1. **Go Backend 职责**
   - 当前设计：租户管理 + 部署管理 + 配额控制 + 作业管理 + 审计
   - 建议：考虑拆分为多个微服务或引入消息队列解耦

2. **LiteLLM 必要性**
   - Dynamo Frontend 本身已提供 OpenAI 兼容 API
   - 保留 LiteLLM 用于：多模型路由、成本日志、细粒度限流

### 2.2 架构分层对比

| 层级 | llm-platform 设计 | Dynamo 原生 | 评价 |
|------|-------------------|-------------|------|
| 接入层 | Web UI / SDK (客户端) | 无 | ✅ 简化设计 |
| 同层级服务 | Go Backend + LiteLLM | Frontend | ✅ 双引擎服务 |
| 执行层 | Dynamo | Dynamo | ✅ 保持一致 |
| 数据层 | PostgreSQL + Redis | etcd + NATS | ⚠️ 需统一存储设计 |

---

## 3. API 设计评审

### 3.1 管理面 API 评价

**优点**：

1. **RESTful 设计**：资源 CRUD 操作清晰
2. **统一响应格式**：`success/data/metadata/error` 结构一致
3. **Job 机制**：异步操作通过 Job 机制处理，避免超时

**建议改进**：

```json
// 建议 1: 创建部署 API 简化
// 当前：性能等级 + 高级配置混合
// 建议：分离为两个端点
POST /deployments/simple    // 性能等级模式
POST /deployments/advanced  // 完整配置模式
```

```json
// 建议 2: 部署状态响应增强
// 当前：状态字段较简单
// 建议：增加更多运行时信息
{
    "status": "running",
    "conditions": [...],      // Kubernetes style conditions
    "stats": {
        "gpu_utilization": 78.5,
        "queue_depth": 3,
        "requests_total": 12345
    }
}
```

### 3.2 推理面 API 评价

**当前设计**：
- 直接透传到 LiteLLM
- OpenAI 100% 兼容

**建议**：

```
架构建议：
Web UI / SDK → Go Backend (:8001) → Dynamo
              → LiteLLM Gateway (:4000) → Dynamo

LiteLLM 用于：多模型统一路由、成本日志、细粒度限流
```

---

## 4. 数据模型评审

### 4.1 核心表结构评价

**优点**：

1. **租户隔离设计**：`tenant_id` 外键关联，逻辑隔离清晰
2. **部署配置 JSONB**：`deployment_config`、`scaling_config` 使用 JSONB，灵活性高
3. **配额设计**：`quota_usage` 记录详细，支持多维度计费

**建议改进**：

```sql
-- 建议 1: 增加部署版本管理
CREATE TABLE deployment_versions (
    id UUID PRIMARY KEY,
    deployment_id UUID REFERENCES deployments(id),
    version INT NOT NULL,
    spec JSONB NOT NULL,      -- 完整配置快照
    created_at TIMESTAMP,
    UNIQUE(deployment_id, version)
);

-- 建议 2: 增加部署依赖关系
CREATE TABLE deployment_dependencies (
    id UUID PRIMARY KEY,
    parent_deployment_id UUID,
    child_deployment_id UUID,
    dependency_type VARCHAR(50),  -- 'lora', 'base_model', etc.
    UNIQUE(parent_deployment_id, child_deployment_id)
);

-- 建议 3: 模型版本支持
CREATE TABLE model_versions (
    id UUID PRIMARY KEY,
    model_id UUID REFERENCES models(id),
    version VARCHAR(50) NOT NULL,
    model_path VARCHAR(500) NOT NULL,
    model_config JSONB,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP,
    UNIQUE(model_id, version)
);
```

### 4.2 Redis 缓存设计评价

**当前设计**：

```
session:{session_id} → {user_id, tenant_id, expires_at}
apikey:{key_hash} → {user_id, tenant_id, allowed_models, rpm, tpm}
deployment:status:{deployment_id} → {status, replicas, resource_usage}
ratelimit:{tenant_id}:{endpoint} → count
```

**建议增强**：

```redis
# 建议 1: 增加 KV Cache 路由信息缓存
kvpool:health:{pool_id} → {capacity, used, score}  TTL: 10s

# 建议 2: 增加部署端点发现缓存
deployment:endpoint:{deployment_id} → {frontend_url, backends[]}  TTL: 5min

# 建议 3: 增加限流细化
ratelimit:{tenant_id}:{model_id}:{endpoint} → {count, window_start}
```

---

## 5. 异步作业机制评审

### 5.1 当前设计评价

**优点**：

1. **Redis Queue 方案**：简单高效
2. **进度跟踪**：`progress` 字段详细
3. **状态机清晰**：`pending → running → completed/failed`

**建议改进**：

```go
// 建议 1: 增加作业优先级
type Job struct {
    Priority    int       `json:"priority"`    // 0-9, 9 最高
    MaxRetries int       `json:"max_retries"` // 失败重试次数
    RetryDelay Duration  `json:"retry_delay"` // 重试间隔
}

// 建议 2: 增加作业依赖
type JobDependency struct {
    JobID       string
    DependsOn   []string  // 前置作业列表
    WaitTimeout Duration   // 等待超时
}

// 建议 3: 作业分类
const (
    JobTypeDeploymentCreate  = "deployment_create"
    JobTypeDeploymentScale   = "deployment_scale"
    JobTypeDeploymentUpdate   = "deployment_update"
    JobTypeDeploymentDelete   = "deployment_delete"
    JobTypeModelImport        = "model_import"
    JobTypeModelVersionCreate = "model_version_create"
)
```

### 5.2 Worker 实现建议

```go
// 建议：引入幂等性设计
type JobWorker struct {
    redis        *redis.Client
    db           *gorm.DB
    k8sClient    *kubernetes.Clientset
    
    // 新增：幂等性保证
    idempotencyStore *IdempotencyStore
}

func (w *JobWorker) processJob(job *Job) error {
    // 1. 检查幂等性
    key := fmt.Sprintf("idempotent:%s:%s", job.Type, job.TargetID)
    if w.idempotencyStore.Exists(key) {
        return nil  // 已处理，直接返回
    }
    
    // 2. 标记处理中
    w.idempotencyStore.MarkProcessing(key)
    defer w.idempotencyStore.MarkDone(key)
    
    // 3. 执行处理
    return w.doProcess(job)
}
```

---

## 6. 配额控制机制评审

### 6.1 两层配额控制设计

**当前设计**：

```
第一层：Go Backend 配额拦截（租户级强控制）
第二层：LiteLLM 细粒度限流（模型/Key 级控制）
```

**优点**：
- 分层控制，职责清晰
- 双重保障

**建议改进**：

```go
// 建议 1: 配额策略细化
type QuotaPolicy struct {
    TenantID       string
    Policies []QuotaPolicyItem
}

type QuotaPolicyItem struct {
    Resource       string  // "tokens", "requests", "gpu_hours"
    Limit           int64
    Window          Duration  // "1h", "1d", "1m"
    BurstAllowance  int64     // 突发允许
}
```

### 6.2 配额超限响应

**建议**：

```json
// 超限时返回
{
    "error": {
        "code": "QUOTA_EXCEEDED",
        "message": "Monthly token quota exceeded",
        "details": {
            "limit": 1000000,
            "used": 1000500,
            "reset_at": "2026-02-01T00:00:00Z",
            "upgrade_url": "/billing/upgrade"
        }
    }
}

// HTTP 状态码建议：429 Too Many Requests
// 配合 Retry-After header
```

---

## 7. 安全设计评审

### 7.1 当前安全措施

- JWT Token 认证
- API Key 认证
- Secret 管理（K8s）
- RBAC 配置

### 7.2 建议增强

```yaml
# 建议 1: 网络策略细化
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: llm-platform-policy
spec:
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8001  # Go Backend
        - protocol: TCP
          port: 4000  # LiteLLM
```

```go
// 建议 2: API 限流增强
type RateLimitConfig struct {
    GlobalLimit    int           // 全局限制
    PerTenantLimit int           // 单租户限制
    PerKeyLimit    int           // 单 Key 限制
    Window         time.Duration // 滑动窗口
    BurstSize      int           // 突发容量
}
```

---

## 8. 可观测性评审

### 8.1 当前指标设计

Dynamo 原生指标：
- `dynamo_frontend_requests_total`
- `dynamo_frontend_time_to_first_token_seconds`
- `dynamo_frontend_inter_token_latency_seconds`
- `dynamo_component_router_kv_hit_rate`

### 8.2 建议增强

```yaml
# 建议：llm-platform 业务指标
metrics:
  # 业务指标
  - llm_platform_deployments_total{status, tier}
  - llm_platform_deployment_ready_seconds{deployment_id}
  - llm_platform_jobs_total{type, status}
  - llm_platform_jobs_duration_seconds{type}
  
  # 配额指标
  - llm_platform_quota_usage_tokens{tenant_id}
  - llm_platform_quota_remaining{tenant_id, resource}
  
  # 成本指标
  - llm_platform_cost_estimate_total{tenant_id}
  - llm_platform_cost_by_model{tenant_id, model_id}
  
  # 安全指标
  - llm_platform_auth_failures_total{reason}
  - llm_platform_rate_limit_hits_total{tenant_id}
```

---

## 9. 架构成熟度与建议

### 9.1 架构成熟度评估

| 维度 | 当前状态 | 目标状态 | 差距 |
|------|----------|----------|------|
| 核心功能 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | MVP 已完成 |
| 扩缩容 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 需 Planner 集成 |
| 多租户 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 基本完善 |
| 安全 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 需增强 |
| 可观测 | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 需告警闭环 |
| 成本控制 | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 需精细化 |

### 9.2 建议优先级

**P1（重要）**：

1. **部署版本管理**：支持配置回滚
2. **配额策略细化**：支持多维度配额
3. **异步作业增强**：优先级、依赖、幂等性

**P2（优化）**：

1. **告警闭环**：从监控到告警到自动处理
2. **成本预估**：实时成本计算
3. **多集群支持**：跨集群部署管理

### 9.3 架构演进路线

```
Phase 1（当前 MVP）：
├── 单集群部署
├── 单租户/多租户隔离
├── 基本配额控制
└── Dynamo 原生能力

Phase 2（下一版本）：
├── 多集群联邦
├── 精细化配额 + 计费
├── 作业依赖 + 优先级
└── 部署版本管理

Phase 3（长期目标）：
├── 跨云部署
├── 智能扩缩容（Planner）
├── 成本优化建议
└── 多模型智能路由
```

---

## 10. 结论

### 10.1 架构评价

llm-platform 的架构设计整体**优秀**，充分借鉴了 Dynamo 的核心能力，并在此基础上做了大量用户友好的抽象。核心亮点：

1. **性能等级抽象**：极大降低用户门槛
2. **分层设计**：管理面/推理面/执行面分离，同层级服务简化架构
3. **异步作业机制**：避免长时间操作超时
4. **LiteLLM 集成**：多模型统一路由、成本日志、细粒度限流

### 10.2 核心建议

1. **增强作业系统**：支持优先级、依赖、幂等性
2. **完善配额体系**：多维度、细粒度
3. **安全加固**：网络策略、API 安全

### 10.3 下一步行动

- [ ] 异步作业增强详细设计（5 天）
- [ ] 配额系统细化设计（3 天）
- [ ] 安全加固方案设计（3 天）

---

**评审结论**：llm-platform 架构设计**通过评审**，建议进入实现阶段。建议按 P1/P2 优先级逐步实施增强功能。
