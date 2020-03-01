---
title: docker实战系列三之image镜像使用
date: 2020-02-19 14:20:05
tags: [docker, 容器, image, 镜像]
---
## 镜像image的特点

容器目的就是运行应用或者服务，这意味着容器的镜像中必须包含应用/服务运行所必需的操作系统和应用文件。

但是，容器又追求快速和小巧，这意味着构建镜像的时候通常需要裁剪掉不必要的部分，保持较小的体积。

例如，Docker 镜像通常不会包含 6 个不同的 Shell 让读者选择——通常 Docker 镜像中只有一个精简的Shell，甚至没有 Shell。

镜像中还不包含内核——容器都是共享所在 Docker 主机的内核。所以有时会说容器仅包含必要的操作系统（通常只有操作系统文件和文件系统对象）。
>提示：Hyper-V 容器运行在专用的轻量级 VM 上，同时利用 VM 内部的操作系统内核。

Docker 官方镜像 Alpine Linux 大约只有 4MB，可以说是 Docker 镜像小巧这一特点的比较典型的例子。

但是，镜像更常见的状态是如 Ubuntu 官方的 Docker 镜像一般，大约有 110MB。这些镜像中都已裁剪掉大部分的无用内容。

Windows 镜像要比 Linux 镜像大一些，这与 Windows OS 工作原理相关。比如，未压缩的最新 Microsoft .NET 镜像（microsoft/dotnet:latest）超过 1.7GB。Windows Server 2016 Nano Server 镜像（microsoft/nanoserver:latest）在拉取并解压后，其体积略大于 1GB。


## 获取image
Docker 主机安装之后，本地并没有镜像。docker image pull 是下载镜像的命令。镜像从远程镜像仓库服务的仓库中下载。默认情况下，镜像会从 [Docker Hub](https://hub.docker.com/) 的仓库中拉取。docker image pull busybox:latest 命令会从 Docker Hub 的 busybox 仓库中拉取标签为 latest 的镜像。

```
$ docker pull
"docker pull" requires exactly 1 argument.
See 'docker pull --help'.

Usage:  docker pull [OPTIONS] NAME[:TAG|@DIGEST]

Pull an image or a repository from a registry
```


## image registry

Docker 镜像存储在镜像仓库服务（Image Registry）当中。镜像仓库服务包含多个镜像仓库（Image Repository）。我们可以搭建私有的registry，比如harbor。
同样，一个镜像仓库中可以包含多个镜像。可能这听起来让人有些迷惑，所以下图展示了包含 3 个镜像仓库的镜像仓库服务，其中每个镜像仓库都包含一个或多个镜像。
![regsitry](dk4.png)

## image命名和标签

只需要给出镜像的名字和标签，就能在官方仓库中定位一个镜像（采用“:”分隔）。从官方仓库拉取镜像时，docker image pull 命令的格式如下。
```
docker image pull <repository>:<tag>
```

在之前的 Linux 示例中，通过下面的两条命令完成 Alpine 和 Ubuntu 镜像的拉取。
```
docker image pull alpine:latest
docker image pull ubuntu:latest
```

这两条命令从 alpine 和 ubuntu 仓库拉取了标有“latest”标签的镜像。

下面来介绍一下如何从官方仓库拉取不同的镜像。
```
$ docker image pull mongo:3.3.11
//该命令会从官方Mongo库拉取标签为3.3.11的镜像

$ docker image pull redis:latest
//该命令会从官方Redis库拉取标签为latest的镜像

$ docker image pull alpine
//该命令会从官方Alpine库拉取标签为latest的镜像
```
关于上述命令，需要注意以下几点。

首先，如果没有在仓库名称后指定具体的镜像标签，则 Docker 会假设用户希望拉取标签为 latest 的镜像。

其次，标签为 latest 的镜像没有什么特殊魔力！标有 latest 标签的镜像不保证这是仓库中最新的镜像！例如，Alpine 仓库中最新的镜像通常标签是 edge。通常来讲，使用 latest 标签时需要谨慎！

从非官方仓库拉取镜像也是类似的，读者只需要在仓库名称面前加上 Docker Hub 的用户名或者组织名称。

下面通过示例来展示如何从 jishengqiu 仓库中拉取 v1 这个镜像，其中镜像的拥有者是 Docker Hub 账户 jishengqiu，一个不应该被信任的账户。
$ docker image pull jishengqiu/hello-docker:v1
//该命令会从以我自己的 Docker Hub 账号为命名空间的 hello-docker 库中下载标签为 v1 的镜像

在之前的 Windows 示例中，使用下面的两条命令拉取了 PowerShell 和 .NET 镜像。
> docker image pull microsoft/powershell:nanoserver

> docker image pull microsoft/dotnet:latest

第一条命令从 microsoft/powershell 仓库中拉取了标签为 nanoserver 的镜像，第二条命令从 microsoft/dotnet 仓库中拉取了标签为 latest 的镜像。

如果希望从第三方镜像仓库服务获取镜像（非 Docker Hub），则需要在镜像仓库名称前加上第三方镜像仓库服务的 DNS 名称。

假设上面的示例中的镜像位于 Google 容器镜像仓库服务（GCR）中，则需要在仓库名称前面加上 gcr.io，如 docker pull gcr.io/nigelpoulton/tu-demo:v2（这个仓库和镜像并不存在）。

可能需要拥有第三方镜像仓库服务的账户，并在拉取镜像前完成登录。


## 镜像（image）和层（layer）
Docker 镜像由一些松耦合的只读镜像层组成。如下图所示。
![imagelayer](dk5.png)
Docker 负责堆叠这些镜像层，并且将它们表示为单个统一的对象。

查看镜像分层的方式可以通过 docker image inspect 命令。下面同样以 redis:latest 镜像为例。
```
[vagrant@docker-host ~]$ docker pull redis
Using default tag: latest
latest: Pulling from library/redis
bc51dd8edc1b: Pull complete  // layer1
37d80eb324ee: Pull complete  // layer2
392b7748dfaf: Pull complete  // layer3
48df82c3534d: Pull complete  // layer4
2ec2bb0b4b0e: Pull complete  // layer5
1302bce0b2cb: Pull complete  // layer6
Digest: sha256:7b84b346c01e5a8d204a5bb30d4521bcc3a8535bbf90c660b8595fad248eae82
Status: Downloaded newer image for redis:latest
docker.io/library/redis:latest
[vagrant@docker-host ~]$ docker inspect redis
[
      ......
      skip
      ......
      
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:488dfecc21b1bc607e09368d2791cb784cf8c4ec5c05d2952b045b3e0f8cc01e",
                "sha256:6d35c327901c03885dc2646367382a9cf90b77e07fb5bc1347905eed9a9f464b",
                "sha256:35e1c7a160662fc0b9a97d6cd893978b6a531d81bede805e47fcae1a06ebe751",
                "sha256:9f839e56c43407961d93dcb97372ffbe5abf7b313f0db9d4328770b1e5caaeab",
                "sha256:d7344f36256ca9a11964e67a4b16fb51e147c69bf4f7b9f3e150f5d13c64f781",
                "sha256:0233556febffa2d711e3f9d34f3daaab58063987b910232728bbea71ba5906bb"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        }
    }
]

```

**所有的 Docker 镜像都起始于一个基础镜像层，当进行修改或增加新的内容时，就会在当前镜像层之上，创建新的镜像层**
*简而言之。每一个新的image，都是站在巨人的肩膀上产生的*

## 制作一个自定义的image最佳实践[Dockerfile](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
docker给我们提供了busybox镜像，聚成了一百多个最常用linux命令和工具的软件工具箱，它在单一的可执行文件中提供了精简的Unix工具集。BusyBox可运行于多款POSIX环境的操作系统中，如 Linux(包括Android),hurd,freebsa等。busybox既包含了一些简单实用工具，如cat 和 echo ,也包含了一些更大，更复杂的工具，如 grep ,find ,mount 以及telnet。
优势在于镜像非常小巧，下面举个栗子来演示制作一个镜像
```
$ docker pull busybox //拉取busybox镜像
$ docker image ls
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
busybox                 latest              6d5fcfe5ff17        8 weeks ago         1.22MB
//会发现busybox非常小巧
$ mkdir myfirstimage //创建一个工作目录
$ cd myfirstimage  //进入当前工作目录
$ touch Dockerfile // 创建Dockerfile
$ vim Dockerfile //编辑Dockerfile文件
  //Dockerfile的内容如下
  FROM busybox
  CMD ["echo","hello docker"]
$ docker build -t jishengqiu/firstimage .  //表示从当前目录的Dockerfile构建镜像 jishengqiu表示docker hub账号   firstimage表示仓库名firstimage  -t默认的tag为latest
$ docker  image ls
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
jishengqiu/firstimage   latest              cd312ca5fb9d        4 minutes ago       1.22MB
busybox                 latest              6d5fcfe5ff17        8 weeks ago         1.22MB

```
通过上面的例子，可以对镜像的构建初步认识，自定义镜像是from busybox，新的镜像大小可以看到基本与busybox大小相近，当然这取决于Dockerfile中的镜像构建逻辑来决定的

## 推送镜像到镜像仓库服务
 首先需要注册`docker hub`账号，例如jishengqiu
 然后使用`docker image push jishengqiu/firstimage`来推送自定义的镜像，这个动作就类似将本地的代码提交到远程git仓库。

## 共享镜像层 share layer
多个镜像之间可以并且确实会共享镜像层。这样可以有效节省空间并提升性能，注意那些以 Already exists 结尾的行。Docker 很聪明，可以识别出要拉取的镜像中，哪几层已经在本地存在。如前所述，Docker 在 Linux 上支持很多存储引擎（Snapshotter）。每个存储引擎都有自己的镜像分层、镜像层共享以及写时复制（CoW）技术的具体实现。但是，其最终效果和用户体验是完全一致的。尽管 Windows 只支持一种存储引擎，还是可以提供与 Linux 相同的功能体验。



## 删除镜像
当不再需要某个镜像的时候，可以通过 `docker image rm `命令从 Docker 主机删除该镜像。其中，rm 是 remove 的缩写。如果被删除的镜像上存在运行状态的容器，那么删除操作不会被允许。再次执行删除镜像命令之前，需要停止并删除该镜像相关的全部容器。
一种删除某 Docker 主机上全部镜像的快捷方式是在 docker image rm 命令中传入当前系统的全部镜像 ID，可以通过 docker image ls 获取全部镜像 ID（使用 -q 参数）。
```
$ docker image rm $(docker image ls -q) -f

[vagrant@docker-host ~]$ docker image ls -q
44d36d2c2374
ccc6e87d482b
6d5fcfe5ff17
fce289e99eb9
[vagrant@docker-host ~]$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redis               latest              44d36d2c2374        2 weeks ago         98.2MB
ubuntu              latest              ccc6e87d482b        5 weeks ago         64.2MB
busybox             latest              6d5fcfe5ff17        7 weeks ago         1.22MB
hello-world         latest              fce289e99eb9        13 months ago       1.84kB
[vagrant@docker-host ~]$docker image rm $(docker image ls -q) -f
[vagrant@docker-host ~]$docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
```
可以看到 `docker image ls -q `命令只返回了系统中本地拉取的全部镜像的 ID 列表。将这个列表作为参数传给 docker image rm会删除本地系统中的全部镜像。

下一篇博客，介绍docker的容器用法

*如有错误，请指正*