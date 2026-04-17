# Dynamo 核心架构理解

本文档帮助读者深入理解 NVIDIA Dynamo 的核心架构设计，作为 llm-platform 架构设计的基础参考。

---

## 1. Dynamo 概述

Dynamo 是一个分布式 LLM 推理运行时，专为大规模语言模型推理设计，核心特性：

| 特性 | 说明 |
|------|------|
| **延迟稳定性** | 在突发和混合长度流量下保持 TTFT 和 ITL 可预测 |
| **GPU 效率** | 解耦 prefill 和 decode，实现独立扩展 |
| **计算复用** | 通过 KV-aware 路由和缓存生命周期管理减少 KV 重复计算 |
| **运维韧性** | 将 worker 崩溃、重启和过载视为正常操作事件 |
| **部署可移植性** | 支持 Kubernetes 原生控制路径和非 Kubernetes 运行时模式 |

---

## 2. 三平面架构

Dynamo 采用三平面协作架构，每一层承担不同的职责：

### 2.1 Request Plane（关键路径）

```
Client → Frontend → Router → Prefill Worker → Decode Worker
```

**职责**：处理推理请求的关键路径

| 组件 | 职责 |
|------|------|
| Frontend | HTTP Server，OpenAI 兼容 API，请求预处理，模型验证 |
| Router | KV-aware 路由选择，负载均衡 |
| Prefill Worker | 计算 prompt KV 状态 |
| Decode Worker | 生成输出 token |

### 2.2 Control Plane（自适应路径）

```
Planner → Dynamo Operator → DynamoGraph → Grove/KAI → K8s
```

**职责**：部署、扩缩容、SLA 优化

| 组件 | 职责 |
|------|------|
| Planner | 根据 SLA 目标计算扩展目标 |
| Dynamo Operator | K8s Operator，管理 CRD 生命周期 |
| DynamoGraph | 声明式部署配置 |
| Grove/KAI | 拓扑感知调度 |

### 2.3 Storage & Events Plane（状态路径）

```
KV Events → KVBM (Block Manager) → NIXL → Remote Storage
```

**职责**：KV Cache 管理和传输

| 组件 | 职责 |
|------|------|
| KVBM | KV 块复用、逐出、卸载/召回 |
| NIXL | 跨 worker GPU-to-GPU KV 传输 |
| Remote Storage | 分层存储（G1-G4） |

---

## 3. 核心 CRD 设计

### 3.1 CRD 概览

| CRD | 全称 | 用途 |
|-----|------|------|
| DGD | DynamoGraphDeployment | 完整推理管线部署 |
| DCD | DynamoComponentDeployment | 单组件部署 |
| DM | DynamoModel | LoRA/Adapter 管理 |
| DGDR | DynamoGraphDeploymentRequest | SLA 驱动部署请求 |
| DGDSA | DynamoGraphDeploymentScalingAdapter | 扩缩容适配器 |
| DC | DynamoCheckpoint | Checkpoint 管理 |

### 3.2 DynamoGraphDeployment 结构

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
    VllmDecodeWorker:
      componentType: worker
      subComponentType: decode  # 解耦模式
      replicas: 2
    VllmPrefillWorker:
      componentType: worker
      subComponentType: prefill  # 解耦模式
      replicas: 2
```

**设计亮点**：
- 声明式：描述期望状态，Operator 负责实现
- 组合式：Frontend + Workers 可灵活组合
- 解耦支持：`subComponentType` 支持 prefill/decode 分离

### 3.3 DynamoGraphDeploymentRequest (DGDR)

DGDR 是用户友好的部署方式，只需指定 SLA 目标：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: my-dgdr
spec:
  model: Qwen/Qwen3-32B-FP8
  backend: vllm
  workload:
    isl: 4000  # Input Sequence Length
    osl: 500   # Output Sequence Length
  sla:
    ttft: 600.0  # Time to First Token (ms)
    itl: 20.0    # Inter Token Latency (ms)
```

**工作流程**：
1. DGDR 创建 → Operator 生成 DGD spec
2. 自动 profiling 获取性能数据
3. 生成最优 DGD 配置
4. 可选 autoApply 直接部署

---

## 4. 部署模式

### 4.1 Aggregated Serving（聚合部署）

**架构**：
```
Frontend → Worker (Prefill + Decode 在同一 GPU)
```

**适用场景**：
- 小型到中型模型（70B 以下）
- 开发和测试环境
- 低到中等流量

**特点**：
- 简单部署
- 无 KV 传输开销
- GPU 利用率可能不是最优

### 4.2 Disaggregated Serving（解耦部署）

**架构**：
```
Frontend → Router → Prefill Worker → (NIXL) → Decode Worker
```

**适用场景**：
- 生产环境
- 高吞吐量需求
- 大型模型（70B+）

**关键要求**：
- 必须使用 RDMA（InfiniBand/RoCE）进行 KV cache 传输
- 无 RDMA 会导致 40x 性能下降

### 4.3 模式对比

| 维度 | Aggregated | Disaggregated |
|------|------------|---------------|
| 架构复杂度 | ⭐ | ⭐⭐⭐ |
| GPU 利用率 | 中等 | 高 |
| KV 传输开销 | 无 | +200-500ms (RDMA) |
| 独立扩缩容 | 不支持 | 支持 |
| 适用规模 | < 70B | 70B+ |

---

## 5. KVBM 分层存储

### 5.1 四层架构

| Tier | 介质 | 延迟 | 容量 | 用途 |
|------|------|------|------|------|
| G1 | GPU HBM | ~ns | 最小 | 活跃 KV cache |
| G2 | Pinned DRAM | ~us | 中等 | RDMA 传输 staging |
| G3 | NVMe/SSD | ~ms | 大 | Warm block 持久化 |
| G4 | S3/MinIO | ~100ms | 无限 | 冷存储/归档 |

### 5.2 KV Cache 生命周期

```
1. Prefill 计算 → KV 进入 G1
2. Decode 完成后 → KV 保留在 G1 或降级到 G2/G3
3. 后续请求匹配 → 从 G2/G3 召回到 G1
4. 长时间未访问 → 降级到 G4 或逐出
```

### 5.3 NIXL 传输

NIXL 提供跨 worker 的 GPU-to-GPU 传输：

```yaml
# NIXL 配置示例
- --kv-transfer-config
- '{"kv_connector":"NixlConnector","kv_role":"kv_both"}'
- --kv-events-config
- '{"publisher":"zmq","topic":"kv-events","endpoint":"tcp://*:20080"}'
```

**传输模式**：
- `get_zcopy`：零拷贝 RDMA GET
- `put_zcopy`：零拷贝 RDMA PUT
- rendezvous 协议：协商传输参数

---

## 6. Dynamo Operator

### 6.1 架构

```
┌─────────────────────────────────────────────────┐
│              Dynamo Operator (Deployment)         │
├─────────────────────────────────────────────────┤
│  DynamoGraphDeploymentController                 │
│    └──  watches: DynamoGraphDeployment CR       │
│         manages: Deployment, Service, Pods       │
├─────────────────────────────────────────────────┤
│  DynamoComponentDeploymentController             │
│    └──  watches: DynamoComponentDeployment CR   │
│         manages: Deployment, Service             │
├─────────────────────────────────────────────────┤
│  DynamoModelController                           │
│    └──  watches: DynamoModel CR                 │
│         manages: LoRA adapter lifecycle          │
└─────────────────────────────────────────────────┘
```

### 6.2 部署模式

| 模式 | 范围 | 推荐度 |
|------|------|--------|
| Cluster-Wide (默认) | 所有命名空间 | ✅ 生产推荐 |
| Namespace-Scoped | 单命名空间 | ⚠️ 已废弃 |
| Hybrid | 混合 | ⚠️ 已废弃 |

### 6.3 扩缩容集成

Dynamo 提供 `DynamoGraphDeploymentScalingAdapter` 连接 K8s HPA/KEDA：

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-deployment-hpa
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: my-deployment-decode
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: dynamo_ttft_p95_seconds
        target:
          type: Value
          value: "500m"
```

---

## 7. 全局规划器 (Global Planner)

### 7.1 为什么需要 Global Planner

单一 DGD 的本地 planner 限制：
- 只扩展自己的部署
- 难以在多个 DGD 之间应用集中式策略
- 无法强制执行共享约束（如总 GPU 预算）

### 7.2 架构

```
GlobalPlanner (--max-total-gpus)
├── DGD model-a: Frontend + Prefill + Decode + Planner
└── DGD model-b: Frontend + Prefill + Decode + Planner
```

### 7.3 使用场景

| 场景 | 方案 |
|------|------|
| 多模型端点 + GPU 预算 | Global Planner 限制总 GPU |
| 单端点 + 多池 | GlobalRouter 智能路由 |
| SLA 驱动扩缩容 | Planner 模式 |

---

## 8. 服务发现

### 8.1 Kubernetes Native 模式

Dynamo 使用 K8s 原生服务发现：

```yaml
# Frontend 通过环境变量发现 Worker
DYN_DISCOVERY_BACKEND: kubernetes
DYN_KUBERNETES_NAMESPACE: my-namespace
```

### 8.2 etcd 模式

```yaml
# 使用 etcd 进行服务发现
DYN_DISCOVERY_BACKEND: etcd
DYN_ETCD_SERVERS: etcd-server:2379
```

---

## 9. 关键配置参考

### 9.1 vLLM Worker 配置

```yaml
VllmDecodeWorker:
  extraPodSpec:
    mainContainer:
      command:
        - python3
        - -m
        - dynamo.vllm
      args:
        - --model
        - Qwen/Qwen3-32B-FP8
        - --tensor-parallel-size
        - "2"
        - --max-model-len
        - "6000"
        - --max-num-seqs
        - "1024"
        - --kv-cache-dtype
        - "fp8"
```

### 9.2 SGLang Worker 配置

```yaml
SglangPrefillWorker:
  extraPodSpec:
    mainContainer:
      command:
        - python3
        - -m
        - dynamo.sglang
      args:
        - --model-path
        - Qwen/Qwen3-0.6B
        - --tp
        - "1"
        - --disaggregation-mode
        - prefill
        - --disaggregation-transfer-backend
        - nixl
```

### 9.3 RDMA 配置

```yaml
env:
  - name: UCX_TLS
    value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"
  - name: UCX_RNDV_SCHEME
    value: "get_zcopy"
  - name: UCX_RNDV_THRESH
    value: "0"

securityContext:
  capabilities:
    add: ["IPC_LOCK"]

resources:
  limits:
    rdma/ib: "2"
```

---

## 10. 监控指标

### 10.1 Frontend 指标

| 指标 | 类型 | 说明 |
|------|------|------|
| `dynamo_frontend_requests_total` | Counter | 总请求数 |
| `dynamo_frontend_time_to_first_token_seconds` | Histogram | TTFT |
| `dynamo_frontend_inter_token_latency_seconds` | Histogram | ITL |
| `dynamo_frontend_queued_requests` | Gauge | HTTP 队列深度 |
| `dynamo_frontend_inflight_requests` | Gauge | 处理中请求 |

### 10.2 Worker 指标

| 指标 | 类型 | 说明 |
|------|------|------|
| `dynamo_component_requests_total` | Counter | 组件总请求数 |
| `dynamo_component_request_duration_seconds` | Histogram | 请求处理时间 |
| `dynamo_component_inflight_requests` | Gauge | 处理中请求 |

### 10.3 Router 指标

| 指标 | 类型 | 说明 |
|------|------|------|
| `dynamo_component_router_requests_total` | Counter | 路由请求总数 |
| `dynamo_component_router_time_to_first_token_seconds` | Histogram | 路由 TTFT |
| `dynamo_component_router_kv_hit_rate` | Gauge | KV 缓存命中率 |

---

## 11. 故障容错

### 11.1 故障层级

| 层级 | 机制 | 效果 |
|------|------|------|
| 请求 | 迁移、取消 | 进行中工作可继续或终止 |
| Worker | 健康检查、优雅关闭 | 失败 Worker 安全停止 |
| 系统 | 请求拒绝/负载卸载 | 防止过载传播 |
| 基础设施 | Discovery 租约过期 | 过期成员移除 |

### 11.2 关键配置

```yaml
# 迁移配置
args:
  - "--migration-limit"
  - "3"

# 优雅关闭
env:
  - name: DYN_GRACEFUL_SHUTDOWN_TIMEOUT
    value: "30"

# 负载卸载
args:
  - "--active-decode-blocks-threshold"
  - "80%"
  - "--active-prefill-tokens-threshold"
  - "4096"
```

---

## 12. 参考链接

- [Dynamo 官方文档](https://docs.dynamo.nvidia.com)
- [Dynamo GitHub](https://github.com/ai-dynamo/dynamo)
- [vLLM 集成](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm)
- [SGLang 集成](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/sglang)
- [TensorRT-LLM 集成](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/trtllm)
