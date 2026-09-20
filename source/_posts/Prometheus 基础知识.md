---
abbrlink: ''
categories:
- - Prometheus
excerpt: 'Prometheus 是一套监控系统兼时间序列数据库，尤其擅长监控动态多变的云环境。它采用维度数据模型，配有强大的查询语言，并将埋点（instrumentation）、指标采集、服务发现、告警等环节整合进同一个生态。'
date: '2026-09-19T19:20:14.894205+08:00'
tags:
- 云原生
- DevOps
- Prometheus
- 运维
title: Prometheus 基础知识
updated: '2026-09-19T19:20:15.756+08:00'
---
# Prometheus 基础知识

## 基本认知

[Prometheus](https://prometheus.io/) 是一套监控系统兼时间序列数据库，尤其擅长监控动态多变的云环境。它采用维度数据模型，配有强大的查询语言，并将埋点（instrumentation）、指标采集、服务发现、告警等环节整合进同一个生态。

Prometheus 提供各种客户端库和服务端组件，串起一条完整的监控流水线：

- **跟踪并暴露指标**（埋点），
- **采集指标**，
- **存储指标**，
- **查询指标**，用于告警、仪表盘等更多场景。

Prometheus 专注于基于**数值指标**（也就是时间序列）的监控，架构简洁，告警规则显式明确。

它明确*不*试图解决以下问题：

- **日志**或单条**事件**的存储与处理（对已发生的单个事件留存带时间戳的详细记录），
- **追踪(trace)** 的存储与处理（跟踪单个用户请求穿越一组系统的完整生命周期），
- 基于**机器学习**或 **AI** 的异常检测，
- **水平可扩展**的集群化存储。

这些能力确实有价值，但 Prometheus 把它们留给其他系统去解决，你可以让这些系统与 Prometheus 并肩运行。

![配图](https://img.fz688.dpdns.org/2026-09-19-1789796729947.svg)

## 系统架构

下图概览了 Prometheus 的整体系统架构。与 Prometheus 集成的方式其实还有很多（比如通过 [OTLP](https://opentelemetry.io/docs/specs/otel/protocol/) 把指标推送进来），但下面这种是最常见的原生部署形态：

![配图](https://img.fz688.dpdns.org/2026-09-19-1789797790915.svg)

一个组织通常会运行一台或多台 **Prometheus 服务器**，它们是整个 Prometheus 监控体系的核心。你可以为 Prometheus 服务器配置**服务发现**机制（如 DNS、Consul、Kubernetes 等），用它来发现一组指标来源（即所谓的**目标，target**）；当然，需要时也完全可以静态配置目标。随后，Prometheus 会通过 HTTP 定期从这些目标**拉取**（或称"抓取"，scrape）[文本格式](https://prometheus.io/docs/instrumenting/exposition_formats/)的指标，并把采集到的数据写入本地文件系统上的时间序列数据库（**TSDB**）。

被监控的目标可以是下面两种之一：

- 一个**完成埋点的应用**，直接跟踪并暴露与自身相关的 Prometheus 指标；
- 一个 **exporter**，即一类中间软件，负责把现有系统（如数据库服务器、Linux 主机或网络设备）的指标转换或生成为 Prometheus 指标暴露格式。

Prometheus 服务器随后让采集到的数据可被查询：既可以通过内置 Web UI，也可以借助 [Grafana](https://grafana.com/) 或 [Perses](https://perses.dev/) 这类仪表盘工具，还可以直接调用它的 [HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/)。

**注意：** 每次抓取只会把目标上每条时间序列的**当前值**传给 Prometheus，所以抓取间隔决定了最终落库数据的采样频率。目标进程自身并不保留任何历史指标数据。

你还可以配置 Prometheus 服务器基于采集到的数据**生成告警**。不过 Prometheus 并不会直接把告警通知发给人，而是把原始告警转交给独立运行的 **Prometheus Alertmanager**。一个组织里往往由一台 Alertmanager 接收来自多台（甚至全部）Prometheus 服务器的告警，由它集中完成告警的分组、节流和路由。最后，Alertmanager 通过邮件、Slack、PagerDuty 或其他通知服务把通知发出去。

## 核心特性

Prometheus 集多项关键特性于一身，让你能高效地监控自己的系统：

- **带维度（基于标签）的数据模型**，可以按关键特征切分所跟踪的指标；
- 专为处理时间序列优化的**强大查询语言**——PromQL；
- **时间序列处理**与**告警**融为一体；
- 与**服务发现机制**集成，从容应对动态环境；
- 运维**简单**；
- 以 Go 实现的**高效**系统。

后续章节会逐项展开讨论这些特性。

## 时间序列数据模型

Prometheus 存储的是**时间序列**（time series），即沿着时间戳不断采样的数值流：

![配图](https://img.fz688.dpdns.org/2026-09-19-1789798900147.svg)

从宏观上看，每条时间序列由一个**标识符**和一组**采样值**组成：

![配图](https://img.fz688.dpdns.org/2026-09-19-1789798911909.svg)

自 Prometheus 3.0 起，数据模型允许在指标名称和标签名称中使用任意 UTF-8 字符。不过，使用这套扩展字符集有一些注意事项，下文会详细说明

### 序列的标识

Prometheus 用**指标名称**（metric name）加一组称为**标签**（label）的键值对来唯一标识每条序列。例如，上图中的一个序列标识符是：

`http_requests_total{job="api-server",instance="10.0.0.1:443",method="GET"}`
Prometheus 会在 TSDB 中于首次见到某个序列标识符时自动创建并为其建立索引，因此你无需预先定义任何显式的 schema。

#### 指标名称

指标名称标识我们所测量系统的某个整体层面。例如，指标名称 `http_requests_total` 表示某个服务进程处理的 HTTP 请求总数，而 `process_resident_memory_bytes` 则表示某个进程当前占用的常驻内存大小（以字节计）。

#### 标签

标签允许你把一个指标切分、细分成多个子维度。例如，上例中的 `instance` 标签说明该指标来自哪个具体实例（进程），`job` 标签表示该实例所属的作业（job），也就是一组进程；`method` 标签则按进程内使用的 HTTP 方法进一步细分该指标。

标签有多种来源：

- **执行抓取的 Prometheus 服务器：** 有些标签由 Prometheus 服务器根据其服务发现和抓取配置附加，这类标签称为**"目标标签"（target labels）**，因为它们作用于被抓取的目标整体。
- **被抓取的进程：** 另一些标签则直接来自完成埋点的进程内部。

在上面的例子中，`job` 和 `instance` 很可能是目标标签，而 `method` 标签则来自被抓取的进程。

在更广阔的 Prometheus 生态中，标签还有其他来源，指标最终如何打标签取决于你的具体部署方式。

### 序列的采样值

采样值（sample）构成一条序列数据的主体，会随时间不断追加到已建立索引的序列上：

- **时间戳**始终是毫秒精度的 64 位整数 [Unix 时间戳](https://en.wikipedia.org/wiki/Unix_time)。
- **采样值**可以是以下两种之一：
  - 一个 64 位浮点数；
  - 一整个高分辨率的*原生直方图*（native histogram）。

大多数采样值就是简单的浮点数。*原生直方图*是另一种采样类型，它把一整个高分辨率直方图（观测值在众多桶上的分布）存进单个采样值。该采样类型在 Prometheus 3.8.0 中已成为稳定特性。Prometheus 还支持一种较旧的**经典（classic）**直方图表示方式，即把直方图拆散到多条独立的时间序列上（每个桶一条序列）。我们会在埋点课程和 PromQL 查询语言课程中更详细地介绍这两种表示。采样值（sample）构成一条序列数据的主体，会随时间不断追加到已建立索引的序列上：

- **时间戳**始终是毫秒精度的 64 位整数 [Unix 时间戳](https://en.wikipedia.org/wiki/Unix_time)。
- **采样值**可以是以下两种之一：
  - 一个 64 位浮点数；
  - 一整个高分辨率的*原生直方图*（native histogram）。

大多数采样值就是简单的浮点数。*原生直方图*是另一种采样类型，它把一整个高分辨率直方图（观测值在众多桶上的分布）存进单个采样值。该采样类型在 Prometheus 3.8.0 中已成为稳定特性。Prometheus 还支持一种较旧的**经典（classic）**直方图表示方式，即把直方图拆散到多条独立的时间序列上（每个桶一条序列）。

### Prometheus 3.0 新特性：完整 UTF-8 支持

为了提升与 [OpenTelemetry](https://opentelemetry.io/) 指标源的兼容性，Prometheus 3.0 开始支持在指标名称和标签名称中使用任意 [UTF-8](https://en.wikipedia.org/wiki/UTF-8) 字符，因此从技术上讲，这些标识符不再受限于上图所示的原始字符集。不过，**官方仍然建议指标和标签名称使用原始字符集**，以确保与那些尚不支持在这类标识符中使用 UTF-8 字符的其他系统和工具保持兼容。

使用扩展字符集还会在 PromQL 中带来一些不便：数据选择器需要更多的引号，语法也略微繁琐。

例如，对于只使用原始字符集的指标，选择器可以这样写：

`my_metric{my_label="value"}`
一旦引入了此前不支持的字符（本例中用点号替代下划线），就不得不改成下面的语法：

`{"my.metric", "my.label"="value"}`
可以看到，指标名称现在必须写进标签匹配器列表里，而且指标名称和 `my.label` 标签名都得加引号。这种语法写起来、读起来都更费劲，所以在决定让标识符突破原始字符集之前，请务必把这一点考虑进去。

## 指标传输格式

想要暴露 Prometheus 指标的服务，只需提供一个 HTTP 端点（通常是 `/metrics`），以 Prometheus 基于文本的暴露格式输出指标即可。这类端点的输出是人类可读的，最简单的形式如下：

```bash
## HELP http_requests_total The total number of processed HTTP requests.
# TYPE http_requests_total counter
http_requests_total{status="200"} 8556
http_requests_total{status="404"} 20
http_requests_total{status="500"} 68
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 32
```

上例中，该端点暴露了一个按响应状态码拆分的 HTTP 请求计数器，以及目标进程当前打开的文件描述符数量。

想在浏览器里看看"真实"目标输出的实时指标，可以访问[官方运行的演示服务的 metrics 端点](http://demo.promlabs.com:10000/metrics)。

带注释的 `# HELP` 和 `# TYPE` 行为指标名称提供了可选的说明文字和指标类型元数据。该格式中每个非注释行代表一条采样，由指标名称、可选标签和采样值组成。这种按行组织的格式让各类系统和服务都能轻松暴露自己的指标。

关于暴露格式的[完整细节](https://prometheus.io/docs/instrumenting/exposition_formats/#text-based-format)，请参阅 Prometheus 官方文档。

### TODO

后续会在文章展开：

1、如何在自己的应用代码中跟踪并暴露这类指标，

2、如何为无法直接修改代码的第三方系统暴露指标——Exporter。

## 查询语言 PromQL

为了让采集到的数据发挥作用，Prometheus 实现了自有的查询语言 **PromQL**。借助 PromQL，我们可以对时间序列数据进行灵活高效的计算，看清系统里正在发生什么。与 SQL 类语言不同，PromQL 只用于读取数据，不负责插入、更新或删除数据（这些发生在查询引擎之外）。

PromQL 在 Prometheus 服务器内部执行如图：

![配图](https://img.fz688.dpdns.org/2026-09-19-1789814438055.svg)

假设你有一组 HTTP 请求计数器时间序列，指标名称为 `http_requests_total`，`status` 标签表示响应状态码，`path` 标签表示 HTTP 路径：

```bash
http_requests_total{status="200", path="/path-a"}   4714
http_requests_total{status="200", path="/path-b"}   7739
http_requests_total{status="200", path="/path-c"}   9605
http_requests_total{status="403", path="/path-a"}      7
http_requests_total{status="403", path="/path-b"}      3
http_requests_total{status="403", path="/path-c"}      0
http_requests_total{status="500", path="/path-a"}    887
http_requests_total{status="500", path="/path-b"}      5
http_requests_total{status="500", path="/path-c"}    110
```

下面的查询会选出所有返回 `500` 状态码的 HTTP 请求的累计总数：

```bash
http_requests_total{status="500"}
```

由于累计计数器的绝对值通常没什么用处，下面这条查询可以告诉你每条被选中计数器序列平均每秒的增长速度（基于 5 分钟窗口）：

```bash
rate(http_requests_total{status="500"}[5m])
```

再进一步，可以这样计算各路径 `status="500"` 错误速率占同一 HTTP 路径总请求速率的比例：

```bash
  sum by(path) (rate(http_requests_total{status="500"}[5m]))
/
  sum by(path) (rate(http_requests_total[5m]))
```

以上只是 PromQL 中常见的几种写法，这门语言还有更多特性和能力。

想深入了解 PromQL 的工作原理、学会自己构建查询，可以先从[PromQL 速查表（Promlabs）][PromQL 速查表（Promlabs）]入手

## 内置告警

### 一体化告警

Prometheus 把**时间序列数据的采集**与**处理同一套主动告警体系**整合在了一起。其理念是：把关于系统的各类数据尽可能收进同一个数据模型，然后在它之上构建**一体化**的查询。同一种查询语言既用于即席查询和仪表盘，也用于定义告警规则。这与历史上的割裂格局形成对照：过去，Nagios、Icinga 这类故障检测系统周期性运行检查脚本、几乎不保留历史数据，而独立的时序数据库只是被动地存储指标，两者各管一段。

例如，下面这条告警规则（作为规则配置文件的一部分加载进 Prometheus）会在某条路径上返回 `500` 状态码的 HTTP 请求超过总流量 5% 时触发告警：

```yaml
alert: Many500Errors
## This is the PromQL expression that forms the "heart" of the alerting rule.
expr: |
  (
      sum by(path) (rate(http_requests_total{status="500"}[5m]))
    /
      sum by(path) (rate(http_requests_total[5m]))
  ) * 100 > 5
for: 5m
labels:
  severity: "critical"
annotations:
  summary: "Many 500 errors for path {{$labels.path}} ({{$value}}%)"
```

`expr` 字段中的 PromQL 表达式是告警规则的核心，其余基于 YAML 的配置项则用于控制告警元数据、路由标签等。这样就能基于采集到的数据实现精准、可靠的告警。

### TODO

后续文章会讲解如何为自己的系统和服务搭建告警体系。

## 服务发现集成

现代动态 IT 环境给监控系统带来了新的挑战：

- 云厂商上的**虚拟机**按需扩容、缩容。
- **容器编排器**（如 Kubernetes、Docker Swarm、Mesos）把服务实例动态调度到各台主机上。
- **微服务** 化的趋势让需要运维和监控的单个服务数量不断增长。

于是问题来了：监控系统如何才能读懂这个动态变化的世界？它怎么知道当前应该存在哪些机器或服务实例、它们是什么身份、又该从哪里拉取指标？让运维人员静态维护这些信息已经不现实——既太复杂，变化又太快。

为了解决这个问题，Prometheus 可以与你基础设施中的各类服务发现提供方集成，动态发现并持续更新自己所监控的目标列表：

![配图](https://img.fz688.dpdns.org/2026-09-19-1789815870579.svg)

Prometheus 支持多种内置的服务发现机制，例如：

- 发现云厂商上的**虚拟机**（AWS、Azure、Google 等）；
- 发现**集群编排器**上的服务实例（Kubernetes、Marathon 等）；
- 通过 DNS、Consul、Zookeeper 等**通用查找手段**或自定义发现机制来发现目标。

Prometheus 把服务发现用于三个彼此关联的不同目的：

- 构建**应当存在哪些目标**的视图（从而能记录这一信息，并在目标缺失时告警）；
- 获取**如何通过 HTTP 从目标拉取指标**的技术信息；
- 用关于目标的**标签化元数据**丰富从该目标采集到的序列。

就这样，Prometheus 把服务发现当作事实来源，在尽量降低管理开销的同时，可靠地监控动态环境。

### TODO

后续文章将讲解如何配置和使用服务发现集成

## 简单的运维

### 运维简单

从核心设计上看，Prometheus 概念简单，易于运维。

### 用 Go 编写

Prometheus 用 [Go](https://golang.org/) 编写，官方发布的静态二进制文件部署起来不依赖外部运行时（如 JVM）、解释器（如 Python 或 Ruby）或系统共享库。

### 节点独立

每台 Prometheus 服务器都独立于其他 Prometheus 服务器采集数据、评估告警规则，数据只存储在本地，没有紧耦合的集群或复制机制。

### 简单的高可用方案

虽然服务器节点彼此独立，你仍然可以搭建高可用（HA）告警方案：运行两台配置完全相同的 Prometheus 服务器，让它们计算同样的告警。Alertmanager 会基于标签集对重复的通知去重：

![配图](https://img.fz688.dpdns.org/2026-09-19-1789816158039.svg)

#### TODO

后续文章会讲解如何搭建高可用的 Prometheus 与 Alertmanager 部署

### 简单性的边界

当然，大规模部署或有特殊需求的 Prometheus 环境依然可能变复杂。Prometheus 也提供了一些接口，用于在外部弥补自身的某些局限，比如持久化的长期存储。但它的基本构件始终是简单的。

## 高效实现

Prometheus 需要同时从大量系统和服务中采集细致的维度化数据。为此，以下组件经过了高度优化：

- 抓取并解析传入的指标；
- 时间序列数据库的写入与读取；
- 基于 TSDB 数据评估 PromQL 表达式。

根据经验数据，单台大规模 Prometheus 服务器每秒可摄入多达 100 万个时间序列采样，磁盘上每个采样只占 1-2 字节。它还能同时应对数百万条并发活跃（即同时出现在对所有目标的一轮抓取中）的时间序列。

## 小结

知识讲解就到这里了，同时也方便自己复习八股文用

## 参考资料

https://training.promlabs.com/training/introduction-to-prometheus/training-overview/introduction
