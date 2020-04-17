---
title: docker实战系列八之部署个人博客
date: 2020-03-14 20:17:27
tags:
---
## 准备镜像
```
[vagrant@docker-host ~]$docker pull wordpress
[vagrant@docker-host ~]$docker pull mysql:5.6
```

## 运行MySQL容器
```
[vagrant@docker-host ~]$ docker run -d  --name mysql1  -e MYSQL_ROOT_PASSWORD=root mysql:5.6

- docker run ：启动容器 
- -d：后台运行容器 
- –name wordpress-mysql：指定容器的名字，本文设置为wordpress-mysql 
- -e MYSQL_ROOT_PASSWORD=root：指定容器的环境参数，此处初始化MySQL的root密码 
- mysql：镜像的名字，首先从docker宿主机本地加载，其次从dockerHub上加载

```

## 运行WordPress容器
```
 [vagrant@docker-host ~]$ docker run -d --name wordpress -p 80:80 --link mysql1:mysql -e WORDPRESS_DB_NAME=wordpress -e WORDPRESS_DB_PASSWORD=root wordpress

- docker run ：启动容器 
- -d：后台运行容器 
- –name wordpress：指定容器的名字，本文设置为wordpress-wordpress 
- –link mysql1:mysql：容器关联，现在启动的容器内部可以通过mysql来访问mysql数据库的功能 
- -p 80:80：端口映射，这里将容器内的80端口映射到docker宿主机的80端口 
- wordpress：镜像的名字，首先从docker宿主机本地加载，其次从dockerHub上加载

```

## 验证安装
打开浏览器输入docker宿主机的ip，如http://192.168.205.13
![wordpress](1.png)

选择对应语言，设置主题，密码
![wordpress](2.png)

安装成功
![wordpress](3.png)


## 验证数据库数据
```
[vagrant@docker-host ~]$ docker exec -it mysql1  mysql -proot
Warning: Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 16
Server version: 5.6.46 MySQL Community Server (GPL)

Copyright (c) 2000, 2019, Oracle and/or its affiliates. All rights reserved.

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
| wordpress          |
+--------------------+
4 rows in set (0.00 sec)

mysql> use wordpress
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql>
mysql> show tables;
+-----------------------+
| Tables_in_wordpress   |
+-----------------------+
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+
12 rows in set (0.00 sec)

mysql>
mysql>
mysql>
mysql> select * from wp_users limit 10;
+----+------------+------------------------------------+---------------+--------------+----------+---------------------+---------------------+-------------+--------------+
| ID | user_login | user_pass                          | user_nicename | user_email   | user_url | user_registered     | user_activation_key | user_status | display_name |
+----+------------+------------------------------------+---------------+--------------+----------+---------------------+---------------------+-------------+--------------+
|  1 | admin      | $P$BgH5K/PmA3CBmDjliue6mr.9S6nxlG/ | admin         | admin@qq.com |          | 2020-04-17 08:49:29 |                     |           0 | admin        |
+----+------------+------------------------------------+---------------+--------------+----------+---------------------+---------------------+-------------+--------------+
1 row in set (0.00 sec)

```

## 总结
通过上面的小案例，我们会发现使用docker部署应用非常高效，快速搭建个人博客；对比传统部署模式，需要本地安装`php`,`mysql`等相关依赖环境，才可以进行部署。



*如有错误，请指正*
