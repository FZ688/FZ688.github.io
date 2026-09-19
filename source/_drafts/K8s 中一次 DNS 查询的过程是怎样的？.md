---
abbrlink: 服务发现是 Kubernetes 整体架构中的关键一环。它让进来的请求能够路由到集群中正确的负载上，而 DNS 在这个过程中扮演着核心角色。
categories: []
date: '2026-09-19T02:16:54.728449+08:00'
tags: []
title: K8s 中一次 DNS 查询的过程是怎样的？
updated: '2026-09-19T02:16:57.152+08:00'
---
# K8s 中一次 DNS 查询

服务发现是 Kubernetes 整体架构中的关键一环。它让进来的请求能够路由到集群中正确的负载上，而 DNS 在这个过程中扮演着核心角色。

理解 Kubernetes 中 DNS 与服务发现的工作方式，对我们排查问题很有帮助。它能让我们更清楚地掌握集群内的流量走向，并诊断可能出现的各种故障。

**先给出结论。** 当一个 Pod 发起 DNS 查询时，查询首先被送往该 Pod 所在节点上的 DNS 缓存。如果缓存里没有所请求主机名对应的 IP 地址，查询会被转发给集群 DNS 服务器。在 Kubernetes 中，正是这台服务器负责服务发现。

集群 DNS 服务器通过查询 Kubernetes 的服务注册表来确定 IP 地址。这份注册表保存着服务名到对应 IP 地址的映射。凭借它，集群 DNS 服务器就能把正确的 IP 地址返回给发起请求的 Pod。

如前所述，DNS 是服务发现不可或缺的组成部分。Kubernetes 的服务发现依靠 Service 实现。每个 Service 拥有一个 IP，访问该 IP 时，连接会被转发到支撑这个 Service 的健康 Pod 上。

在 Kubernetes 中创建 Service 时，集群 DNS 服务器会为它创建一条 **A 记录**。这条记录把 Service 的 DNS 名称映射到它的 IP 地址，Pod 于是可以通过 DNS 名称来访问 Service。每当 Service 的 IP 地址变化，DNS 服务器也会更新这条 A 记录，确保 DNS 名称始终指向正确的 IP。

一个 Service 的定义大致如下：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: foo
  namespace: bar
spec:
  ports:
    - port: 80
      name: http
```

这个例子中创建出的 A 记录和 SRV 记录如下：

foo.bar.svc.cluster.local                  30   A   10.129.1.26
_http._tcp.nginx.default.svc.cluster.local 3600 SRV 0 100 80 10-129-1-26.foo.bar.svc.cluster.local.

要拼出这个 Service 的全限定域名（FQDN），需要用到服务名（`foo`）、命名空间（`bar`）和集群域（`cluster.local`）。

集群中任何负载现在都可以通过这个 DNS 名称解析出该 Service 的 IP 地址。

当 Pod 发起 DNS 查询时，查询首先被送往 Pod 内的**本地 DNS 解析器**。这个解析器依据 resolv.conf 配置文件工作。该文件中，nodelocaldns 服务器被配置为默认的递归 DNS 解析器，充当缓存。

如果缓存里没有所请求主机名对应的 IP 地址，查询会被转发给集群 DNS 服务器（[CoreDNS](https://coredns.io/)）。

这台 DNS 服务器通过查询 Kubernetes 服务注册表来确定 IP 地址。注册表中保存着服务名到 IP 地址的映射，因此集群 DNS 服务器能把正确的 IP 返回给发起请求的 Pod。

凡是被查询、但不在 Kubernetes 服务注册表中的域名，都会被转发到上游 DNS 服务器。

下面逐步拆解每一个组件。

当 Pod 要向同一 Kubernetes 集群内的 Service 发送 API 请求时，必须先解析出该 Service 的 IP 地址。为此，Pod 会按照其 [/etc/resolv.conf](https://en.wikipedia.org/wiki/Resolv.conf) 配置文件中指定的 DNS 服务器发起 DNS 查询。

这个文件由 kubelet 负责下发，定义了 Pod 内 DNS 查询的各项设置，其中包含对集群 DNS 服务器的引用。

默认情况下，这个配置文件的内容类似下面这样：

```
search namespace.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.123.0.10
options ndots:5
```

默认情况下，kubelet 提供的 `/etc/resolv.conf` 会把所有 DNS 查询转发给集群的 DNS 服务器（上例中的 10.123.0.10）。kubelet 还会为 DNS 查询定义搜索域（search domain）和 `ndots` 选项。

搜索域规定了当给出不完整的域名（非 FQDN）时应当尝试哪些域名后缀。`ndots` 选项则决定何时直接查询绝对域名、而不是先拼接搜索域。

通过一个例子更容易理解。假设名为 foo 的 Pod 对 `bar.other-ns` 发起 DNS 查询。若 `ndots` 选项为 5（默认值——[原因见此](https://github.com/kubernetes/kubernetes/issues/33554#issuecomment-266251056)），解析器会先数一数域名里点的个数。

如果点少于 5 个，就会先拼接搜索域再去 DNS 服务器查询；如果有 5 个或更多点，则按原样查询域名，不拼接搜索域。本例中 `bar.other-ns` 的点少于 5 个，因此会先拼接搜索域再查询。

默认的搜索域是：

在找到有效响应之前，解析器会把这些搜索域逐个拼接到域名后面依次查询：

bar 这个 Service 会监听在 `bar.other-ns.svc.cluster.local` 上，于是匹配成功，返回正确的 A 记录。

要改变 Pod DNS 解析器的行为，可以修改 Pod 的 DNS 配置：

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster-domain.example
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

上例中 `dnsPolicy` 设为 "None"，意味着该 Pod 不使用集群提供的默认 DNS 设置，而是通过 `dnsConfig` 字段为 Pod 指定自定义 DNS 设置。

`nameservers` 字段指定 Pod 进行 DNS 查询时使用的 DNS 服务器；`searches` 字段指定处理不完整域名时使用的搜索域。

`options` 字段为 DNS 解析器指定自定义选项，例如上例中的 `ndots` 和 `edns0`。

这些设置将取代集群提供的默认值，供 Pod 的 DNS 解析器使用。关于 Pod DNS 配置的更多信息，参见[官方文档](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config)。

在 Kubernetes 1.13 及更早的集群中，承担权威 DNS 服务器（[authoritative DNS server](https://www.nslookup.io/learning/recursive-vs-authoritative-dns/)）角色的是 kube-dns。从 Kubernetes 1.13 开始，[CoreDNS 取代 kube-dns](https://kubernetes.io/blog/2018/12/03/kubernetes-1-13-release-announcement/#coredns-is-now-the-default-dns-server-for-kubernetes)，成为处理权威 DNS 查询的默认组件。

DNS 服务器会把所有 Service 纳入其权威 [DNS zone](https://www.nslookup.io/learning/what-is-a-dns-zone/)，从而为 Kubernetes 服务完成域名到 IP 的解析。Kubernetes 中的权威 DNS 服务器存在多种软件实现。

CoreDNS 是流行的选择：它支持直接从 Kubernetes 服务注册表构建 DNS zone，还提供缓存、转发、日志等附加功能。

CoreDNS 配置文件的一个示例：

```nginx
.:53 {
    errors
    health {
        lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    forward . /etc/resolv.conf
    cache 30
}
```

需要重点关注的是 kubernetes zone 配置和 forward 语句。

关于修改 kube-dns 配置的更多信息，参见[这份文档](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)。

DNS 查询是网络通信中常见且必不可少的环节，必须被快速处理，否则会引发性能问题。缓慢的 DNS 查询造成的问题往往难以诊断和排查。

为了提升 Kubernetes 集群中 DNS 查询的性能，可以在每个节点上加一层缓存，即 [nodelocaldns](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) 组件。它负责缓存 DNS 查询的响应。

缓存未命中时，它会把查询转发给权威域名服务器（CoreDNS）。响应会被存入本地缓存，用于应答同一节点上相同或其他 Pod 的后续查询。

这减少了 Pod 与 DNS 服务器之间的网络流量，意味着更低的延迟和更快的 DNS 查询。nodelocaldns 的职能也常常由 CoreDNS 本身承担。

在 Kubernetes 中，DNS 记录的存活时间（TTL）由所使用的 DNS 服务器实现决定。

默认情况下，CoreDNS 把 DNS 记录的 TTL 设为 30 秒。也就是说，一条 DNS 查询被解析后，响应最多缓存 30 秒就会被视为过期。可以通过 CoreDNS 配置文件中的 `ttl` 选项修改 TTL。

TTL 是个重要参数，它决定了一条 DNS 响应在必须重新发起查询之前被视为有效的时间长短。

较短的 TTL 能提升 DNS 响应的准确性，但会增加 DNS 服务器的负载；较长的 TTL 能减轻 DNS 服务器的负载，但当底层 DNS 记录发生变更时，也可能导致响应过期或不准确。

因此，应当根据集群的具体需求选择合适的 TTL。

到目前为止我们只讨论了用 A 记录解析 IP 地址。Kubernetes 还使用 SRV（服务）记录来解析命名端口的端口号。客户端通过向 DNS 服务器查询相应的 SRV 记录，就能发现服务的端口号。

```
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  ports:
    - port: 80
      name: http
```

这个 Service 暴露了容器端口 80，并将其命名为 "http"。因为端口有了名字，Kubernetes 会生成一条名称为 `_<port>._<proto>.<service>.<ns>.svc.<zone>` 的 SRV 记录。

本例中，这条 SRV 记录的名称是 `_http._tcp.nginx.default.svc.cluster.local`。对它发起 DNS 查询会返回该命名服务的端口号和 IP 地址：

dig +short SRV _http._tcp.nginx.default.svc.cluster.local
0 100 80 10-129-1-26.nginx.default.svc.cluster.local.

某些服务（例如 Kerberos）就使用 SRV 记录来发现 KDC（Key Distribution Center，密钥分发中心）服务器。
