---
name: minikube-model-mount
description: minikube 环境下将本地模型挂载到 Pod
version: 1.0.0
---

# minikube 模型挂载 Skill

在 minikube 环境下将本地下载的模型挂载到 Kubernetes Pod。

## 背景问题

minikube 使用 docker 驱动时，不支持 `minikube mount` 命令（9p 文件系统）。需要通过 `docker cp` 将模型复制到 minikube 容器内，再使用 hostPath 挂载到 Pod。

## 工作流程

### 1. 下载模型到本地

使用 hf-download skill 下载模型到本地目录：

```bash
hf download Qwen/Qwen3-0.6B --local-dir ./models/Qwen3-0.6B
```

### 2. 复制模型到 minikube 容器

```bash
# 创建目标目录
docker exec minikube mkdir -p /mnt/models

# 复制模型目录
docker cp ./models/Qwen3-0.6B minikube:/mnt/models/
```

### 3. 验证复制成功

```bash
docker exec minikube ls -la /mnt/models/Qwen3-0.6B/
```

应包含：
- `config.json` - 模型配置
- `model.safetensors` - 模型权重
- `tokenizer.json`, `tokenizer_config.json` - tokenizer 文件

### 4. 配置 Pod 挂载

在 Kubernetes YAML 中使用 hostPath：

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: my-container
        volumeMounts:
        - name: model-storage
          mountPath: /mnt/models
      volumes:
      - name: model-storage
        hostPath:
          path: /mnt/models
          type: Directory
```

### 5. DynamoGraphDeployment 配置示例

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: sglang-demo
  namespace: dynamo-system
spec:
  services:
    Frontend:
      componentType: frontend
      extraPodSpec:
        mainContainer:
          env:
            - name: DYN_LOCAL_MODEL_STORE
              value: /mnt/models
          volumeMounts:
            - name: model-storage
              mountPath: /mnt/models
        volumes:
          - name: model-storage
            hostPath:
              path: /mnt/models
              type: Directory
    decode:
      componentType: worker
      extraPodSpec:
        mainContainer:
          args:
            - --model-path
            - /mnt/models/Qwen3-0.6B
          volumeMounts:
            - name: model-storage
              mountPath: /mnt/models
        volumes:
          - name: model-storage
            hostPath:
              path: /mnt/models
              type: Directory
```

## 关键注意事项

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Frontend 无法识别模型 | 未设置 `DYN_LOCAL_MODEL_STORE` | 添加环境变量指向模型目录 |
| 模型路径错误 | 使用 HF ID 而非本地路径 | 使用 `/mnt/models/{model_name}` 格式 |
| hostPath 不可访问 | 模型未复制到 minikube | 执行 `docker cp` 复制模型 |
| 权限问题 | 文件属主不正确 | 检查容器内文件权限 |

## 模型大小参考

| 模型 | 大小 | 复制时间 (SSD) |
|------|------|----------------|
| Qwen3-0.6B | ~1.5GB | ~10秒 |
| Qwen2.5-7B | ~15GB | ~2分钟 |
| Llama-3.1-8B | ~16GB | ~2分钟 |

## 清理

```bash
# 删除 minikube 内的模型
docker exec minikube rm -rf /mnt/models/Qwen3-0.6B
```
