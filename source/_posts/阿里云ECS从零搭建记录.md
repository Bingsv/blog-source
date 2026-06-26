---
title: 阿里云ECS从零搭建记录
date: 2026-06-27 01:06:13
tags: [阿里云, ECS, 运维, Linux, 云计算]
---

## 背景

作为一名准大四学生，暑假开始系统学习运维。第一步是在云上搞一台属于自己的 Linux 服务器。

我选择了阿里云的学生优惠——**"学生专属算力包"**，有 850 元额度，对学习来说完全够用。

## 选型过程

### 学生认证

之前阿里云学生认证过期了，重新验证了下，很快通过。

### 选哪个实例

学生计划里有几个选项：

- e实例 2核2G
- 轻量应用服务器 2核2G
- c7实例 2核4G

最终选了 **e实例 2核2G**，因为：
1. 学生优惠覆盖，基本免费
2. 学习用足够了，后面不够再升配

### 选配置

可选规格中选了 ecs.e 系列，可用区选的是**华东1（杭州）**，地域离得近延迟低。

**注意**：系统盘默认 40G ESSD Entry，不支持变更，够用就行。

## 网络配置（踩坑）

### 专有网络 vs 公网IP

购买页面默认只有专有网络和虚拟交换机选项，**没有公网IP**。一开始以为漏掉了什么配置。

解决方案：先买实例，之后**单独创建弹性公网 IP（EIP）**再绑定。

### EIP 创建

创建弹性 IP 时几个关键选项：
- **防护类型**：默认
- **地址池**：默认
- **流量计费**：按使用流量（省钱）

### 带宽选择

带宽选 3Mbps 后发现无法使用学生优惠券——改成**按使用流量计费**就好了，平时学习测试流量不大。

## SSH 连接踩坑

### Windows 上 SSH

Windows 上用 **Git Bash** 自带的 ssh 命令，不需要额外装。

### PEM 密钥

阿里云生成密钥的时候下载了一个 `.pem` 文件。连接命令：

```bash
ssh -i ~/.ssh/bing-key.pem root@47.99.166.175
```

**第一个坑**：路径写错了，`C:Users` 没有斜杠，应该用 `/c/Users/...` 这种 Unix 路径格式。

**第二个坑**：密钥权限太开放，需要在 Git Bash 里执行：

```bash
chmod 400 ~/.ssh/bing-key.pem
```

连上之后看到了熟悉的 Linux 欢迎界面，那一刻还是很开心的 😄

## 安全加固

1. 创建普通用户 `bing`，日常工作不用 root
2. 禁止 root SSH 登录：

```bash
sed -i 's/^#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

**关于 `^#` 的含义**：`^#` 是正则表达式，表示行首的注释符 `#`。整句意思：找到被注释掉的 `PermitRootLogin yes`，改为 `PermitRootLogin no`。

## Docker 安装踩坑

### 第一个坑：官方源被墙

添加 Docker 官方源时报错：
```
Curl error (56): Failure when receiving data from the peer
OpenSSL SSL_read: SSL_ERROR_SYSCALL, errno 104
```

这是国内访问 Docker 官方源被墙的经典问题。

**解决**：换阿里云 Docker 镜像源，秒装成功。

### 第二个坑：Docker Hub 拉镜像超时

```
docker run hello-world
Error response from daemon: 
  Get "https://registry-1.docker.io/v2/": 
  net/http: request canceled while waiting for connection
```

**解决**：配置 Docker 镜像加速器（阿里云容器镜像服务提供免费的加速地址）：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://<你的阿里云加速地址>.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

配好后 `docker run hello-world` 终于跑起来了 🎉

## 下一步计划

- [ ] Nginx 安装配置
- [ ] Docker Compose 
- [ ] 把博客部署到这台服务器上
- [ ] 监控告警
- [ ] CI/CD

---

> 把踩过的坑记录下来，下次再遇到就不慌了。这就是运维的日常。