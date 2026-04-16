---
name: github-clone
description: 从 GitHub 下载开源项目到本地
version: 1.0.0
---

# GitHub Clone Skill

从 GitHub 下载开源项目到本地。

## 使用场景

当用户需要从 GitHub 下载开源项目时使用此 skill。

## 工作流程

1. **搜索项目地址**
   - 使用 `google_search` 工具搜索项目的 GitHub 仓库链接
   - 确认正确的仓库 URL (格式: `https://github.com/{org}/{repo}`)

2. **配置代理 (如需要)**
   - 检查环境变量代理: `HTTP_PROXY`, `HTTPS_PROXY`
   - 检查 git 代理: `git config --global --get http.proxy`
   - 如果环境变量有代理但 git 没有,配置 git 代理:
     ```bash
     git config --global http.proxy http://PROXY_HOST:PORT
     git config --global https.proxy http://PROXY_HOST:PORT
     ```
   - 默认代理地址: `10.74.176.8:11080`

3. **测试网络连通性**
   - 使用 curl 测试代理是否可用:
     ```bash
     curl -I --proxy http://PROXY_HOST:PORT --connect-timeout 10 https://github.com
     ```
   - 如果返回 `HTTP/1.1 200 Connection Established` 表示代理正常

4. **克隆仓库**
   - 目标目录: 当前工作目录下的 `github/` 子目录
   - 首次尝试浅克隆 (快速获取):
     ```bash
     mkdir -p github && git clone --depth 1 https://github.com/{org}/{repo}.git github/{repo}
     ```
   - 如果用户需要完整历史,删除后重新完整克隆:
     ```bash
     rm -rf github/{repo} && git clone https://github.com/{org}/{repo}.git github/{repo}
     ```
   - 如果克隆中断,增加缓冲区:
     ```bash
     git config --global http.postBuffer 524288000
     ```

5. **验证结果**
   - 检查目录是否存在
   - 验证克隆类型:
     ```bash
     git rev-parse --is-shallow-repository
     ```
     - 返回 `true`: 浅克隆
     - 返回 `false`: 完整克隆

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| `unexpected disconnect while reading sideband packet` | 连接中断 | 增大 http.postBuffer,使用浅克隆 |
| 超时 | 仓库太大或网络慢 | 使用 `--depth 1` 浅克隆 |
| 无法连接 | 未配置代理 | 配置 git 全局代理 |

## 示例

```
用户: 帮我下载 nvidia 开源的 dynamo 项目

执行步骤:
1. 搜索 "nvidia dynamo github" 找到 https://github.com/ai-dynamo/dynamo
2. 配置 git 代理 (如果未配置)
3. 测试代理连通性
4. 克隆到 github/dynamo
5. 验证克隆成功
```
