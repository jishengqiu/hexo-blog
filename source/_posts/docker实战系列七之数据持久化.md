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
可以看到，persistent数据库依然存在，说明数据持久化正常。

*如有错误，请指正*
