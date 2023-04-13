---
title: Docker
date: 2023-04-13 00:19:13
categories:
- Env
tags:
- 笔记
- Docker
---

# Docker启动失败

- failed to start daemon: error while opening volume store metadata database (/var/lib/docker/volumes/metadata.db): timeout

>  运行：
>
> ```shell
> ps axf | grep docker | grep -v grep | awk '{print "kill -9 " $1}' | sudo sh
> ```
>
> ```shell
> systemctl start docker
> ```
