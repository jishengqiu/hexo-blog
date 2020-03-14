---
title: docker实战系列七之数据持久化
date: 2020-03-03 15:15:58
tags: [docker, 文件, 数据持久化, mount]
---


## 数据持久化使用场景

前面的文章讲过docker容器的生命周期，当容器删除掉以后，容器内部的文件数据会随之删除。在某些场景下，需要即使容器被意外删除了，数据也不会丢失，当重新启动新容器时，可以灌入数据继续提供服务，典型的应用有数据库服务。



## Data Volume

回归mysql官方提供的[Dockerfile](https://github.com/docker-library/mysql/blob/master/5.7/Dockerfile)，如下图所示，就指定了`VOLUME /var/lib/mysql`，即mysql容器创建后数据会写入宿主机的`/var/lib/mysql`目录保存，容器即使本删除了，宿主机的`Volume`依然存在，可以创建新的容器恢复数据

下面举个例子，实例一演示数据持久化Volume不会随着容器删除而删除
第一步，创建mysql1的容器
```
[vagrant@docker-host ~]$ docker run --name mysql1 -d -e MYSQL_ROOT_PASSWORD=root mysql:5.7
8df818507aff66886d14e44a7e9260a71b0b2d24f3cfe72810fab3bf84c861d5
[vagrant@docker-host ~]$
[vagrant@docker-host ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                 NAMES
8df818507aff        mysql:5.7           "docker-entrypoint.s…"   4 seconds ago       Up 4 seconds        3306/tcp, 33060/tcp   mysql1

```
第二步，查看docker volume
```
[vagrant@docker-host ~]$ docker volume ls
DRIVER              VOLUME NAME
local               369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1
[vagrant@docker-host ~]$ docker volume inspect 369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1
[
    {
        "CreatedAt": "2020-03-03T09:37:03Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1/_data",
        "Name": "369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1",
        "Options": null,
        "Scope": "local"
    }
]
```

第三步，删除容器mysql1，然后确认volume是否删除
```
[vagrant@docker-host ~]$ docker container stop mysql1
mysql1
[vagrant@docker-host ~]$ docker container rm mysql1

[vagrant@docker-host ~]$ docker volume ls
DRIVER              VOLUME NAME
local               369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1
[vagrant@docker-host ~]$ docker volume inspect 369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1
[
    {
        "CreatedAt": "2020-03-03T09:37:03Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1/_data",
        "Name": "369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1",
        "Options": null,
        "Scope": "local"
    }
]

```
说明volume并没有随着容器的删除而删除，volume的名称不友好，可以再创建容器时通过-v参数来指定


实例二，演示为容器指定Volume

第一步，创建容器,并使用-v参数指定volume
```
[vagrant@docker-host ~]$ docker run --name mysql2 -v mysql:/var/lib/mysql -d -e MYSQL_ROOT_PASSWORD=root mysql:5.7
a64471fef31123dbae1fd66205a5ea55a5faa9ca31c7a6dfb0cc3cd1010af661
[vagrant@docker-host ~]$ docker volume ls
DRIVER              VOLUME NAME
local               369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1
local               mysql
```

第二步，进入容器内部访问mysql2
```
[vagrant@docker-host ~]$ docker exec -it mysql2 mysql -proot
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.7.29 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.02 sec)

```

第三步，创建一个数据库persistent
```
mysql> create database persistent;
Query OK, 1 row affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| persistent         |
| sys                |
+--------------------+
5 rows in set (0.01 sec)
```
第四步，删除容器mysql2
```
mysql> exit;
Bye
[vagrant@docker-host ~]$ docker container stop mysql2
mysql1
[vagrant@docker-host ~]$ docker container rm mysql2
mysql1
[vagrant@docker-host ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```


第五步，重新创建一个容器mysql3，通过-v参数指定volume
```
[vagrant@docker-host ~]$  docker run --name mysql3 -v mysql:/var/lib/mysql -d -e MYSQL_ROOT_PASSWORD=root mysql:5.7
[vagrant@docker-host ~]$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                 NAMES
34bb95b24a37        mysql:5.7           "docker-entrypoint.s…"   4 seconds ago       Up 3 seconds        3306/tcp, 33060/tcp   mysql3

[vagrant@docker-host ~]$ docker volume ls
DRIVER              VOLUME NAME
local               369e383578406e084269c649fcea1146b203b94f78183161ea1874bd4607d2a1
local               mysql
```

第六步，进入容器内部访问mysql3

```
[vagrant@docker-host ~]$ docker exec -it mysql3 mysql -proot
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.29 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| persistent         |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql>

```
可以看到，persistent数据库依然存在，说明通过-v参数指定Volume可以达到数据持久化。


##Bind Mounting
上面讲的Data Volume是指将容器产生的数据持久化。在某些场景下，需要将本地的文件同步到容器中，容器在运行过程中能够读取到，从而使本地的文件内容跟容器中的文件一致。
第一步，利用`nginx`制作一个自定义镜像`my-nginx`
```
[vagrant@docker-host ~]$mkdir my-nginx
[vagrant@docker-host ~]cd  my-nginx
[vagrant@docker-host my-nginx]touch Dockerfile  ##创建Dockerfile

##Dockerfile的文件内容

FROM nginx:1.16
WORKDIR /usr/share/nginx/html
COPY index.html index.html

[vagrant@docker-host my-nginx]touch index.html  ##创建nginx的欢迎页面，用于替换默认的欢迎页面

##index的文件内容

hello docker!
```

第二步，创建启动容器web1，使用-p参数将容器内部80端口映射到宿主机`docker-host`的8011端口，这样就可以不用通过容器的ip加端口访问了
```
[vagrant@docker-host my-nginx]$ docker run --name web1 -p 8011:80 -d jishengqiu/my-nginx
6f841a8653c445ec0679c92c19a4f591aaa80efe31ad6fe6cb3a19d7011d31ef
[vagrant@docker-host my-nginx]$
[vagrant@docker-host my-nginx]$
[vagrant@docker-host my-nginx]$
[vagrant@docker-host my-nginx]$
[vagrant@docker-host my-nginx]$ docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                  NAMES
6f841a8653c4        jishengqiu/my-nginx   "nginx -g 'daemon of…"   5 seconds ago       Up 4 seconds        0.0.0.0:8011->80/tcp   web1
[vagrant@docker-host my-nginx]$ curl 127.0.0.1:8011  
hello docker
```
说明，本地的index.html文件，通过COPY的方式给到容器内部使用，但这并没有达到想要的效果，当本地的`index.html`文件改动，容器也是实时感知这样的变化呢？先停掉并删掉容器web1
```
[vagrant@docker-host my-nginx]$ docker rm -f web1
web1
[vagrant@docker-host my-nginx]$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[vagrant@docker-host my-nginx]$
```
第三步，创建运行容器web2,使用-v参数指定同步路径
```
[vagrant@docker-host my-nginx]$ docker run --name web2 -v $(pwd):/usr/share/nginx/html -p 8011:80 -d jishengqiu/my-nginx
a93c961303a63e5eb543a4a2272870acd6259dd9850827e16958039818b8c8d5
[vagrant@docker-host my-nginx]$ curl 127.0.0.1:8011
hello docker

```
第三步,修改本地宿主机的`index.html`，再次访问`nginx`，观察文件内容是否同步

```
[vagrant@docker-host my-nginx]$ cat index.html
hello docker
武汉加油，中国加油！
[vagrant@docker-host my-nginx]$ curl 127.0.0.1:8011
hello docker
武汉加油，中国加油！
[vagrant@docker-host my-nginx]$

```

说明提供-v参数，来绑定同步路径，可以达到容器能够实时获取到本地的改动。

## 小结

提供上面2个小节的演示数据持久化，docker容器为开发者提供很多的便利，比如开发程序员本地开发写代码自测，然后将代码同步给容器内部，模拟生产环境测试，返回预期效果给开发者。



*如有错误，请指正*
