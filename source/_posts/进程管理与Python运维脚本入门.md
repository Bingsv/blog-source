---
title: 进程管理与Python运维脚本入门
date: 2026-07-04 19:02:29
categories: [运维]
tags: [Linux, 进程管理, systemd, Python, Shell]
---

## 进程管理：服务器上的"任务管理器"

每个运行中的程序都是一个**进程**。Nginx 是进程、Docker 是进程、你的 SSH 连接也是进程。

### ps — 查看进程

```bash
ps aux
```

| 列 | 含义 |
|------|------|
| USER | 谁启动的 |
| PID | 进程 ID（唯一编号） |
| %CPU | CPU 占用 |
| %MEM | 内存占用 |
| STAT | S=睡眠 R=运行 Z=僵尸 |

在你的服务器上跑 `ps aux | grep nginx`，能看到：

```
nginx: master process    ← 主进程，负责管理
nginx: worker process    ← 工作进程，处理请求
nginx: worker process    ← 同上（多核就多个）
```

### top / htop — 实时监控

```bash
top       # 自带，功能基础
htop      # 需要装，彩色 + 鼠标可点 + 按 F9 杀进程
```

`htop` 比 `top` 好太多，装上就不想用 top 了。

### kill — 给进程发信号

```bash
kill PID        # 默认发 SIGTERM（礼貌请退出）
kill -9 PID     # SIGKILL（强制杀掉，不给商量）
```

有趣的是，`kill` 不是"杀掉"的意思，是"发信号"。你每天用的 `docker restart` 本质就是给容器进程发 SIGHUP 信号。

---

## systemd / systemctl — 管理后台服务

Linux 启动的第一个进程就是 systemd（PID=1）。它负责启动和管理所有后台服务。

```bash
systemctl status fail2ban    # 看状态
systemctl is-enabled docker  # 是否开机自启？
```

常用命令：

| 命令 | 作用 |
|------|------|
| `systemctl status` | 看运行状态 |
| `systemctl start/stop/restart` | 启/停/重启 |
| `systemctl enable/disable` | 开/关 开机自启 |

你服务器上的服务层级：

```
systemd（PID 1）
├── sshd       ← SSH 服务
├── fail2ban   ← 防暴力破解
├── docker     ← Docker 引擎（自启）
├── cron       ← 定时任务
└── docker compose 管理
    ├── blog-nginx
    └── glances
```

---

## Nginx 日志分析脚本

运维经常被问："昨天有多少人访问？404 最多的页面是什么？有没有人在扫服务器？"

写个脚本一键出报告：

```bash
#!/bin/bash
LOG=$(docker logs blog-nginx 2>&1)

echo "总访问量: $(echo "$LOG" | grep -cE 'GET|POST')"

# 访问最多的 IP
echo "$LOG" | grep -oP '^\d+\.\d+\.\d+\.\d+' | sort | uniq -c | sort -rn | head -5

# 状态码分布
echo "$LOG" | grep -E '"GET|"POST' | awk '{print $9}' | sort | uniq -c | sort -rn

# 404 最多的页面
echo "$LOG" | grep ' 404 ' | awk '{print $7}' | sort | uniq -c | sort -rn | head -5
```

跑出来的结果很有意思——有人在扫你的服务器：

```
/.env          ← 想偷环境变量
/.git/config   ← 想偷 Git 仓库
```

公网 IP 就是这样，每分钟都有人敲门。fail2ban 挡住了 SSH，Nginx 挡不住 HTTP 扫描，但这种低水平扫描也不用担心——要是你有个正式 HTTPS 证书 + 域名，对方都不知道你 IP。

---

## Python 运维脚本

Shell 脚本适合简单任务，逻辑一复杂就难读。Python 更适合写稍微复杂一点的脚本。

### 健康检查：Shell → Python

Shell 版：
```bash
df -h / | awk 'NR==2{print "磁盘: "$3"/"$2" ("$5")"}'
```

Python 版：
```python
import os
result = os.popen("df -h /").read()
data = result.split("\n")[1].split()
print(f"磁盘: {data[2]}/{data[1]} ({data[4]})")
```

同样的结果，但 Python 的变量名和 `f"..."` 格式化字符串比 Shell 可读太多。

### 网站监控脚本

用 Python 的 `requests` 库，几行代码就能定时检查网站是否活着：

```python
import requests

urls = [
    ("博客", "https://47.99.166.175"),
    ("监控", "http://47.99.166.175:8080"),
    ("GitHub", "https://github.com"),
]

for name, url in urls:
    try:
        r = requests.get(url, timeout=10, verify=False)
        print(f"✅ {name} HTTP {r.status_code}")
    except:
        print(f"❌ {name} 挂了")
```

加到 crontab，每 30 分钟自动跑一次：

```
*/30 * * * * python3 ~/monitor.py >> ~/monitor.log
```

现在你的自动化体系：

```
cron 定时任务
├── 03:00  → backup.sh      → 每日备份
├── 每30分 → monitor.py     → 网站监控
└── 手动   → check.sh/log-report.sh
```

---

## 总结

| 学到什么 | 命令/工具 |
|----------|-----------|
| 进程查看 | `ps` `top` `htop` |
| 进程控制 | `kill` `kill -9` |
| 服务管理 | `systemctl start/stop/enable/status` |
| 日志分析 | `grep` `awk` `sort` `uniq -c` |
| Python 运维 | `os.popen` `requests` `f-string` |

> Shell 做小事快，Python 做复杂事清楚。两个都会，运维日常就够用了。

---

*[脚本源码](https://github.com/Bingsv/ops-portfolio)*