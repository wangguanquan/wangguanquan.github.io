---
layout: post
title: BIND9 源码编译安装
categories: Blog
description: BIND9 源码编译安装以及集群部署
keywords: BIND,DNS 
---

BIND（Berkeley Internet Name Domain）是现今互联网上最常使用的DNS软件，使用BIND作为服务器软件的DNS服务器约占所有DNS服务器的九成。BIND现在由互联网系统协会（Internet Systems Consortium）负责开发与维护。

20世纪80年代，柏克莱加州大学计算机系统研究小组的四个研究生Douglas B Terry、Mark Painter、David W. Riggle和周松年（Songnian Zhou）一同编写了BIND的第一个版本，并随4.3BSD发布。

安装BIND可以使用源码编译或者直接使用yum,brew等工具安装，我这里介绍源码安装。

### 编译前环境准备

前往官网[下载源码](https://www.isc.org/bind/)，当前有4个版本

VERSION|STATUS|RELEASE DATE|EOL DATE
-------|------|------------|---------
9.17.1|Development|April 2020|TBD
9.16.3|	Current-Stable|May 2020|TBD
9.14.12|Approaching EOL|May 2020|May 2020
9.11.19|Current-Stable, ESV|May 2020|December 2021

- Development 为开发版，不适用于生产环境
- Current-Stable 稳定版本
- Approaching EOL 即将停更版本
- Current-Stable, ESV 带扩展功能的的稳定版本

线上使用下载稳定版本即可，点击下载会弹出一个窗口，这里提供windows或是*nux版本，根据操作系统选择下载。

![download](/images/posts/bind9/bind9%20download.png)

本文主要介绍BIND9在linux系统上的安装步骤，最后会稍微提一下windows上的安装步骤

## linux 安装

### 编译安装

1. 解压

```
tar -xvf bind-9.16.3.tar.xz
```

2. 编译安装

编译前可以使用`./configure -h`查看编译参数，我这里使用最简单的编译参数，如下我指定了程序安装位置以及系统配置文件位置。

```
$ cd bind-9.16.3
$ ./configure --prefix=/usr/local/bind9 --sysconfdir=/etc/named/ --enable-threads
$ make && make install
```

当然编译过程中肯定会有一些错误，缺少什么就安装什么。我记得有`PLY`，`libuv`，`openssl`，`libcap`等，需要python2.7及更高版本。

*自行编译bind源码包没有配置文件*

3. 创建用户

生产环境创建一个named用户/组

```
$ groupadd  -r  -g  53  named
$ useradd  -r  -u  53   -g  53  named
```

### 添加PATH

```
#1、将bind下配置文件加入PATH中
$ vim /etc/profile.d/named.sh
export PATH=/usr/local/bind9/bin:/usr/local/bind9/sbin:$PATH
$ . /etc/profile.d/named.sh
 
#2、导出库文件搜索路径
$ vim /etc/ld.so.conf.d/named.conf
/usr/local/bind9/lib
$ ldconfig -v
 
#3、导出头文件搜索路径
$ ln -sv /usr/local/bind9/include /usr/include/named
"/usr/include/named" -> "/usr/local/bind9/include"
 
#4、导出帮助文档搜索路径（非必须）
$ vim /etc/man.config 
MANPATH /usr/local/bind9/share/man
```

### 配置

#### 1. 主配置
```
$ cd /etc/named
$ vim named.conf

options {
  listen-on port 53 { any; };
  listen-on-v6 port 53 { ::1; };
  directory "/var/named";   # zone配置根目录
  dump-file "/var/named/data/cache_dump.db";
  statistics-file "/var/named/data/named_stats.txt";
  memstatistics-file "/var/named/data/named_mem_stats.txt";
  allow-query { any; };
  recursion yes; # 允许递归查询

  forward first;
  forwarders {
    114.114.114.114; # 电信DNS
  };
};

logging {
  channel queries_log {
    file "data/named.run" versions 3 size 300m; # 这里的路径是相对于上面的directory路径
    print-time yes;                  # 日志文件每300MB切割一次
    print-category yes;
    print-severity yes;
    severity info;
  };

  channel query-errors_log {
    file "data/query-errors.run" versions 5 size 20m;
    print-time yes;
    print-category yes;
    print-severity yes;
    severity dynamic;
  };

  category queries { queries_log; };
  category resolver { queries_log; };
  category query-errors {query-errors_log; };
};


zone "." IN { # 根
  type hint;
  file "named.ca";
};

include "/etc/named/named.rfc1912.zones";
```

#### 2. rndc配置

rndc是一个管理程序，可以用它来刷新配置，停止服务，强制同步等
```
rndc-confgen  > /etc/named/rndc.conf
```

打开rndc.conf文件，找到# Use with the following in named.conf, adjusting the allow list as needed:注释，复制其下所有行到named.conf并放开注释。

最终的named.conf文件像下面这样

```
key "rndc-key" {
  algorithm hmac-sha256;
  secret "...";
};

controls {
  inet 127.0.0.1 port 953
  allow { 127.0.0.1; } keys { "rndc-key"; };
};

options {
  ...
};

logging {
  ...
};


zone "." IN { # 根
  type hint;
  file "named.ca";
};

include "/etc/named/named.rfc1912.zones";
```

#### 3. zones 配置

```
$ vim named.rfc1912.zones

zone "abc.com" IN {
  type master;
  file "abc.com.zone"; # 这里的文件路径是相对于directory路径
  allow-update { none; };
  forwarders { }; # 如果本地无法解析则不递归到其它DNS
};

zone "7.168.192.in-addr.arpa" IN {
  type master;
  file "7.168.192.loopback";
};
```

#### 4. 配置zone

所有的zone配置都放在directory路径下。因为我们在options配置了`directory "/var/named";`，所以我们需要把zone放到/var/named目录下

```
$ vim /var/named/abc.com.zone

$TTL 10M  ;time to live  信息存放在高速缓存中的时间长度，以秒为单位
@       IN      SOA     abc.com.      admin.abc.com. (
                        1   ;序列号
                        1H  ;1小时后刷新
                        5M  ;15分钟后重试
                        7D  ;1星期后过期
                        1D );否定缓存TTL为1天
        IN      NS      ns.abc.com.
ns      IN      A       192.168.7.134
www       IN      A      192.168.7.133
@         IN      A      192.168.7.133
api       IN      A      192.168.7.133
a         IN      A      192.168.7.134
```

SOA记录：start of authority，起始授权机构，用于标示一个区的开始，其格式如下：

zone IN SOA Hostname Contact (
              SerialNumber
              Refresh
              Retry
              Expire
              Minimum )
              
- Hostname 存放本 Zone 的域名服务器的主机名
- Contact 管理域的管理员的邮件地址
- SerialNumber 本区配置数据的序列号，用于从服务器判断何时获取最新的区数据
- Refresh 辅助域名服务器多长时间更新数据库
- Retry 若辅助域名服务器更新数据失败，多长时间再试
- Expire 若辅助域名服务器无法从主服务器上更新数据，原有的数据何时失效
- Minimum 设置被缓存的否定回答的存活时间


配置反向解析

```
$TTL 10M
@       IN SOA  abc.com.  admin.abc.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
           IN       NS       ns.abc.com.
133        IN       PTR     www.abc.com.
           IN       PTR     @.abc.com.
           IN       PTR     api.abc.com.
134        IN       PTR     a.abc.com.
```


#### 5. 根区域配置

在联网的情况下直接将查询根的结果导入根区域配置文件，我们在named.rfc1912.zones文件中配置了根文件named.ca

```
$ dig -t NS . > /var/named/named.ca
```

### 授权

接下来我们更改所有配置文件的用户为named用户

```
$ chown -R named:named /etc/named
$ chown -R named:named /var/named
```

### 启动服务

到此，我们的安装配置工作已经结束，愉快的启动吧。

```
$ named-checkconf
$ named -u named
```

第一行是检查配置是否正确，第二行指定named用户启动named服务。

### 修改首选DNS

```
$ vim /etc/resolv.conf

nameserver 192.168.7.134  # 这里配置我们自己的DNS服务器地址，必须放在最前面。
nameserver 114.114.114.114 # 当首选不可用时使用备选DNS
```

### 测试

```
$ dig www.abc.com

; <<>> DiG 9.16.3 <<>> www.abc.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25494
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.abc.com.			IN	A

;; ANSWER SECTION:
www.abc.com.		600	IN	A	192.168.7.133

;; Query time: 0 msec
;; SERVER: 192.168.7.134#53(192.168.7.134)
;; WHEN: 三 6月 17 16:57:24 CST 2020
;; MSG SIZE  rcvd: 84
```

在ANSWER SECTION里能够看到正确的解析，正是我们在abc.com.zone配置的IP地址。

### 一些重要的命令

以下是rndc的命令

1. `status` 查看服务状态
2. `reload` 重新加载服务，修改配置或添加域名后刷新配置，也可以指定某个zone文件。使用和`reconfig`相当
3. `flush` 刷新服务器缓存
4. `querylog [ on | off ]` 打开/关闭查询日志

刷新DNS缓存

- windows `ipconfig /flushdns`
- linux `nscd -i hosts`
- mac `dscacheutil -flushcache`

## windows

以管理员身份运行`BINDInstall.exe`

![BINDInstall](/images/posts/bind9/bind9%20win.png)

设置密码点击Install按钮即自动安装。

配置Path，将安装路径(C:\Program Files\ISC BIND 9\bin)配置到Path方便使用命令。

WINDOwS差不多就这些吧，我也没有完全安装。。。

### 主从 DNS 服务搭建

1. 主从 DNS 的搭建开始的时候其实是和单机搭建一样的，可以将配置文件从主机完成复制到从机，然后修改主机配置"/etc/named/named.rfc1912.zones"

```
zone "abc.com" IN {
  type master;
  file "abc.com.zone"; # 这里的文件路径是相对于directory路径
  allow-update { 192.168.7.133; };
  allow-transfer { 192.168.7.133; };    # 允许同步DNS的辅助服务器IP
  also-notify { 192.168.7.133; };
  notify yes;                           # 启用变更通告，当主文件变更，通知从进行比较同步
};

zone "7.168.192.in-addr.arpa" IN {
  type master;
  file "7.168.192.loopback";
  allow-update { 192.168.7.133; };
  allow-transfer { 192.168.7.133; };    # 允许同步DNS的辅助服务器IP
  also-notify { 192.168.7.133; };
  notify yes;                           # 启用变更通告，当主文件变更，通知从进行比较同步
};
```

重启主机服务

2. 修改从服务器的"/etc/named/named.rfc1912.zones"

```
zone "abc.com" IN {
  type slave;
  file "slaves/abc.com.zone";
  masters { 192.168.7.134; };   # 指定主服务器的 IP
  masterfile-format text;       # 指定区域文件的格式为text，不指定有可能会为乱码
};

zone "7.168.192.in-addr.arpa" IN {
  type slave;
  file "7.168.192.loopback";
  masters { 192.168.7.134; };   # 指定主服务器的 IP
  masterfile-format text;       # 指定区域文件的格式为text，不指定有可能会为乱码
};
```

我们不需要再去配置 abc.com.zone 文件，直接启动从的 dns 服务，所有的zone文件会从主节点同步过来。

到此主从DNS服务已经搭建完成。