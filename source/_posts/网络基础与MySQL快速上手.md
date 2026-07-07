---
title: 网络基础与MySQL快速上手
date: 2026-07-07 13:14:12
categories: [运维]
tags: [网络, TCP, DNS, MySQL, Docker]
---

## 网络基础

运维每天和网络打交道——SSH 要用 TCP、配域名要用 DNS、排查不通要用端口测试。

### TCP 三次握手

建立连接前，客户端和服务器要确认**双方都能收发**：

```
客户端                          服务器
   │                               │
   │──── SYN (我想连你) ──────────→│   第1次
   │                               │
   │←── SYN+ACK (收到，你呢) ──────│   第2次
   │                               │
   │──── ACK (收到了，开聊) ──────→│   第3次
   │                               │
   ╞══════ 连接建立 ═══════════════╡
```

面试官问"为什么是三次不是两次"——答：两次只能证明客户端能发、服务器能收；服务器不知道客户端能不能收。第三次确认客户端收到了服务器的回应。

### 在自己服务器上看

```bash
ss -t | awk '{print $1}' | sort | uniq -c
```

输出：

```
3 ESTAB     ← 3 个已建立的连接
1 State     ← 表头
```

`ss` 是 `netstat` 的替代品，更快更现代。`ESTAB` = 连接中，`LISTEN` = 在监听，`TIME-WAIT` = 刚关闭还在等。

### DNS 解析

把域名翻译成 IP：

```bash
nslookup github.com
# Name: github.com → Address: 20.205.243.166
```

服务器上 `/etc/resolv.conf` 存了 DNS 服务器地址：

```
nameserver 100.100.2.136   ← 阿里云内网 DNS
```

> 这就是为什么 GitHub 在国内 ping 不通但翻墙能访问——DNS 解析出的 IP 被墙了。

### 端口测试

测试一个端口通不通，用 `nc`（netcat）：

```bash
nc -zv 47.99.166.175 443    # ✅ Connection succeeded
nc -zv 47.99.166.175 3306   # ❌ TIMEOUT
```

| 工具 | 测什么 | 什么情况用 |
|------|--------|-----------|
| `ping` | ICMP 协议 | 测延迟，但通了不代表服务活着 |
| `nc -zv` | TCP 端口 | 测端口通不通，这个通才是真的通 |
| `curl -v` | HTTP/HTTPS | 看完整请求响应过程 |

### 双层防火墙

云服务器通常有两层防火墙：

```
互联网 → 阿里云安全组（第一层）→ 服务器 iptables（第二层）→ 你的应用
```

安全组 3306 没开 → `nc 3306` 超时。开了 443 → 浏览器能访问博客。

---

## MySQL 快速上手

### 用 Docker 一行跑起来

```bash
docker run -d --name mysql-learn \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=Admin@123 \
  -e MYSQL_DATABASE=opsdb \
  mysql:8.0
```

### 基本 CRUD 操作

```sql
-- 建表
CREATE TABLE servers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    ip VARCHAR(20),
    status VARCHAR(20)
);

-- 插入数据
INSERT INTO servers (name, ip, status) VALUES
    ('博客服务器', '47.99.166.175', '运行中'),
    ('测试服务器', '10.0.0.1', '已下线');

-- 查询
SELECT * FROM servers;
SELECT name, ip FROM servers WHERE status = '运行中';

-- 更新
UPDATE servers SET status = '已下线' WHERE name = '测试服务器';

-- 删除
DELETE FROM servers WHERE name = '测试服务器';
```

### 进入 MySQL 容器的两种方式

```bash
# 方式 1：直接连
docker exec -it mysql-learn mysql -uroot -pAdmin@123 opsdb

# 方式 2：进容器再连
docker exec -it mysql-learn bash
mysql -uroot -pAdmin@123 opsdb
```

---

运维不一定需要精通 SQL，但至少要会**装数据库**、**连进去**、**查数据**。面试时被问到"用过数据库吗"，能说"我自己在服务器上用 Docker 跑了一个 MySQL，基本的增删改查都会"。

