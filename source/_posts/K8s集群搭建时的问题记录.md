---
abbrlink: ''
categories: []
date: '2026-09-15T11:21:34.569478+08:00'
tags: []
title: K8s集群搭建时的问题记录
updated: '2026-09-15T11:56:59.925+08:00'
---
物理机时代应用硬件系统一起绑定，虚拟化时代隔离了硬件，容器化时代只隔离应用与配置——**容器是云架构的最佳载体**

1、rancher agent 这类 hostNetwork Pod 要配 `dnsPolicy: ClusterFirstWithHostNet` 才能用集群 DNS。

2、证书相关

K8s组件之间（apiserver↔etcd↔kubelet）也全走 HTTPS 验证，用的是**装集群时自动生成的自签证书**（不是买的），默认一年有效期。**过期不换**导致 **组件互认失败**，严重甚至 **整个集群瘫痪**（比网站证书过期严重得多）。

点了 Rotate 后其他组件都成功，唯独 kube-controller-manager（没换成功，集群卡在 Provisioning。排查：

```bash
cd /var/lib/rancher/rke2/server/tls   
for i in $(find ./ -name "*.crt"); do  #  找出所有 .crt 证书
  openssl x509 -in $i --noout -enddate
done
```

一扫就知道哪张还是旧日期 ，之后 定向重转：`rke2 cert rotate --service kube-controller-manager`。
