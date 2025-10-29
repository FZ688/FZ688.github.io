---
abbrlink: ''
categories:
- - 学习记录
date: '2025-10-30T00:56:42.038360+08:00'
tags:
- Linux
- Docker
title: 记一次解决：新版Docker Desktop下，容器默认挂载的数据卷无法访问
updated: '2025-10-30T00:56:44.462+08:00'
---
我在 WSL2上使用Docker Desktop运行 Docker。以docker部署elasticsearch容器为例,它有一个“es-plugins” 的卷。当我运行`docker volume inspect es-plugins`时，得到以下输出：
