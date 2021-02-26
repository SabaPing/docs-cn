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

> **注意：**
> 
> 本文只针对 TiUP 部署的 TiDB 集群。使用 TiDB Operator 部署在 k8s 的 TiDB 集群也能 集成到 Datadog，详见后续文档。

> **事先准备：**
>
> 进行本文操作之前，必须先[使用 TiUP 部署 TiDB 集群](/production-deployment-using-tiup.md)

> **示例集群：**
> 
> 下文使用名为 `tidb-test` 的集群作为示例集群，它是一个3个节点的 TiFlash 集群。请用户根据实际情况确定集群。

## 第一步：使用 TiUP 检查集群状态

检查集群状态，明确集群拓扑结构。在 shell 中输入以下命令：

{{< copyable "shell-regular" >}}

```shell
tiup cluster display tidb-test
```

会显示一下信息：

```shell
Starting component `cluster`: /root/.tiup/components/cluster/v1.3.2/tiup-cluster display tidb-test
Cluster type:       tidb
Cluster name:       tidb-test
Cluster version:    v4.0.10
SSH type:           builtin
Dashboard URL:      http://172.16.5.211:2379/dashboard
ID                  Role          Host          Ports                            OS/Arch       Status   Data Dir                      Deploy Dir
--                  ----          ----          -----                            -------       ------   --------                      ----------
172.16.4.192:9093   alertmanager  172.16.4.192  9093/9094                        linux/x86_64  Up       /tidb-data/alertmanager-9093  /tidb-deploy/alertmanager-9093
172.16.4.192:3000   grafana       172.16.4.192  3000                             linux/x86_64  Up       -                             /tidb-deploy/grafana-3000
172.16.4.192:2379   pd            172.16.4.192  2379/2380                        linux/x86_64  Up       /tidb-data/pd-2379            /tidb-deploy/pd-2379
172.16.5.211:2379   pd            172.16.5.211  2379/2380                        linux/x86_64  Up|L|UI  /tidb-data/pd-2379            /tidb-deploy/pd-2379
172.16.5.220:2379   pd            172.16.5.220  2379/2380                        linux/x86_64  Up       /tidb-data/pd-2379            /tidb-deploy/pd-2379
172.16.5.220:9090   prometheus    172.16.5.220  9090                             linux/x86_64  Up       /tidb-data/prometheus-9090    /tidb-deploy/prometheus-9090
172.16.4.192:4000   tidb          172.16.4.192  4000/10080                       linux/x86_64  Up       -                             /tidb-deploy/tidb-4000
172.16.5.211:4000   tidb          172.16.5.211  4000/10080                       linux/x86_64  Up       -                             /tidb-deploy/tidb-4000
172.16.5.220:4000   tidb          172.16.5.220  4000/10080                       linux/x86_64  Up       -                             /tidb-deploy/tidb-4000
172.16.5.211:9000   tiflash       172.16.5.211  9000/8123/3930/20170/20292/8234  linux/x86_64  Up       /tidb-data/tiflash-9000       /tidb-deploy/tiflash-9000
172.16.4.192:20160  tikv          172.16.4.192  20160/20180                      linux/x86_64  Up       /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
172.16.5.211:20160  tikv          172.16.5.211  20160/20180                      linux/x86_64  Up       /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
172.16.5.220:20160  tikv          172.16.5.220  20160/20180                      linux/x86_64  Up       /tidb-data/tikv-20160         /tidb-deploy/tikv-20160
Total nodes: 13
```

这里需要重点关注 `Host` 一列和 `Data Dir` 一列。`Host` 一列显示了集群中有哪些节点，在后续步骤中每个节点都要做 Datadog 相关的配置；`Data Dir` 一列显示了日志路径，使用 TiUP 部署的 TiDB 集群的各个组件都会把日志保存在 `Data Dir` 路径下，后续步骤中需要配置 Datadog agent 采集这些日志。

> **注意：**
>
> 下面的 agent 配置，必须对每个 `Host` 都做。

## 第二步：确保 Datadog Agent 在每个节点上正常运行

> **注意：**
>
> 请先确保已经安装 Datadog agent，详见 [Getting Started with the Agent](https://docs.datadoghq.com/getting_started/agent/) 。


在 shell 中输入以下命令：

{{< copyable "shell-regular" >}}

```shell
sudo datadog-agent status
```

检查返回结果是否正常。

## 第三步：配置 Agent 日志采集

根据 [Datadog 的日志采集文档](https://docs.datadoghq.com/agent/logs/?tab=tailfiles#custom-log-collection) ，做以下配置。

- 创建 TiDB 配置目录，并通过 vim 编写 yaml 配置文件：

  {{< copyable "shell-regular" >}}
  
  ```shell
  mkdir /etc/datadog-agent/conf.d/tidb.d
  vim /etc/datadog-agent/conf.d/tidb.d/log_collection.yaml
  ```

- 写入以下配置：

  > **注意：**
  >
  > 根据第一步获得的 `Data Dir` 路径，修改配置中的路径。

  {{< copyable "" >}}

  ```yaml
  logs:
   # pd log
   - type: file
     path: "/tidb-deploy/pd-2379/log/pd.log"
     service: "tidb-cluster"
     source: "pd"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
   - type: file
     path: "/tidb-deploy/pd-2379/log/pd_stderr.log"
     service: "tidb-cluster"
     source: "pd"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
  
   # tikv log
   - type: file
     path: "/tidb-deploy/tikv-20160/log/tikv.log"
     service: "tidb-cluster"
     source: "tikv"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
   - type: file
     path: "/tidb-deploy/tikv-20160/log/tikv_stderr.log"
     service: "tidb-cluster"
     source: "tikv"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
  
   # tidb log
   - type: file
     path: "/tidb-deploy/tidb-4000/log/tidb.log"
     service: "tidb-cluster"
     source: "tidb"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
   - type: file
     path: "/tidb-deploy/tidb-4000/log/tidb_slow_query.log"
     service: "tidb-cluster"
     source: "tidb"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: '#\sTime:'
     tags:
       - "custom_format:tidb_slow_query"
   - type: file
     path: "/tidb-deploy/tidb-4000/log/tidb_stderr.log"
     service: "tidb-cluster"
     source: "tidb"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
  
   # tiflash log
   - type: file
     path: "/tidb-deploy/tiflash-9000/log/tiflash.log"
     service: "tidb-cluster"
     source: "tiflash"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
   - type: file
     path: "/tidb-deploy/tiflash-9000/log/tiflash_cluster_manager.log"
     service: "tidb-cluster"
     source: "tiflash"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2}\,\d{3}
     tags:
       - "custom_format:tiflash_cluster_manager"
   - type: file
     path: "/tidb-deploy/tiflash-9000/log/tiflash_error.log"
     service: "tidb-cluster"
     source: "tiflash"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
   - type: file
     path: "/tidb-deploy/tiflash-9000/log/tiflash_stderr.log"
     service: "tidb-cluster"
     source: "tiflash"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
   - type: file
     path: "/tidb-deploy/tiflash-9000/log/tiflash_tikv.log"
     service: "tidb-cluster"
     source: "tiflash"
     log_processing_rules:
       - type: multi_line
         name: new_log_start_with_datetime
         pattern: \[\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2}\.\d{3}\s\+\d{2}:\d{2}\]
  ```
  
  配置中对多行日志配置了聚合，可以参考官方文档 [Multi-line aggregation](https://docs.datadoghq.com/agent/logs/advanced_log_collection/?tab=configurationfile#multi-line-aggregation) 。

- 最后开启 agent 日志采集：

  Agent 默认不采集日志，通过修改配置开启。

  {{< copyable "shell-regular" >}}

  ```shell
  vim /etc/datadog-agent/datadog.yaml
  ```
  
  将 `logs_enabled` 一行改为 `true`:
  
  {{< copyable "" >}}

  ```yaml
  logs_enabled: true
  ```   

## 第四步：配置 Agent 指标抓取

为了理解如何让 Datadog agent 采集 TiDB 集群的指标，必须先了解 TiDB 暴露、存储指标的背景知识。

上文提到，TiUP 维护的 TiDB 集群默认集成 Prometheus 和 Grafana。TiDB 的每个关键组件，都集成了轻量的 http 服务， http 服务中又必然有一个指标接口。对指标接口发起 `GET` 请求，能获 OpenMetrics 标准格式的指标数据。Prometheus 作为一个指标收集、存储组件，每隔一段时间主动的访问 TiDB 的指标接口，抓取并保存数据。

如果要把指标数据集成到 Datadog，Datadog agent 扮演了类似 Prometheus 的角色。它也会周期性的抓取 TiDB 的指标接口，并把数据发往 Datadog 中央服务。

使用 Datadog agent 的 OpenMetrics 模块，配置指标采集。

- 进入 OpenMetrics 配置目录，创建并编辑 yaml 配置文件：

  {{< copyable "shell-regular" >}}

  ```shell
  vim /etc/datadog-agent/conf.d/openmetrics.d/tidb.yaml
  ```

- 写入以下配置：

  > **注意：**
  >
  > 根据第一步获得的 `Host` ，修改配置中的端口。

  {{< copyable "" >}}

  ```yaml
  init_config:
  instances:
    - prometheus_url: http://localhost:2379/metrics
      namespace: tidb-cluster
      metrics: ['*']
      max_returned_metrics: 100000
      send_distribution_buckets: true
      send_distribution_counts_as_monotonic: true
    - prometheus_url: http://localhost:10080/metrics
      namespace: tidb-cluster
      metrics: ['*']
      max_returned_metrics: 100000
      send_distribution_buckets: true
      send_distribution_counts_as_monotonic: true
    - prometheus_url: http://localhost:20180/metrics
      namespace: tidb-cluster
      metrics: ['*']
      max_returned_metrics: 100000
      send_distribution_buckets: true
      send_distribution_counts_as_monotonic: true
    - prometheus_url: http://localhost:20292/metrics
      namespace: tidb-cluster
      metrics: ['*']
      max_returned_metrics: 100000
      send_distribution_buckets: true
      send_distribution_counts_as_monotonic: true
    - prometheus_url: http://localhost:8234/metrics
      namespace: tidb-cluster
      metrics: ['*']
      max_returned_metrics: 100000
      send_distribution_buckets: true
      send_distribution_counts_as_monotonic: true
  ```

  配置中对 histogram 类型的指标开启了特殊配置，可以参考官方文档 [Mapping Prometheus Metrics to Datadog Metrics # Histogram](https://docs.datadoghq.com/integrations/guide/prometheus-metrics/#histogram) 。

## 第五步：重启 Agent

在 shell 中输入以下命令：

{{< copyable "shell-regular" >}}

```shell
sudo systemctl restart datadog-agent
```

然后检查 agent 是否重启成功：

{{< copyable "shell-regular" >}}

```shell
sudo datadog-agent status
```

## 第六步：通过 Datadog Web App 配置日志解析

上面五个步骤，已经将数据上报到了 Datadog 中央服务。这时候已经能在 Datadog app 网页上查到数据。为了获得更好的体验，还要做两个修改。

为了让 Datadog 识别日志中的时间和 level ，需要使用 Datadog 的日志 [processing pipeline](https://docs.datadoghq.com/logs/processing/processors/?tab=ui) 功能，配置 [grok parser](https://docs.datadoghq.com/logs/processing/parsing/?tab=matcher) 和 [remapper](https://docs.datadoghq.com/logs/processing/processors/?tab=ui#remapper) 。

进入 Datadog app 首页，左边选择 Logs > Configuration，创建以下 pipeline：

![datadog-log-pipeline](/media/tiup/tiup-datadog-pipeline.png)

Pipeline 中的 grok parser需要解析三种不同格式的log：

![datadog-log-grok](/media/tiup/tiup-datadog-grok.png)

Parsing rules 详细配置如下：

{{< copyable "" >}}

```
extractDateLevel1 \[%{date("yyyy/MM/dd HH:mm:ss.SSS ZZ"):date}]\s+\[%{word:level}\]\s+%{data}
extractDateLevel2 %{date("yyyy-MM-dd HH:mm:ss,SSS"):date}\s+<%{word:level}>\s+%{data}
extractDate \#\sTime:\s%{date("yyyy-MM-dd'T'HH:mm:ss.SSSSSSSSZZ"):date}\s+%{data}
```

创建一个 date remapper：

![datedog-log-remapper-date](/media/tiup/tiup-datadog-remapper-date.png)

创建一个 level remapper：

![datedog-log-remapper-date](/media/tiup/tiup-datadog-remapper-level.png)

## 第七步：通过 Datadog Web App 配置 Histogram 计算

对于 histogram 类型的 metric，Datadog 需要显式开启 percentile 的计算。

进入 Datadog app 首页，左边选择 Metrics > Summary，点击页面上方的 Calculate Percentiles 按钮，对所有 histogram 类型的指标都开启 percentile 计算：

![datedog-histogram](/media/tiup/tiup-datadog-histogram.png)

## Datadog App 简单使用

完成上面七个步骤后，已经能在 Datadog app 中使用 TiDB 监控数据了。下面展示部分使用截图。

日志查询：

![datedog-app-logs](/media/tiup/tiup-datadog-app-logs.png)

单个日志详情：

![datedog-app-logs-single](/media/tiup/tiup-datadog-app-logs-single.png)

指标（ETCD写磁盘耗时P95）查询：

![datedog-app-metrics](/media/tiup/tiup-datadog-app-metrics.png)

用户还可以根据 Datadog 文档做更定制化的配置，例如配置报警 (Monitors)，看板 (Dashboards) 等等。

## 未来计划

上面的集成过程，有大量人工配置，在集群节点较多情况下容易出错。未来可能会为 Datadog 开发集成功能，让用户做到一键接入。