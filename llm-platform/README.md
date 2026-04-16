# NVIDIA Dynamo LLM 推理服务平台生产最佳实践

本文档基于 NVIDIA Dynamo 项目，总结基于 Dynamo 实现 LLM 推理服务平台的生产最佳实践。

## 目录

1. [Dynamo 架构概述](#1-dynamo-架构概述)
2. [部署架构模式](#2-部署架构模式)
3. [Kubernetes 部署配置](#3-kubernetes-部署配置)
4. [后端引擎集成](#4-后端引擎集成)
5. [RDMA 与 KV Cache 传输](#5-rdma-与-kv-cache-传输)
6. [模型缓存与加载](#6-模型缓存与加载)
7. [弹性伸缩配置](#7-弹性伸缩配置)
8. [全局规划器与多池架构](#8-全局规划器与多池架构)
9. [可观测性与监控](#9-可观测性与监控)
10. [故障容错机制](#10-故障容错机制)
11. [生产环境清单](#11-生产环境清单)

---

## 1. Dynamo 架构概述

### 1.1 设计目标

Dynamo 是一个分布式 LLM 推理运行时，设计目标包括：

| 目标 | 说明 |
|------|------|
| **延迟稳定性** | 在突发和混合长度流量下保持 TTFT 和 ITL 可预测 |
| **GPU 效率** | 解耦 prefill 和 decode，实现独立扩展 |
| **计算复用** | 通过 KV-aware 路由和缓存生命周期管理减少 KV 重复计算 |
| **运维韧性** | 将 worker 崩溃、重启和过载视为正常操作事件 |
| **部署可移植性** | 支持 Kubernetes 原生控制路径和非 Kubernetes 运行时模式 |

### 1.2 三大平面架构

Dynamo 采用三平面协作架构：

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

### 1.3 核心组件

| 组件 | 类型 | 职责 |
|------|------|------|
| **Frontend** | HTTP Server | OpenAI 兼容 API、请求预处理、模型验证 |
| **Router** | 请求路由 | KV-aware 路由选择、负载均衡 |
| **Prefill Worker** | 计算节点 | 计算 prompt KV 状态 |
| **Decode Worker** | 计算节点 | 生成输出 token |
| **Planner** | 控制器 | 根据 SLA 目标计算扩展目标 |
| **KVBM** | 缓存管理 | KV 块复用、逐出、卸载/召回 |
| **NIXL** | 传输层 | 跨 worker GPU-to-GPU KV 传输 |

---

## 2. 部署架构模式

### 2.1 聚合部署 (Aggregated Serving)

**适用场景**：
- 小型到中型模型（70B 以下）
- 开发和测试环境
- 低到中等流量
- 优先考虑简单性而非最大吞吐量

**架构**：
```
Frontend → Worker (Prefill + Decode 在同一 GPU)
```

**示例配置** (`agg.yaml`)：
```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag

    VllmDecodeWorker:
      componentType: worker
      replicas: 1
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
```

### 2.2 聚合路由部署 (Aggregated + Router)

**适用场景**：
- 需要高可用性的中等流量
- 需要水平扩展
- 想要负载均衡但不需要解耦复杂性

**架构**：
```
Frontend (with KV Router) → Worker Pool (多个聚合 Worker)
```

### 2.3 解耦部署 (Disaggregated Serving)

**适用场景**：
- 生产环境部署
- 高吞吐量需求
- 大型模型（70B+）
- 需要最大 GPU 利用率

**架构**：
```
Frontend → Router → Prefill Worker → (NIXL Transfer) → Decode Worker
```

**关键要求**：
- **必须使用 RDMA**（InfiniBand/RoCE）进行 KV cache 传输
- 无 RDMA 会导致 40x 性能下降（TTFT 从 200-500ms 增加到 10+ 秒）

**示例配置** (`disagg.yaml`)：
```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag

    VllmDecodeWorker:
      componentType: worker
      subComponentType: decode
      replicas: 1
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --disaggregation-mode
            - decode

    VllmPrefillWorker:
      componentType: worker
      subComponentType: prefill
      replicas: 1
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --disaggregation-mode
            - prefill
            - --kv-transfer-config
            - '{"kv_connector":"NixlConnector","kv_role":"kv_both"}'
```

### 2.4 架构选择指南

| 场景 | 推荐架构 |
|------|----------|
| 开发/测试 | `agg.yaml` |
| 生产+负载均衡 | `agg_router.yaml` |
| 高性能/解耦 | `disagg_router.yaml` |
| SLA 优化 | `disagg_planner.yaml` |

---

## 3. Kubernetes 部署配置

### 3.1 前置条件

| 组件 | 版本要求 | 说明 |
|------|----------|------|
| Kubernetes | v1.24+ | 支持 Dynamo Operator |
| kubectl | v1.24+ | K8s 命令行工具 |
| Helm | v3.0+ | 包管理器 |

### 3.2 平台安装

**路径 A：生产安装（推荐）**

```bash
export NAMESPACE=dynamo-system
export RELEASE_VERSION=0.x.x

# 获取并安装 Helm chart
helm fetch https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-${RELEASE_VERSION}.tgz
helm install dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace ${NAMESPACE} --create-namespace
```

**路径 B：从源码构建**

```bash
export NAMESPACE=dynamo-system
export DOCKER_SERVER=nvcr.io/nvidia/ai-dynamo/
export IMAGE_TAG=0.x.x

# 构建并推送 Operator 镜像
cd deploy/operator
docker build -t $DOCKER_SERVER/kubernetes-operator:$IMAGE_TAG .
docker push $DOCKER_SERVER/kubernetes-operator:$IMAGE_TAG
cd -

# 安装 Platform
cd deploy/helm/charts
helm dep build ./platform/
helm install dynamo-platform ./platform/ \
  --namespace "${NAMESPACE}" \
  --set "dynamo-operator.controllerManager.manager.image.repository=${DOCKER_SERVER}/kubernetes-operator" \
  --set "dynamo-operator.controllerManager.manager.image.tag=${IMAGE_TAG}"
```

### 3.3 验证安装

```bash
# 检查 CRD
kubectl get crd | grep dynamo

# 检查 operator 和平台 pods
kubectl get pods -n ${NAMESPACE}
# 预期：dynamo-operator-*、etcd-*、nats-* pods Running
```

### 3.4 DGD 服务配置详解

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  services:
    ServiceName:
      # 基础配置
      componentType: frontend | worker | default | planner
      replicas: 1
      
      # 可选：子类型（用于解耦部署）
      subComponentType: prefill | decode
      
      # Kubernetes 命名空间（逻辑隔离）
      dynamoNamespace: my-logical-namespace
      
      # 资源请求
      resources:
        limits:
          gpu: "2"              # GPU 数量
          memory: "40Gi"
          cpu: "16"
        requests:
          custom:
            ephemeral-storage: "2Gi"
      
      # 环境变量
      envs:
        - name: DYN_LOG
          value: "debug"
        - name: HF_HOME
          value: /opt/models
      
      # 从 Secret 注入环境变量
      envFromSecret: hf-token-secret
      
      # 挂载点配置
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      
      # PVC 配置
      pvcs:
        - name: model-cache
          create: false
      
      # 探针配置
      readinessProbe:
        exec:
          command: ["/bin/sh", "-c", 'grep "Worker.*initialized" /tmp/worker.log']
        initialDelaySeconds: 60
        periodSeconds: 60
      
      # Pod 级联配置
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          imagePullPolicy: IfNotPresent
          workingDir: /workspace
          command: ["/bin/sh", "-c"]
          args:
            - python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B
```

### 3.5 多节点部署

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-multinode
spec:
  services:
    my-service:
      multinode:
        nodeCount: 2          # 物理节点数
      resources:
        limits:
          gpu: "4"            # 每节点 GPU 数
      extraPodSpec:
        mainContainer:
          args:
            - "--tp-size"
            - "8"             # 必须等于 nodeCount × gpu
```

**后端自动行为**：

| 后端 | 多节点策略 |
|------|------------|
| vLLM | 使用 Ray 进行分布式 TP/PP 执行 |
| SGLang | 注入分布式初始化参数 |
| TRT-LLM | 使用 MPI 包装器启动 |

### 3.6 拓扑感知调度

```yaml
spec:
  topologyConstraint:
    packDomain: rack           # 打包域：rack, block, zone
  services:
    VllmWorker:
      # ...
```

---

## 4. 后端引擎集成

### 4.1 vLLM

**关键参数**：

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `--model` | 模型名称或路径 | `Qwen/Qwen3-32B-FP8` |
| `--tensor-parallel-size` | 张量并行度 | `2`, `4`, `8` |
| `--pipeline-parallel-size` | 流水线并行度 | `1`, `2` |
| `--max-model-len` | 最大序列长度 | `6000`, `32000` |
| `--max-num-seqs` | 最大并发序列数 | `1024` |
| `--kv-cache-dtype` | KV 缓存数据类型 | `fp8`, `fp16` |
| `--disaggregation-mode` | 解耦模式 | `prefill`, `decode` |

**共享内存配置**：
```yaml
sharedMemory:
  size: 16Gi        # vLLM 需要 16Gi
```

**聚合部署示例**：
```yaml
VllmDecodeWorker:
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
        - "--tensor-parallel-size"
        - "2"
        - "--pipeline-parallel-size"
        - "1"
        - "--max-model-len"
        - "6000"
        - "--max-num-seqs"
        - "1024"
        - "--kv-cache-dtype"
        - "fp8"
```

### 4.2 SGLang

**关键参数**：

| 参数 | 说明 |
|------|------|
| `--model-path` | 模型路径 |
| `--tp` | 张量并行度 |
| `--disaggregation-mode` | `prefill` 或 `decode` |
| `--disaggregation-transfer-backend` | `nixl` |
| `--disaggregation-bootstrap-port` | 引导端口 |

**解耦部署示例**：
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
        - --disaggregation-bootstrap-port
        - "12345"
```

### 4.3 TensorRT-LLM

**关键参数**：

| 参数 | 说明 |
|------|------|
| `--model-path` | 模型路径 |
| `--served-model-name` | 服务模型名称 |
| `--extra-engine-args` | 引擎配置文件路径 |

**共享内存配置**：
```yaml
sharedMemory:
  size: 80Gi        # TRT-LLM 需要 80Gi
```

---

## 5. RDMA 与 KV Cache 传输

### 5.1 为什么 NVLink 不能跨 Pod 工作

| 限制 | 说明 |
|------|------|
| **进程隔离** | Pod 在独立的 Linux 命名空间，无法直接访问对方内存 |
| **GPU 分区** | K8s device plugin 通过 `CUDA_VISIBLE_DEVICES` 分配 GPU |
| **内存注册** | NVLink 需要 `cudaDeviceEnablePeerAccess()` 调用，跨进程不可能 |

**NVLink 仅在 Pod 内部有效**（TP/EP 并行）。

### 5.2 支持的传输方式

| 传输方式 | 带宽 | 延迟 | 节点内 | 跨节点 | GPU Direct |
|----------|------|------|--------|--------|------------|
| **NVLink** | 450-900 GB/s | ~µs | ✅ | ❌ | ✅ |
| **InfiniBand RDMA** | 20-50 GB/s | ~1 µs | ✅ | ✅ | ✅ |
| **RoCE RDMA** | 10-25 GB/s | ~2 µs | ✅ | ✅ | ✅ |
| **TCP** | 1-3 GB/s | ~50 µs | ✅ | ✅ | ❌ |

### 5.3 UCX 配置参考

**核心传输配置**：
```yaml
env:
  - name: UCX_TLS
    value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"
```

| 传输 | 说明 | 使用场景 |
|------|------|----------|
| `rc_x` | 可靠连接（加速） | 主要 RDMA 传输 |
| `rc` | 可靠连接（标准） | RDMA 回退 |
| `dc_x` | 动态连接（加速） | 可扩展 RDMA |
| `cuda_copy` | GPU↔Host 内存暂存 | GPU 缓冲区必需 |
| `cuda_ipc` | CUDA IPC（节点内） | Pod 内 GPU 传输 |
| `tcp` | TCP 套接字 | RDMA 不可用时回退 |

**Rendezvous 协议设置**：
```yaml
env:
  - name: UCX_RNDV_SCHEME
    value: "get_zcopy"    # 零拷贝 RDMA GET
  - name: UCX_RNDV_THRESH
    value: "0"            # 所有消息使用 rendezvous
```

### 5.4 生产环境完整配置

```yaml
env:
  # 传输选择 - 带 GPU 支持的 RDMA
  - name: UCX_TLS
    value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"

  # 大传输的 rendezvous
  - name: UCX_RNDV_SCHEME
    value: "get_zcopy"
  - name: UCX_RNDV_THRESH
    value: "0"

  # 内存注册优化
  - name: UCX_IB_REG_METHODS
    value: "odp,rcache"

  # RDMA 设置
  - name: UCX_IB_GID_INDEX
    value: "3"           # RoCE v2 GID 索引（集群特定）

# Pod 安全上下文
securityContext:
  capabilities:
    add: ["IPC_LOCK"]      # RDMA 内存锁定必需

# 资源请求
resources:
  limits:
    rdma/ib: "2"          # RDMA 资源（匹配 TP 大小）
  requests:
    rdma/ib: "2"
```

### 5.5 AWS EFA 配置

> ⚠️ **关键**：在 AWS Ubuntu 24.04 + Kernel ≥6.8 上，`get_zcopy` 会导致崩溃。

**必需配置**：
```yaml
env:
  - name: UCX_TLS
    value: "srd,cuda_copy,tcp"    # SRD 是 EFA 的 RDMA 传输
  - name: UCX_RNDV_SCHEME
    value: "auto"                  # 不要使用 get_zcopy
  - name: UCX_RNDV_THRESH
    value: "8192"
```

### 5.6 性能预期

| 配置 | TTFT 开销 | 来源 |
|------|-----------|------|
| 聚合（基线） | 0 | 无 KV 传输 |
| 解耦 + InfiniBand RDMA + GPUDirect | +200-500ms | 预期值 |
| 解耦 + RoCE RDMA + GPUDirect | +300-800ms | 预期值 |
| 解耦 + 主机暂存（无 GPUDirect） | +1-3s | 预期值 |
| 解耦 + AWS EFA（无 GPUDirect） | ~3x 慢于聚合 | 实测 |
| 解耦 + TCP 回退 | **+90-100s** | 实测 ~98s TTFT |

### 5.7 诊断检查清单

- [ ] `rdma/ib` 资源可见：`kubectl get nodes -o jsonpath='{..allocatable.rdma/ib}'`
- [ ] UCX 看到 RDMA 设备：`ucx_info -d | grep "Transport: rc"`
- [ ] UCX 看到 GPU 内存：`ucx_info -d | grep "memory types.*cuda"`
- [ ] NIXL 使用 UCX 初始化：`kubectl logs <pod> | grep "Backend UCX"`
- [ ] 传输带宽 > 1 GB/s（Grafana 指标）

---

## 6. 模型缓存与加载

### 6.1 PVC + 下载 Job（推荐）

**步骤 1：创建共享 PVC**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache
spec:
  accessModes:
    - ReadWriteMany          # 多节点同时挂载必需
  resources:
    requests:
      storage: 100Gi
```

**步骤 2：下载模型 Job**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: model-download
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: downloader
          image: python:3.12-slim
          command: ["sh", "-c"]
          args:
            - |
              pip install huggingface_hub hf_transfer
              HF_HUB_ENABLE_HF_TRANSFER=1 huggingface-cli download \
                $MODEL_NAME --revision $MODEL_REVISION
          env:
            - name: MODEL_NAME
              value: "Qwen/Qwen3-32B-FP8"
            - name: HF_HOME
              value: /cache/huggingface
          volumeMounts:
            - name: model-cache
              mountPath: /cache/huggingface
      volumes:
        - name: model-cache
          persistentVolumeClaim:
            claimName: model-cache
```

**步骤 3：挂载到 DGD**
```yaml
spec:
  pvcs:
    - create: false
      name: model-cache
  services:
    VllmWorker:
      volumeMounts:
        - name: model-cache
          mountPoint: /opt/models
      envs:
        - name: HF_HOME
          value: /opt/models
```

### 6.2 编译缓存

```yaml
spec:
  pvcs:
    - name: model-cache
      create: false
    - name: compilation-cache
      create: false
  services:
    VllmWorker:
      volumeMounts:
        - name: model-cache
          mountPoint: /home/dynamo/.cache/huggingface
        - name: compilation-cache
          mountPoint: /home/dynamo/.cache/vllm
```

### 6.3 Model Express (P2P 分发)

**安装**：
```bash
helm install dynamo-platform dynamo-platform-${RELEASE_VERSION}.tgz \
  --namespace ${NAMESPACE} \
  --set "dynamo-operator.modelExpressURL=http://model-express-server.model-express.svc.cluster.local:8080"
```

**配置 Worker**：
```yaml
services:
  VllmWorker:
    envs:
      - name: VLLM_LOAD_FORMAT
        value: mx-target
```

### 6.4 何时使用何种方案

| 场景 | 推荐方案 |
|------|----------|
| 小型集群，简单设置 | PVC + Download Job |
| 大型集群，多节点 | Model Express |
| 模型已在共享存储（NFS） | PVC |
| 跨集群频繁更新模型 | Model Express |

---

## 7. 弹性伸缩配置

### 7.1 Dynamo Planner（推荐用于 LLM）

**配置模式**：

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| `throughput` | 基于队列/利用率阈值的静态扩展 | 开箱即用，无需配置 |
| `latency` | 更积极的低延迟阈值 | 延迟敏感型工作负载 |
| `sla` | 基于回归模型的 SLA 驱动扩展 | 需要精确 TTFT/ITL 控制 |

**部署示例**（通过 DGDR）：
```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: my-dgdr
spec:
  model: Qwen/Qwen3-32B-FP8
  backend: vllm
  workload:
    isl: 4000
    osl: 500
  sla:
    ttft: 600.0
    itl: 20.0
  features:
    planner:
      mode: disagg
      backend: vllm
      optimization_target: sla
      enable_throughput_scaling: true
```

### 7.2 KEDA（推荐用于通用场景）

**安装**：
```bash
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace
```

**ScaledObject 示例**：
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-deployment-scaler
spec:
  scaleTargetRef:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeploymentScalingAdapter
    name: my-deployment-decode
  minReplicaCount: 1
  maxReplicaCount: 10
  pollingInterval: 15
  cooldownPeriod: 60
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus-server:9090
      metricName: dynamo_ttft_p95
      query: |
        histogram_quantile(0.95,
          sum(rate(dynamo_frontend_time_to_first_token_seconds_bucket{
            dynamo_namespace="default-my-deployment"}[5m]))
          by (le)
        )
      threshold: "0.5"
```

### 7.3 HPA（Kubernetes 原生）

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
        selector:
          matchLabels:
            dynamo_namespace: "default-my-deployment"
      target:
        type: Value
        value: "500m"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
```

### 7.4 伸缩指标选择

| 服务类型 | 推荐指标 | Dynamo 指标 |
|----------|----------|-------------|
| Frontend | CPU 利用率、请求率 | `dynamo_frontend_requests_total` |
| Prefill | 队列深度、TTFT | `dynamo_frontend_queued_requests` |
| Decode | ITL | `dynamo_frontend_inter_token_latency_seconds` |

---

## 8. 全局规划器与多池架构

### 8.1 为什么需要 Global Planner

**单一 DGD 限制**：
- 每个 DGD 的本地 planner 只扩展自己的部署
- 难以在多个 DGD 之间应用集中式扩展策略
- 无法强制执行共享约束（如总 GPU 预算）

### 8.2 部署模式

**模式 1：多模型端点 + GPU 预算**

```
GlobalPlanner (--max-total-gpus)
├── DGD model-a: Frontend + PrefillWorker + DecodeWorker + Planner
└── DGD model-b: Frontend + PrefillWorker + DecodeWorker + Planner
```

**模式 2：单端点 + 多池**

```
Frontend (单一公共端点)
    ↓
GlobalRouter (池选择)
    ├── Prefill Pool 0 (TP1, 短 ISL)
    ├── Prefill Pool 1 (TP2, 长 ISL)
    └── Decode Pool 0 (TP1)
```

### 8.3 完整示例架构

```
DGD gp-ctrl:      Frontend + GlobalRouter + GlobalPlanner
DGD gp-prefill-0: LocalRouter + VllmPrefillWorker (TP1) + Planner
DGD gp-prefill-1: LocalRouter + VllmPrefillWorker (TP2) + Planner  
DGD gp-decode-0: LocalRouter + VllmDecodeWorker (TP1) + Planner
```

### 8.4 GlobalRouter 配置

```json
{
  "num_prefill_pools": 2,
  "num_decode_pools": 1,
  "prefill_pool_dynamo_namespaces": [
    "${K8S_NAMESPACE}-gp-prefill-0",
    "${K8S_NAMESPACE}-gp-prefill-1"
  ],
  "decode_pool_dynamo_namespaces": [
    "${K8S_NAMESPACE}-gp-decode-0"
  ],
  "prefill_pool_selection_strategy": {
    "ttft_min": 10, "ttft_max": 3000, "ttft_resolution": 2,
    "isl_min": 0, "isl_max": 32000, "isl_resolution": 2,
    "prefill_pool_mapping": [[0,1],[0,1]]
  },
  "decode_pool_selection_strategy": {
    "itl_min": 10, "itl_max": 500, "itl_resolution": 2,
    "context_length_min": 0, "context_length_max": 32000,
    "decode_pool_mapping": [[0,0],[0,0]]
  }
}
```

### 8.5 Planner 配置（委托模式）

```json
{
  "environment": "global-planner",
  "global_planner_namespace": "${K8S_NAMESPACE}-gp-ctrl",
  "backend": "vllm",
  "mode": "prefill",
  "enable_load_scaling": false,
  "enable_throughput_scaling": true,
  "throughput_metrics_source": "router",
  "ttft": 2000,
  "max_gpu_budget": -1,
  "prefill_engine_num_gpu": 1,
  "model_name": "${MODEL_NAME}"
}
```

### 8.6 GlobalPlanner 标志

| 标志 | 说明 |
|------|------|
| `--max-total-gpus N` | 拒绝超过 N 总 GPU 的请求 |
| `--managed-namespaces NS...` | 只接受列出的 Dynamo 命名空间的扩展请求 |
| `--no-operation` | 日志扩展请求但不执行（dry-run） |

---

## 9. 可观测性与监控

### 9.1 指标端点

| 组件 | 端点 | 端口 |
|------|------|------|
| Frontend | `/metrics` | 8000 (或 `DYN_HTTP_PORT`) |
| Backend Worker | `/metrics` | `DYN_SYSTEM_PORT` (默认 9090) |
| NIXL Telemetry | `/metrics` | `NIXL_TELEMETRY_PROMETHEUS_PORT` (默认 19090) |

### 9.2 核心指标

**Frontend 指标**：
- `dynamo_frontend_requests_total` - 总请求数
- `dynamo_frontend_time_to_first_token_seconds` - TTFT
- `dynamo_frontend_inter_token_latency_seconds` - ITL
- `dynamo_frontend_queued_requests` - HTTP 队列中的请求
- `dynamo_frontend_inflight_requests` - 处理中的请求

**Worker 指标**：
- `dynamo_component_requests_total` - 组件总请求数
- `dynamo_component_request_duration_seconds` - 请求处理时间
- `dynamo_component_inflight_requests` - 组件中处理中的请求

**Router 指标**：
- `dynamo_component_router_requests_total` - 路由请求总数
- `dynamo_component_router_time_to_first_token_seconds` - 路由 TTFT
- `dynamo_component_router_kv_hit_rate` - KV 缓存命中率

### 9.3 Prometheus 配置

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorNamespaceSelector.matchLabels=null
```

### 9.4 NIXL Telemetry

```yaml
env:
  - name: NIXL_TELEMETRY_ENABLE
    value: "y"
  - name: NIXL_TELEMETRY_PROMETHEUS_PORT
    value: "19090"
```

### 9.5 Grafana Dashboard

```bash
kubectl apply -n monitoring \
  -f deploy/observability/k8s/grafana-dynamo-dashboard-configmap.yaml
```

**Dashboard 面板**：
- Frontend 请求率
- TTFT / ITL 延迟
- 请求持续时间
- 输入/输出序列长度
- GPU 利用率（DCGM）
- 节点 CPU 利用率和系统负载
- 每 Pod 容器 CPU 使用率
- 内存使用率

---

## 10. 故障容错机制

### 10.1 故障层级

| 层级 | 机制 | 实际效果 |
|------|------|----------|
| 请求 | 迁移、取消 | 进行中的工作可以继续或终止 |
| Worker | 健康检查、优雅关闭 | 失败/终止的 Worker 安全停止接收新流量 |
| 系统 | 请求拒绝/负载卸载 | 防止过载传播到 Worker |
| 基础设施 | Discovery 租约过期、事件路径恢复 | 过期成员被移除，流量重新路由 |

### 10.2 请求迁移

```yaml
args:
  - "--migration-limit"
  - "3"          # 允许最多 3 次迁移
```

**工作原理**：
1. Worker 故障检测
2. Frontend 发现故障 Worker 被移除
3. 进行中的请求重新路由到健康 Worker
4. 保留部分生成状态

### 10.3 优雅关闭

**配置**：
```yaml
env:
  - name: DYN_GRACEFUL_SHUTDOWN_TIMEOUT
    value: "30"
```

**行为**：
1. 接收 SIGTERM 信号
2. 立即停止接收新请求
3. 排空进行中的请求（或超时）
4. 清理资源

### 10.4 健康检查

```yaml
env:
  - name: DYN_HEALTH_CHECK_ENABLED
    value: "true"
  - name: DYN_CANARY_WAIT_TIME
    value: "10"
  - name: DYN_HEALTH_CHECK_REQUEST_TIMEOUT
    value: "3"
```

### 10.5 请求拒绝（负载卸载）

```yaml
args:
  - "--active-decode-blocks-threshold"
  - "50"
  - "--active-prefill-tokens-threshold"
  - "4096"
```

当超过阈值时返回 HTTP 503。

---

## 11. 生产环境清单

### 11.1 基础设施检查

- [ ] Kubernetes 集群 v1.24+
- [ ] GPU 节点可用（nvidia.com/gpu）
- [ ] RDMA 设备插件安装（用于解耦部署）
- [ ] etcd 和 NATS 可用（平台安装）
- [ ] Prometheus 和 Grafana 部署
- [ ] ReadWriteMany 存储类可用

### 11.2 镜像准备

- [ ] Dynamo Frontend 镜像
- [ ] Worker 运行时镜像（vLLM/SGLang/TRT-LLM）
- [ ] 所有镜像推送到可用 registry

### 11.3 部署前配置

- [ ] HuggingFace token secret 创建
- [ ] 模型缓存 PVC 创建
- [ ] 模型预下载（如使用 PVC）
- [ ] RDMA 网络验证（用于解耦部署）
- [ ] 拓扑配置（如需要）

### 11.4 部署验证

```bash
# 1. 检查 CRD
kubectl get crd | grep dynamo

# 2. 检查 operator
kubectl get pods -n dynamo-system

# 3. 部署 DGD
kubectl apply -f my-deployment.yaml -n my-namespace

# 4. 检查 pods
kubectl get pods -n my-namespace

# 5. 查看 logs
kubectl logs -f -l nvidia.com/dynamo-component=frontend -n my-namespace

# 6. 端口转发测试
kubectl port-forward svc/my-deployment-frontend 8000:8000 -n my-namespace
curl http://localhost:8000/v1/models

# 7. 发送测试请求
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "Hello!"}]}'
```

### 11.5 性能验证

```bash
# 1. 安装 AIPerf
pip install aiperf

# 2. 运行基准测试
aiperf profile \
  -m Qwen/Qwen3-32B-FP8 \
  --endpoint-type chat \
  -u http://my-deployment-frontend:8000 \
  --isl 4000 --osl 500 \
  --num-requests 800 \
  --concurrency 56 \
  --streaming

# 3. 检查指标
kubectl port-forward svc/prometheus-server 9090:9090 -n monitoring
# 访问 http://localhost:9090 查看 Prometheus
```

### 11.6 解耦部署 RDMA 验证

```bash
# 1. 检查 RDMA 设备
kubectl debug node/<node> -it --image=ubuntu:22.04 -- ibv_devinfo

# 2. 检查 UCX 初始化日志
kubectl logs <worker-pod> | grep -i "NIXL\|UCX"
# 期望输出：NIXL INFO Backend UCX was instantiated

# 3. 检查 RDMA 资源分配
kubectl get pod <worker-pod> -o yaml | grep -A5 "resources:"

# 4. 验证 UCX 传输
kubectl exec <worker-pod> -- ucx_info -d | grep cuda
# 期望：看到 cuda 内存类型支持
```

### 11.7 监控配置

- [ ] PodMonitor 自动创建（通过 operator）
- [ ] Prometheus 抓取配置
- [ ] Grafana Dashboard 导入
- [ ] 告警规则配置（如需要）

### 11.8 生产检查清单

| 类别 | 检查项 |
|------|--------|
| **高可用** | 多副本部署、前端负载均衡 |
| **扩展性** | HPA/KEDA/Planner 配置 |
| **性能** | RDMA 配置、KV 缓存命中率 |
| **可观测性** | 指标、日志、追踪 |
| **安全** | Secret 管理、RBAC、网络策略 |
| **成本** | GPU 预算、资源限制、自动伸缩 |

---

## 参考链接

- [Dynamo 官方文档](https://github.com/ai-dynamo/dynamo)
- [AIConfigurator](https://github.com/ai-dynamo/aiconfigurator)
- [vLLM 部署示例](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy)
- [SGLang 部署示例](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/sglang/deploy)
- [Global Planner 示例](https://github.com/ai-dynamo/dynamo/tree/main/examples/global_planner)
- [NIXL 项目](https://github.com/ai-dynamo/nixl)
- [Grove](https://github.com/NVIDIA/grove)
- [KAI Scheduler](https://github.com/NVIDIA/KAI-Scheduler)
