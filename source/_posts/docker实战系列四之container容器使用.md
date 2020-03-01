---
title: docker实战系列四之container容器使用
date: 2020-02-21 15:34:30
tags:
---
## 理解容器
`docker`中的`image`类比为`java`中`class`，`container`类比为`java`中`object`。还有一个比喻可以把`image`理解为工程上的模具，而容器就是使用这个模具“生产”出来的产品。
![dk](dk1.jpg)


## 运行容器
启动容器的简便方式是使用`docker container run`命令
```
$ docker container run 可以简写为docker run 
"docker container run" requires at least 1 argument.
See 'docker container run --help'.

Usage:  docker container run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container
```

回顾上篇博文中，使用`busybox`作为`base image`制作的`firstimage`镜像，可以用下面命令启动起来
```
$ docker container run jishengqiu/firstimage //可以看到，输出hello docker，即是Dockerfile中要执行的CMD命令
hello docker

$ docker container stop //停止容器id or name 
$ docker container rm //删除容器，不能删除正在运行的容器
$ docker ps //查看容器进程，没有正在运行的容器，因为不是后台执行的
$ docker ps -a //显示所有容器，可以看到刚才打印hello docker容器，STATUS为Exited
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS                           PORTS               NAMES
4f4c821173b8        jishengqiu/firstimage   "echo 'hello docker'"    3 seconds ago       Exited (0) 2 seconds ago                             interesting_nightingale
```
`CONTAINER ID` 唯一标记一个容器
`IMAGE`  表示容器使用的镜像
`COMMAND` 容器运行时执行的命令
`CREATED`容器创建的时间
`STATUS`  容器运行的状态
`PORTS`  容器对外暴露的端口,可以使用-p参数指定
`NAMES`   容器的名称，唯一，可以使用--name参数指定


## 操作容器
启动容器，能不能进到这个容器里面呢？答案是肯定的。
```
$  docker container run -it jishengqiu/firstimage sh //交互式运行 docker container run --help可以查看-i -t的含义，sh表示容器运行起来时要执行的命令
   # 进入sh窗口，可以运行常用的linux命令
  / # ip a //查看容器的ip
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
  23: eth0@if24: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue
      link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
      inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
         valid_lft forever preferred_lft forever
  / #
  / # exit
$ docker ps  
```
很多场景，启动容器以后不期望立马就退出，而是希望它能后台一直提供服务。为了让大家直观的有一个感受，用一个while循环来保持命令一直不会中止
了解shell的同学应该对`while true ; do sleep 1000 ;done `并不陌生，下面来演示下效果
```
$ docker container run jishengqiu/firstimage  sh -c  "while true ; do sleep 1000 ;done "
//运行容器，命令窗口一直卡在这里，点什么也没有反应了，tip再开一个bash终端来stop掉这个容器
第二个终端窗口运行 $ docker ps //找到容器ID
第二个终端窗口运行 $ docker container stop 572edccc63c4 //关闭容器
```
此时，第一个终端窗口退出run，可以正常操作了。这样的效果是不友好的，我们期望运行容器并且不影响当前的操作。这个时候需要用到docker run -d 顾名思义daemon，后台运行容器。
下面演示下效果，操作如下
```
$ docker container run -d jishengqiu/firstimage  sh -c  "while true ; do sleep 1000 ;done "
0aaf51556af568ec293b1bcbd3a0644cd92cd039e493984c7b1bea50c2cd2fbf
$ docker ps   //显示有一个ID为0aaf51556af5状态Up的容器
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS               NAMES
0aaf51556af5        jishengqiu/firstimage   "sh -c 'while true ;…"   3 seconds ago       Up 2 seconds                            kind_ride
```


## 使用容器来搭建Web服务器
为了更好的演示访问容器，我选择了`nginx:1.16`镜像
```
$ docker pull nginx:1.16 //指定tag1.16，默认是latest
$ docker container run  -d  nginx:1.16 // 后台运行容器
$ docker ps 
 CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
 de64367e37cf        nginx:1.16          "nginx -g 'daemon of…"   34 seconds ago      Up 33 seconds       80/tcp              adoring_morse
//怎么访问这个容器呢？我们知道要访问一个服务，需要知道这个服务的ip和端口，那么怎么查看容器的ip呢？
//1） 方式1 进入容器内查看ip
$ docker run -it nginx:1.16 bash
  # ip a //这种方式受限于image是否包含了常用的linux工具包，比如运行的nginx容器，交互式运行以后，发现这些基本命令是用不了的
//2) 方式2 使用docker inspect查看容器的基本信息
$ docker inspect de64367e37cf
[
           ....... skip .......
           
            "SandboxKey": "/var/run/docker/netns/6e573254c86e",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "222455dbd6a3800e875f6191d46e921b09525def18bb6d9f13f13be13d36c260",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.3", //容器的ip地址
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:03",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "a064357504457e0744d2ee5ab17f93ea14fccf7dac5e3306515935b02acae799",
                    "EndpointID": "222455dbd6a3800e875f6191d46e921b09525def18bb6d9f13f13be13d36c260",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03",
                    "DriverOpts": null
                }
            }
        }
    }
]

现在容器的ip找到了172.17.0.3，端口默认是80，是否可以访问到nginx服务呢？关于docker的网络会在后面的文章中介绍，这些先演示下效果。

$ curl 172.17.0.3:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

说明，运行的nginx服务可以正常对外提供访问
```
## 容器生命周期
下面来介绍一下容器的生命周期，从创建、运行、休眠，直至销毁的整个过程。

先交互式启动一个`name`为`lifecycle`的容器，使用`--name`可以指定容器的名称，使用的镜像为`ubuntu`最新的版本，容器启动后执行bash命令，进入容器内部，创建一个文件，输入"hello docker"。
保存文件，`exit`退出容器。然后停止`lifecycle`容器，再重启这个容器，交互式进入容器，发现创建的`note.txt`文件依然存在，内容也没有丢失。

```
[vagrant@docker-host lifecycle]$ docker run -it --name lifecycle ubuntu  bash
root@eac01c285982:/#
root@eac01c285982:/# touch note.txt
root@eac01c285982:/# echo "hello docker"> note.txt
root@eac01c285982:/# cat note.txt
hello docker
//ctrl+PQ退出容器内部

[vagrant@docker-host lifecycle]$ docker stop lifecycle  //停止容器
[vagrant@docker-host lifecycle]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[vagrant@docker-host lifecycle]$ docker ps -a  //-a表示列出所有的容器
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                          PORTS               NAMES
eac01c285982        ubuntu              "bash"              13 minutes ago      Exited (0) About a minute ago                       lifecycle

[vagrant@docker-host lifecycle]$ docker start lifecycle   //重启容器
[vagrant@docker-host lifecycle]$ docker exec -it lifecycle bash  //交互式进入lifecycle容器，执行bash命令
root@eac01c285982:/# ll  
-rw-r--r--.   1 root root   13 Feb 24 08:53 note.txt  // 文件依然存在
root@eac01c285982:/# cat note.txt   //说明使用docker container stop停止容器运行并不会损毁容器或者其中的数据
hello docker
```
尽管上面的示例阐明了容器的持久化特性，后面我们还是介绍`mount`挂载卷（`volume`）才是在容器中存储持久化数据的首选方式。

彻底删除容器，`docker container rm `命令后面添加 `-f` 参数来一次性删除运行中的容器是可行的。但是，删除容器的*最佳方式*还是分两步，先停止容器然后删除。这样可以给容器中运行的应用/进程一个停止运行并清理残留数据的机会。
```
[vagrant@docker-host lifecycle]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
eac01c285982        ubuntu              "bash"              24 minutes ago      Up 10 minutes                           lifecycle

[vagrant@docker-host lifecycle]$
[vagrant@docker-host lifecycle]$ docker container stop lifecycle
lifecycle
[vagrant@docker-host lifecycle]$
[vagrant@docker-host lifecycle]$
[vagrant@docker-host lifecycle]$ docker container rm lifecycle
lifecycle
[vagrant@docker-host lifecycle]$docker ps -a 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

```

使用docker container rm -f 会一次性删除正在运行容器，这种方式是不优雅的

总结一下容器的生命周期。根据需要多次停止、启动、暂停以及重启容器，并且这些操作执行得很快。

容器及其数据是安全的。直至明确删除容器前，容器都不会丢弃其中的数据。就算容器被删除了，如果将容器数据存储挂载本地宿主机的磁盘上的卷中，数据也会被保存下来。当我们再次创建新的容器，使用-volume参数指定挂载点依然可以恢复删除容器之前的数据，后面的章节我们会演示这种效果


## 小技巧

### 快速清理退出的容器
`docker container rm $(docker container ls -aq)`如果将 $(docker container ls -aq) 作为参数传递给 docker container rm 命令，等价于将系统中每个容器的 ID 传给该命令。-f 标识表示强制执行，所以即使是处于运行状态的容器也会被删除。谨慎使用。
演示效果如下
```
[vagrant@docker-host ~]$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
3678ced351e4        ubuntu              "/bin/bash"         25 seconds ago      Exited (0) 25 seconds ago                       test2
bd96ff7d0ad4        ubuntu              "/bin/bash"         29 seconds ago      Exited (0) 29 seconds ago                       test1
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$ docker container rm $(docker container ls -aq)
3678ced351e4
bd96ff7d0ad4
[vagrant@docker-host ~]$ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

### 利用重启策略进行容器的自我修复
容器支持的重启策略包括` always、unless-stopped` 和 `on-failed`。
+ `always` 策略是一种简单的方式。除非容器被明确停止，比如通过 `docker container stop `命令，否则该策略会一直尝试重启处于停止状态的容器。
+ `always` 和 `unless-stopped` 的最大区别，就是那些指定了 `--restart unless-stopped `并处于 `Stopped (Exited) `状态的容器，不会在 `Docker daemon` 重启的时候被重启。
+ `on-failure` 策略会在退出容器并且返回值不是 0 的时候，重启容器。

后面的章节会有专题来演示效果。

*如有错误，请指正*