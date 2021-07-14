---
title: Docker学习
date: 2021-07-09 15:08:22
tags: Docker
categories: 工具
description:
---

## Docker简介

Docker由Go语言编写而成，其结构图如下：

![](https://docs.docker.com/engine/images/architecture.svg)

- Docker daemon：Docker后台（dockerd），用于监听Docker API请求，并管理Docker对象（如images、containers、networks、volumes）
- Docker client：Docker客户端，我们访问Docker daemon时，我们的计算机就是Docker client。
- Docker Registry：存储Docker images
- Images：镜像，用于实例化容器。
- Containers：运行中的镜像，容器，可以理解成机器上的另一个进程。

mac安装docker：https://docs.docker.com/docker-for-mac/install/

## 启动

使用以下命令来启动一个docker镜像：

```bash
docker run -d -p 80:80 docker/getting-started
```

- `-d` - run the container in detached mode (in the background)
- `-p 80:80` - map port 80 of the host to port 80 in the container
- `docker/getting-started` - the image to use



## 参考资料

- [Docker官方文档](https://docs.docker.com/get-started/overview/)
- [Docker —— 从入门到实践](https://yeasy.gitbook.io/docker_practice/)

