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
readmore: true
author: FZ688
excerpt: ' 我在 WSL2上使用Docker Desktop运行 Docker。以docker部署elasticsearch容器为例,它有一个“es-plugins” 的卷。当我运行docker volume inspect es-plugins时，得到以下输出：[{{"CreatedAt": "2025-10-28T18:15:37Z","Driver": "local","Labels": null,"Mountpoint": "/var/lib/docker/volumes/es-plugins/_data","Name": "es-plugins","Options": null,"Scope": "local"}}]显示挂载到了/var/lib/docker/volumes/es-plugins/_data这个路径，然而我不能cd进入该路径，因为它不存在。'
---
## 问题：使用Docker Desktop安装容器后，无法进入数据卷挂载的宿主机目录

我在 WSL2上使用Docker Desktop运行 Docker。以docker部署elasticsearch容器为例,它有一个“es-plugins” 的卷。当我运行`docker volume inspect es-plugins`时，得到以下输出：

![image-20251030200710215](https://img.fz688.dpdns.org/2025-10-30-1761826030310.png)

```sh

[
    {
        "CreatedAt": "2025-10-28T18:15:37Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/es-plugins/_data",
        "Name": "es-plugins",
        "Options": null,
        "Scope": "local"
    }
]
```

显示挂载到了 `/var/lib/docker/volumes/es-plugins/_data` 这个路径，然而我不能`cd`进入该路径，因为它不存在。

![image-20251030202506155](https://img.fz688.dpdns.org/2025-10-30-1761827106221.png)

## 解决过程：

### 旧版本的Docker Desktop解决方式

网上大部分教程都说：在windows资源管理器路径里可以访问数据卷

在 Windows 文件资源管理器中输入：

- 对于 Docker 版本 20.10.+： `\\wsl$\docker-desktop-data\data\docker\volumes`
- 对于 Docker 引擎 v19.03： `\\wsl$\docker-desktop-data\version-pack-data\community\docker\volumes\`

每个卷都有一个目录。

### 新版本的Docker Desktop解决方式

#### 搜索

我的环境是WSL2（Ubuntu24.02） + Docker Desktop（4.49.0）

![image-20251030205313833](https://img.fz688.dpdns.org/2025-10-30-1761828793905.png)

继续查阅搜索发现： Docker Desktop从4.30.0开始，使用WSL2运行的docker中，“docker-desktop-data”储存位置消失了，更新日志显示：

[Release notes | Docker Docs](https://docs.docker.com/desktop/release-notes/#4300)

![image-20251030204024698](https://img.fz688.dpdns.org/2025-10-30-1761828024872.png)

`docker-desktop-data`这个存储路径，它实际上被移动到了

`\\wsl.localhost\docker-desktop\mnt\docker-desktop-disk` 这个路径下

![image-20251030204514489](https://img.fz688.dpdns.org/2025-10-30-1761828314626.png)

#### 解决：

而挂载的数据卷呢，就存储在`\\wsl.localhost\docker-desktop\mnt\docker-desktop-disk\data\docker\volumes\`这个目录下

![image-20251030204933052](https://img.fz688.dpdns.org/2025-10-30-1761828573141.png)

找到的 `\\wsl.localhost\docker-desktop\mnt\docker-desktop-disk\data\docker\volumes\...` 路径，是 Windows 系统通过 WSL 机制，对 Linux 虚拟机内文件系统的**共享访问入口**。相当于 Windows 把 Linux 虚拟机的文件目录 “映射” 到了自身可访问的路径下，因此直接在 Windows 命令行（如 PowerShell）中无法访问 Linux 格式的 `/var/lib/docker/volumes/...` 路径，必须通过 WSL 共享路径才能找到实际文件。







参考链接：[新版Docker Desktop的docker-desktop-data位置消失 - 哔哩哔哩](https://www.bilibili.com/opus/1055381775162802176)
