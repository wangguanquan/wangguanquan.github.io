---
layout: post
title: Burp Suite(v2020.2)实操介绍
categories: 测试
description: Burp Suite渗透测试
keywords: Burp Suite,渗透测试,测试
---

Burp Suite是web应用程序渗透测试集成平台。从应用程序攻击表面的最初映射和分析，到寻找和利用安全漏洞等过程，所有工具为支持整体测试程序而无缝地在一起工作。

平台中所有工具共享同一robust框架，以便统一处理HTTP请求、持久性、认证、上游代理、日志记录、报警和可扩展性。Burp Suite允许攻击者结合手工和自动技术去枚举、分析、攻击Web应用程序。

Burp Suite具有以下功能：
- Proxy——是一个拦截HTTP/S的代理服务器，作为一个在浏览器和目标应用程序之间的中间人，允许你拦截，查看，修改在两个方向上的原始数据流。
- Target——直译为"靶/目标"，它合并了原来的"Spider"和"Scanner"，集成抓虫和漏洞扫描功能。
- Intruder——是一个定制的高度可配置的工具，对web应用程序进行自动化攻击，如：枚举标识符，收集有用的数据，以及使用fuzzing 技术探测常规漏洞。
- Repeater——是一个靠手动操作来补发单独的HTTP 请求，并分析应用程序响应的工具。
- Sequencer——是一个用来分析那些不可预知的应用程序会话令牌和重要数据项的随机性的工具。
- Decoder——是一个进行手动执行或对应用程序数据者智能解码编码的工具。
- Comparer——是一个实用的工具，通常是通过一些相关的请求和响应得到两项数据的一个可视化的“差异”。
- Extender——扩展/插件管理，在这里可以添加管理插件

## 1. 程序启动

程序启动后有三个选项，Temporary project，New project on disk和Open existing project。

- Temporary project——临时项目，会话结束后会被清除。如果只是临时测试一个网站可以使用临时项目。
- New project on disk——创建磁盘项目，项目每隔一段时间会自动保存当前状态，下次可以通过Open existing project重新打开。

![](/images/posts/burp/5ec9d39d2c815.png)

第二步是选择配置，一般默认就好了
![](/images/posts/burp/5ec9d59341409.png)

## 2. 实操

启动后映入眼帘的就是下面这个窗口，差不多就是我们上面列出来的功能，每个功能对应一个Tab标签
![](/images/posts/burp/5ec9d6110cf9b.png)

要对一个网站进行渗透测试第一步是设置代理，因为Burp Suite是通过代理来拦截请求。当然也可以在Dashboard中"New scan“主动抓取目标网站

### 2.1 设置代理

Burp Suite代理端口默认是8080，你可以在Proxy中设置其它端口。

![](/images/posts/burp/5ec9d879afc9f.png)

如上图，在Proxy -> Options中有详细的代理设置，比如我这里勾选了一个正则用来过滤图片等静态资源。这是App中的设置，另外还需要在系统中配置此代理，以下拿macOS系统为例。打开“系统偏好设置” -> “网络” -> “以太网” -> “高级” -> "代理” -> “网页代理”，选中网页代理，配置上代理地址就可以了。

![](/images/posts/burp/5ec9d9fa73aaa.png)

*在拦截前请确保Proxy为"Intercept is on"状态。*

### 2.2 拦截请求

下面我们访问http://192.168.1.153:8080/ 为例，在浏览器访问该地址时浏览器会被夯住，返回Burp Suite会出现下面这个面板，让我们选择前进还是丢弃此次请求。
![](/images/posts/burp/5ec9df9162333.png)

Burp Suite会拦截所有请求，对于我们关注的请求选择Forword其它请求选择Drop。

*在v2020.2中我们可以关闭代理*

### 2.3. 扫描

通过上面的操作，我们可以拿到需要扫描的目标请求，然后我们就可以在Target页中对目标网站进行扫描了。
![](/images/posts/burp/5ec9e5c3968cb.png)

扫描前我们可以把目标地址加到"Scope"中，过滤掉不关注的目标，使界面更清爽

![](/images/posts/burp/5ec9e70d54e46.png)

点击顶部的Filter过滤，勾选"Show only in-scope items"
![](/images/posts/burp/5ec9e707aa892.png)

Burp Suite扫描分两种：主动扫描和被动扫描

- 主动扫描（Active Scanning）

当使用主动扫描模式时，Burp 会向应用发送新的请求并通过payload验证漏洞。这种模式下的操作，会产生大量的请求和应答数据，直接影响系统的性能，通常使用在非生产环境。它对下列的两类漏洞有很好的扫描效果：
1. 客户端的漏洞，像XSS、Http头注入、操作重定向；
2. 服务端的漏洞，像SQL注入、命令行注入、文件遍历。

对于第一类漏洞，Burp在检测时，会提交一下input域，然后根据应答的数据进行解析。在检测过程中，Burp会对基础的请求信息进行修改，即根据漏洞的特征对参数进行修改，模拟人的行为，以达到检测漏洞的目的。 对于第二类漏洞，一般来说检测比较困难，因为是发生在服务器侧。比如说SQL注入，有可能是返回数据库错误提示信息，也有可能是什么也不反馈。Burp在检测过程中，采用各个技术来验证漏洞是否存在，比如诱导时间延迟、强制修改Boolean值，与模糊测试的结果进行比较，已达到高准确性的漏洞扫描报告。

- 被动扫描（Passive Scanning）

当使用被动扫描模式时，Burp不会重新发送新的请求，它只是对已经存在的请求和应答进行分析，这对系统的检测比较安全，尤其在你授权访问的许可下进行的，通常适用于生成环境的检测。一般来说，下列这些漏洞在被动模式中容易被检测出来：
1. 提交的密码为未加密的明文。
2. 不安全的Cookie的属性，比如缺少的HttpOnly和安全标志。
3. cookie的范围缺失。
4. 跨域脚本包含和站点引用泄漏。
5. 表单值自动填充，尤其是密码。
6. SSL保护的内容缓存。
7. 目录列表。
8. 提交密码后应答延迟。
9. session令牌的不安全传输。
10. 敏感信息泄露，像内部IP地址，电子邮件地址，堆栈跟踪等信息泄漏。
11. 不安全的ViewState的配置。
12. 错误或者不规范的Content-type指令。

虽然被动扫描模式相比于主动模式有很多的不足，但同时也具有主动模式不具备的优点，除了前文说的对系统的检测在我们授权的范围内比较安全外，当某种业务场景的测试，每测试一次都会导致业务的某方面问题时，我们也可以使用被动扫描模式，去验证问题是否存在，减少测试的风险。

### 2.4. 主动扫描

在Target -> Site map -> 右键目标scope 选择"Actively scan this host"
![](/images/posts/burp/5ec9e9ade2e17.png)

点击Dashboard就能看到扫描的进度和实时结果
![](/images/posts/burp/5ec9ea098086f.png)

### 2.5 SQL注入

扫描后如果出现“SQL injection"说明可能出现了SQL注入，如下图
![](/images/posts/burp/5ec9ec6706581.png)

此时我们可以使用sqlmap来进一步来验证

我们可以使用gason+sqlmap或‘SQLiPy Sqlmap Integration’插件，我们分别介绍两种方式的方法

- gason+sqlmap方式

gason是一个jar包，可以[点击这里](https://code.google.com/archive/p/gason/)下载，sqlmap可以在github[下载源码](https://github.com/sqlmapproject/sqlmap)

按照下图所示，通过jar包导入gason插件
![](/images/posts/burp/5ec9ffd5aa48d.png)
在Errors中没有报错误就表明插件安装成功
![](/images/posts/burp/5ec9fdf31a95d.png)

安装完插件我们就可以在具体的请求中进行SQL注入测试了
![](/images/posts/burp/5eca00647baa8.png)
![](/images/posts/burp/5eca00913f36e.png)

在弹出的窗口中我们指定sqlmap路径(在github下载的源码sqlmap/sqlmap.py)点击Run就可以进行测试了
![](/images/posts/burp/5eca016c0b499.png)

顶部会追加一个Tab页来显示结果

## 导出报告

做完所有测试后我们需要一份完整的报告，Burp Suite提供html和xml两种格式的报告

在Target页右键目标网站选择Issues -> Report issues for this host，然后按照提示一路操作即可。
![](/images/posts/burp/5eca02c2ee08f.png)