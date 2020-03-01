---
title: docker实战系列一之docker环境搭建
date: 2020-02-18 13:58:09
tags: [docker, ci, gitlab, devops]
---
## 环境准备
   
   欢迎各位加入学习docker的大军，让大家一起努力，在容器技术火爆的今天，能够与时俱进。
   

### 实验环境介绍
   
   初次想接触或者学习docker的朋友，没有mac环境或者linux环境，于是写一篇面向windows爱好者来一起学习docker，通过创捷虚拟机的方式来获得linux环境，能够更好的模拟真实使用docker的生产环境，实战价值高。
    
### 版本介绍
   
- 一台windows 10 64位
- [vagrant 2.2.7](https://www.vagrantup.com/downloads.html)
- [virtual box 6.1.2](https://www.virtualbox.org/wiki/Downloads)
- [centos 7](http://www.vagrantbox.es/)(借助virtual box创建centos7虚拟机，不需要下载centos7.iso)
- [docker 19.03](https://www.docker.com/get-started)
        
## 安装vagrant

   ![vagrant下载](vg1.png)
   下载对应的版本，然后双击安装，一直到确认安装完成。在windows下安装vagrant，为了写入相应配置到环境变量，系统可能会要求重新启动。在命令行中，输入vagrant，查看程序是不是已经运行了。如果不行，请检查一下$PATH里面是否包含vagrant所在的路径
   验证安装是否成功，打开cmd或者bash窗口
```
$ vagrant -v
```

![vagrant版本](vg2.png)

## 安装virtual box

   ![virtual box](vb1.png)
   下载对应的版本，然后双击安装，一直到确认安装完成
   注意virtual box模拟将创建的虚拟机文件放在系统用户盘下,当创建的虚拟机过多时，会导致系统盘磁盘空间不足（创建一个虚拟机，大约需要4G的磁盘空间），可以调整下虚拟机的文件存放路径，如下
   第一步，打开virtual box，找到全局设定
   ![step1](vb2.png)
   第二步，修改默认虚拟电脑的位置，选择一个磁盘空间大的即可
   ![step2](vb3.png)
   
## 通过vagrant创建centos7虚拟机
   开始之前，先创建工作目录，比如
   ```
   [Admin@HP docker]$pwd
   /e/study/docker
   ```
### 3种方式创建centos7虚拟机
   + 使用http绝对地址，vagrant box add centos7 https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box
   + 使用本地文件，使用下载工具下载https://github.com/tommy-muehle/puppet-vagrant-boxes/releases/download/1.1.0/centos-7.0-x86_64.box，然后通过vagrant box add centos7  ./centos-7.0-x86_64.box
   + 使用仓库地址，比如vagrant box add ubuntu/precise64


### 初始化镜像

```
  $ cd 工作目录，比如e://study/docker
  $ vagrant init centos7
```

### 启动系统
```
$ vagrant up
```

![启动虚拟机](vg3.png)

### ssh登录虚拟机
1、直接在当前目录，通过vagrant自带的ssh命令登录
```
 $ vagrant ssh
```

2、使用第三方的ssh远程工具，比如xshell,SecureCRT等

![登录虚拟机](vg4.png)

登录进去之后，可以刷刷的玩Linux命令了，激动不

### vagrant常用命令
   
    vagrant box add	添加box的操作
    vagrant init	初始化box的操作，会生成vagrant的配置文件Vagrantfile
    vagrant up	启动本地环境
    vagrant ssh	通过ssh登录本地环境所在虚拟机
    vagrant halt	关闭本地环境
    vagrant suspend	暂停本地环境
    vagrant resume	恢复本地环境
    vagrant reload	修改了Vagrantfile后，使之生效（相当于先 halt，再 up）
    vagrant destroy	彻底移除本地环境
    vagrant box list	显示当前已经添加的box列表
    vagrant box remove	删除相应的box
    vagrant package	打包命令，可以把当前的运行的虚拟机环境进行打包
    vagrant plugin	用于安装卸载插件
    vagrant status	获取当前虚拟机的状态
    vagrant global-status	显示当前用户Vagrant的所有环境状态
    
    
## 安装docker
  外国的东西，一般先lou一眼官网说明。打开docker官网，找到docker社区版 [Get Docker Engine - Community for CentOS](https://docs.docker.com/install/linux/docker-ce/centos/)
  **step1 卸载旧版本**
  ```
  $ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```
**step2 安装工具包和设置仓库**
 ```
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
  
$ sudo yum-config-manager \
      --add-repo \
      https://download.docker.com/linux/centos/docker-ce.repo
```
**step3 安装docker社区版**

```
$ sudo yum install docker-ce docker-ce-cli containerd.io
```
**step4 检查安装是否成功**
```
$ docker -v
$ docker version // 检查docker daemon进程
```
>注意linux用户权限问题sudo groupadd docker  
>sudo usermod -aG docker 用户名，vagrant创建的虚拟机用户名默认为vagrant

**step5 启动docker服务**
```
$ sudo systemctl start docker
```
**step6 验证docker服务是否正常**
```
$ sudo docker run hello-world
```

![docker验证](dk1.png)

## 使用`Vagrantfile`快速搭建docker环境
每次需要一台linux的docker实验环境，都需要按照上面的步骤来安装docker，似乎有些笨重机械，能不能通过脚本来完成自动安装呢？下面介绍使用`Vagrantfile`来创建一台安装好docker的机器
```
$ mkdir docker-host
$ cd docker-host
$ cat Vagrantfile  // 将准备好的Vagrantfile放入当前目录
## 注意以下是Vagrantfile的内容，boxes是一个数组，可以一次创建多台安装好docker的实验环境的linux机器
## 刚兴趣的读者可以先学习下Vagrantfile的配置文件介绍

# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.6.0"

boxes = [
    {
        :name => "docker-host",     # 注意修改
        :eth1 => "192.168.205.13",  # 注意修改
        :mem => "1024",
        :cpu => "1",
        :port => "6777"            # 指将虚拟机的80端口映射到本地window宿主机的6777端口
                                   # 比如当前配置这样本地window使用浏览器就可以通过localhost:6777访问创建的虚拟机中的192.168.205.13:80
                                   # 实质是port mapping的过程
    }
]
Vagrant.configure(2) do |config|

  config.vm.box = "centos/7"

  boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "vmware_fusion" do |v|
          v.vmx["memsize"] = opts[:mem]
          v.vmx["numvcpus"] = opts[:cpu]
        end

        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
          v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
        end

        config.vm.network :private_network, ip: opts[:eth1]
        config.vm.network :forwarded_port, guest: 80, host: opts[:port]
      end
  end
      config.vm.synced_folder ".", "/vagrant", disabled: true   #需要disabled的理由是因为vagrant为我们创建的默认用户名是vagrant，它禁止vagrant目录映射
      config.vm.synced_folder "./labs","/home/vagrant/labs"     # 将本地当前目录的lab文件夹映射到虚拟机内部的labs文件夹                                                  
      config.vm.provision "shell",privileged:true, path: "./setup.sh"  #是指虚拟创建完成需要执行的外部脚本
end

$ cat setup.sh  //查看setup.sh文件内容

#/bin/sh
 
# install some tools
sudo yum install -y git vim gcc glibc-static telnet bridge-utils
 
# install docker
curl -fsSL get.docker.com -o get-docker.sh
sh get-docker.sh
 
# start docker service
sudo groupadd docker
sudo usermod -aG docker vagrant
sudo systemctl start docker
 
rm -rf get-docker.sh


$ vagrant up  // 启动虚拟机

$ docker version //验证docker环境
```
现在准备实验环境的步骤简化为，第一步，创建工作目录；第二步，放入编辑好的配置文件；第三步，然后启动虚拟机；第四步，完成后验证`docker`的安装是否成功即可。


## 使用第三方ssh工具远程vagrant创建的虚拟机
使用`xshell`远程登录虚拟机`ssh -p 127.0.0.1:2222`
![s1](s1.png)
默认的用户名为`vagrant`
![s2](s2.png)
Password选项选不了，需要提供秘钥登录？上面的安装过程中根本没有提到秘钥文件，这可怎么办呢？原因在于`vagrant`创建的虚拟机默认允许的是私钥登录，查看ssh配置文件，如下图
![s3](s3.png)

具体步骤如下先在本地`docker-host`目录下，打开`bash`窗口，运行`vagrant ssh`进入到虚拟机内部，然后修改配置，重新服务。虚拟机内部的修改方式如下

```
   $ cd /etc/ssh
   $ sudo vim sshd_config #开启密码访问
   # To disable tunneled clear text passwords, change to no here!
   PasswordAuthentication yes
   #PermitEmptyPasswords no
   #PasswordAuthentication no
   #重启服务
   $ sudo systemctl restart sshd.service
```
修改完毕后，再次使用xshell登录，默认用户名，密码都是vagrant
![s4](s4.png)

登录成功！查看docker安装是否成功
![s5](s5.png)


## 小结
本文主要介绍了，从零开始学习docker的环境搭建，通过动手实践，即使没有mac或者linux机器，也可以轻松拥有一个docker学习环境。下一篇，将介绍docker的基本使用。

*如有错误，请指正*