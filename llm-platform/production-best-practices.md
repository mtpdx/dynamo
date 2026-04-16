# 生产环境最佳实践与故障容错

本文档涵盖 NVIDIA Dynamo LLM 推理服务平台的生产环境最佳实践和故障容错指南。

## 1. 生产环境架构设计

### 1.1 推荐架构模式

```
                    ┌─────────────────────────────────────────────┐
                    │              Load Balancer                   │
                    │         (云厂商 LB / Nginx)                 │
                    └─────────────────┬───────────────────────────┘
                                      │
                    ┌─────────────────▼───────────────────────────┐
                    │         Dynamo Frontend (N replicas)         │
                    │           OpenAI Compatible API              │
                    └─────────────────┬───────────────────────────┘
                                      │
                    ┌─────────────────▼───────────────────────────┐
                    │           Global Router (可选)               │
                    │         请求路由 + KV-aware 调度              │
                    └───────┬─────────────────────────┬───────────┘
                            │                         │
            ┌───────────────▼───────────┐   ┌───────▼──────────────┐
            │    Prefill Pool 0 (TP1)   │   │   Prefill Pool 1     │
            │    [Worker × N replicas]  │   │   [TP2, 更大规模]    │
            └───────────────┬───────────┘   └───────┬──────────────┘
                            │                         │
                            └─────────┬───────────────┘
                                      │ KV Transfer (RDMA)
                            ┌─────────▼───────────────────────────┐
                            │        Decode Pool (TP4)            │
                            │      [Worker × M replicas]          │
                            └────────────────────────────────────┘
```

### 1.2 高可用设计原则

| 原则 | 实现方式 |
|------|----------|
| **冗余** | 每个组件至少 2 个副本 |
| **隔离** | 关键组件独立部署 |
| **优雅降级** | 过载时请求拒绝而非崩溃 |
| **自动恢复** | 故障自动检测和转移 |
| **可观测性** | 完整指标、日志、追踪 |

## 2. 容量规划

### 2.1 估算模型

```
总吞吐量 (tokens/s) = GPU数 × tokens/s/gpu × 利用率

GPU需求 = 目标吞吐量 / (tokens/s/gpu × 利用率)
```

### 2.2 性能参考数据

基于 H100/H200 实测：

| 模型 | GPUs | 精度 | ISL/OSL | tokens/s/gpu | TTFT (ms) | ITL (ms) |
|------|------|------|---------|--------------|------------|-----------|
| Llama-3.1-70B | 8 | FP8 | 4K/500 | ~450 | ~450 | ~18 |
| Qwen3-32B | 8 | FP8 | 4K/500 | ~320 | ~320 | ~16 |
| DeepSeek-R1 | 8 | FP8 | 4K/500 | ~400 | ~400 | ~17 |

### 2.3 扩展性规划

```bash
# 根据目标 QPS 估算
目标 QPS = 100
平均输出长度 = 500 tokens
ITL = 17 ms

每 GPU 吞吐量 = 1000 / ITL / 1000 = ~59 req/s/gpu (假设高并发)
需要的 GPU = 目标 QPS / 每 GPU吞吐量 = 100 / 59 ≈ 2 GPUs

# 考虑突发流量
峰值因子 = 2.0
建议 GPU = 4 GPUs
```

## 3. 部署最佳实践

### 3.1 镜像管理

```bash
# 1. 使用特定版本标签（不用 latest）
image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0

# 2. 在测试环境验证后再生产
# 3. 维护内部镜像仓库加速拉取
# 4. 配置 imagePullSecrets
```

### 3.2 资源配置模板

**vLLM Worker（高性能）**：
```yaml
resources:
  limits:
    gpu: "4"
    memory: "64Gi"
    cpu: "16"
  requests:
    gpu: "4"
    memory: "64Gi"
    cpu: "8"
sharedMemory:
  size: 16Gi
```

**vLLM Worker（内存优化）**：
```yaml
resources:
  limits:
    gpu: "2"
    memory: "128Gi"
    cpu: "16"
  requests:
    gpu: "2"
    memory: "96Gi"
    cpu: "8"
sharedMemory:
  size: 16Gi
```

### 3.3 探针配置

```yaml
readinessProbe:
  exec:
    command: ["/bin/sh", "-c", 'grep "Worker.*initialized" /tmp/worker.log']
  initialDelaySeconds: 120  # 大模型加载需要时间
  periodSeconds: 30
  timeoutSeconds: 10
  failureThreshold: 3

livenessProbe:
  exec:
    command: ["/bin/sh", "-c", 'ps aux | grep "dynamo" | grep -v grep']
  initialDelaySeconds: 60
  periodSeconds: 60
  failureThreshold: 3
```

### 3.4 Pod 中断预算

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-deployment-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      nvidia.com/dynamo-component: worker
```

## 4. 运维最佳实践

### 4.1 日志管理

```yaml
# 结构化日志配置
env:
  - name: DYN_LOG
    value: "info"
  - name: DYN_LOG_FORMAT
    value: "json"
```

**日志级别指南**：
- `debug`：问题诊断
- `info`：正常运行
- `warn`：异常但可恢复
- `error`：需要关注

### 4.2 指标监控

关键监控指标：

| 指标 | 告警阈值 | 说明 |
|------|----------|------|
| TTFT_p99 | > 2000ms | 首个 token 延迟 |
| ITL_p99 | > 100ms | token 间延迟 |
| queue_depth | > 50 | 请求堆积 |
| gpu_util | < 50% | GPU 利用率 |
| requests_failed | > 1% | 失败率 |

### 4.3 告警规则示例

```yaml
groups:
  - name: dynamo-alerts
    rules:
      - alert: HighTTFT
        expr: histogram_quantile(0.99, rate(dynamo_frontend_time_to_first_token_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "TTFT p99 超过 2 秒"

      - alert: LowGPUUtilization
        expr: avg(dcgm_gpu_utilization{instance=~"worker.*"}) < 50
        for: 10m
        labels:
          severity: info
        annotations:
          summary: "GPU 利用率低于 50%"

      - alert: HighQueueDepth
        expr: dynamo_frontend_queued_requests > 50
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "请求队列过深"
```

## 5. 故障容错

### 5.1 故障场景与响应

| 场景 | 检测方式 | 响应策略 |
|------|----------|----------|
| Worker 崩溃 | 健康检查失败 | 请求迁移到其他 Worker |
| GPU 故障 | XID 错误 | 节点标记，流量重路由 |
| 网络分区 | Discovery 租约过期 | Worker 移除，重路由 |
| Frontend 故障 | 健康检查 | 负载均衡器重路由 |
| Pod 驱逐 | K8s 事件 | 优雅关闭，迁移请求 |

### 5.2 请求迁移配置

```yaml
# 启用请求迁移
args:
  - "--migration-limit"
  - "3"          # 最多 3 次迁移

# 迁移超时
env:
  - name: DYN_MIGRATION_TIMEOUT
    value: "30"
```

### 5.3 优雅关闭

```yaml
# 停止接收新请求后等待时长
env:
  - name: DYN_GRACEFUL_SHUTDOWN_TIMEOUT
    value: "60"

# 滚动更新配置
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 0
    maxUnavailable: 1
```

### 5.4 负载卸载

```yaml
# 防止过载的配置
args:
  - "--active-decode-blocks-threshold"
  - "80%"        # 80% 时开始拒绝
  - "--active-prefill-tokens-threshold"
  - "4096"
```

**拒绝策略**：
- 返回 HTTP 503 Service Unavailable
- 在响应头添加 Retry-After
- 客户端自动重试

## 6. 安全最佳实践

### 6.1 网络策略

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: dynamo-network-policy
spec:
  podSelector:
    matchLabels:
      nvidia.com/dynamo-component: worker
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              nvidia.com/dynamo-component: frontend
      ports:
        - protocol: TCP
          port: 8000
  egress:
    - to:
        - podSelector:
            matchLabels:
              nvidia.com/dynamo-component: etcd
      ports:
        - protocol: TCP
          port: 2379
```

### 6.2 Secret 管理

```bash
# 使用 Kubernetes Secret 存储敏感信息
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n my-namespace

# 使用 External Secrets Operator (ESO) 管理
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: hf-token-external
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: hf-token-secret
  data:
    - secretKey: HF_TOKEN
      remoteRef:
        key: secret/huggingface
        property: token
```

### 6.3 RBAC 配置

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dynamo-operator
rules:
  - apiGroups: ["nvidia.com"]
    resources: ["dynamographdeployments", "dynamocomponentdeployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["services", "pods", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

## 7. 性能优化

### 7.1 前缀缓存

对于有重复前缀的工作负载：

```bash
# 启用前缀缓存
args:
  - "--enable-prefix-caching"

# 对于 vLLM
  - "--enable-prefix-caching"
```

### 7.2 KV 缓存优化

```yaml
# 优化 KV 缓存利用率
args:
  - "--kv-cache-dtype"
  - "fp8"              # 更小的数据类型
  - "--max-model-len"
  - "32000"            # 根据实际需求设置
```

### 7.3 批处理优化

```yaml
args:
  - "--max-num-seqs"
  - "1024"             # 最大并发序列
  - "--max-num-batched-tokens"
  - "8192"             # 最大批量 token 数
```

## 8. 灾难恢复

### 8.1 备份策略

| 数据 | 备份方式 | 频率 |
|------|----------|------|
| 模型权重 | PVC 快照/异地复制 | 每周 |
| 配置 | GitOps (FluxCD) | 每次变更 |
| Checkpoint | Dynamo Checkpoint CR | 按需 |

### 8.2 恢复步骤

```bash
# 1. 验证集群健康
kubectl get nodes
kubectl get pods -n dynamo-system

# 2. 重新部署
kubectl apply -f backup/deployments.yaml

# 3. 验证服务
kubectl get dgd
curl http://my-deployment-frontend:8000/health

# 4. 运行冒烟测试
kubectl apply -f tests/smoke-test.yaml
```

### 8.3 定期演练

- [ ] 每月一次故障注入演练
- [ ] 每季度一次灾难恢复演练
- [ ] 记录和复盘改进

## 9. 成本优化

### 9.1 资源优化

```yaml
# 使用 Spot/Preemptible 实例
nodeSelector:
  node.kubernetes.io/lifecycle: spot

# 配置预算
resources:
  limits:
    gpu: "2"           # 不要过度分配
```

### 9.2 自动伸缩

```yaml
# KEDA 配置
spec:
  minReplicaCount: 1
  maxReplicaCount: 10
  pollingInterval: 30
  cooldownPeriod: 300
  
  triggers:
  - type: prometheus
    metadata:
      metricName: dynamo_queued_requests
      threshold: "5"
```

### 9.3 监控成本指标

```promql
# 每请求成本
(sum(rate(dynamo_frontend_request_duration_seconds_sum[1h])) * gpu_cost_per_second) 
/ sum(rate(dynamo_frontend_requests_total[1h]))

# 每 Token 成本
(total_gpu_cost_per_hour * 3600) / total_tokens_per_hour
```

## 10. 升级和维护

### 10.1 升级策略

```bash
# 1. 阅读发布说明
# https://github.com/ai-dynamo/dynamo/releases

# 2. 在测试环境验证
# 3. 备份当前配置
kubectl get dgd -o yaml > backup/dgd-backup.yaml

# 4. 分阶段升级
# - 先升级 operator
helm upgrade dynamo-platform dynamo-platform-${NEW_VERSION}.tgz

# - 验证 operator 健康
kubectl rollout status deployment/dynamo-operator -n dynamo-system

# 5. 滚动更新 worker
kubectl rollout restart deployment/my-deployment-worker -n my-namespace
```

### 10.2 维护窗口

```yaml
# 计划内维护
spec:
  terminationGracePeriodSeconds: 3600  # 1 小时优雅关闭
```

### 10.3 版本兼容性矩阵

| Dynamo 版本 | Kubernetes | Helm | 后端 |
|-------------|------------|------|------|
| 1.0.x | 1.24+ | 3.0+ | vLLM 0.12+, SGLang 0.4+ |
| 0.9.x | 1.24+ | 3.0+ | vLLM 0.11+, SGLang 0.3+ |

## 11. 参考检查清单

### 部署前
- [ ] 镜像验证
- [ ] 资源配置
- [ ] RDMA 验证（解耦）
- [ ] 模型缓存设置
- [ ] Secret 创建

### 部署后
- [ ] 健康检查通过
- [ ] 基准测试达标
- [ ] 指标告警配置
- [ ] 日志收集配置
- [ ] 备份策略验证

### 生产环境
- [ ] 高可用配置
- [ ] 容量规划
- [ ] 监控仪表板
- [ ] 灾难恢复计划
- [ ] 安全审计
