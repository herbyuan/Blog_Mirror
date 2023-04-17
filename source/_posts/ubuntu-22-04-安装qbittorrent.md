---
title: Ubuntu 22.04 安装qBittorrent
date: 2023-04-17 10:11:56
tags: 编译
---

# Ubuntu 22.04 安装qBittorrent

这期来讲一下怎么在 Ubuntu 安装最新版本的 qBittorrent.

## 方法一: apt 直接安装旧版

``` sh
sudo apt-get update
sudo apt install qbittorent
```

这样可以安装旧版本的 qBittorrent, 应该是 4.4.X.

## 方法二: 添加 PPA 软件源

安装add-apt-repository命令

```sh
sudo apt-get update && sudo apt-get install software-properties-common -y
```

添加qbittorrent的PPA软件源

```sh
sudo add-apt-repository ppa:qbittorrent-team/qbittorrent-stable
```

安装qbittorrent-nox, 这里 `nox` 指的是没有图形界面

```sh
sudo apt-get update && sudo apt-get install qbittorrent-nox -y
```

## 方法三: 自行编译源文件

方法二可以节约大量时间, 能采用就尽量采用. 网络连不到 PPA 的时候可以尝试更换一下 DNS. 实在连不上或者闲着没事才搞这种方法.

大致来说 qBittorrent 的编译需要依赖 libtorrent 和 Qt5. 而这两个玩意都不能用 apt 安装.

以 `4.5.2` 版本为例, 所有需要的文件我都打包在[这里](qbittorrent_source.zip), 可以直接下载或者去官网自行下载.

### 安装 Qt5

```sh
sudo apt install build-essential cmake automake autoconf git
sudo apt-get install qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools
sudo apt-get install qtcreator
sudo apt-get install qt5*
sudo apt install libqt5svg5-dev
sudo apt install pkg-config
```

速度慢的话替换成国内镜像.



需要先安装 BOOST 和 openssl, 这两者都不能用 apt 安装的.

#### 编译 boost

libtorrent使用 boost C++库作为基础库而开发的, 所以需要先编译boost库. 注意 boost 版本 >=1.58 才可以在libtorrent库中使用.

进入 boost 目录中, 正常编译:

```sh
sudo ./bootstrap.sh --with-libraries=all --with-toolset=gcc
sudo ./b2
```

这里推荐不要用 `./b2 install`, 因为我用了之后编译会出意想不到的问题.

最后成功了会显示:

```
The following directory should be added to compiler include paths:
    /home/herbert/libtorrent/boost_1_82_0

The following directory should be added to linker library paths:
    /home/herbert/libtorrent/boost_1_82_0/stage/lib
```

boost 库编译完成后可以看到它提示需要将 boost 库的头文件路径和库文件路径添加到系统的包含目录中, 便于其他软件编译时进行调用.

用 vim 打开 `/etc/profile`, 在文件最后把之前编译 boost 之后的 boost 相关头文件路径, 库文件路径, 链接库路径加上:

```sh
export   CPLUS_INCLUDE_PATH=$CPLUS_INCLUDE_PATH:/home/herbert/libtorrent/boost_1_82_0

export   LIBRARY_PATH=$LIBRARY_PATH:/home/kherbert/libtorrent/boost_1_82_0/stage/lib

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/herbert/libtorrent/boost_1_82_0/stage/lib
``` 

然后终端中使用source命令使/etc/profile配置的路径生效

```sh
source /etc/profile
```

#### 编译openssl

编译安装openssl:

```sh
./config
sudo make install
```

这一步不太会出问题, 就是有可能有的 `so` 文件不能正确链接, 使用 `locate XXX.so` 找到文件后 `sudo ln -s XXX.so.xxx XXX.so` 建立连接.

#### 编译 libtorrent

**注意: 编译 qB 4.5.2 一定要 libtorrent >= 1.18 && < 2 或者 >= 2.0.7!**

进入 libtorrent 文件夹, 输入命令:

```sh
sudo ./configure
```

我遇到了以下错误:

```
$ ./configure
Checking for boost libraries:
checking for boostlib >= 1.58 (105800)... yes
checking whether g++ supports C++11 features with -std=c++11... yes
checking whether the Boost::System library is available... yes
configure: error: Could not find a version of the Boost::System library!
```

不使用 b2 编译时, 在configure阶段, 每次都提示找不到boost系统库文件, 这个问题的解决方案是在 configure 阶段添加指定 system library 路径.

```sh
./configure --with-boost-libdir=/home/herbert/libtorrent/boost_1_74_0/stage/lib
```

然后

```sh
sudo cmake
```

漫长的等待......

```sh
sudo make install
```

### 编译 qBittorrent

终于可以编译目标文件了......

还要安装一些包

```sh
sudo apt install libqt5svg5-dev libboost-dev libboost-system-dev zlib1g-dev
```

进入解压后的 qBittorrent 文件夹, 输入命令:

```sh
./configure
```

理论上应该没有问题了, 如果之前 BOOST 没有加载到目录里面的话可以加上参数 `--with-boost-libdir=/home/herbert/libtorrent/boost_1_74_0/stage/lib`.

如果要没有图形界面的版本, 加上 ` --disable-gui`.

```sh
sudo make
```

漫长的等待......
> 据说可以用 `make -j$(nproc)`, 使用多个核心,但是我没有尝试.

```sh
sudo make install
```

大功告成, 直接在程序启动台就可以看到 qbittorrent 了.

------

## 设置开机启动

```sh
sudo apt-get install vim -y 
vim /etc/systemd/system/qbittorrent-nox.service
```

输入以下内容

```
[Unit]
Description=qBittorrent-nox
After=network.target
[Service]
User=root
Type=forking
RemainAfterExit=yes
ExecStart=/usr/bin/qbittorrent-nox -d
[Install]
WantedBy=multi-user.target
```

修改qbittorrent-nox.service文件后重新载入

``` sh
sudo systemctl daemon-reload
```

启动

``` sh
sudo systemctl start qbittorrent-nox
```

停止

```sh
sudo systemctl stop qbittorrent-nox
```

设置开机启动

```sh
sudo systemctl enable qbittorrent-nox
```

查看状态

```sh
sudo systemctl status qbittorrent-nox
```

登录 web 管理界面

默认账号：admin 密码：adminadmin
默认登陆网址：http://127.0.0.1:8080

注意默认情况下 qBittorrent 开启了 Host Header 属性验证, 如果做了端口转发登录时可能显示 `Unauthorized`. 这时候用原先端口登录后`设置 - Web UI - 安全` 取消勾选 `启用 Host Header 属性验证` 就可以正常登录了.



