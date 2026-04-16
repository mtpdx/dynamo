# Dynamo LLM 平台快速参考

本附录提供常用的配置模板和命令参考。

## A. 完整 DGD 配置示例

### A.1 vLLM 聚合部署

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg
spec:
  pvcs:
    - name: model-cache
      create: false
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      envFromSecret: hf-token-secret
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0

    VllmDecodeWorker:
      componentType: worker
      replicas: 2
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "2"
      sharedMemory:
        size: 16Gi
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0
          workingDir: /workspace/examples/backends/vllm
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - "Qwen/Qwen3-32B-FP8"
            - "--tensor-parallel-size"
            - "2"
            - "--max-model-len"
            - "6000"
            - "--max-num-seqs"
            - "1024"
            - "--kv-cache-dtype"
            - "fp8"
```

### A.2 vLLM 解耦部署（RDMA）

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
spec:
  pvcs:
    - name: model-cache
      create: false
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0

    VllmPrefillWorker:
      componentType: worker
      subComponentType: prefill
      replicas: 2
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "2"
          rdma/ib: "2"
      sharedMemory:
        size: 16Gi
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0
          workingDir: /workspace/examples/backends/vllm
          securityContext:
            capabilities:
              add: ["IPC_LOCK"]
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - "Qwen/Qwen3-32B-FP8"
            - "--tensor-parallel-size"
            - "2"
            - "--disaggregation-mode"
            - prefill
          env:
            - name: UCX_TLS
              value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"
            - name: UCX_RNDV_SCHEME
              value: "get_zcopy"
            - name: UCX_RNDV_THRESH
              value: "0"

    VllmDecodeWorker:
      componentType: worker
      subComponentType: decode
      replicas: 1
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "4"
          rdma/ib: "4"
      sharedMemory:
        size: 16Gi
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0
          workingDir: /workspace/examples/backends/vllm
          securityContext:
            capabilities:
              add: ["IPC_LOCK"]
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - "Qwen/Qwen3-32B-FP8"
            - "--tensor-parallel-size"
            - "4"
            - "--max-num-seqs"
            - "1024"
            - "--disaggregation-mode"
            - decode
          env:
            - name: UCX_TLS
              value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"
            - name: UCX_RNDV_SCHEME
              value: "get_zcopy"
            - name: UCX_RNDV_THRESH
              value: "0"
```

### A.3 SGLang 解耦部署

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: sglang-disagg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.0.0

    SglangPrefillWorker:
      componentType: worker
      subComponentType: prefill
      replicas: 1
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.0.0
          command:
            - python3
            - -m
            - dynamo.sglang
          args:
            - --model-path
            - "Qwen/Qwen3-0.6B"
            - --served-model-name
            - "Qwen/Qwen3-0.6B"
            - --tp
            - "1"
            - --disaggregation-mode
            - prefill
            - --disaggregation-transfer-backend
            - nixl
            - --disaggregation-bootstrap-port
            - "12345"

    SglangDecodeWorker:
      componentType: worker
      subComponentType: decode
      replicas: 1
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.0.0
          command:
            - python3
            - -m
            - dynamo.sglang
          args:
            - --model-path
            - "Qwen/Qwen3-0.6B"
            - --served-model-name
            - "Qwen/Qwen3-0.6B"
            - --tp
            - "1"
            - --disaggregation-mode
            - decode
            - --disaggregation-transfer-backend
            - nixl
            - --disaggregation-bootstrap-port
            - "12345"
```

### A.4 多节点 vLLM 部署

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-multinode
spec:
  pvcs:
    - name: model-cache
      create: false
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0

    VllmDecodeWorker:
      componentType: worker
      replicas: 1
      multinode:
        nodeCount: 2
      envFromSecret: hf-token-secret
      resources:
        limits:
          gpu: "4"
      sharedMemory:
        size: 16Gi
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - "Qwen/Qwen3-32B-FP8"
            - --tensor-parallel-size
            - "8"
```

## B. Secret 和 PVC 创建

### B.1 HuggingFace Token Secret

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n my-namespace
```

### B.2 Model Cache PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 100Gi
```

## C. 常用命令

### C.1 部署管理

```bash
# 部署
kubectl apply -f my-deployment.yaml -n my-namespace

# 查看状态
kubectl get dgd -n my-namespace
kubectl get pods -n my-namespace

# 查看日志
kubectl logs -f -l nvidia.com/dynamo-component=frontend -n my-namespace
kubectl logs -f deployment/my-deployment-vllm-worker -n my-namespace

# 删除
kubectl delete -f my-deployment.yaml -n my-namespace
```

### C.2 端口转发和测试

```bash
# 端口转发
kubectl port-forward svc/my-deployment-frontend 8000:8000 -n my-namespace

# 测试 API
curl http://localhost:8000/v1/models
curl http://localhost:8000/health

# 发送测试请求
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

### C.3 扩展管理

```bash
# 手动扩展（通过 DGDSA）
kubectl scale dgdsa my-deployment-decode --replicas=3 -n my-namespace

# 查看 DGDSA
kubectl get dgdsa -n my-namespace

# 查看 HPA/KEDA
kubectl get hpa -n my-namespace
kubectl get scaledobject -n my-namespace
```

### C.4 诊断命令

```bash
# 检查 RDMA 资源
kubectl get nodes -o jsonpath='{.items[*].status.allocatable.rdma/ib}'

# UCX 信息
kubectl exec -it <pod> -- ucx_info -d | grep -i "transport\|cuda"

# NIXL 日志
kubectl logs <pod> | grep -i "NIXL\|UCX"

# 指标检查
curl http://localhost:8000/metrics | grep dynamo_
curl http://localhost:9090/metrics | grep dynamo_
```

## D. 环境变量参考

### D.1 Frontend

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `DYN_HTTP_PORT` | 8000 | HTTP 端口 |
| `DYN_LOG` | info | 日志级别 |
| `DYN_DISCOVERY_BACKEND` | kubernetes | 服务发现后端 |
| `DYN_ROUTER_MODE` | - | 路由模式（kv, round-robin） |

### D.2 Worker

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `DYN_SYSTEM_PORT` | -1 | 系统指标端口（-1 禁用） |
| `DYN_LOG` | info | 日志级别 |
| `DYN_DISCOVERY_BACKEND` | kubernetes | 服务发现后端 |
| `HF_HOME` | - | HuggingFace 缓存路径 |

### D.3 UCX/NIXL

| 变量 | 说明 |
|------|------|
| `UCX_TLS` | 传输列表 |
| `UCX_RNDV_SCHEME` | rendezvous 协议 |
| `UCX_RNDV_THRESH` | rendezvous 阈值 |
| `NIXL_TELEMETRY_ENABLE` | 启用 NIXL 遥测 |
| `NIXL_TELEMETRY_PROMETHEUS_PORT` | NIXL 指标端口 |

## E. 指标参考

### E.1 Prometheus 查询示例

```promql
# 请求率
rate(dynamo_frontend_requests_total[5m])

# TTFT p99
histogram_quantile(0.99, rate(dynamo_frontend_time_to_first_token_seconds_bucket[5m]))

# ITL p99
histogram_quantile(0.99, rate(dynamo_frontend_inter_token_latency_seconds_bucket[5m]))

# 队列深度
dynamo_frontend_queued_requests

# 进行中请求
dynamo_frontend_inflight_requests

# KV 命中率
rate(dynamo_component_router_kv_hit_rate_sum[5m]) / rate(dynamo_component_router_kv_hit_rate_count[5m])
```

### E.2 Grafana 变量

```
# Dynamo 命名空间过滤
dynamo_namespace="$namespace-$dynamoNamespace"

# 模型过滤
model="$model_name"
```

## F. 故障排除

### F.1 Pod 无法启动

```bash
# 查看事件
kubectl describe pod <pod-name> -n my-namespace

# 查看日志
kubectl logs <pod-name> --previous -n my-namespace

# 常见原因
# - 镜像拉取失败：检查 imagePullSecrets
# - 资源不足：检查 GPU 资源请求
# - 挂载失败：检查 PVC 是否存在
```

### F.2 TTFT 过高（解耦部署）

```bash
# 1. 检查 RDMA 是否激活
kubectl logs <worker-pod> | grep "Backend UCX"
# 期望：NIXL INFO Backend UCX was instantiated

# 2. 检查 UCX 传输
kubectl exec <worker-pod> -- env | grep UCX

# 3. 检查 RDMA 资源分配
kubectl get pod <worker-pod> -o yaml | grep -A5 "resources:"

# 如果 RDMA 未激活
# - 添加 rdma/ib 资源请求
# - 添加 IPC_LOCK capability
# - 配置 UCX 环境变量
```

### F.3 性能低于预期

```bash
# 1. 检查 GPU 利用率
kubectl port-forward svc/prometheus-server 9090:9090
# 访问 http://localhost:9090 查看 DCGM 指标

# 2. 检查 KV 缓存命中率
curl -s localhost:8000/metrics | grep kv_hit

# 3. 运行基准测试验证
aiperf profile -m <model> --endpoint-type chat -u http://localhost:8000 ...
```
