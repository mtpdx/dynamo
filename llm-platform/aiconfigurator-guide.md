# AIConfigurator 自动化配置指南

本文档介绍如何使用 AIConfigurator 自动化生成 Dynamo 部署配置。

## 1. AIConfigurator 概述

AIConfigurator 是一个性能优化工具，可以自动为 Dynamo 部署找到最优配置：

- **自动确定**：最佳 prefill/decode worker 数量
- **并行设置**：tensor/pipeline parallel 配置
- **SLA 合规**：满足 TTFT 和 TPOT 目标
- **性能提升**：相比手动配置提升高达 1.7x 吞吐量

## 2. 安装

```bash
pip install aiconfigurator
```

## 3. 快速开始

### 3.1 基础用法

```bash
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 8 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --isl 4000 \
  --osl 500 \
  --ttft 600 \
  --tpot 16.67 \
  --save-dir ./results
```

### 3.2 参数说明

| 参数 | 说明 | 示例 |
|------|------|------|
| `--model` | HuggingFace 模型 ID | `Qwen/Qwen3-32B-FP8` |
| `--total-gpus` | 可用 GPU 总数 | `8`, `16`, `32` |
| `--system` | GPU 系统类型 | `h200_sxm`, `h100_sxm`, `a100_sxm` |
| `--backend` | 推理后端 | `vllm`, `trtllm`, `sglang` |
| `--backend-version` | 后端版本 | `0.12.0` |
| `--isl` | 输入序列长度 | `4000` |
| `--osl` | 输出序列长度 | `500` |
| `--ttft` | TTFT SLA 目标 (ms) | `600` |
| `--tpot` | TPOT SLA 目标 (ms) | `16.67` |
| `--save-dir` | 结果保存目录 | `./results` |

## 4. 完整示例：vLLM on H200

### 4.1 运行 AIConfigurator

```bash
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --system h200_sxm \
  --total-gpus 8 \
  --isl 4000 \
  --osl 500 \
  --ttft 600 \
  --tpot 25 \
  --backend vllm \
  --backend-version 0.12.0 \
  --generator-dynamo-version 1.0.0 \
  --generator-set K8sConfig.k8s_namespace=$YOUR_NAMESPACE \
  --generator-set K8sConfig.k8s_pvc_name=$YOUR_PVC \
  --save-dir ./results_vllm
```

### 4.2 输出解读

AIConfigurator 输出聚合和解耦两种架构的比较：

```
agg Top Configurations: (Sorted by tokens/s/gpu)
| Rank | backend | tokens/s/gpu | TTFT  | request_latency | concurrency | total_gpus | replicas | gpus/replica | parallel | bs |
|------|---------|-------------|-------|-----------------|-------------|------------|---------|--------------|----------|----|
|  1   |   vllm  |    322.69   | 546.92|     12490.03    |  64 (=32x2) |     8      |    2    |      4       |  tp4pp1  | 32 |

disagg Top Configurations: (Sorted by tokens/s/gpu)
| Rank | backend | tokens/s/gpu | TTFT  | (p)workers | (p)gpus/worker | (d)workers | (d)gpus/worker | (d)parallel | (d)bs |
|------|---------|-------------|-------|-------------|-----------------|-------------|-----------------|-------------|-------|
|  1   |   vllm  |    446.85   | 453.18|     2       |    2 (=2x1)     |     1      |    4 (=4x1)     |    tp4pp1   |   76  |
```

### 4.3 关键指标

| 指标 | 说明 | 优化方向 |
|------|------|----------|
| `tokens/s/gpu` | 每 GPU 吞吐量效率 | 越高越好 |
| `tokens/s/user` | 每用户生成速度 | 越高越好 |
| `TTFT` | 首个 token 时间 | 越低越好 |
| `concurrency` | 总并发请求数 | 根据工作负载选择 |
| `total_gpus` | 使用的总 GPU 数 | 根据预算选择 |

### 4.4 部署生成配置

```bash
# 使用生成的配置部署
kubectl apply -f ./results_vllm/disagg/top1/disagg/k8s_deploy.yaml
```

## 5. 自定义实验

### 5.1 实验配置 YAML

```yaml
# custom_exp.yaml
exps:
  - exp_tp2
  - exp_tp4

exp_tp2:
  mode: "patch"
  serving_mode: "agg"
  model_path: "Qwen/Qwen3-32B-FP8"
  total_gpus: 8
  system_name: "h200_sxm"
  backend_name: "vllm"
  backend_version: "0.12.0"
  isl: 4000
  osl: 500
  ttft: 600
  tpot: 16.67
  config:
    agg_worker_config:
      tp_list: [2]

exp_tp4:
  mode: "patch"
  serving_mode: "agg"
  model_path: "Qwen/Qwen3-32B-FP8"
  total_gpus: 8
  system_name: "h200_sxm"
  backend_name: "vllm"
  backend_version: "0.12.0"
  isl: 4000
  osl: 500
  ttft: 600
  tpot: 16.67
  config:
    agg_worker_config:
      tp_list: [4]
```

### 5.2 运行实验

```bash
aiconfigurator cli exp --yaml-path custom_exp.yaml --save-dir ./results_custom
```

## 6. 常见使用场景

### 6.1 严格延迟 SLA（实时聊天）

```bash
aiconfigurator cli default \
  --model meta-llama/Llama-3.1-70B \
  --total-gpus 16 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --ttft 200 --tpot 8
```

### 6.2 高吞吐量（批处理）

```bash
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 32 \
  --system h200_sxm \
  --backend trtllm \
  --ttft 2000 --tpot 50
```

### 6.3 请求延迟约束（端到端 SLA）

```bash
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 16 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --request-latency 12000 \
  --isl 4000 --osl 500
```

## 7. 调整实际工作负载

### 7.1 长输出（聊天/代码生成）

```bash
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 8 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --isl 2000 \
  --osl 2000 \
  --ttft 1000 \
  --tpot 10 \
  --save-dir ./results_long_output
```

### 7.2 生产级配置调优

```bash
aiconfigurator cli default \
  --model Qwen/Qwen3-32B-FP8 \
  --total-gpus 8 \
  --system h200_sxm \
  --backend vllm \
  --backend-version 0.12.0 \
  --isl 4000 --osl 500 \
  --ttft 600 --tpot 16.67 \
  --save-dir ./results_tuned \
  --generator-set Workers.agg.kv_cache_free_gpu_memory_fraction=0.85 \
  --generator-set Workers.agg.max_num_seqs=2048
```

## 8. 部署后验证

### 8.1 AIPerf 基准测试

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: aiperf-benchmark
  namespace: my-namespace
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: aiperf
        image: python:3.10
        command:
        - /bin/bash
        - -c
        - |
          pip install aiperf
          aiperf profile \
            -m Qwen/Qwen3-32B-FP8 \
            --endpoint-type chat \
            -u http://dynamo-agg-frontend:8000 \
            --isl 4000 --isl-stddev 0 \
            --osl 500 --osl-stddev 0 \
            --num-requests 800 \
            --concurrency 56 \
            --streaming \
            --extra-inputs "ignore_eos:true"
```

```bash
kubectl apply -f k8s_bench.yaml
kubectl logs -f -l job-name=aiperf-benchmark
```

### 8.2 预期结果对比

| 指标 | AIConfigurator 预测 | 实际结果 | 状态 |
|------|---------------------|----------|------|
| TTFT (ms) | 509 | 209 | ✅ |
| ITL/TPOT (ms) | 16.49 | 15.06 | ✅ |
| 吞吐量 (req/s) | ~6.3 | 6.9 | ✅ |

> **注意**：实际吞吐量通常达到 AIConfigurator 预测的 85-90%，ITL/TPOT 是最准确的指标。

## 9. 解耦部署 RDMA 要求

> ⚠️ **关键**：解耦部署**必须使用 RDMA**。没有 RDMA，性能下降 40 倍。

### 9.1 前置条件

1. RDMA 网络（InfiniBand 或 RoCE）
2. RDMA 设备插件安装
3. ETCD 和 NATS 部署

### 9.2 生成配置中的 RDMA 设置

AIConfigurator 生成的 `k8s_deploy.yaml` 已包含 RDMA 配置：

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
  requests:
    rdma/ib: "2"
```

### 9.3 验证 RDMA 激活

```bash
kubectl logs <prefill-worker-pod> | grep -i "UCX\|NIXL"
```

期望输出：`NIXL INFO Backend UCX was instantiated`

如果只看到 TCP 传输，RDMA 未激活。

## 10. 故障排除

### 10.1 模型未找到

使用完整 HuggingFace 路径：
```bash
# 错误
aiconfigurator --model QWEN3_32B

# 正确
aiconfigurator --model Qwen/Qwen3-32B-FP8
```

### 10.2 后端版本不匹配

```bash
aiconfigurator cli support --model <model> --system <system> --backend <backend>
```

### 10.3 Pod 崩溃

```bash
# 检查镜像访问
kubectl describe pod <pod> | grep -i "failed\|error"

# 检查 HuggingFace token
kubectl get secret hf-token-secret -o yaml

# 检查共享内存配置
# vLLM 需要 16Gi, TRT-LLM 需要 80Gi
```

### 10.4 性能低于预测

1. 确保有足够的预热请求（40+）
2. 检查集群是否有竞争负载
3. 验证 KV 缓存内存分数是否优化
4. 从集群内部运行基准测试以消除网络延迟

## 11. 参考链接

- [AIConfigurator CLI 指南](https://github.com/ai-dynamo/aiconfigurator/blob/main/docs/cli_user_guide.md)
- [Dynamo 部署指南](https://github.com/ai-dynamo/aiconfigurator/blob/main/docs/dynamo_deployment_guide.md)
- [支持矩阵](https://github.com/ai-dynamo/aiconfigurator/blob/main/src/aiconfigurator/systems/support_matrix.csv)
