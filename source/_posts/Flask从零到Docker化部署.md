---
title: Flask从零到Docker化部署
date: 2026-07-09 02:38:32
categories: [运维]
tags: [Python, Flask, Docker, Nginx, 反向代理]
---

## 为什么学 Flask

之前博客是纯静态 HTML，没有"活"的后台。学 Flask 是为了写一个能返回实时数据的 Web 应用——比如查服务器状态、磁盘、内存这些实时变化的数据。

## 第一个 Flask 应用

```python
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/api/status')
def status():
    disk = os.popen("df -h / | awk 'NR==2{print $3\"/\"$2\" (\"$5\")\"}'").read().strip()
    mem  = os.popen("free -h | awk 'NR==2{print $3\"/\"$2}'").read().strip()
    load = os.popen("cat /proc/loadavg").read().strip().split()[:3]
    return jsonify({'disk': disk, 'memory': mem, 'load': ' '.join(load)})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

`os.popen` 可以调用 Linux 命令拿到输出——这和 Shell 脚本里 `$(命令)` 一样，但在 Python 里更方便处理数据。

## Docker 化

```dockerfile
FROM python:3.9-alpine
WORKDIR /app
COPY . .
RUN pip install -i https://mirrors.aliyun.com/pypi/simple/ flask
EXPOSE 5000
CMD ["python", "app.py"]
```

### 踩坑：pip 安装被墙

用 `pip install flask` 从官方 PyPI 拉，结果被墙超时。加 `-i https://mirrors.aliyun.com/pypi/simple/` 切到阿里云镜像，秒装成功。

## Nginx 反向代理

Flask 在容器内部跑在 `:5000` 端口，外面访问要走 Nginx：

```nginx
location /api/ {
    proxy_pass http://flask-api:5000;
}
```

### 踩坑：proxy_pass 末尾的斜杠

```nginx
# ❌ 带斜杠——/api/status 变成 /status，Flask 返回 404
proxy_pass http://flask-api:5000/;

# ✅ 不带斜杠——/api/status 原样传给 Flask
proxy_pass http://flask-api:5000;
```

### 踩坑：容器名 vs 服务名

nginx 配置里 `proxy_pass http://flask-api:5000` 用的是**容器名**。因为两个容器在同一个 docker compose 网络里，容器名就是 DNS 域名。

## API 端点

| 端点 | 功能 |
|------|------|
| `GET /api/status` | 磁盘、内存、负载 |
| `GET /api/backup` | 最近备份信息 |
| `GET /api/uptime` | 服务器运行时间 |
| `GET /api/time` | 服务器当前时间 |

返回都是 JSON 格式，可以直接用在前端或者监控脚本里。

## 最终架构

```
浏览器 → https://47.99.166.175
              │
         Nginx (:443)
         ├── /       → 博客静态文件
         └── /api/   → Flask (:5000)
                        ├── /api/status
                        ├── /api/backup
                        ├── /api/uptime
                        └── /api/time
```

静态博客 + 动态 API 挂在一起，用 docker compose 一个命令全管。

## 完整 docker-compose

```yaml
services:
  nginx:
    image: nginx:alpine
    ports: ["80:80", "443:443"]
    volumes: [博客文件, 证书, nginx配置]

  flask-api:
    image: flask-api:v4
    # 不需要 ports——Nginx 内部转发够了

  glances:
    image: nicolargo/glances:latest
    ports: ["8080:61208"]
```

Flask 容器不用映射端口——Nginx 通过内部网络 `flask-api:5000` 就能访问它。外面只开了 80/443。

---

> 从静态博客到动态 API，从手动部署到 Docker Compose 一键编排。运维和开发之间的那条线，越来越模糊了。

