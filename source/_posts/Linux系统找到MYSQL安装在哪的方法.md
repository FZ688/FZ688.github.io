---
abbrlink: ''
categories:
- - 工作记录
date: '2025-09-24T14:45:11.643357+08:00'
excerpt: title: Linux系统查找MySQL安装路径 author: FZ688 readmore: true 为什么我要知道MYSQL安装在哪？ 最近公司给了台堡垒机服务器，我需要去连接服务器查看MySQL数据库表。因为这个堡垒机只能通过浏览器的远程vpn访问，ssh连接工具都无法使用，只能用它那个浏览器界面了去服务器找Mysql安装在哪才能连接到数据库😢。 1. whereis / where...
tags:
- Linux
- MySQL
title: Linux系统找到MYSQL安装路径
updated: '2025-09-24T15:06:19.277+08:00'
---
---
title: Linux系统查找MySQL安装路径
author: FZ688
readmore: true
---
## 为什么我要知道MYSQL安装在哪？

最近公司给了台堡垒机服务器，我需要去连接服务器查看MySQL数据库表。因为这个堡垒机只能通过浏览器的远程vpn访问，ssh连接工具都无法使用，只能用它那个浏览器界面了去服务器找Mysql安装在哪才能连接到数据库😢。

## 1. whereis / where/ which

whereis命令可以搜索指定的文件名，并返回其所在的位置。我们可以使用whereis命令查找MySQL服务的位置。

where命令和whereis命令类似，也用于查找命令的位置。但是，where命令会返回所有匹配项，而不仅仅是第一个匹配项。

which命令可以查找执行文件的路径。我们可以使用which命令查找mysql服务的位置。通过这种方式可以直接查找mysql执行文件的位置。

```shell
whereis mysql
where mysql
which mysql
```

很遗憾，通过这些命令呢，还是找不到MySQL安装路径。

## 2.find命令

既然上面3个命令都无法成功，那只能用最暴力的方式了。

find命令可以在指定的路径下搜索匹配的文件或目录。我们可以使用find命令查找MySQL的安装路径。

打开终端窗口，输入如下命令:

```shell
find / -name mysql
```

其中，/表示搜索的起始路径，-name mysql表示要搜索的文件名。在执行命令后，终端窗口会输出所有匹配的路径名称，其中包括MySQL服务的安装路径。

这个方法最暴力也直观，最后发现原来是通过**docker容器**部署的。

通过这件事呢，给了我点**经验**，以后只要where命令找不到，基本可以确定是MySQL是安装在容器里的。
