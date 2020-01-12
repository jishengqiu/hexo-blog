---
title: Git进阶篇idea终端启用Git bash
date: 2019-11-27 15:50:54
tags:
---
## 安装`git bash`
你必须安装 gitbash，可以参考我的另一篇博客[Git使用教程之环境安装](https:www.baidu.com)

## 设置`Terminal`插件
打开设置（快捷键：Ctrl + Alt + S），进入 Plugins,搜索栏搜索 Terminal，查看 Terminal插件是否打勾选中，如果没有，请打勾。
如图
![idea插件配置](idea-plugins.jpg)

## 设置`Terminal`脚本路径
进入设置（快捷键：Ctrl + Alt + S），进入 Tools字段，再进入 Terminal字段，在 Shell path那一栏中，输入你主机 Git Bash的安装位置
如图
![idea插件配置](idea-terminal.jpg)

> "E:\Program Files\Git\bin\sh.exe" -login -i
## 测试
![Terminal终端](test_ok.jpg)
OK,done!
