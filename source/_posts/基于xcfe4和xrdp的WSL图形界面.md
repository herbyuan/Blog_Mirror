---
title: 基于xcfe4和xrdp的WSL图形界面
date: 2023-03-23 13:47:41
tags: WSL
updated:
type:
comments:
description:
keywords:
top_img:
mathjax:
katex:
aside:
aplayer:
highlight_shrink:
---


# 基于xcfe4和xrdp的WSL图形界面

如果只要使用单个应用, 微软官方有[教程](https://learn.microsoft.com/zh-cn/windows/wsl/tutorials/gui-apps), 直接照着做就好了. 

## 更新本地软件数据库

``` sh
sudo apt-get update
sudo apt update
```

## 安装桌面 `xfce4` 和远程服务器 `xrdp`

``` sh
sudo apt install -y xfce4  xrdp
```

如果有这个选项, 选择 `lightdm` 据说能有更好的体验. 

<img src="lightdm.png" width="60%">


有人说还要 `sudo apt install xorg`, 但是我不安装好像也没有问题, 好像自动也安装了. 

## 解决和windows的端口冲突

`xrdp` 服务器默认使用 `3389` 端口, 会和本机Windows的远程桌面冲突, 这里可以改为 `3390`. 

``` sh
sudo vim /etc/xrdp/xrdp.ini
```

把 `port = 3389` 改成 `port = 3390` (或者任意一个你喜欢且不冲突的端口). 

<img src="xrdp_ini.png" width="60%">

> 注意, 如果 `wsl2` 使用了虚拟交换机的桥接网络, 则虚拟机会有一套自己的ip地址, 使用这个ip在局域网内的其他电脑也是可以连接到虚拟机的. 但即便如此, 也尽量不要让端口冲突, 因为本机使用 `127.0.0.1` 时既可以连接宿主机又可以连接虚拟机. 

## 指定桌面软件

``` sh
echo "xfce4-session" > ~/.xsession
```

## 修改xfce4锁屏相关配置,不然可能出现自动锁屏后黑屏,也无法唤醒

``` sh
sudo vim /etc/X11/xrdp/xorg.conf
```

在末端追加

``` sh
Section "ServerFlags"
        Option  "BlankTime"  "0"
        Option  "StandbyTime"  "0"
        Option  "SuspendTime"  "0"
        Option  "OffTime"  "0"
EndSection
```

## 启动xrdp服务器

```sh
sudo service xrdp start
```

或者

```sh
sudo /etc/init.d/xrdp start
```

<img src="start_xrdp.png" width="60%">

## 使用 windows 远程桌面连接

打开Windows远程连接,计算机处输入 `localhost:3390` ,用户名和登录密码对应 wsl 的用户和密码. 

<img src="login.png" width="60%">

登陆后就有了图形界面, 虽然卡卡的但是可以看了. 

<img src="screenshot.png" width="60%">

好像双击不能运行文件. . . . . . 
