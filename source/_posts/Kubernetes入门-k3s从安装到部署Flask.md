---
title: Kubernetes入门—k3s从安装到部署Flask
date: 2026-07-21 01:30:00
categories: [运维]
tags: [Kubernetes, k3s, Docker, 容器编排, Flask]
---

## 为什么要学 K8s

Docker Compose 适合单机跑几个容器，但 Kubernetes 才是容器编排的事实标准。面试时"你会 K8s 吗"是标配问题。

k3s 是 Rancher 出的轻量版 Kubernetes，专为小机器和边缘场景设计，2核2G 就能跑。它打包了 containerd、kubelet、kubectl 等全套组件到一个 60MB 的二进制文件里。

本文记录在阿里云 ECS（2核2G，Rocky Linux 8）上从零安装 k3s 并部署 Flask API 的完整过程。

## 安装过程

### 踩坑记录

第一个坑：最新版 k3s 不支持 Rocky Linux 8 的 cgroup v1：

```
Error: kubelet is configured to not run on a host using cgroup v1
```

解决：降级到 `v1.30.6+k3s1`。

第二个坑：k3s 默认用 containerd 运行时，拉取 Docker Hub 镜像在国内超时。解决：用 `--docker` 参数让 k3s 复用 Docker 的阿里云镜像加速。

### 最终安装命令

```bash
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | \
  INSTALL_K3S_MIRROR=cn \
  INSTALL_K3S_VERSION=v1.30.6+k3s1 \
  INSTALL_K3S_EXEC="--docker" \
  sh -
```

第三个坑：k3s 自带的 Traefik 会抢 80/443 端口，导致已有的 Nginx 博客 404。解决：删掉 Traefik，继续用 Docker Compose 的 Nginx 做入口。

## 核心概念

K8s 有几十个概念，但入门只需要三个：

### Pod — 最小调度单位

Pod 是 K8s 里最小的东西，一个 Pod 包装一个或多个容器。每个 Pod 有自己的集群内部 IP。

```bash
kubectl run my-first-pod --image=nginx:alpine --port=80
```

Pod 的致命弱点：删了就没了，不会自动重启。

### Service — 网络入口

Service 给 Pod 提供固定的 IP 和端口，不随 Pod 的创建销毁而改变。

```bash
kubectl expose pod my-first-pod --type=NodePort --port=80
```

NodePort 类型会自动分配 30000-32767 的端口，外部可以直接访问。

### Deployment — Pod 管家

Deployment 保证"我说跑几个 Pod 就跑几个，挂了自动拉起来"。

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 2        # 我要 2 个
  selector:
    matchLabels:
      app: nginx
  template:           # Pod 模板
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
```

**自愈实测**：手动 `kubectl delete pod` 杀掉一个 Pod，Deployment 立刻创建新的补上。

## 部署 Flask API

之前 Docker Compose 里跑的 Flask API，迁移到 K8s：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-api
  template:
    metadata:
      labels:
        app: flask-api
    spec:
      containers:
      - name: flask-api
        image: flask-api:v4
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: flask-api-svc
spec:
  type: NodePort
  selector:
    app: flask-api
  ports:
  - port: 5000
    targetPort: 5000
```

部署后验证：

```bash
curl localhost:32473/api/status
→ {"disk":"17.4G/39.8G","load":"0.01, 0.18, 0.47","memory":"1023.6M/1.6G"}
```

Flask API 在 K8s 上正常返回数据。

## K8s vs Docker Compose

| 场景 | Docker Compose | Kubernetes |
|------|---------------|------------|
| 单机部署 | 简洁够用 | 配置多 |
| 多节点集群 | 不行 | 天生支持 |
| 自动重启 | restart: always | Deployment 自愈 |
| 滚动更新 | 手动 | kubectl rollout |
| 服务发现 | container_name | Service + DNS |
| 面试加分 | 😅 | 🔥 |

## 总结

从零到在 K8s 上跑起 Flask API，核心就三句话：

1. **Pod** — 容器的壳，IP 是临时的
2. **Service** — 给 Pod 提供固定入口
3. **Deployment** — 管 Pod 的，自愈 + 滚动更新

整套配置和 YAML 文件已推到 [ops-portfolio](https://github.com/Bingsv/ops-portfolio) 仓库。
