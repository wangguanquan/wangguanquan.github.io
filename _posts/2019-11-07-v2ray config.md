---
layout: post
title: 安装并配置v2ray
categories: Blog
description: 配置v2ray
keywords: v2ray
---

## 安装v2ray

首先需要海外或香港节点服务器

```
wget https://install.direct/go.sh
bash go.sh
```

## 设置SSL/TLS协议

**必须要有域名**

### 证书生成
TLS 是证书认证机制，所以使用 TLS 需要证书，证书也有免费付费的，同样的这里使用免费证书，证书认证机构为 Let's Encrypt。 证书的生成有许多方法，这里使用的是比较简单的方法：使用 acme.sh 脚本生成，本部分说明部分内容参考于acme.sh README。
证书有两种，一种是 ECC 证书（内置公钥是 ECDSA 公钥），一种是 RSA 证书（内置 RSA 公钥）。简单来说，同等长度 ECC 比 RSA 更安全,也就是说在具有同样安全性的情况下，ECC 的密钥长度比 RSA 短得多（加密解密会更快）。但问题是 ECC 的兼容性会差一些，Android 4.x 以下和 Windows XP 不支持。只要您的设备不是非常老的老古董，强烈建议使用 ECC 证书。
以下将给出这两类证书的生成方法，请大家根据自身的情况自行选择其中一种证书类型。

证书生成只需在服务器上操作。


#### 安装 acme.sh
执行以下命令，acme.sh 会安装到 ~/.acme.sh 目录下。

```
curl  https://get.acme.sh | sh
```

安装过程中如果出错请根据提示安装依懒

#### 使用 acme.sh 生成证书

```
sudo ~/.acme.sh/acme.sh --issue -d domain.com --standalone -k ec-256
```

-k 表示密钥长度，后面的值可以是 ec-256 、ec-384、2048、3072、4096、8192，带有 ec 表示生成的是 ECC 证书，没有则是 RSA 证书。在安全性上 256 位的 ECC 证书等同于 3072 位的 RSA 证书。
-d 指定你自己的域名

证书更新

由于 Let's Encrypt 的证书有效期只有 3 个月，因此需要 90 天至少要更新一次证书，acme.sh 脚本会每 60 天自动更新证书。也可以手动更新。

手动更新 ECC 证书，执行：

```
sudo ~/.acme.sh/acme.sh --renew -d domain.com --force --ecc
```

如果是 RSA 证书则执行：

```
sudo ~/.acme.sh/acme.sh --renew -d domain.com --force
```

由于本例中将证书生成到`/etc/v2ray/`文件夹，更新证书之后还得把新证书生成到`/etc/v2ray`。

#### 安装证书和密钥
将证书和密钥安装到 /etc/v2ray 中：

```
sudo ~/.acme.sh/acme.sh --installcert -d domain.com --fullchainpath /etc/v2ray/v2ray.crt --keypath /etc/v2ray/v2ray.key --ecc
```

## 配置v2ray

### 服务端

```
vim /etc/v2ray/config.json 
```

```
{
  "inbounds": [{
    "port": 443, // 建议使用此端口
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "43f7489e-f477-4a5c-be9d-5b3794f3f70b", // 生成一个新的UUID填进来
          "level": 1,
          "alterId": 64
        }
      ]
    },
    "streamSettings": {
        "network": "tcp",
        "security": "tls", // security 要设置为 tls 才会启用 TLS
        "tlsSettings": {
          "certificates": [
            {
              "certificateFile": "/etc/v2ray/v2ray.crt", // 证书文件
              "keyFile": "/etc/v2ray/v2ray.key" // 密钥文件
            }
          ]
        }
      }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  },{
    "protocol": "blackhole",
    "settings": {},
    "tag": "blocked"
  }],
  "routing": {
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "blocked"
      }
    ]
  }
}
```

## 客户端安装

在[此地址](https://github.com/v2ray/v2ray-core/releases)下载各端的v2ray。

下载完成后进入目录，然后修改config.json文件

```
{
  "inbounds": [
    {
      "port": 1080, // 代理端口，socks5代理设置为此值
      "protocol": "socks",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"] // 加入tls
      },
      "settings": {
        "auth": "noauth"
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "domain.com", // tls 需要域名，所以这里应该填自己的域名
            "port": 443, // 端口与服务器v2ray设置相同
            "users": [
              {
                "id": "23ad6b10-8d1a-40f7-8ad0-e3e35cd38297", // 端口与服务器v2ray设置相同
                "alterId": 64
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls" // 客户端的 security 也要设置为 tls
      }
    }
  ]
}
```

## 设置代理

代理服务器地址填写`127.0.0.1`端口填写inbounds.port值，我这里是1080。

配置完上面的项目你就可以科学上网了。
