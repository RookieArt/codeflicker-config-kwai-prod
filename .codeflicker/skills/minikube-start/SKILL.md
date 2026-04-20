---
name: minikube-start
description: 启动 minikube 并确保集群完全就绪
version: 1.0.0
---

# minikube 启动 Skill

启动 minikube 并确保所有系统组件正常运行。

## 启动步骤

### 1. 启动 minikube

```bash
minikube start
```

### 2. 检查节点状态

```bash
kubectl get nodes
kubectl get po -A
```

## 常见问题：节点 NotReady（CNI 未就绪）

### 现象

```
NAME       STATUS     ROLES           AGE   VERSION
minikube   NotReady   control-plane   ...   v1.28.0
```

`kubectl describe node minikube` 显示：

```
KubeletNotReady: container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

### 原因

minikube 使用 docker 驱动时，默认不安装 CNI 网络插件，导致节点无法就绪，coredns 等 Pod 一直 Pending。

### 解决方案：安装 Flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

等待约 60 秒，Flannel 拉取镜像并初始化后，节点会自动变为 Ready：

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   ...   v1.28.0
```

### 验证集群就绪

```bash
kubectl get po -A
```

所有 Pod 均应为 `Running` 状态，包括：
- `kube-flannel/kube-flannel-ds-*`
- `kube-system/coredns-*`
- `kube-system/storage-provisioner`

## 注意事项

| 场景 | 说明 |
|------|------|
| 首次启动 / 重建集群 | 需要安装 Flannel，否则节点永远 NotReady |
| 重启已有集群 | Flannel 已安装，节点会自动就绪，无需重新安装 |
| 镜像拉取时间 | Flannel 需拉取两个镜像，视网络情况需 30~120 秒 |
