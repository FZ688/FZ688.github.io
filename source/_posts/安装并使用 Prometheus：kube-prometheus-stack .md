---
abbrlink: ''
categories:
- - Prometheus
excerpt: '上一篇Prometheus 基础知识讲过 Prometheus 的架构：Prometheus 服务器负责采集和存储指标，Alertmanager 负责告警，再加上 Grafana 做可视化——一套能用的监控体系至少是这三件套'
date: '2026-07-20T19:20:14.894205+08:00'
tags:
- 云原生
- DevOps
- Prometheus
- 运维
title: Kubernetes 集群监控体系——Prometheus+Grafana+Alertmanager
updated: '2026-07-20T19:20:15.756+08:00'
---
## 架构

上一篇[《Prometheus 基础知识》](https://blog.fz688.dpdns.org/2026/09/19/Prometheus-%E5%9F%BA%E7%A1%80%E7%9F%A5%E8%AF%86/)讲过 Prometheus 的架构：Prometheus 服务器负责采集和存储指标，Alertmanager 负责告警，再加上 Grafana 做可视化——一套能用的监控体系至少是这三件套。
如果我们是在一台 Linux 虚拟机上学习，可以下载官方二进制包，5 分钟跑起一个 Prometheus。但我们的目标是 **Kubernetes 集群监控**：集群里几十个组件（kubelet、API Server、CoreDNS、容器、节点……）都要采集，还要预置几十张仪表盘和上百条告警规则。这时候手动一个个装就不现实了。
社区最常见的做法是 **kube-prometheus-stack**——一个 Helm Chart，一条命令把整套装齐：

| 组件                  | 作用                                            | 本次安装的版本   |
| --------------------- | ----------------------------------------------- | ---------------- |
| Prometheus Operator   | 用 Kubernetes 的方式管理 Prometheus（后面详讲） | v0.94.0          |
| Prometheus            | 监控核心：采集 + 存储 + PromQL                  | v3.14.0          |
| Alertmanager          | 告警的分组、抑制、静默、路由                    | v0.34.0          |
| Grafana               | 可视化仪表盘                                    | v13.2.2          |
| node-exporter         | 采集节点（Linux 主机）指标                      | v1.12.1          |
| kube-state-metrics    | 把 K8s 对象状态转成指标                         | v2.20.0          |
| 预置告警规则 + 仪表盘 | 数百条规则、几十张仪表盘开箱即用                | kubernetes-mixin |

用 mermaid 画一下装完之后的整体架构：

![image-20260920143706604](https://img.fz688.dpdns.org/2026-09-20-1789886226824.png)

> 图里实线是"数据流向"（指标被 Prometheus 抓走、告警发给 Alertmanager、Grafana 查 Prometheus 画图），虚线是"控制流向"（Operator 感知配置变化并自动生效）。

![image-20260920152524098](https://img.fz688.dpdns.org/2026-09-20-1789889124778.png)

## 安装

### 添加 Chart 仓库

```bash
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```

> `prometheus-community` 是我们给这个仓库起的本地别名。kube-prometheus-stack 由 Prometheus 社区（prometheus-operator 项目）维护，托管在 GitHub Pages 上。

搜索确认 chart 存在、看有哪些版本：

```bash
$ helm search repo kube-prometheus-stack --versions | head -5
NAME                                        CHART VERSION   APP VERSION     DESCRIPTION
prometheus-community/kube-prometheus-stack  91.4.1          v0.94.0         kube-prometheus-stack collects Kubernetes manif...
prometheus-community/kube-prometheus-stack  91.3.0          v0.93.1         ...
```

> CHART VERSION 是 Helm 包的版本，APP VERSION 是里面 Prometheus Operator 应用的版本——**两者独立演进**，看版本号时别混。本次使用 `91.4.1`（安装时的最新版）。

### 写 values.yaml

Helm Chart 默认参数对学习环境并不友好：默认用 ClusterIP 集群外访问不了、Grafana 密码随机等。所以我们写一份自定义 values（文件保存为 `kps-values.yaml`）：

```yaml
# kps-values.yaml 

# Grafana 
grafana:
  adminPassword: Grafana12345        # 管理员密码。不写则 Helm 会生成随机密码存进 Secret
  service:
    type: NodePort                   # 默认 ClusterIP 只有集群内能访问；NodePort 在宿主机开端口
    nodePort: 30330                  # 指定 NodePort 端口号（默认随机分配 30000-32767）
  persistence:
    enabled: true                    # 持久化仪表盘/数据源配置，Pod 重建不丢
    size: 2Gi
  resources:
    requests: { cpu: 100m, memory: 256Mi }
    limits:   { cpu: 500m, memory: 512Mi }

# Prometheus 本体
prometheus:
  prometheusSpec:
    retention: 3d                    # 本地数据保留期，默认 10d。测试环境为了省磁盘，设置3天
    scrapeInterval: 30s              # 全局抓取间隔。生产常见 15s-60s
    resources:
      requests: { cpu: 250m, memory: 512Mi }
      limits:   { cpu: "1", memory: 2Gi }
    # 以下三个 false 非常重要！后面解释
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false

# Alertmanager
alertmanager:
  service:
    type: NodePort
    nodePort: 30993
  alertmanagerSpec:
    retention: 120h                  # 告警痕迹（静默记录等）保留 5 天
    resources:
      requests: { cpu: 50m, memory: 128Mi }
      limits:   { cpu: 200m, memory: 256Mi }

#  预置告警规则裁剪 
defaultRules:
  create: true                       # 保留官方预置规则（kubernetes-mixin）
  rules:
    etcd: false                      # 下面 5 个 false 的原因见下文
    kubeControllerManager: false
    kubeProxy: false
    kubeSchedulerAlerting: false
    kubeSchedulerRecording: false
    windows: false                   # 集群里没有 Windows 节点

# 不需要的组件直接关闭
kubeEtcd: { enabled: false }
kubeControllerManager: { enabled: false }
kubeProxy: { enabled: false }
kubeScheduler: { enabled: false }
```

#### 为什么要关掉 etcd / scheduler / controller-manager / proxy？

这四个是 **K8s 控制平面组件**，它们的指标暴露在 `127.0.0.1`（或 HTTPS + 特殊证书）上：

- 云上托管的 K8s（ACK/EKS/GKE）**根本看不到**控制平面，想监控都不行；
- 本机 Docker Desktop 里，Prometheus Pod 想抓 `kube-scheduler` 的指标需要访问节点本机回环地址 + 客户端证书，默认配置抓不到，只会产生一排永远 DOWN 的 target 和一堆 Firing 的告警噪音。

所以官方 chart 把这些开关做成显式的：**没有的组件直接不装采集、不建 ServiceMonitor、不加载相关告警规则**。三处开关是配套的（`kubeScheduler.*` 关采集 + `defaultRules.rules.kubeScheduler*` 关规则），只关一处会留下"规则在但没数据"的永久告警。

#### `SelectorNilUsesHelmValues: false` 为什么要这么设置？

Prometheus Operator 通过**标签选择器**决定"哪些 ServiceMonitor 归我管"。这个 chart 出于多租户安全考虑，默认把选择器设成 `release=kps`（你的 Release 名）——**只有带着这个标签的 ServiceMonitor 才会被采纳**。

副作用是：以后你自己写了一个 ServiceMonitor（比如监控自己的应用），忘了打 `release: kps` 标签，Prometheus 就是不抓它，而且 **targets 页面毫无痕迹**（它根本不认为那是它的目标）。这是"为什么我的 ServiceMonitor 不生效"的第一大原因。

三个 `false` 的意思就是：**不按 Release 名过滤，集群里所有 ServiceMonitor/PodMonitor/规则都归这个 Prometheus 管**。学习环境单实例，为了省心，防止出现那些莫名其妙的"不生效"情况。

### 执行安装

```bash
# 创建独立命名空间，然后安装（release 名取 kps）
$ kubectl create namespace monitoring
namespace/monitoring created

$ helm install kps prometheus-community/kube-prometheus-stack \
    -n monitoring \
    -f kps-values.yaml
NAME: kps
LAST DEPLOYED: Sat Sep 19 20:15:26 2026
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=kps"

```

> `STATUS: deployed` 只表示 **Helm 把资源都提交给 K8s 了**，不代表 Pod 已经跑起来——镜像还在拉、CRD 还在注册。所以下一步一定是观察。

### 观察部署过程

```bash
$ kubectl get pods -n monitoring -w     # -w 持续 watch，Ctrl+C 退出
```

大概2-3分钟全部就绪

![image-20260920145542858](https://img.fz688.dpdns.org/2026-09-20-1789887343028.png)

### 确认服务端口

```bash
$ kubectl get svc -n monitoring
```

![image-20260920145846577](https://img.fz688.dpdns.org/2026-09-20-1789887526915.png)

至此安装完成。三个入口：

| 服务         | 本机地址               | 登录                 |
| ------------ | ---------------------- | -------------------- |
| Prometheus   | http://localhost:30990 | 无需登录             |
| Grafana      | http://localhost:30330 | admin / Grafana12345 |
| Alertmanager | http://localhost:30993 | 无需登录             |

## WebUI使用

###  Prometheus：先看它抓取了哪些目标

打开 http://localhost:30990/query ，这是查询首页（Table / Graph / Explain 三个视图）：

![image-20260920150605290](https://img.fz688.dpdns.org/2026-09-20-1789887965591.png)

第一次进来是空的（"No data queried yet"）。我们先输入 **`up`**——它不是exporter 暴露的，而是 Prometheus 自己对每个抓取目标合成的"心跳"：抓取成功=1，失败=0：

![up 查询结果：14 条序列全部为 1](https://img.fz688.dpdns.org/2026-09-20-1789887912419.png)

- **Result series: 14** —— 当前集群里有 14 个抓取目标，全部值为 `1`（健康）。任何一个变成 0，就代表对应目标抓取失败，这是排障第一入口。
- 每条序列的标签里藏着身份：`job="kps-grafana"`、`job="kubelet"`、`job="coredns"`、`job="kps-kube-state-metrics"`、`job="apiserver"`…… **job 名 = ServiceMonitor 生成的抓取任务名**。
- 注意 `job="kps-kube-prometheus-stack-prometheus"` 这条——**Prometheus 自己也在抓自己**（自省指标）。
- 很多教程让你查 `up{job="prometheus"}`，在 kps 部署下会**查不到任何东西**——因为 job 名带了 release 前缀。

再画一张图。切到 Graph 标签，输入：

```promql
rate(process_cpu_seconds_total{job="kps-kube-prometheus-stack-prometheus"}[1m])
```

![Prometheus 自身 CPU 使用率曲线](https://img.fz688.dpdns.org/2026-09-20-1789887912325.png)

> **读图**：过去 1 小时 Prometheus 进程自身的 CPU 大约稳定在 **0.02~0.03 核**（一个核的 2.5%-3%），中间那个突刺到 0.05 是一次查询/压缩操作。`rate(...[1m])` 的含义是"取 1 分钟窗口内计数的每秒平均增长"，counter 类型指标必配 rate（后续将promql再继续深入）。

### Targets 页：抓取目标的健康状态检查

Status → Target health（或直接访问 `/targets`）：

![Targets 页面：所有目标 UP](https://img.fz688.dpdns.org/2026-09-20-1789887912375.png)

- 按 **scrape pool**（抓取池）分组，每个池对应一个 job；
- **Last scrape** 显示最近一次抓取的耗时和距今时间（30s 间隔，所以最多 30s 前一定抓过一轮）；
- **State = UP（绿色）**。如果抓取失败，这里会变红色 DOWN，并直接显示错误信息（如 `connection refused`、`context deadline exceeded`）——**排障的时候一般从这一页开始看**。
- 分组标题 `serviceMonitor/monitoring/kps-grafana/0` 的含义：这个目标来自 `monitoring` 命名空间里名为 `kps-grafana` 的 ServiceMonitor 的第 0 个端点。此处再次体现了"Operator 把 K8s 对象翻译成抓取配置"。

### Service Discovery 页：看懂 Operator 的"翻译结果"

Status → Service discovery（`/service-discovery`）：

![Service Discovery 页面：Discovered labels](https://img.fz688.dpdns.org/2026-09-20-1789887912505.png)

这一页展示每个目标**在进入抓取配置之前，服务发现阶段附带的全部元数据标签**：

- `__address__="10.1.1.15:3000"`（目标地址，Pod IP + 端口）、`__scheme__`、`__metrics_path__` 这些双下划线开头的是"抓取控制标签"；
- `__meta_kubernetes_pod_name`、`__meta_kubernetes_endpoints_label_app_kubernetes_io_managed_by="Helm"` 这些 `__meta_` 开头的是从 K8s API 发现的元数据；
- 右上角 `1 / 24`：发现阶段有 24 个候选标签，经过 relabeling（标签重写）筛选后只剩 1 个进入最终抓取——**relabeling 是 ServiceMonitor 的底层机制，后续会讲**，这里先混个眼熟。

###  Grafana：登录与开箱即用的仪表盘

浏览器打开 http://localhost:30330 ：

![Grafana 登录页](https://img.fz688.dpdns.org/2026-09-20-1789887912450.png)

输入 `admin / Grafana12345`（先前我们在 kps-values.yaml 里设置的）：

![Grafana 首页](https://img.fz688.dpdns.org/2026-09-20-1789887914346.png)

左侧菜单从上到下：Dashboards（仪表盘）、Explore（临时查询，相当于 Prometheus 查询页的增强版）、Alerting（Grafana 自带的告警系统，注意它和 Alertmanager 是两套东西）、Connections（数据源等连接管理）、Administration（管理后台）。

也可以在左上角设置里将语言改成简体中文，可能会起到方便查看的效果？因为中文翻译仍然无法全面覆盖，我还是选择默认的英文，看着更连贯些

![image-20260920152837730](https://img.fz688.dpdns.org/2026-09-20-1789889318095.png)

**安装完后第一步永远是检查数据源**。Connections → Data sources：

![数据源列表：Prometheus 和 Alertmanager 已自动注入](https://img.fz688.dpdns.org/2026-09-20-1789887914459.png)

两个数据源已经自动配好：

- `Prometheus`（带 default 标记），URL 是 `http://kps-kube-prometheus-stack-prometheus.monitoring:9090` —— **K8s 集群内部 DNS**。Grafana Pod 访问 Prometheus 走的是集群内网络，所以 URL 不是 NodePort 暴露的localhost:30990；
- `Alertmanager`，URL 同理。

点进 Prometheus 数据源看详情，注意顶部这条提示：

![数据源详情：Provisioned data source 提示](https://img.fz688.dpdns.org/2026-09-20-1789887914589.png)

> **Provisioned data source**："本数据源由配置文件注入，不能在 UI 里修改"。
>
> 这是 Helm 部署的一个核心特征：数据源不是我们点鼠标加的，而是 Grafana 容器启动时从挂载的配置文件（由 Chart 生成、sidecar 自动加载）注入的。好处是**可复现**——删掉 Pod 重建，配置原样回来；代价是 UI 里改不了，要改就得改 values 再 `helm upgrade`。这就是"配置即代码"（GitOps 思想）的直观体现。

**第二步：看开箱自带的仪表盘**。Dashboards 列表：

![仪表盘列表：kubernetes-mixin 全家桶](https://img.fz688.dpdns.org/2026-09-20-1789887914669.png)

这一长串（Alertmanager/Overview、CoreDNS、Kubernetes / API server、Kubernetes / Compute Resources 系列、Node Exporter 系列……）都是 chart 预置的，源自社区标准项目 **kubernetes-mixin**（标签也标着 `kubernetes-mixin`）。挺方便的，不用手动安装

挑最常用的 **Kubernetes / Compute Resources / Cluster**（集群资源总览）打开：

![集群资源总览仪表盘](https://img.fz688.dpdns.org/2026-09-20-1789887914750.png)

- 顶部一排大数字：CPU Utilisation 6.15%、CPU Requests Commitment 20.5%、CPU Limits Commitment 70.3%、Memory Utilisation 80.3%…… 这是集群级别的指标；
- 中间 CPU Usage 时序图显示 **No data**，这不是故障，这个面板读的是 recording rule（预聚合指标）+ `cluster` 变量过滤，单节点学习环境里变量取值和面板默认过滤条件对不上。
- 底部 CPU Quota 表格按命名空间统计：`workshop-infra`（我们的 MySQL/Redis/MinIO）、`monitoring`、`kube-system`、`cattle-system`（Rancher）…… 注意这已经是**用监控数据反观整个集群**了。

> Node Exporter Full（社区仪表盘 ID 1860）是最著名的节点仪表盘，kps 已预置 Node Exporter / Nodes 系列卷展——多节点集群里它最常用，本机单节点只有一张卡，但是还是点开感受下。

![image-20260920154414324](https://img.fz688.dpdns.org/2026-09-20-1789890254502.png)

![image-20260920154612085](https://img.fz688.dpdns.org/2026-09-20-1789890372902.png)

#### TODO

后续再出个 Grafana 专篇讲下怎么导入社区仪表盘和自制仪表盘

### Alertmanager

打开 http://localhost:30993 ：

![Alertmanager 首页：Watchdog 心跳告警](https://img.fz688.dpdns.org/2026-09-20-1789887914826.png)

**这张图上有两个真实告警，都值得来讲下**：

1. **`Watchdog`（severity="none"）**——一条**故意永远 Firing 的告警**。它的作用是"心跳校验"：正常情况下你**永远应该能在 Alertmanager 里看到它**。如果哪天 Watchdog 消失了，说明 Prometheus→Alertmanager 的链路断了。这是 Google SRE 的经典实践，防止"告警系统自己挂了却没人知道"。
2. **`AlertmanagerClusterCrashlooping`（severity="critical"）**——安装当晚 Alertmanager 容器重启过几次触发的真实告警，第二天自愈。读者在自己环境里大概率也会遇到类似"历史告警"，学会看 `startsAt`/`endsAt` 时间戳判断是否为当前问题即可。

## 梳理原理

梳理一下原理， 此次通过helm安装了10个CRD，

![image-20260920155435073](https://img.fz688.dpdns.org/2026-09-20-1789890875255.png)

给 K8s 扩充了 `ServiceMonitor`、`PodMonitor`、 `PrometheusRule` 等这些新的资源对象。

然后chart 会替我们把监控栈自己的每个组件都接好了监控，比如ServiceMonitor：

![image-20260920155815936](https://img.fz688.dpdns.org/2026-09-20-1789891096123.png)

Prometheus Operator在这套架构的工作方式：把 Prometheus 实例和监控配置都变成 K8s 自定义资源（`Prometheus`/`ServiceMonitor`/`PrometheusRule` 等 CRD），Operator 控制器持续 watch 并渲染成配置，热加载生效。

![image-20260920160004642](https://img.fz688.dpdns.org/2026-09-20-1789891204860.png)

Prometheus的配置由 Operator 生成的 Secret 动态渲染。因此：一切监控配置的修改都应该通过创建/修改 CRD 对象完成。下图是经过 Operator 翻译出来的一份配置

![image-20260920160218373](https://img.fz688.dpdns.org/2026-09-20-1789891338674.png)

里面的 `scrape_configs` 就是被翻译后的抓取配置——每个 ServiceMonitor 变成了一段 job 配置。

## 小结

本篇从零完成了一套生产级监控栈的部署，并且对它的内部结构进行分析。



## 参考资料

- kube-prometheus-stack 官方仓库：https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
- Prometheus Operator 文档：https://prometheus-operator.dev/
