---
name: hf-download
description: 从 HuggingFace 或镜像站下载开源模型到本地
version: 1.0.0
---

# HuggingFace 模型下载 Skill

从 HuggingFace 或镜像站下载开源模型到本地。

## 使用场景

当用户需要从 HuggingFace 下载开源模型时使用此 skill。

## 工作流程

1. **确认模型信息**
   - 模型 ID 格式: `{org}/{model_name}` (如 `Qwen/Qwen3-0.6B`)
   - 确认下载目标目录

2. **配置环境变量**
   - 使用国内镜像站:
     ```bash
     export HF_ENDPOINT=https://hf-mirror.com
     ```
   - 配置代理 (推荐使用 `10.66.70.240:11080`):
     ```bash
     export HTTP_PROXY=http://10.66.70.240:11080
     export HTTPS_PROXY=http://10.66.70.240:11080
     ```

3. **检查 huggingface_hub 安装**
   ```bash
   pip install -U huggingface_hub
   ```

4. **下载模型**
   - 使用 `hf download` 命令 (新版本 CLI):
     ```bash
     HF_ENDPOINT=https://hf-mirror.com hf download {model_id} --local-dir ./{model_name}
     ```
   - 完整示例:
     ```bash
     HTTP_PROXY=http://10.66.70.240:11080 \
     HTTPS_PROXY=http://10.66.70.240:11080 \
     HF_ENDPOINT=https://hf-mirror.com \
     hf download Qwen/Qwen3-0.6B --local-dir ./Qwen3-0.6B
     ```

5. **验证下载**
   - 检查目录结构,应包含:
     - `config.json` - 模型配置
     - `model.safetensors` 或 `pytorch_model.bin` - 模型权重
     - `tokenizer.json`, `tokenizer_config.json` - tokenizer 文件
     - `vocab.json`, `merges.txt` - 词表文件 (如适用)

## 代理选择

| 代理地址 | 端口 | 说明 |
|----------|------|------|
| `10.66.70.240` | `11080` | 推荐,速度较快 |
| `10.74.176.8` | `11080` | 备选 |

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `huggingface-cli: command not found` | 旧版 CLI 已弃用 | 使用 `hf` 命令代替 |
| `Permission denied: ~/.cache/huggingface` | 缓存目录权限问题 | 使用 `--local-dir` 指定目录 |
| 下载速度慢 | 未使用代理或代理不佳 | 切换代理地址 |
| 连接超时 | 网络问题 | 检查代理配置,使用镜像站 |

## 示例

```
用户: 下载 Qwen/Qwen3-0.6B 模型

执行步骤:
1. 确认模型 ID: Qwen/Qwen3-0.6B
2. 配置代理和镜像环境变量
3. 运行: hf download Qwen/Qwen3-0.6B --local-dir ./Qwen3-0.6B
4. 验证下载完成
```

## Python API 方式

如需在代码中下载:

```python
from huggingface_hub import snapshot_download

snapshot_download(
    repo_id="Qwen/Qwen3-0.6B",
    local_dir="./Qwen3-0.6B",
    endpoint="https://hf-mirror.com",
)
```

## Kubernetes 部署集成

下载模型后用于 Kubernetes 部署：

1. **复制到 minikube**（docker 驱动）：
   ```bash
   docker exec minikube mkdir -p /mnt/models
   docker cp ./Qwen3-0.6B minikube:/mnt/models/
   ```

2. **在 Pod 中使用 hostPath 挂载**：
   ```yaml
   volumeMounts:
     - name: model-storage
       mountPath: /mnt/models
   volumes:
     - name: model-storage
       hostPath:
         path: /mnt/models
         type: Directory
   ```

详见 `minikube-model-mount` skill。
