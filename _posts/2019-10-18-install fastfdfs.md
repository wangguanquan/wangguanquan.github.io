---
layout: post
title: Install fastdfs + nginx + imagemagic
categories: Blog
description: 详细列出安装流程
keywords: fastdfs, imagemagic, nginx
---

# 1. 安装FastDFS

### 1.1 下载安装 libfastcommon
libfastcommon是从 FastDFS 和 FastDHT 中提取出来的公共 C 函数库，基础环境，安装即可 。

##### 1.1.1 下载最新版 libfastcommon

```
# wget 下载
wget https://github.com/happyfish100/libfastcommon/archive/V1.0.40.tar.gz

# curl 下载
curl https://codeload.github.com/happyfish100/libfastcommon/tar.gz/V1.0.40 > V1.0.40.tar.gz
```

##### 1.1.2 解压

```
tar -xvf V1.0.40.tar.gz
cd libfastcommon-1.0.40
```

##### 1.1.3 编译、安装

```
./make.sh
./make.sh install
```

### 1.2 下载安装 FastDFS

##### 1.2.1 下载最新版 FastDFS
```
# wget 下载
wget https://github.com/happyfish100/fastdfs/archive/V6.00.tar.gz

# curl 下载
curl https://codeload.github.com/happyfish100/fastdfs/tar.gz/V6.00 > V6.00.tar.gz
```

##### 1.2.2 解压

```
tar -xvf V6.00.tar.gz
cd fastdfs-6.00
```

##### 1.2.3 编译、安装

```
./make.sh
./make.sh install
```

### 1.3 修改配置
通过上一步安装完成后fdfs默认被安装在`/etc/fdfs`目录下, 请先cd到此目录,ls可见当前目录已有如下样例配置文件

```
client.conf.sample
storage.conf.sample
storage_ids.conf.sample
tracker.conf.sample
```

##### 1.3.1 配置Tracker

```
sudo cp tracker.conf.sample tracker.conf
sudo vim tracker.conf
```
这里仅列出必要修改点，其它的默认即可

```
# 存放data和log，必须确定run_by_user参数配置的用户有读写权限
base_path=/opt/fastdfs
# 如何选择上传文件的存储group
# 0: 轮询
# 1: 制定group名称
# 2: 负载均衡, 选择空闲空间最大的存储group
store_lookup=2
# 预留的存储空间 G(GB) M(MB) K(KB) 默认byte(B)，% 表示比例，如下，预留的存储空间为10%，即只占用90%
reserved_storage_space = 10%
# 运行用户组，默认当前用户组
run_by_group=
# 运行用户，默认当前用户
run_by_user=
# tracker的http端口(最好不要配置为80或其它较常用的端口)
http.server_port=3315
```

##### 1.3.2 配置Storage

```
sudo cp storage.conf.sample storage.conf
sudo vim storage.conf
```
这里仅列出必要修改点，其它的默认即可

```
# storage server 服务端口
port=23000
# Storage 数据和日志目录地址(根目录必须存在，子目录会自动生成)
base_path=/opt/fastdfs/storage
# 逐一配置 store_path_count 个路径，索引号基于 0。
# 如果不配置 store_path0，那它就和 base_path 对应的路径一样。
store_path0=/opt/fastdfs/storage
# tracker地址，可以配置多个tracker_server
tracker_server=tracker.server1:22122
tracker_server=tracker.server2:22122
# http端口(最好不要配置为80或其它较常用的端口)
http.server_port=8888
```

### 1.4 防火墙中打开跟踪端口

```
sudo vim /etc/sysconfig/iptables

添加如下端口行：
# tracker默认的 22122
-A INPUT -m state --state NEW -m tcp -p tcp --dport 22122 -j ACCEPT
# Storage默认的 23000
-A INPUT -m state --state NEW -m tcp -p tcp --dport 23000 -j ACCEPT

重启防火墙：
sudo service iptables restart
```

### 1.5 启动tracker和storage
```
# 启动
fdfs_trackerd /etc/fdfs/tracker.conf start
fdfs_storaged /etc/fdfs/storage.conf start
# 查看日志
tail -n10 /opt/fastdfs/logs/trackerd.log
tail -n10 /opt/fastdfs/storage/logs/storaged.log
# 如果日志显示有错误信息，需要根据信息来查找错误原因
```

### 1.6 测试
测试使用`fdfs_test`命令，使用命令前需要配置`client.conf`文件

```
sudo cp client.conf.sample client.conf
sudo vim client.conf
```
这里仅列出必要修改点，其它的默认即可

```
# Client日志路径，任意位置均可，保证有读写权限
base_path=~/fastdfs/client
# tracker地址，可以配置多个tracker_server
tracker_server=tracker.server1:22122
tracker_server=tracker.server2:22122
# Tracker端口
http.tracker_server_port=3315
```

测试上传命令`fdfs_test /etc/fdfs/client.conf upload ~/test.png`，成功后会出现以下信息

```
This is FastDFS client test program v5.12

Copyright (C) 2008, Happy Fish / YuQing

FastDFS may be copied only under the terms of the GNU General
Public License V3, which may be found in the FastDFS source kit.
Please visit the FastDFS Home Page http://www.csource.org/ 
for more detail.

[2019-10-18 14:05:57] DEBUG - base_path=/opt/fastdfs/storage, connect_timeout=10, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=0, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0

tracker_query_storage_store_list_without_group: 
	server 1. group_name=, ip_addr=xxx.xxx.xxx.xxx, port=23000
	server 2. group_name=, ip_addr=xxx.xxx.xxx.xxx, port=23000

group_name=group1, ip_addr=xxx.xxx.xxx.xxx, port=23000
storage_upload_by_filename
group_name=group1, remote_filename=M00/00/00/wKgHhl2pVruARflDAAHI8Ld-dIU264.png
source ip address: xxx.xxx.xxx.xxx
file timestamp=xxxx
file size=xxxx
file crc32=xxxx
example file url: http://xxx.xxx.xxx.xxx/group1/M00/00/00/wKgHhl2pVruARflDAAHI8Ld-dIU264.png
storage_upload_slave_by_filename
group_name=group1, remote_filename=M00/00/00/wKgHhl2pVruARflDAAHI8Ld-dIU264_big.png
source ip address: xxx.xxx.xxx.xxx
file timestamp=xxxx
file size=xxxx
file crc32=xxxx
example file url: http://xxx.xxx.xxx.xxx/group1/M00/00/00/wKgHhl2pVruARflDAAHI8Ld-dIU264_big.png
```

删除测试文件`fdfs_test /etc/fdfs/client.conf delete group1 M00/00/00/wKgHhl2pVruARflDAAHI8Ld-dIU264.png`

### 1.7 集群监控
可以通过fdfs_monitor监控当前集群情况
```
fdfs_monitor /etc/fdfs/client.conf
```

# 2. 安装Nginx
安装Nginx作为服务器以支持Http方式访问文件。同时，后面安装fastdfs-nginx-module模块也需要Nginx环境。Nginx只需要安装到StorageServer所在的服务器即可，用于访问文件。

### 2.1 下载nginx

```
# wget 下载
wget -c https://nginx.org/download/nginx-1.15.1.tar.gz

# curl 下载
curl -O https://nginx.org/download/nginx-1.15.1.tar.gz
```

### 2.2 解压
```
tar -xvf nginx-1.15.1.tar.gz
cd nginx-1.15.1
```

### 2.3 安装
```
# 建立nginx用户，为了限制nginx有太大权限访问数据
groupadd nginx
useradd -g nginx nginx --shell=/sbin/nologin

./configure
make && make install
 
# 更改nginx目录权限
chown -R nginx:nginx /usr/local/nginx
```

**如果安装过程提示错误请参照[Installing NGINX Dependencies](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/#sources)**

### 2.4 最简配置nginx.conf
```
sudo vim conf/nginx.conf
```
```$xml
user  nginx;
worker_processes  1;
events {
    worker_connections  1024;
}
 
http {
    
    server {
        listen 8888;
        server_name fastdfs;
        location /M00 {
            alias /opt/fastdfs/storage/data;
        }
    }
    
    server { # 配置访问路由和负载均衡
        listen       80;
        server_name  image.ttzero.org; # 这里配置自己的域名，如果没有域名可随意配置一个域名然后在`/etc/hosts`映射到本地即可

        location /group1 {
            proxy_pass         http://image.group1; 
            proxy_set_header   Host             $host; 
            proxy_set_header   X-Real-IP        $remote_addr; 
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
        location /group2 {
            proxy_pass         http://image.group2; 
            proxy_set_header   Host             $host; 
            proxy_set_header   X-Real-IP        $remote_addr; 
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }
    }

    upstream image.group1 {
        server xxxx.xxxx.xxxx.xxx:8888; # storage机器的IP
        server xxxx.xxxx.xxxx.xxx:8888; # storage机器的IP
    }
    
    upstream image.group2 {
        server xxxx.xxxx.xxxx.xxx:8888; # storage机器的IP
        server xxxx.xxxx.xxxx.xxx:8888; # storage机器的IP
    }
}
```

### 2.5 测试及运行
```
./sbin/nginx -t

# 输出如下信息表示成功
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful

# 启动nginx
./sbin/nginx

# 其它命令
./sbin/nginx -s reload # 服务重启
./sbin/nginx -s stop   # 服务停止

# 下载我们上面所上传的文件
curl -O http://image.ttzero.org/group1/M00/00/00/wKgHhl2pVruARflDAAHI8Ld-dIU264.png

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   169  100   169    0     0  23201      0 --:--:-- --:--:-- --:--:-- 24142
# OK可以下载了
```

### 2.6 设置开机启动

```
vim /etc/rc.local

添加一行：
/usr/local/nginx/sbin/nginx
```

# 3. 配置fastdfs-nginx-module模块
FastDFS 通过 Tracker 服务器，将文件放在 Storage 服务器存储， 但是同组存储服务器之间需要进行文件复制， 有同步延迟的问题。
假设 Tracker 服务器将文件上传到了 192.168.100.1，上传成功后文件 ID已经返回给客户端。
此时 FastDFS 存储集群机制会将这个文件同步到同组存储 192.168.100.2，在文件还没有复制完成的情况下，
客户端如果用这个文件 ID 在 192.168.100.2 上取文件,就会出现文件无法访问的错误。
而 fastdfs-nginx-module 可以重定向文件链接到源服务器取文件，避免客户端由于复制延迟导致的文件无法访问错误。

### 3.1 下载最新版 fastdfs-nginx-module
```
# wget 下载
wget https://github.com/happyfish100/fastdfs-nginx-module/archive/V1.20.tar.gz

# curl 下载
curl https://codeload.github.com/happyfish100/fastdfs-nginx-module/tar.gz/V1.20 > V1.20.tar.gz
```

### 3.2 解压

```
tar -xvf V1.20.tar.gz
```

### 3.3 nginx中添加模块

```
# 先跳到第2步下载的nginx源码中
cd /opt/soft/nginx-1.15.1
./configure --prefix=/usr/local/nginx --add-module=../fastdfs-nginx-module-1.20/src
make && make install

# 查看编译选项
/usr/local/nginx/sbin/nginx -V

nginx version: nginx/1.15.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-28) (GCC) 
configure arguments: --prefix=/usr/local/nginx --add-module=../fastdfs-nginx-module-1.20/src
```

如果执行make命令提示如下错误时
```
In file included from ../fastdfs-nginx-module-1.20/src/ngx_http_fastdfs_module.c:6:
../fastdfs-nginx-module-1.20/src/common.c:26:10: fatal error: 
      'fastdfs/fdfs_define.h' file not found
#include "fastdfs/fdfs_define.h"
         ^~~~~~~~~~~~~~~~~~~~~~~
```
造成原因是找不到fastdfs/fdfs_define.h文件，打开`fastdfs-nginx-module-1.20/src/config`文件修改`ngx_module_incs`和`CORE_INCS`这两个值
值修改为本机的include目录，比如我机器fastdfs include文件放在/usr/include/fastdfs目录，那将`/usr/include`
添换原来的`ngx_module_incs`和`CORE_INCS`，并重新执行[1.2.3](#1.2.3)即可。

### 3.4 配置mod_fastdfs.conf
复制 fastdfs-nginx-module 源码中的配置文件到/etc/fdfs 目录， 并修改

```
sudo cp ../fastdfs-nginx-module-1.20/src/mod_fastdfs.conf /etc/fdfs/
sudo vim /etc/fdfs/mod_fastdfs.conf
```
修改如下配置，其它默认
```
# 配置tracker地址
tracker_server=tracker.server1:22122
tracker_server=tracker.server2:22122
# 如果文件ID的uri中包含/group**，则要设置为true
url_have_group_name = true
# Storage 配置的store_path0路径，必须和storage.conf中的一致
store_path0=/opt/fastdfs/storage
```

复制 FastDFS 的部分配置文件到/etc/fdfs 目录
```
# 配置位置放在fastdfs源文件下conf目录
cd /opt/soft/fastdfs-5.12/conf
sudo cp anti-steal.jpg http.conf mime.types /etc/fdfs/
```

### 3.5 修改nginx.conf
```
cd /usr/local/nginx
# 配置配置文件
cp ./conf/nginx.conf ./conf/nginx.no-fdfs.conf
vim conf/nginx.conf
# 将原有fastdfs下面加`ngx_fastdfs_module`即可
server {
  listen 8888;
  server_name fastdfs;
  location /group1/M00 {
     alias /opt/fastdfs/storage/data;
     ngx_fastdfs_module; # 添加这一行
  }
}

# 测试
./sbin/nginx -t
# 输出如下信息即表示成功
ngx_http_fastdfs_set pid=21069  # 模块添加成功
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful

# 重启
./sbin/nginx -s reload
```

如果有多台storage机器，那么每台机器都需要装nginx和ngx_fastdfs_module模块，步骤同上