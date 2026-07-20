---
title: Prometheus+Grafana监控体系从零搭建
date: 2026-07-18 11:00:00
categories: [运维]
tags: [Prometheus, Grafana, 监控, Docker, Node Exporter, Nginx]
---

## 为什么要搞监控

之前服务器上跑着 Glances（:8080），能实时看 CPU、内存、磁盘——但关掉页面就没了。真正生产环境需要三样东西：

1. **历史数据**——昨天凌晨 3 点 CPU 有没有突然飙高？看曲线就知道
2. **自动告警**——半夜服务挂了，总不能靠人盯着
3. **可视化面板**——一眼看清整个系统状态，不用挨个敲命令

Prometheus + Grafana 就是这个领域的标准组合。本文记录我在阿里云 ECS 上从零搭建的完整过程。

## 架构设计

```
┌──────────────────────────────────────┐
│            ECS (2核2G)               │
│                                      │
│  docker compose 新增 4 个容器:        │
│                                      │
│  ┌──────────┐  ┌──────────┐         │
│  │Prometheus│  │ Grafana  │         │
│  │  :9090   │  │  :3000   │         │
│  │ 指标采集  │  │ 可视化   │         │
│  └────┬─────┘  └────▲─────┘         │
│       │ 每15秒抓取   │ 查询           │
│       ▼              │               │
│  ┌──────────┐        │               │
│  │Node      │  ┌─────┴──────┐       │
│  │Exporter  │  │Nginx       │       │
│  │:9100     │  │Exporter    │       │
│  │主机指标  │  │:9113       │       │
│  └──────────┘  │Nginx指标   │       │
│                └────────────┘       │
└──────────────────────────────────────┘
```

## 第一步：准备 Nginx stub_status

Prometheus 采集 Nginx 指标需要一个入口。在 Nginx 配置里加上 `/nginx_status` 端点：

```nginx
server {
    listen 80;

    location /nginx_status {
        stub_status on;
        access_log off;
        allow 127.0.0.1;
        allow 172.16.0.0/12;
        deny all;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

访问 `/nginx_status` 会看到这样的数据：

```
Active connections: 2
server accepts handled requests
 152 152 218
Reading: 0 Writing: 1 Waiting: 1
```

## 第二步：docker-compose 编排

四个服务的 compose 文件：

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    extra_hosts:
      - "host.docker.internal:host-gateway"
    command:
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'

  node_exporter:
    image: prom/node-exporter:latest
    pid: "host"
    network_mode: "host"
    volumes:
      - /proc:/host/proc:ro
      - /:/rootfs:ro

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:latest
    ports: ["9113:9113"]
    command:
      - '-nginx.scrape-uri=http://host.docker.internal/nginx_status'
    extra_hosts:
      - "host.docker.internal:host-gateway"

  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    volumes:
      - ./grafana/datasources/datasource.yml:/etc/grafana/provisioning/datasources/datasource.yml:ro
```

## 第三步：Prometheus 配置

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['host.docker.internal:9100']

  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']
```

**关键坑**：node_exporter 用了 `network_mode: host`，所以不在 Docker 网络里。Prometheus 容器内要用 `host.docker.internal` 才能访问它。同时要在 compose 里给 prometheus 服务加 `extra_hosts` 做映射。

## 第四步：告警规则

配了 5 条规则，覆盖最常见的故障场景：

```yaml
groups:
  - name: basic_alerts
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels: { severity: critical }

      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels: { severity: warning }

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
        for: 5m
        labels: { severity: warning }

      - alert: HighDiskUsage
        expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 85
        for: 5m
        labels: { severity: warning }

      - alert: DiskFull
        expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 95
        for: 1m
        labels: { severity: critical }
```

告警在 `http://47.99.166.175:9090/alerts` 可以看到。下一步接 Alertmanager + 企业微信通知。

## 第五步：Grafana 可视化

Grafana 启动后自动连 Prometheus。通过 API 导入了两个官方 Dashboard：

- **Node Exporter Full** (ID 1860)：CPU/内存/磁盘/网络/负载 全貌
- **NGINX Exporter** (ID 12708)：Nginx 连接数、请求速率

## 踩坑记录

| 坑 | 原因 | 解决 |
|-----|------|------|
| node_exporter 抓不到 | host 网络不在 Docker 网桥里 | 用 `host.docker.internal` + extra_hosts |
| nginx_status 返回 301 | 放在 443 server 块，80 全跳 HTTPS | 放在 80 server 块的 return 之前 |
| Grafana 刚启动 000 | 第一次初始化插件和数据库 | 等 10 秒再访问 |

## 效果

最终所有服务正常运行：

- **Prometheus** (:9090)：三个目标全是绿色 UP
- **Grafana** (:3000)：CPU/内存/磁盘/网络的实时曲线一目了然  
- 时序数据持续采集中，保留 15 天

## 总结

从 Glances 到 Prometheus + Grafana，监控体系上了一个台阶。核心就三件事：

1. **Exporter** 暴露指标
2. **Prometheus** 采集 + 存储 + 告警
3. **Grafana** 可视化

整个搭建花了两小时，踩了几个网络相关的坑，但最终所有组件都在 docker compose 里干净地跑着。配置文件和告警规则已全部推到 [ops-portfolio](https://github.com/Bingsv/ops-portfolio) 仓库的 `monitoring/` 目录。
