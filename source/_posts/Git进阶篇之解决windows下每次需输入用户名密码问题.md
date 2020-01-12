---
title: Git进阶篇之解决windows下每次需输入用户名密码问题
date: 2019-11-27 17:46:35
tags:
---
## 准备工作
  - 安装git
  - 配置ssh key
  - 略懂shell脚本语法
  - 略懂ssh agent的工作原理
  
## 确认ssh agent是否开启

> eval "$(ssh-agent -s)"

若显示 `Agent pid xxxx`表示已启用.

## 添加ssh key

> ssh-add ~/.ssh/id_rsa

## 修改远程项目地址为ssh传输方式

在项目目录下执行
> git remote add origin git@github.com:username/repository_name.git

或者修改项目.git目录下的config文件remote节点
``` bash
修改前
[remote "origin"]
        url = https://github.com/xxxx/xxx.git
        fetch = +refs/heads/*:refs/remotes/origin/*
修改后
[remote "origin"]
        url = git@github.com/xxxx/xxxxx.git
        fetch = +refs/heads/*:refs/remotes/origin/*
```
*https与ssh免密的方式不一样，一般使用ssh方式*

## 打开git bash自动去ssh-add

### 配置shell脚本
`windows`下安装好`Git bash`之后，我们会在C:\Users\Admin(用户名)\目录下，找到.ssh文件夹用户存放ssh-keygen生成的秘钥文件，同时会有.bash_profile，.bashrc文件。一般会在.bash_profile文件中显式调用.bashrc。登陆linux启动bash时首先会去读取~/.bash_profile文件，这样~/.bashrc也就得到执行了，你的个性化设置也就生效了。
*用户登录bash login --> 加载 ~/.bash_profile --> bash_profile中配置了首先是使~/.bashrc生效*
.bashrc文件中增加自定义的函数脚本ssh-add ~/.ssh/id_rsa，git每一次启动bash时自动执行

**talk is cheap,show me the code!**

```bash
#!/bin/sh

if [ -f ~/.agent.env ]; then
  . ~/.agent.env >/dev/null
 if ! kill -0 $SSH_AGENT_PID >/dev/null 2>&1; then
              echo “Stale agent file found. Spawning new agent…”
              eval `ssh-agent |tee ~/.agent.env`
              ssh-add
 fi
else
	echo "Starting ssh-agent..."
	eval `ssh-agent |tee ~/.agent.env`
	ssh-add ~/.ssh/jisheng_qiu@kingdee.com
fi

## 还可以在此处自定义别名
## 例子中定义了路径，语言，命令别名（使用rm删除命令时总是加上-i参数需要用户确认，使用ls命令列出文件列表时加上颜色显示）
alias rm='rm -i'
alias ls='/bin/ls -F --color=tty --show-control-chars'

```

### 测试效果
idea中打开终端`Terminal`,此时已经完成了`ssh-agent`的动作，进入到`git`项目的目录下，运行`git pull`命令，完美发现不用输入用户名密码了

### Terminal启动指定固定目录
比如负责的模块是用户管理模块，在当下微服务如此流行的情况，这个模块下的项目会有很多，比如用户基本信息服务，用户开户服务，用户支付服务等等，通常情况我会把这些项目从远程克隆到我本地的一个文件下比如`D:\work\product\`下，那使用idea会打开这个工作空间，希望`Terminal`打开后，自动定位到这个目录下，该怎么做呢？答案其实很简单，只需在上面的`.bashsc`文件文件中，增加脚本
```bash
#!/bin/sh

.....(省略其他脚本)

## 增加内容
proj="D:\work\product"
cd $proj

```

> 下一篇介绍`Git批量操作`，比如批量git pull该怎么实现？