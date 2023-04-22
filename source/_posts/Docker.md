---
title: Docker
date: 2023-04-13 00:19:13
categories:
- Env
tags:
- 笔记
- Docker
- Ubuntu
- Linux
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



# 端口占用

- 查找被占用的端口

  ```shell
  netstat -tln	//查看所有端口使用情况
  ```

  ```shell
  netstat -tln | grep 80	//只查看80端口的使用情况
  ```

- 查看那哪个端口被占用

  ```shell
  lsof -i :80
  ```

- 杀进程

  ```shell
  kill -9 进程id
  ```

  
