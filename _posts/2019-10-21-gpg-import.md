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