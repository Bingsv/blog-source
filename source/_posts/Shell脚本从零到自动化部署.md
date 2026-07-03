---
title: Shell脚本从零到自动化部署
date: 2026-07-01 15:41:11
categories: [运维]
tags: [Shell, 运维, 自动化, Linux, Bash]
---

## 从一行命令到脚本

运维最开始的起点很简单：SSH 登录服务器，敲命令。

但敲久了就发现，有些命令组合你每天都要跑——查磁盘、看内存、确认容器活着没。每次都手打一遍？太蠢了。

**Shell 脚本就是把重复的手动操作，写成一个文件，以后一行跑完。**

---

## 第一版：能跑就行

最开始写的健康检查脚本长这样：

```bash
#!/bin/bash
echo "磁盘使用率"
df -h /
echo "内存使用率"
free -h
echo "容器状态"
docker ps
echo "Nginx 响应"
curl -o /dev/null -s -w "%{http_code}" https://localhost -k
```

能用。但输出乱，看不懂哪里有问题。

---

## 学了三个东西，脚本突然不一样了

### 1. 变量

不用把 IP 地址、阈值写死在代码各处，改一处就行：

```bash
DISK_WARN=80           # 磁盘告警阈值
ECS_HOST="47.99.166.175"
```

### 2. 条件判断

```bash
if [ "$PCT" -gt "$DISK_WARN" ]; then
    echo "磁盘告警！"
else
    echo "磁盘正常"
fi
```

Shell 的 if 语法和大多数语言不一样，记几个最常用的：

| 写法 | 含义 |
|------|------|
| `-gt` | 大于 (greater than) |
| `-lt` | 小于 (less than) |
| `-eq` | 等于 (equal) |
| `-ne` | 不等于 (not equal) |
| `-n "$VAR"` | 变量非空 |

### 3. 函数

把每个检查「打包」成独立函数，想加检查就加一个函数，结构清晰：

```bash
check_disk() {
    # 磁盘检查逻辑
}

check_memory() {
    # 内存检查逻辑
}

# 调用所有检查
check_disk
check_memory
```

---

## 第二版：彩色输出 + 告警阈值

```
磁盘使用率
  状态: ✅ 正常  已用: 6.8G / 总计: 40G (17%)

内存使用率
  状态: ✅ 正常  已用: 557Mi / 总计: 1.6Gi (33%)

Nginx 响应
  Nginx: ✅ 正常 (HTTP 200)
```

一眼看出哪个指标出问题了。绿色的 ✅ 正常，红色的 ⚠ 告警。

颜色实现很简单：

```bash
echo -e "\033[32m ✅ 正常 \033[0m"   # 绿色
echo -e "\033[31m ⚠ 告警 \033[0m"   # 红色
```

`\033[32m` 开启绿色，`\033[0m` 关闭颜色。32 = 绿，31 = 红，33 = 黄。

[完整脚本已上传 GitHub](https://github.com/Bingsv/ops-portfolio/blob/master/check.sh)

---

## 进阶：博客一键部署脚本

有了健康检查还不够。每次更新博客要：

1. 本地 `hexo generate`
2. `scp` 上传到 ECS
3. SSH 上去 `docker compose restart`

也是重复劳动，写个脚本一劳永逸：

```bash
#!/bin/bash
# 博客一键部署：生成 → 上传 → 重启 → 验证

hexo generate                          # 第 1 步
scp -r public/* bing@服务器:~/blog-static/   # 第 2 步
ssh bing@服务器 "docker compose restart nginx" # 第 3 步
curl -s -o /dev/null -w "%{http_code}" https://服务器 -k  # 第 4 步：验证
```

[部署脚本也上传了](https://github.com/Bingsv/ops-portfolio/blob/master/deploy-blog.sh)

---

## 踩过的坑

### 1. docker compose 用服务名，不是容器名

```bash
# ❌ 错误
docker compose restart blog-nginx

# ✅ 正确
docker compose restart nginx
```

`docker-compose.yml` 里 `services:` 下面定义的叫**服务名**，`container_name` 只是容器显示名。compose 命令只看服务名。

### 2. chmod 权限

脚本写完必须加执行权限：

```bash
chmod +x script.sh
```

否则会报 `Permission denied`。这其实是在鸟哥第 5 章里讲的——Linux 文件权限基础。

### 3. `$?` 判断上一条命令是否成功

```bash
scp -r public/* bing@服务器:~/blog-static/
if [ $? -ne 0 ]; then
    echo "上传失败"
    exit 1
fi
```

`$?` 是上一条命令的**退出码**。0 = 成功，非 0 = 失败。

---

## 两条脚本，覆盖一个运维的日常

| 脚本 | 用途 | 频率 |
|------|------|------|
| `check.sh` | 快速确认服务器状态 | 每天 SSH 上去就跑 |
| `backup.sh` | 自动备份配置文件 | cron 凌晨 3 点 |
| `deploy-blog.sh` | 更新博客并上线 | 写了新文章就跑 |

---

## 下一个目标

学了 Shell 基本语法之后，接下来想深入 Docker——自己写 Dockerfile，把博客打包成一个完整的镜像。

> 运维的本质不是写代码，是**把重复的事交给机器做**。Shell 脚本是第一把钥匙。

---

*所有代码：[github.com/Bingsv/ops-portfolio](https://github.com/Bingsv/ops-portfolio)*