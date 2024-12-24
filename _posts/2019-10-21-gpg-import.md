---
layout: post
title: 发布项目到Maven中央仓库(换机器后)
categories: Blog
description: 解决切换机器或换系统后无法Release项目到Maven中央仓库
keywords: sonatype, oss.sonatype, gpg, Maven中央仓库 
---

## 前言

工作原因开发环境由win10切换到Mac，原来维护的两个开源项目[EEC](https://github.com/wangguanquan/eec)和[bree-mybatis-maven-plugin](https://github.com/wangguanquan/bree-mybatis-maven-plugin)
想升级结果推到sonatype后验证未通过，提示"sonatype Failed: Signature Validation"，原因好理解肯定是gpg签名不通过，因为在deploy过程中未提示
我输入密码...

## 修复

#### 查找原有公钥

使用命令`gpg --search-keys`查找自己的公钥

```
bogon:~ $ gpg --search-keys guanquan.wang@yandex.com
gpg: data source: https://keys.openpgp.org:443
(1)	guanquan.wang (2048mail) <guanquan.wang@yandex.com>
	  2048 bit RSA key 5867FE8781F69B9F, 创建于：2019-10-21
Keys 1-1 of 1 for "guanquan.wang@yandex.com".  输入数字以选择，输入 N 翻页，输入 Q 退出 >
```

我们可以看到一个公钥RSA key 5867FE8781F69B9F

#### 从公钥服务器上导入密钥

使用命令`gpg --receive-keys`从公钥服务器上导入密钥


```
bogon:~ $ gpg --receive-keys 5867FE8781F69B9F
gpg: 密钥 5867FE8781F69B9F：公钥 “guanquan.wang (2048mail) <guanquan.wang@yandex.com>” 已导入
gpg: 处理的总数：1
gpg:               已导入：1
```

这里的密钥就是上面查找结果中的RSA key。

#### 从原机器上导出私钥

使用命令`gpg --export-secret-keys`

```
gpg --export-secret-keys --armor --output ./Desktop/guanquan.wang@yandex.com.key
```

#### 导入私钥

将上面得到的guanquan.wang@yandex.com.key复制到新机器上，然后使用命令`gpg --import`导入

```
gpg --import ./guanquan.wang@yandex.com.key
gpg: 密钥 5867FE8781F69B9F：“guanquan.wang <guanquan.wang@yandex.com>” 未改变
gpg: 密钥 5867FE8781F69B9F：私钥已导入
gpg: 处理的总数：1
gpg:              未改变：1
gpg:       读取的私钥：1
gpg:   导入的私钥：1
```

出现上面的信息说明导出成功了，通过上面的操作我们已经完成了所有迁移，重新用Maven命令打包就出弹出GPG的密码输入框了。

如果机器上有多个公钥，你还可以在pom.xml文件里指定公钥。

```
<!-- pom.xml -->

<profile>
    <properties>
        <gpg.keyname>5867FE8781F69B9F</gpg.keyname>
    </properties>
</profile>
```

GPG相关命令
### 生成 gpg 密钥
gpg --gen-key

### 生成吊销证书
gpg --gen-revoke 5867FE8781F69B9F

### 列出所有 gpg 公钥
gpg --list-keys

### 列出所有 gpg 私钥
gpg --list-secret-keys

### 删除 gpg 公钥
gpg --delete-keys 5867FE8781F69B9F

### 删除 gpg 私钥
gpg --delete-secret-keys 5867FE8781F69B9F

### 输出 gpg 公钥 ascii
gpg --armor --output public.key --export 5867FE8781F69B9F

### 输出 gpg 私钥 ascii
gpg --armor --output private.key --export-secret-keys 5867FE8781F69B9F

### 上传 gpg 公钥
gpg --send-keys 5867FE8781F69B9F --keyserver

### 查看 gpg 公钥指纹
gpg --fingerprint 5867FE8781F69B9F

### 导入 gpg 密钥(导入私钥时会自动导入公钥)
gpg --import private.key

### 加密文件
gpg --recipient 5867FE8781F69B9F --output encrypt.file --encrypt origin.file

### 解密文件
gpg --output origin.file --decrypt encrypt.file

### 文件签名，生成二进制的 gpg 文件
gpg --sign file.txt

### 文件签名，生成文本末尾追加 ASCII 签名的 asc 文件
gpg --clearsign file.txt

### 文件签名，生成二进制的 sig 文件
gpg --detach-sign file.txt

### 文件签名，生成 ASCII 格式的 asc 文件
gpg --detach-sign file.txt

### 签名并加密
gpg --local-user 5867FE8781F69B9F --recipient 5867FE8781F69B9F --armor --sign --encrypt file.txt

### 验证签名
gpg --verify file.txt.asc file.txt

### 延期
> gpg 也是使用主密钥和子密钥结合加密的
> pub 和 sub 分别是主公钥和子公钥
> sec 和 ssb 分别是主私钥和子私钥
> 如果有多个子密钥，会显示更多的 sub 和 ssb
> 一个主密钥可以绑定多个子密钥，平时加密解密使用的都是子密钥

`gpg --edit-key guanquan.wang@yandex.com`

sec  rsa2048/5867FE8781F69B9F
创建于：2019-06-10  过期于：2023-09-10  可用于：SC  
信任度：未知        有效性：已过期
ssb  rsa2048/1208CFFABF3A8A6D
创建于：2019-06-10  过期于：2021-06-09  可用于：E   
[ 过期 ] (1). guanquan.wang <guanquan.wang@yandex.com>

### 指定子密钥，不指定则为主密钥
gpg> key 1
sec  rsa4096/1208CFFABF3A8A6D
创建于：2019-06-10  过期于：2021-06-09  可用于：E   
信任度：未知        有效性：未知

### 更新过期时间
gpg> expire

将要变更子密钥的过期时间。
请设定这个密钥的有效期限。
0 = 密钥永不过期
<n>  = 密钥在 n 天后过期
<n>w = 密钥在 n 周后过期
<n>m = 密钥在 n 月后过期
<n>y = 密钥在 n 年后过期
密钥的有效期限是？(0) 1y
密钥于 四 11/13 19:28:25 2025 CST 过期
这些内容正确吗？ (y/N) y

sec  rsa4096/5867FE8781F69B9F
创建于：2019-06-10  有效至：2025-11-13  可用于：SC  
信任度：未知        有效性：未知
ssb  rsa2048/1208CFFABF3A8A6D
创建于：2019-06-10  过期于：2021-06-09  可用于：E   
[ 未知 ] (1). guanquan.wang <guanquan.wang@yandex.com>

### 保存
gpg> save