---
title: hexo在github上搭建个人博客系统
date: 2019-11-27 10:50:11
tags: [hexo, github, 博客, blog]
---


## Hexo介绍

[Hexo](https://hexo.io/)是一款基于`Node.js`的静态博客框架,简单的说,就是可以通过`Hexo`,把`markdown`文件直接转换成一个静态的网站输出到一个文件夹,然后将这个文件夹可以直接上传到服务器或者github,就可以访问啦~


## 安装`Node.js`
`Hexo`基于`Node.js`,所以需要先安装`Node.js`的环境，下载地址[nodejs官网](https://nodejs.org/en/download/)，选择msi版本傻瓜式安装。安装完成后，在cmd窗口运行命令有对应版本显示即成功
> node -v


## 安装git

博客托管于Github,基于Hexo制作成静态博客;安装方法参考我另一篇博客[Git使用教程第一章 环境安装](http://xxx.xx.com)
 

## Hexo使用
安装Hexo:

> npm install-g hexo

创建博客项目目录:
> mkdir hexo-blog    //目录名称可以自己定义，hexo init要求目录为空，故创建一个新目录hexo-blog

初始化项目:

> hexo init

生成静态页面:

> hexo generate or hexo g

本地启动访问静态网站:

> hexo server or hexo s

启动之后会提示本地访问的地址,直接输入地址访问就可以啦;eg:localhost:4000

## Hexo部署

怎样才能让全世界的人都可以看到我的博客呢？Github提供了github pages快速部署你的静态网页，经过hexo编译后，我们可以将生成好的静态网站资源提交到xxx.github.io,这样外网上的用户就可以访问我的博客了。比如[我的博客](https://jishengqiu.github.io/)链接，可以看到访问的是github上的pages了。在初始化博客项目的文件夹hexo-blog目录下找到`_config.yml`,修改deploy节点

```bash
deploy:

     type: git

     repo:https://github.com/您的github项目仓库地址

     branch:master
```
**选择github pages优势**
- 使用零成本: github pages 集成在 github 中, 直接和代码管理绑定在一起, 随着代码更新自动重新部署, 使用非常方便
- 免费: 免费提供 username.github.io 的域名, 免费的静态网站服务器
- 无数量限制: github pages 没有使用的数量限制, 每一个 github repository 都可以部署为一个静态网站

注意,我们这里的`branch`需要设置为master,否则,css,js,图片等资源文件加载不到。为了通过hexo直接把项目直接发布到github仓库的master分支,需要再安装一个git模块:

> npm install hexo-deployer-git --save-dev

执行如下命令,静态站点就push到github上面去啦:

> hexo deploy or hexo d

这个命令实际上是把我们刚才执行`hexo generate`所生成的`public`文件夹里面的所有内容,既是静态站点,push到了github的master分支上面;

每次部署的步骤,可以按照这个步骤来:

```bash
# hexo clean  # 如果不是重新调整了目录结构，不建议执行这个命令，会重新生成文章的时间

hexo generate

hexo deploy
```
---
>抛出一个问题：如果每一次部署都需要重复这个步骤是比较繁琐，能不能实现自动化部署呢？只用关心写md文件就好了。答案是肯定的，下一篇博客介绍一款自动化集成工具[Travis CI](https:travis-ci.org)






