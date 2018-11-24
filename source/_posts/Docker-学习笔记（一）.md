title: Docker 学习笔记（一）
author: ACE NI
date: 2018-06-28 22:31:11
tags:
---
Linux 容器不是模拟一个完整的操作系统，而是对进程进行隔离。

- 启动快
- 占用资源少
- 体积小

Docker 属于 Linux 容器的一种封装，提供简单易用的容器使用接口。

- 提供一次性的环境
- 提供弹性的云服务
- 组建微服务架构

<!--more-->

```
# Uninstall old versions
$ yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine
                  
# Install Docker CE
$ yum install -y yum-utils device-mapper-persistent-data lvm2
  
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

$ yum-config-manager --enable docker-ce-edge

$ yum-config-manager --enable docker-ce-test

$ yum install docker-ce

$ systemctl start docker

$ docker version
$ docker info

```

Docker 把应用程序及其依赖，打包在 image 文件里面。

```
# 列出本机的所有 image 文件。
$ docker image ls

# 删除 image 文件
$ docker image rm [imageName]
```

image 文件是通用的，一台机器的 image 文件拷贝到另一台机器，照样可以使用。一般来说，为了节省时间，我们应该尽量使用别人制作好的 image 文件，而不是自己制作。即使要定制，也应该基于别人的 image 文件进行加工，而不是从零开始制作。

```
# 配置国内镜像仓库
$ vi /etc/docker/daemon.json
{
    "registry-mirrors": ["https://registry.docker-cn.com"]
}

$ systemctl restart docker
```

`docker container run` 命令会从 image 文件，生成一个正在运行的容器实例。

注意，`docker container run` 命令具有自动抓取 image 文件的功能。如果发现本地没有指定的 image 文件，就会从仓库自动抓取。

对于那些不会自动终止的容器，必须使用`docker container kill [containerID]` 命令手动终止。

**image 文件生成的容器实例，本身也是一个文件，称为容器文件。**也就是说，一旦容器生成，就会同时存在两个文件： image 文件和容器文件。而且关闭容器并不会删除容器文件，只是容器停止运行而已。

```
# 列出本机正在运行的容器
$ docker container ls

# 列出本机所有容器，包括终止运行的容器
$ docker container ls --all
```

终止运行的容器文件，依然会占据硬盘空间，可以使用`docker container rm [containerID]` 命令删除。


RUN 命令在 image 文件的构建阶段执行，执行结果都会打包进入 image 文件；
CMD 命令则是在容器启动后执行。另外，一个 Dockerfile 可以包含多个RUN命令，但是只能有一个CMD命令




**（1）docker container start**

前面的docker container run命令是新建容器，每运行一次，就会新建一个容器。同样的命令运行两次，就会生成两个一模一样的容器文件。如果希望重复使用容器，就要使用docker container start命令，它用来启动已经生成、已经停止运行的容器文件。

```
$ docker container start [containerID]
```

**（2）docker container stop**

前面的docker container kill命令终止容器运行，相当于向容器里面的主进程发出 SIGKILL 信号。而docker container stop命令也是用来终止容器运行，相当于向容器里面的主进程发出 SIGTERM 信号，然后过一段时间再发出 SIGKILL 信号。

```
$ docker container stop [containerID]
```

这两个信号的差别是，应用程序收到 SIGTERM 信号以后，可以自行进行收尾清理工作，但也可以不理会这个信号。如果收到 SIGKILL 信号，就会强行立即终止，那些正在进行中的操作会全部丢失。

**（3）docker container logs**

docker container logs命令用来查看 docker 容器的输出，即容器里面 Shell 的标准输出。如果docker run命令运行容器的时候，没有使用-it参数，就要用这个命令查看输出。

```
$ docker container logs [containerID]
```

**（4）docker container exec**

docker container exec命令用于进入一个正在运行的 docker 容器。如果docker run命令运行容器的时候，没有使用-it参数，就要用这个命令进入容器。一旦进入了容器，就可以在容器的 Shell 执行命令了。
```
$ docker container exec -it [containerID] /bin/bash
```

**（5）docker container cp**

docker container cp命令用于从正在运行的 Docker 容器里面，将文件拷贝到本机。下面是拷贝到当前目录的写法。

```
$ docker container cp [containID]:[/path/to/file] .
```
