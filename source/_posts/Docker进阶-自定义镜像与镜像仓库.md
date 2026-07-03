---
title: Docker进阶-自定义镜像与镜像仓库
date: 2026-07-03 15:33:25
categories: [运维]
tags: [Docker, Dockerfile, 镜像仓库, 阿里云, 容器]
---

## 上篇回顾

之前已经学会了拉镜像、跑容器、用 Compose 管理多个服务。但一直用的是别人的镜像（`nginx:alpine`），博客文件靠挂载目录。

这次学两件事：**自己写镜像 + 把镜像存到仓库**。

---

## 第一部分：写自己的 Dockerfile

### 为什么要自己写镜像

之前跑 Nginx 是这样：

```bash
docker run -d -p 80:80 -v ~/blog-static:/usr/share/nginx/html nginx:alpine
```

每次都依赖外部文件挂载，换个机器就得重新准备文件。如果能**把博客文件打进镜像内部**，镜像自带一切，到任何地方 `docker run` 直接能跑。

### 第一个 Dockerfile

```dockerfile
FROM nginx:alpine

# 把博客文件复制进镜像
COPY blog-static/ /usr/share/nginx/html/

# 把 nginx 配置也复制进去
COPY nginx.conf /etc/nginx/conf.d/default.conf

# 把 SSL 证书也复制进去
COPY certs/ /etc/nginx/certs/

# 声明端口（文档性质）
EXPOSE 80 443
```

每条指令解释：

| 指令 | 含义 |
|------|------|
| `FROM` | 基于哪个基础镜像。所有 Dockerfile 第一行必须是 FROM |
| `COPY` | 把宿主机文件复制到镜像内部 |
| `EXPOSE` | 声明容器监听哪些端口，文档用途，不真正打开端口 |
| `RUN` | 构建时执行命令（如 `apt install`） |
| `CMD` | 容器启动时执行的默认命令 |
| `WORKDIR` | 设置工作目录 |
| `ENV` | 设置环境变量 |

### 构建镜像

```bash
docker build -t my-blog:v1 .
```

- `-t` = tag，给镜像命名（名字:版本）
- `.` = 用当前目录的 Dockerfile

### 踩坑：证书没打进镜像

v1 构建成功，但容器启动报错：

```
cannot load certificate "/etc/nginx/certs/fullchain.pem": No such file
```

排查：Dockerfile 里没有 `COPY certs/`，修复后 v2 正常。

**教训**：镜像构建时不会自动帮你猜缺什么，少一个 COPY 就少一个文件。

### 验证镜像

```bash
docker images | grep my-blog
docker run -d --name test -p 80:80 -p 443:443 my-blog:v2
curl -k https://localhost
```

---

## 第二部分：镜像存到仓库

### 本地镜像的问题

镜像只在 ECS 本地。换一台服务器就得重新构建，其他人也用不了。

用镜像仓库解决：
- 推到仓库 → 换个机器 `docker pull` → 直接跑
- 版本管理：v1、v2、v3 清晰可查
- 配合 CI/CD：GitHub Actions 构建完自动推送

### Docker Hub or 阿里云 ACR？

| | Docker Hub | 阿里云 ACR |
|------|-----------|------------|
| 免费 | ✅ | ✅ 个人版 |
| 国内速度 | ❌ 基本没法用 | ✅ 很快 |
| 适合谁 | 开源项目 | 国内学习/公司项目 |

### 阿里云 ACR 配置

**1. 创建实例**
- https://cr.console.aliyun.com
- 创建个人版（免费）

**2. 创建命名空间和仓库**
- 命名空间 `bingsv`
- 仓库 `my-blog`
- 代码源选「本地仓库」

**3. 设置访问凭证**
- 左侧「访问凭证」→ 设置固定密码

### 推送镜像到仓库

```bash
# 1. 登录
docker login --username=bingsv crpi-xxx.cn-hangzhou.personal.cr.aliyuncs.com

# 2. 给本地镜像打上仓库标签
docker tag my-blog:v2 crpi-xxx.cn-hangzhou.personal.cr.aliyuncs.com/bingsv/my-blog:v1

# 3. 推送
docker push crpi-xxx.cn-hangzhou.personal.cr.aliyuncs.com/bingsv/my-blog:v1
```

### 验证：从仓库拉取

```bash
# 删掉本地镜像
docker rmi my-blog:v2

# 从仓库拉
docker pull crpi-xxx.cn-hangzhou.personal.cr.aliyuncs.com/bingsv/my-blog:v1

# 确认
docker images | grep my-blog
```

---

## 完整工作流

日常更新博客镜像的标准流程：

```bash
# 1. 构建新版本
docker build -t my-blog:v3 .

# 2. 打远程标签
docker tag my-blog:v3 crpi-xxx.aliyuncs.com/bingsv/my-blog:v3

# 3. 推到阿里云
docker push crpi-xxx.aliyuncs.com/bingsv/my-blog:v3

# 4. 更新 compose
sed -i 's|my-blog:v2|my-blog:v3|' docker-compose.yml

# 5. 重启
docker compose up -d
```

---

## 镜像命名规范

```
crpi-xxx.cn-hangzhou.personal.cr.aliyuncs.com / bingsv / my-blog : v1
                     ↑                              ↑         ↑       ↑
                  仓库地址                        命名空间   仓库名   版本
```

---

## 总结：Docker 技能链条

```
写 Dockerfile → docker build → docker tag → docker push → docker compose up
    (造镜像)      (构建)        (打标签)      (存仓库)        (部署)
```

这条线走通了，Docker 就算真正掌握了。

---

*[Dockerfile 源码](https://github.com/Bingsv/ops-portfolio/blob/master/Dockerfile)*
*[阿里云 ACR](https://cr.console.aliyun.com)*