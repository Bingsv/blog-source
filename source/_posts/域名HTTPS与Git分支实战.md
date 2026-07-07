---
title: 域名HTTPS与Git分支实战
date: 2026-07-07 22:00:40
categories: [运维]
tags: [域名, HTTPS, Git, GitHub Pages, systemd]
---

## 终于有域名了

买了 `bi-ng.top`，首年几块钱。不用再给人看 `47.99.166.175` 这种裸 IP 了。

### DNS 解析配置

阿里云域名解析 → 添加两条 CNAME 记录：

| 记录类型 | 主机记录 | 记录值 |
|----------|----------|--------|
| CNAME | @ | Bingsv.github.io |
| CNAME | www | Bingsv.github.io |

`@` 代表域名本身 `bi-ng.top`，`www` 代表 `www.bi-ng.top`。两条 CNAME 把这两个都指向 GitHub Pages。

### GitHub Pages 配置

仓库 Settings → Pages → Custom domain 填 `bi-ng.top` → Save → 勾选 Enforce HTTPS。

等 1-2 分钟，GitHub 自动签发 Let's Encrypt 证书，浏览器访问 `https://bi-ng.top` → 绿锁 🔒。

### 自签名 vs 正式证书

| | 自签名 (ECS) | 正式证书 (GitHub Pages) |
|------|-------------|----------------------|
| 加密 | ✅ | ✅ |
| 浏览器信任 | ❌ 红锁 | ✅ 绿锁 |
| 费用 | 免费 | 免费 |
| 需要域名 | 不需要 | 需要 |

---

## Git 分支实操

之前只有一个 `master` 分支。在真实项目里，改东西不走 master——在分支里改，改好了再合并。

### 完整流程

```bash
git branch draft/fix-title       # 1. 创建分支
git checkout draft/fix-title     # 2. 切换到分支

# 在分支里安心改代码...

git commit -m "修了个 bug"       # 3. 提交
git checkout master              # 4. 切回 master
git merge draft/fix-title        # 5. 合并
git branch -d draft/fix-title    # 6. 删掉用完的分支
```

### 为什么这么干

```
master ────────────────────────────→ 永远稳定
            ↘                 ↗
           draft/fix-title ──→ 在这改，改坏了不影响 master
```

团队里每个人都在自己分支干活，互相不干扰。merge 时有冲突再解决。

### 常用命令

| 命令 | 作用 |
|------|------|
| `git branch 分支名` | 创建分支 |
| `git checkout 分支名` | 切换分支 |
| `git checkout -b 分支名` | 创建 + 切换 |
| `git merge 分支名` | 合并到当前分支 |
| `git branch -d 分支名` | 删除分支 |
| `git branch -a` | 查看所有分支 |

---

## 给监控脚本写 systemd 服务

之前 Python 监控脚本 `monitor.py` 靠 cron 每 30 分钟跑一次。但万一服务器重启了，它不会自动恢复（cron 只管定时触发）。

写一个 systemd 服务文件 `/etc/systemd/system/monitor.service`：

```ini
[Unit]
Description=网站可用性监控
After=network.target docker.service

[Service]
Type=simple
User=bing
ExecStart=/usr/bin/python3 /home/bing/monitor.py
Restart=on-failure
RestartSec=60

[Install]
WantedBy=multi-user.target
```

| 字段 | 含义 |
|------|------|
| `After=network.target` | 等网络就绪再启动 |
| `Restart=on-failure` | 异常退出自动重启 |
| `RestartSec=60` | 挂了等 60 秒再拉起来 |
| `WantedBy=multi-user.target` | 开机自动启动 |

然后：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now monitor
systemctl status monitor
```

---

## 现在你的服务管理体系

```
systemd（开机自启）
├── sshd
├── docker
├── fail2ban
├── cron
└── monitor      ← 今天加的

docker compose（容器化）
├── blog-nginx
└── glances
```

一个是 Linux 原生服务管理，一个是容器编排，各管各的，互不冲突。

---

今天还装了个 MySQL，用 Docker 一行跑起来，建库建表增删改查都通了。这些零散技能加起来，运维日常基本覆盖了。

> 从裸 IP 到 HTTPS 域名，从单分支到分支工作流，从手动脚本到 systemd 服务——看不见的进步，在细节里。

