---
title: docker实战系列五之Dockerfile容器最佳实践
date: 2020-02-25 10:50:21
tags: [docker, 容器, Dockerfile]
---


## `Dockerfile`指令详解
`Dockerfile`中包括`FROM、MAINTAINER、RUN、CMD、EXPOSE、ENV、ADD、COPY、ENTRYPOINT、VOLUME、USER、WORKDIR、ONBUILD`等13个指令。下面一一讲解。

+ `FROM`
格式为`FROM image`或`FROM image:tag`，并且`Dockerfile`中第一条指令必须是FROM指令，且在同一个Dockerfile中创建多个镜像时，可以使用多个`FROM`指令。

+ `MAINTAINER`
格式为`MAINTAINER user_name user_email`，指定维护者信息

+ `RUN`
格式为RUN command或 RUN ["EXECUTABLE","PARAM1","PARAM2".....]，前者在shell终端中运行命令，`/bin/sh -c command`，例如：`/bin/sh -c "echo hello"`;后者使用exec执行，指定其他运行终端使用RUN["/bin/bash","-c","echo hello"]。每条RUN指令将当前的镜像基础上执行指令，并提交为新的镜像，命令较长的时候可以使用`\`来换行。

+ `CMD`
支持三种格式：
`CMD` ["executable","param1","param2"]，使用`exec`执行，这是推荐的方式。
`CMD` command param1 param2 在`/bin/sh`中执行。
`CMD` ["param1","param2"] 提供给`ENTERYPOINT`的默认参数。
`CMD`用于指定容器启动时执行的命令，每个`Dockerfile`只能有一个`CMD`命令，多个`CMD`命令只执行最后一个。若容器启动时指定了运行的命令，则会覆盖掉`CMD`中指定的命令。
  
+ `EXPOSE`
格式为 `EXPOSE port [port2,port3,...]`，例如`EXPOSE 80`这条指令告诉`Docker`服务器暴露80端口，供容器外部连接使用。
在启动容器的使用使用-P，Docker会自动分配一个端口和转发指定的端口，使用-p可以具体指定使用哪个本地的端口来映射对外开放的端口。

+ `ENV`
格式为：`EVN key value` 。用于指定环境变量，这些环境变量，后续可以被`RUN`指令使用，容器运行起来之后，也可以在容器中获取这些环境变量。
例如
`ENV word hello`
`RUN echo $word`

+ `ADD`
格式：ADD src dest
该命令将复制指定本地目录中的文件到容器中的dest中，src可以是是一个绝对路径，也可以是一个URL或一个tar文件，tar文件会自动解压为目录。

+ `COPY`
格式为：`COPY src desc`
复制本地主机`src`目录或文件到容器的`desc`目录，`desc`不存在时会自动创建。

+ `ENTRYPOINT`
格式有两种：
`ENTRYPOINT` ["executable","param1","param2"]
`ENTRYPOINT command param1,param2` 会在shell中执行。
用于配置容器启动后执行的命令，这些命令不能被`docker run`提供的参数覆盖。和`CMD`一样，每个`Dockerfile`中只能有一个`ENTRYPOINT`，当有多个时最后一个生效。

+ `VOLUME`
格式为 `VOLUME` ["/data"]
作用是创建在本地主机或其他容器可以挂载的数据卷，用来存放数据。

+ `USER`
格式为：`USER username`
指定容器运行时的用户名或UID，后续的RUN也会使用指定的用户。要临时使用管理员权限可以使用`sudo`。在USER命令之前可以使用RUN命令创建需要的用户。
例如：`RUN groupadd -r docker && useradd -r -g docker docker`

+ `WORKDIR`
格式： `WORKDIR /path`
为后续的`RUN CMD ENTRYPOINT`指定配置工作目录，可以使用多个`WORKDIR`指令，若后续指令用得是相对路径，则会基于之前的命令指定路径。

+ `ONBUILD`
格式`ONBUILD [INSTRUCTION]`
该配置指定当所创建的镜像作为其他新建镜像的基础镜像时所执行的指令。
例如下面的`Dockerfile`创建了镜像A：
`ONBUILD ADD . /app`
`ONBUILD RUN python app.py`

则基于镜像A创建新的镜像时，新的Dockerfile中使用from A 指定基镜像时，会自动执行ONBBUILD指令内容，等价于在新的要构建镜像的Dockerfile中增加了两条指令：
```
FROM A
ADD ./app
RUN python app.py
```

## 使用Dockerfile构建镜像
创建好`Dockerfile`之后，通过`docker build`命令来创建镜像，该命令首先会上传Dockerfile文件给Docker服务器端，服务器端将逐行执行`Dockerfile`中定义的指令。
通常建议放置`Dockerfile`的目录为空目录。另外可以在目录下创建`.dockerignore`文件，让`Docker`忽略路径下的文件和目录，这一点与`Git`中的配置很相似。

通过 `-t` 指定镜像的标签信息，例如：`docker build -t jishengqiu/firstimage . `其中"."指定的是Dockerfile所在的路径
                                                                                                                                                                                    
## 小试牛刀
回顾我们前面的博客，自己构建的第一个镜像`firstimage`，打开Dockerfile文件，现只使用FROM，CMD2个指令

```
$ cd myfirstimage
$  vim Dockefile
   FROM busybox
   CMD ["echo","hello docker"]
```

有了上面的理论基础，演示如何构建一个基于C的hello world镜像，准备工作

```
$ mkdir helloworld // 创建工作目录
$ cd helloworld    // 进入工作目录
$ touch Dockerfile // 创建Dockerfile文件

```

> step1 在创建的centos机器上，安装 gcc

查看有没有安装

```
[vagrant@docker-host helloworld]$ rpm -qa gcc glibc-static
glibc-static-2.17-292.el7.x86_64
gcc-4.8.5-39.el7.x86_64

```

没有的话，进行安装即可

```
[vagrant@docker-host helloworld]$ yum install gcc glibc-static 


```
> step2 编辑 C 源代码文件

```
[vagrant@docker-host helloworld] cat hello.c
#include <stdio.h>

int main()
{
   printf("Hello Docker! \n");
   return 0;
}

```
> step3 将源代码文件编译为可执行的二进制文件
```
[vagrant@docker-host helloworld]$  gcc --static hello.c -o hello

```

编译好后，测试一下
```
[vagrant@docker-host helloworld]$ ll
total 852
-rw-rw-r--. 1 vagrant vagrant     41 Feb 25 08:22 Dockerfile  
-rwxrwxr-x. 1 vagrant vagrant 861112 Feb 25 09:08 hello     // 二进制可执行文件
-rw-rw-r--. 1 vagrant vagrant     79 Feb 25 09:08 hello.c  // 源码文件
[vagrant@docker-host helloworld]$ ./hello
Hello Docker!

```
> step4 编写Dockerfile

```
[vagrant@docker-host helloworld]$ cat Dockerfile
FROM scratch   //Docker提供的一个特殊镜像即scratch空镜像，空间非常小，这镜像是虚拟的，表示不需要任何镜像作为base image
MAINTAINER JasonChiu  jishengqiu@github.com  //指定维护者的信息
ADD hello /app/  //将当前目录的hello  拷贝到容器的根目录下app文件夹 *这里一定要注意app后面的 / 不能省略*
WORKDIR /app   //指定工作目录
CMD ["echo","first cmd"]   //测试cmd覆盖
CMD ["echo","second cmd"]  //测试cmd覆盖
CMD ["./hello"]
```

> step5 构建镜像

```
[vagrant@docker-host helloworld]$ docker build -t jishengqiu/helloworld .
Sending build context to Docker daemon  864.8kB
Step 1/7 : FROM scratch
 --->
Step 2/7 : MAINTAINER JasonChiu  jishengqiu@github.com
 ---> Using cache
 ---> e6d4a7b7fa2f
Step 3/7 : ADD hello /app/
 ---> 7c6626cabbd0
Step 4/7 : WORKDIR app
 ---> Running in 3de66973208c
Removing intermediate container 3de66973208c
 ---> ee4df475c200
Step 5/7 : CMD ["echo","first cmd"]
 ---> Running in 590eb0c13cda
Removing intermediate container 590eb0c13cda
 ---> 2c5766a4458a
Step 6/7 : CMD ["echo","second cmd"]
 ---> Running in 2ffbb25e5b96
Removing intermediate container 2ffbb25e5b96
 ---> c45425474542
Step 7/7 : CMD ["./hello"]
 ---> Running in ea42145e101e
Removing intermediate container ea42145e101e
 ---> 0ff1ba535f03
Successfully built 0ff1ba535f03
Successfully tagged jishengqiu/helloworld:latest
[vagrant@docker-host helloworld]$
```

> step6 查看本地镜像仓库验证

```
[vagrant@docker-host helloworld]$ docker image ls
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
jishengqiu/helloworld   latest              0ff1ba535f03        10 minutes ago      861kB
```
![dk1](dk1.jpg)

空间是非常小的，基本是等于可执行文件hello的大小


> step7 利用新的镜像运行一个容器

```
[vagrant@docker-host helloworld]$ docker run  jishengqiu/helloworld
Hello Docker!  //只输出了Hello Docker,说明first cmd和second cmd都被覆盖了
```


## 小结
本章节主要介绍了`docker`的构建镜像的最佳实践`Dockerfile`,这样的话镜像的构建会变的透明化，对镜像的维护起来也更加简单，只修改这个文件即可。同时分享也更加简单快捷，因为只要分享这个文件即可。读者如果有兴趣，可以构建`java，python`版本的`hello world`镜像
Dockerfile相关的语法和命令介绍，读者可以参考官方文档进一步学习和深入，感兴趣还可以进一步研究官方的镜像Dockerfile的源码，比如[MySql5.7](https://github.com/docker-library/mysql/blob/master/5.7/Dockerfile)的`Dockerfile`。

*参考docker官网资料，如有错误，请指正*


