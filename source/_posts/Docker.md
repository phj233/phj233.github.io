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

- 查看哪个端口被占用

  ```shell
  lsof -i :80
  ```

- 杀进程

  ```shell
  kill -9 进程id
  ```

  

# 本地推送镜像

## 拉取registry镜像并运行

```shell
docker pull registry
..
docker run -d -p 5000:5000 --name="myregistry" -v /myregistry/:/tmp/registry --privileged=true registry 
```

## 拉取原始ubuntu镜像

```shell
docker pull ubuntu
```

## 运行并进入容器

```shell
docker run -d -it --name="myubuntu" ubuntu
..
docker exec -it myubuntu /bin/bash
```

## 安装vim并退出容器

```shell
apt update
..
apt -y install vim
..
exit
```

## 本地提交镜像

docker commit -m="提交信息" -a="作者" 容器ID 创建镜像名:[标签名]

```shell
docker commit -m="add vim to ubuntu" -a="phj233" 1b3236 myubuntu:0.1
```

## 创建daemon.json文件，保存并重启docker

```shell
vim /etc/docker/daemon.json
..
{
  "insecure-registries": ["127.0.0.1:5000"]
}
..
systemctl restart docker
```

## 修改镜像tag

```shell
docker tag myubuntu:0.1 127.0.0.1:5000/myubuntu:0.1
```

## 推送

```shell
docker push 127.0.0.1:5000/myubuntu:0.1
```

若失败可执行

```shell
systemctl daemon-reload
systemctl restart docker
```

## 查看推送镜像

```shell
root@Az-JP:/# curl -XGET http://127.0.0.1:5000/v2/_catalog
{"repositories":["myubuntu"]}
```



