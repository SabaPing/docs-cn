---
title: 使用 Datadog 监控 TiUP 部署的 TiDB 集群
summary: 使用 Datadog 监控 TiUP 部署的 TiDB 集群。
---

# 使用 Datadog 监控 TiUP 部署的 TiDB 集群

TiDB 自带两个监控功能 -- Grafana 和 TiDB Dashboard。TiUP 在部署 TiDB 集群的同时也会部署 Prometheus 和 Grafana 实例，以此提供一定的指标 (metric) 查询能力；TiDB 的 PD 组件集成了 TiDB Dashboard，上面可以查看集群实时状态、慢 SQL 等重要信息，同时还有简单的日志搜索功能。

Datadog 是一个针对云原生应用的监控服务。它通过 SaaS 数据分析平台，向用户提供对服务器、数据库、工具和应用的监控服务。

但是在某些情况下，用户不希望使用上面两个监控功能，而是想使用 Datadog 监控 TiDB 集群。主要原因有两个：

- 部分用户已经使用了完整 Datadog 生态系统，而自带的 Grafana 和 TiDB Dashboard 是两个独立的功能。如果三个界面同时使用，会导致数据分散，效率降低；
- Datadog 的监控产品十分强大，用户体验优秀。如果把 TiDB 监控整合到 Datadog，能获得更完善的集群监控能力。

本文介绍如何将 TiUP 部署、运维的 TiDB 集群集成到 Datadog。

> **注意一：**
> 
> 本文只针对 TiUP 部署的 TiDB 集群。使用 TiDB Operator 部署在 k8s 的 TiDB 集群也能 集成到 Datadog，详见[另一篇文档](nothing-here!)。

> **事先准备：**
>
> 进行本文操作之前，必须先[使用 TiUP 部署 TiDB 集群](/production-deployment-using-tiup.md)

## 第 1 步：使用 TiUP 检查集群状态


