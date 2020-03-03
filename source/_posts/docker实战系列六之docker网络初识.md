---
title: docker实战系列六之docker网络
date: 2020-02-26 11:12:15
tags: [docker, 网络, network]
---



## linux网络知识

**终端访问服务器，网络传输是基于数据包来传输的**
![network](bb.png)

**数据包是如何进行传输的，这需要用到网络分层的概念**
![network](10.png)

**网络路由**
![network](11.png)
网络路由就像地图搜索目的地，去往服务器的路径很多，路由会帮助选择一条最优的路径来传输
**NAT地址转换**
![network](12.png)
**linux 网络命名空间 Network namespaces**
Linux命名空间是一个相对较新的内核功能，对于实现容器至关重要。 命名空间将全局系统资源包装到一个抽象中，该抽象只会与命名空间中的进程绑定，从而提供资源隔离。
 
命名空间和`cgroups`是软件集装箱化（`Docker`）的大部分新趋势的主要内核技术之一。 简单来说，`cgroups`是一种计量和限制机制，它们控制您可以使用多少系统资源（CPU，内存）。 另一方面，命名空间限制了您可以看到的内容。 由于命名空间进程有自己的系统资源视图。 Linux内核提供了6种类型的命名空间：pid，net，mnt，uts，ipc和user。 例如，pid命名空间中的进程只能看到同一命名空间中的进程。 感谢mnt命名空间，可以将进程附加到自己的文件系统（如`chroot`）。 在本文中，我仅关注网络命名空间。

什么是网络命名空间，直观来说，当运行命令 `ip a`看到的网络接口列表，ip，mac等信息就是一个独立的网络命名空间，如下图所示
![network](13.png)

下面用一个实际的例子来演示

第一步，使用` ip netns`相关命令来创建2个`linux network namespaces`，分别为`test1`，`test2`
```
[vagrant@docker-host ~]$ sudo ip netns ls
[vagrant@docker-host ~]$ sudo ip netns add test1
[vagrant@docker-host ~]$ sudo ip netns add test2
[vagrant@docker-host ~]$ sudo ip netns ls
test2
test1

[vagrant@docker-host ~]$ sudo ip netns exec test1 ip a           ##在test1网络空间，执行ip a命令，只有一个回环接口lo，并且状态为DOWN
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

[vagrant@docker-host ~]$
[vagrant@docker-host ~]$ sudo ip netns exec test2 ip a          ##在test2网络空间，执行ip a命令，只有一个回环接口lo，并且状态为DOWN
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    

[vagrant@docker-host ~]$ip a                                    ##在宿主机本地网络空间，执行ip a命令，发现有很多接口，重点关注docker0接口
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:8a:fe:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 53137sec preferred_lft 53137sec
    inet6 fe80::5054:ff:fe8a:fee6/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:59:96:0d brd ff:ff:ff:ff:ff:ff
    inet 192.168.205.30/24 brd 192.168.205.255 scope global noprefixroute eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe59:960d/64 scope link
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:30:ec:77:9e brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:30ff:feec:779e/64 scope link
       valid_lft forever preferred_lft forever
```

这个时候，创建的2个`namespaces`的状态是`DOWN`的，并且也没有具体的IP地址

第二步，将test1,test2的lo端口up起来，状态为`UNKNOWN`是由于我们本地宿主机的lo接口也是`UNKNOWN`
```
[vagrant@docker-host ~]$ sudo ip netns exec test1 ip link set dev lo up
[vagrant@docker-host ~]$ sudo ip netns exec test2 ip link set dev lo up
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$ sudo ip netns exec test1 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
[vagrant@docker-host ~]$ sudo ip netns exec test2 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
[vagrant@docker-host ~]$

```


第三步，创建一对`veth`的接口分别为`veth-test1`,`veth-test2`，2个网络空间需要连通可以理解成2个电脑需要连通就需要有一个网线来连接这2台设备。

```
[vagrant@docker-host ~]$sudo ip link add veth-test1 type veth peer name veth-test2
[vagrant@docker-host ~]

```

第四步，分别将`veth-test1`,`veth-test2`分配给`test1`，`test2`网络空间

```
[vagrant@docker-host ~]$ sudo ip link set veth-test1 netns test1
[vagrant@docker-host ~]$ sudo ip link set veth-test2 netns test2
[vagrant@docker-host ~]$ sudo ip netns exec test1 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
102: veth-test1@if101: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 42:c3:a2:8c:c8:54 brd ff:ff:ff:ff:ff:ff link-netnsid 1
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$ sudo ip netns exec test2 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
101: veth-test2@if102: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 46:e6:6e:38:96:7d brd ff:ff:ff:ff:ff:ff link-netnsid 0

```
第五步，分别给`veth-test1`,`veth-test2`分配`IP`地址，分配完成后，发现这2个端口的状态为`DOWN`
```
[vagrant@docker-host ~]$ sudo ip netns exec test1 ip addr add 192.168.1.10/24 dev veth-test1
[vagrant@docker-host ~]$ sudo ip netns exec test2 ip addr add 192.168.1.11/24 dev veth-test2

[vagrant@docker-host ~]$ sudo ip netns exec test1 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
102: veth-test1@if101: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 42:c3:a2:8c:c8:54 brd ff:ff:ff:ff:ff:ff link-netnsid 1

[vagrant@docker-host ~]$ sudo ip netns exec test2 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
101: veth-test2@if102: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 46:e6:6e:38:96:7d brd ff:ff:ff:ff:ff:ff link-netnsid 0

```

第六步，分别将`veth-test1`,`veth-test2`端口`up`起来
```
[vagrant@docker-host ~]$ sudo ip netns exec test1 ip link set dev veth-test1 up
[vagrant@docker-host ~]$ sudo ip netns exec test2 ip link set dev veth-test2 up
[vagrant@docker-host ~]$ sudo ip netns exec test1 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
102: veth-test1@if101: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 42:c3:a2:8c:c8:54 brd ff:ff:ff:ff:ff:ff link-netnsid 1
[vagrant@docker-host ~]$ sudo ip netns exec test2 ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
101: veth-test2@if102: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 46:e6:6e:38:96:7d brd ff:ff:ff:ff:ff:ff link-netnsid 0
```
第七步，查看`test1`,`test2`的`ip`地址以及端口状态
```
[vagrant@docker-host ~]$ sudo ip netns exec test1 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
102: veth-test1@if101: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 42:c3:a2:8c:c8:54 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 192.168.1.10/24 scope global veth-test1
       valid_lft forever preferred_lft forever
    inet6 fe80::40c3:a2ff:fe8c:c854/64 scope link
       valid_lft forever preferred_lft forever
       
[vagrant@docker-host ~]$ sudo ip netns exec test2 ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
101: veth-test2@if102: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 46:e6:6e:38:96:7d brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.1.11/24 scope global veth-test2
       valid_lft forever preferred_lft forever
    inet6 fe80::44e6:6eff:fe38:967d/64 scope link
       valid_lft forever preferred_lft forever
```

第八步，测试`test1`,`test2`的是否互通
```
[vagrant@docker-host ~]$ sudo ip netns exec test1 ping 192.168.1.11
PING 192.168.1.11 (192.168.1.11) 56(84) bytes of data.
64 bytes from 192.168.1.11: icmp_seq=1 ttl=64 time=0.053 ms
64 bytes from 192.168.1.11: icmp_seq=2 ttl=64 time=0.046 ms
^C
--- 192.168.1.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.046/0.049/0.053/0.007 ms
[vagrant@docker-host ~]$ sudo ip netns exec test2 ping 192.168.1.10
PING 192.168.1.10 (192.168.1.10) 56(84) bytes of data.
64 bytes from 192.168.1.10: icmp_seq=1 ttl=64 time=0.042 ms
64 bytes from 192.168.1.10: icmp_seq=2 ttl=64 time=0.081 ms
64 bytes from 192.168.1.10: icmp_seq=3 ttl=64 time=0.075 ms
64 bytes from 192.168.1.10: icmp_seq=4 ttl=64 time=0.098 ms
^C
--- 192.168.1.10 ping statistics ---
59 packets transmitted, 59 received, 0% packet loss, time 58083ms
rtt min/avg/max/mdev = 0.033/0.059/0.138/0.025 ms
[vagrant@docker-host ~]$

```
通过这个小栗子，可以了解`linux`的网络命名空间，下文关于容器的网络实验，本质上来的原理就是利于了`linux`的网络命名空间，也就是理解了上面的实验过程和内容，对阅读下面容器的网络是非常有帮助的

## docker网络驱动

### docker自带网络模式

+ `bridge`: 默认的网络驱动模式，此模式会为每一个容器分配` Network namespace`、设置 IP 等，并将一个主机上的 Docker 容器连接到一个虚拟网桥上。当 `Docker server` 启动时，会在主机上创建一个名为` docker0 `的虚拟网桥，此主机上启动的 `Docker` 容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。接下来就要为容器分配 `IP` 了，`Docker` 会从 `RFC1918` 所定义的私有 `IP` 网段中，选择一个和宿主机不同的`IP`地址和子网分配给 `docker0`，连接到 `docker0` 的容器就从这个子网中选择一个未占用的`IP` 使用。如一般 `Docker` 会使用 `172.17.0.0/16` 这个网段，并将 `172.17.42.1/16` 分配给 `docker0 `网桥（在主机上使用 `ifconfig` 命令是可以看到 `docker0 `的，可以认为它是网桥的管理接口，在宿主机上作为一块虚拟网卡使用）

+ `host` : 如果启动容器的时候使用 host 模式，那么这个容器将不会获得一个独立的 `Network Namespace`，而是和宿主机共用一个 `Network Namespace`。容器将不会虚拟出自己的网卡，配置自己的` IP` 等，而是使用宿主机的 `IP `和端口。
  
+ `none` : 这个模式和前两个不同。在这种模式下，`Docker `容器拥有自己的 `Network Namespace`，但是，并不为 `Docker`容器进行任何网络配置。也就是说，这个 Docker 容器没有网卡、`IP`、路由等信息。需要我们自己为 `Docker `容器添加网卡、配置 `IP `等。

### 自定义网络模式

+ `bridge`: 基本上跟默认的`bridge`网络驱动类似，但新增了一些其他功能

+ `overlay`: 覆盖网络将多个`Docker`守护程序连接在一起，并使群集服务能够相互通信。您还可以使用覆盖网络来促进群集服务和独立容器之间或不同`Docker`守护程序上的两个独立容器之间的通信。这种策略消除了在这些容器之间进行操作系统级路由的需要。

+ `macvlan`: `Macvlan`网络允许您为容器分配`MAC`地址，使其在网络上显示为物理设备。`Docker`守护程序通过其`MAC`地址将流量路由到容器。`macvlan` 在处理希望直接连接到物理网络而不是通过`Docker`主机的网络堆栈进行路由的旧应用程序时，使用驱动程序有时是最佳选择

+ 第三方网络插件

`overlay`与`maclan`主要是用于跨主机网络


## docker network常用命令介绍
```
docker network create   // 创建网络资源，默认driver为bridge

docker network connect  // 连接网络资源

docker network ls      // 列出当前的网络资源

docker network rm      // 删除网络资源

docker network disconnect // 断开网络链接

docker network inspect    //查看网络资源的详细信息

```

## bridge网络

本地`bridge`网络，示意图如下

![bridge3](3.png)

根据上图的场景，下面演示实验效果，创建2个容器`test1, test2`，默认会链接到本地宿主机的bridge网络上，具体来说是连接到docker0网络上
![bridge](1.png)



说明test1,test2互相是可以ping通的，那么怎么证明他们就是链接到了`docker0`,我们借助`brctl show`命令来证明，列出当前主机网桥
```
$ sudo brctl show # brctl 工具依赖 bridge-utils 软件包 bridge name bridge id STP enabled interfaces
docker0 8000.000000000000 n
```

![bridge2](2.png)

以上, `docker0` 扮演着`test1，test2`这2个容器的虚拟接口`veth6504554，vetha1b22b4 interface`桥接的角色。当然也可以使用`docker`自带的网络相关命令来查看
![bridge4](4.png)



## host网络
host网络模式需要在容器创建时指定–network=host，表示与宿主机共享本地网络。下面举个实例来说明，如下图

![bridge4](5.png)
图中关键步骤的操作命令如下
```
[vagrant@docker-host ~]$ docker container  run --name=test2  --network host -d  busybox sh -c "while true ; do sleep 1000 ;done "^C
[vagrant@docker-host ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
a60832b76c73        busybox             "sh -c 'while true ;…"   17 seconds ago      Up 17 seconds                           test2
[vagrant@docker-host ~]$ ip a   //查看宿主机的ip
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$ docker exec -it test2  ip a   //查看容器test2的ip，也可以使用docker container inspect查看，也可以使用 docker exec -it test2  sh进入容器内，执行ip a来查看，这里将步骤简化
```

host网络有一个很大的缺点，在于跟本机共享网络资源，也就意味这端口共享的问题，比如有些场景需要启动多个`nginx`容器，那这样就没有办法了，只能将容器的端口一一进行调整。

## none网络
none网络可以理解为一个孤立的网络，它不会有的ip地址等网络相关配置信息

![bridge6](6.png)
图中关键步骤的操作命令如下
```
[vagrant@docker-host ~]$ docker container  run --name=test3  --network none -d  busybox sh -c "while true ; do sleep 1000 ;done "
7ccad41fcd5b559ddb7c277730fbd7ac03687a5f256b78a41817c1d6d42c3fec
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$ docker exec -it test3  ip a
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$ docker network inspect none

```
## 自定义网络模式
### 自定义bridge
创建自定义`bridge`
```
 [vagrant@docker-host ~]$ docker network create my_bridge  //默认的driver
 [vagrant@docker-host ~]$ docker network ls
 NETWORK ID          NAME                DRIVER              SCOPE
 a06435750445        bridge              bridge              local
 e764a36b467f        host                host                local
 3cd0c7bc90e0        my_bridge           bridge              local
 bbd76da25d88        none                null                local

```

创建2个容器，使用`busybox`镜像，指定network为刚才创建的bridge
```
[vagrant@docker-host ~]$ docker container  run --name=test1  --network my_bridge -d  busybox sh -c "while true ; do sleep 1000 ;done "
d444b86c0f8a6f6a1c0fd2b56a1a4edfa691bb128a0d6ca3c9b9812cbd938ed7
[vagrant@docker-host ~]$ docker container  run --name=test2  --network my_bridge -d  busybox sh -c "while true ; do sleep 1000 ;done "
71574b265cc215647b8a9e6a5a75375eba68c1923341a16031d4ae441e2ff7c4

```

查看my_bridge网络的详细信息
```
[vagrant@docker-host ~]$ docker network inspect my_bridge
[
    {
        "Name": "my_bridge",
        "Id": "3cd0c7bc90e0cc8a94f05d0b15f67e39a2d390bf2342d701180648fbc1a4cf52",
        "Created": "2020-02-28T08:25:18.53631711Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "71574b265cc215647b8a9e6a5a75375eba68c1923341a16031d4ae441e2ff7c4": {
                "Name": "test2",
                "EndpointID": "1140c5896d84b460e910b23a1df45d407559532ad94dc9164b8588b0a479d6f7",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "d444b86c0f8a6f6a1c0fd2b56a1a4edfa691bb128a0d6ca3c9b9812cbd938ed7": {
                "Name": "test1",
                "EndpointID": "2f6b60481762cb91ff765f586b571e69dd11c6d5e6d49c42acedf3d37485ecff",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

```

查看容器`test1, test2`是否能够`ping`通
![bridge8](8.png)
说明`test1, test2`之间不仅使用`ip`地址可以`ping`通，`ping test1, test2`的名称也可以`ping`通，这是`docker`自带默认的`bridge`不具有的功能。


现在再创建一个自定义bridge网络
```
 [vagrant@docker-host ~]$ docker network create other_bridge  //默认的driver
 [vagrant@docker-host ~]$ docker network ls
 NETWORK ID          NAME                DRIVER              SCOPE
 a06435750445        bridge              bridge              local
 e764a36b467f        host                host                local
 3cd0c7bc90e0        my_bridge           bridge              local
 bbd76da25d88        none                null                local
 6ebc3d61504d        other_bridge        bridge              local

```
再新建一个容器test3，并且指定network到other_bridge
```
[vagrant@docker-host ~]$ docker container  run --name=test3  --network other_bridge -d  busybox sh -c "while true ; do sleep 1000 ;done "
75a3e813735dcd2f051f3b2c46c203ceefb12cc79d2e605bf095726a70892633
[vagrant@docker-host ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
75a3e813735d        busybox             "sh -c 'while true ;…"   9 seconds ago       Up 8 seconds                            test3
71574b265cc2        busybox             "sh -c 'while true ;…"   30 minutes ago      Up 30 minutes                           test2
d444b86c0f8a        busybox             "sh -c 'while true ;…"   30 minutes ago      Up 30 minutes                           test1
[vagrant@docker-host ~]$

```
测试下test1是否可以ping通test3的ip或者名称
![bridge9](9.png)
根据具体的ip地址也可以看出来，test1,test2在172.18.xx.xx上，test3在172.19.xxx.xx上，所以他们之间是不可达的。


### overlay网络
`overlay`网络实现原理需要有`vxlan`隧道的理论知识，比较复杂，原理篇后面再补充上来。
多机网络，下一篇博客专题介绍

*参考docker官网资料，如有错误，请指正*