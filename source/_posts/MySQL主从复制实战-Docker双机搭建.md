---
title: MySQL主从复制实战—Docker双机搭建
date: 2026-07-23 09:00:00
categories: [运维]
tags: [MySQL, Docker, 主从复制, 数据库]
---

## 为什么要搞主从

单机 MySQL 挂了就没了。主从复制是最基础的数据库高可用方案——主库负责写，从库负责读。

## 核心原理

一句话：**主库把写操作记到 binlog，从库拉取 binlog 重放。**

```
主库 INSERT → binlog → Slave IO线程拉取 → relay log → Slave SQL线程重放 → 从库数据一致
```

三个线程：Master IO线程 / Slave IO线程 / Slave SQL线程。

## Docker 搭建

Master 配置：`server-id=1, log-bin=mysql-bin, binlog-format=ROW`
Slave 配置：`server-id=2, relay-log=mysql-relay-bin, read-only=ON`

Master :3306，Slave :3307。

## 配置步骤

1. Master 创建复制用户 `GRANT REPLICATION SLAVE`
2. `SHOW MASTER STATUS` 记 File + Position
3. Slave `CHANGE REPLICATION SOURCE TO` + `START REPLICA`
4. 验证 `Replica_IO_Running: Yes` + `Replica_SQL_Running: Yes`

## 踩坑

| 坑 | 解决 |
|------|------|
| 从库没数据库 | CREATE DATABASE + 手动建表 |
| Position 错位 | RESET REPLICA ALL 重新对齐 |

## 面试要点

- **原理**：binlog → IO线程 → relay log → SQL线程
- **延迟**：并行复制、半同步、拆小事务
- **主库挂了**：从库关只读切流量，生产用 MHA/Orchestrator
