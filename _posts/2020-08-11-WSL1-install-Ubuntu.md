---
layout: post
title: Windows10安装Ubuntu+图形界面
categories: blog
description: 使用windows10的wsl功能来实现Ubuntu子系统
keywords: wsl,ubuntu
---

WSL(Windows Subsystem for Linux)适用于 Linux 的 Windows 子系统，是一个为在Windows 10和Windows Server 2019上能够原生运行Linux二进制可执行文件（ELF格式）的兼容层。

WSL提供了一个由微软开发的Linux兼容的内核接口（不包含Linux内核代码），然后可以在其上运行GNU用户空间，例如Ubuntu，openSUSE，SUSE Linux Enterprise Server，Debian和Kali Linux。这样的用户空间可能包含Bash shell和命令语言，使用本机GNU/Linux命令行工具（sed，awk等），编程语言解释器（Ruby，Python等），甚至是图形应用程序（使用主机端的X窗口系统）。


## 启用WSL

启用WSL前确保windows10版本大于1607，WSL2系统要求版本2004以上，你可以使用命令`winvar`来查看，如果系统满足就可以使用如下步骤开启wsl了：


1. 打开windows设置 > 更新和安全


2. 找到开发者选项并勾选“开发人员模式”

![开发人员模式](/images/posts/wsl/1.png)


3. 控制面板 > 所有控制面板荐 > 程序和功能 选择左边的“启用或关闭windows功能”


4. 在弹出来的窗口中勾选“适用于Linux的Windows子系统”并确定

![](/images/posts/wsl/2.png)

## 安装Ubuntu

启用WSL功能后你就可以到商店搜索"Ubuntu"，目前有18.04 LTS和20.04 LTS这两个长期稳定版，建议WSL1使用18.04 LTS，因为已知在wsl1上安装20.04有各种问题，当然爱折腾的你可以去尝试一下。

![下载Ubuntu](/images/posts/wsl/3.png)


下载安装好后就可以启动，第一次启动需要几分钟的时间，耐心等待一下即可。

![启动Ubuntu](/images/posts/wsl/4.png)


初始化完成之后会让你创建用户，输入用户名和密码即可。

![](/images/posts/wsl/6.png)


接下来可以使用命令`cat /etc/os-release`来查看系统。

![系统信息](/images/posts/wsl/5.png)


现在Ubuntu已经成功运行了，下面我们可以安装一些额外的东西比较GUI界面和中文环境

## 安装图型界面

#### 更新系统

```
sudo apt update
sudo apt upgrade
```

#### 安装Xfce桌面环境

Xfce 是一个快捷的、轻量级的，功能齐全的桌面环境，不像 Gnome 和 KDE Plasma 等这些重量级的桌面环境，Xfce占用的系统资源要少得多。另外，它拥有更好的模块性和更少的依赖性；它将占用你更少的磁盘空间和更少的安装时间。

```
sudo apt install xfce4
```

#### 安装桌面连接工具

你可以选择在Ubuntu上安装xrdp使用远程桌面连接Ubuntu桌面，或者是在windows上安装[VcXsrv](https://sourceforge.net/projects/vcxsrv/)工具使用XLaunch执行WSL图形界面。

xrdp安装

```
sudo apt install xrdp
```

启动xrdp

```
sudo /etc/init.d/xrdp start
```

![启动xrdp](/images/posts/wsl/8.png)

打开“远程桌面连接”，输入地址`localhost:3390`连接，如果连接失败多半是端口问题，编辑`/etc/xrdp/xrdp.ini`文件，将`port=3389`改为`port=3390`并保存，

修改端口后需要重启xrdp服务

```
sudo /etc/init.d/xrdp restart
```

输入上面设置的用户名密码，现在你应该可以正常进入Ubuntu桌面了。
![远程桌面](/images/posts/wsl/9.png)
![Ubuntu桌面](/images/posts/wsl/10.png)

## 安装中文环境（非必须）

下载中文语言包

```
sudo apt install language-pack-zh-han*
```

修改`/etc/default/locale`文件


```
LANG=zh_CN.UTF-8
LANGUAGE="zh_CN:zh"
```

重启wsl即可生效

## 安装过程中遇到的一些问题即解决方法

1. 启动xrdp时提示`sleep: cannot read realtime clock: Invalid argument`异常

```

wget https://launchpad.net/~rafaeldtinoco/+archive/ubuntu/lp1871129/+files/libc6_2.31-0ubuntu8+lp1871129~1_amd64.deb

sudo dpkg --install libc6_2.31-0ubuntu8+lp1871129~1_amd64.deb

sudo apt-mark hold libc6

sudo apt --fix-broken install

sudo apt full-upgrade
```

2. 执行apt更新或安装app时提示`E: Sub-process /usr/bin/dpkg returned an error code (1)`

提示使用`sudo apt --fix-broken install`命令修复，可以先执行此命令修复一下，如果不能修复可以编辑`/var/lib/dpkg/info/libc6\:amd64.postinst`文件并注释掉`set -e`这一行。

安装过程中如果遇到问题一般根据提示就可以修复了

## 一些使用技巧

1. 与windows交互

在windows系统中通过`\\wsl$\Ubuntu-18.04`访问Ubuntu文件
![](/images/posts/wsl/11.png)

如果不知道wsl后面的值可以通过windows命令执行`wslconfig /l`查看

Ubunut访问windows目录使用`/mnt/` + 盘符（无冒号）+完整路径，比如访问d盘doc目录下的1.txt文件，可以使用命令

```
cat /mnt/d/doc/1.txt
```
2. 停止wsl服务

出于某种原因你可能需要暂时停止wsl服务来释放一些资源，此时你可以使用命令`wslconfig /t Ubuntu-18.04`来停止，系统名可以使用
`/l`命令查看

3. 网络访问

wsl共享windows网络设备，所以如果你在ubunut里启动了nginx服务，你可以在windows下直接使用localhost访问


对于开发人员来说wsl还是非常有用的，至少你有了gcc,nginx,docker,python等一些天然环境，并且可以与windows无缝连接。




