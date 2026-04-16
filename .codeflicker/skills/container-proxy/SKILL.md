---
name: container-proxy
description: Kubernetes 容器内代理配置指南
version: 1.0.0
---

# Kubernetes 容器内代理配置 Skill

在 Kubernetes 容器内配置代理，使容器能够访问外部网络资源。

## 使用场景

当 Kubernetes Pod 内的容器需要访问外部网络（如下载模型、拉取镜像、访问 API）时使用此 skill。

## 代理地址

| 代理地址 | 端口 | 适用场景 |
|----------|------|----------|
| `10.74.176.8` | `11080` | 推荐，通用代理，适合大多数外部访问 |
| `10.66.70.240` | `11080` | 备选，访问 hf-mirror.com 速度快 |

## Kubernetes 配置方式

### 方式一：在 YAML 中配置环境变量

```yaml
spec:
  containers:
  - name: my-container
    env:
    - name: HTTP_PROXY
      value: "http://10.74.176.8:11080"
    - name: HTTPS_PROXY
      value: "http://10.74.176.8:11080"
    - name: NO_PROXY
      value: "localhost,127.0.0.1,.internal,192.168.49.0/24,10.0.0.0/8,.nvidia.com,.nvcr.io"
```

### 方式二：使用 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: proxy-config
  namespace: dynamo-system
data:
  HTTP_PROXY: "http://10.74.176.8:11080"
  HTTPS_PROXY: "http://10.74.176.8:11080"
  NO_PROXY: "localhost,127.0.0.1,.internal,192.168.49.0/24,10.0.0.0/8,.nvidia.com,.nvcr.io"
---
spec:
  containers:
  - name: my-container
    envFrom:
    - configMapRef:
        name: proxy-config
```

### 方式三：DynamoGraphDeployment 配置

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: sglang-demo
  namespace: dynamo-system
spec:
  services:
    decode:
      componentType: worker
      extraPodSpec:
        mainContainer:
          env:
          - name: HTTP_PROXY
            value: "http://10.74.176.8:11080"
          - name: HTTPS_PROXY
            value: "http://10.74.176.8:11080"
          - name: NO_PROXY
            value: "localhost,127.0.0.1,.internal,192.168.49.0/24,10.0.0.0/8,.nvidia.com,.nvcr.io"
```

## NO_PROXY 配置说明

NO_PROXY 用于指定不走代理的地址，必须包含：

| 地址 | 说明 |
|------|------|
| `localhost,127.0.0.1` | 本地回环地址 |
| `.internal` | Kubernetes 内部 DNS 后缀 |
| `192.168.49.0/24` | minikube 集群网络 |
| `10.0.0.0/8` | 内部网络 CIDR |
| `.nvidia.com,.nvcr.io` | NVIDIA 镜像仓库（直连更快） |

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| kubectl 连接 Forbidden | 主机代理设置影响 minikube 连接 | 清除 NO_PROXY 环境变量 |
| 容器无法访问外部网络 | 未配置代理环境变量 | 添加 HTTP_PROXY/HTTPS_PROXY |
| 模型下载超时 | 使用错误的代理地址 | 切换到合适的代理 |
| 内部服务调用经过代理 | NO_PROXY 未排除内部地址 | 添加内部地址到 NO_PROXY |

## 主机环境与容器环境

**主机环境执行 kubectl：**
- 不需要代理（连接本地 minikube）
- 如遇 Forbidden 错误，清除代理环境变量：`unset HTTP_PROXY HTTPS_PROXY NO_PROXY`

**容器内环境：**
- 需要配置代理访问外部网络
- 通过 Pod YAML 的 env 字段配置

## 验证代理配置

```bash
# 进入容器验证
kubectl exec -n dynamo-system <pod-name> -- env | grep -i proxy

# 测试网络连通性
kubectl exec -n dynamo-system <pod-name> -- curl -s https://huggingface.co
```

## Dynamo Frontend 特殊配置

Dynamo Frontend 组件需要在启动时从 HuggingFace 下载模型配置，需要配置代理和本地模型存储路径：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
spec:
  services:
    Frontend:
      componentType: frontend
      extraPodSpec:
        mainContainer:
          env:
            # 本地模型存储路径，避免 Frontend 尝试下载
            - name: DYN_LOCAL_MODEL_STORE
              value: /mnt/models
            # 代理配置（用于下载 tokenizer 等）
            - name: HTTP_PROXY
              value: "http://10.74.176.8:11080"
            - name: HTTPS_PROXY
              value: "http://10.74.176.8:11080"
            - name: NO_PROXY
              value: "localhost,127.0.0.1,.internal,192.168.49.0/24,10.0.0.0/8,.nvidia.com,.nvcr.io"
```

**重要**：如果不设置 `DYN_LOCAL_MODEL_STORE`，Frontend 会尝试从 HuggingFace 下载模型配置，可能导致模型无法被识别。
