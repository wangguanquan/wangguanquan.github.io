---
layout: post
title: EEC vs easyexcel
categories: Excel
description: Excel操作工具EEC和alibaba开源easyexcel工具的性能对比
keywords: excel, eec, easyexcel
---

本文主要对比Excel操作的两款工具easyexcel和eec的性能和内存，以下列出测试环境

硬件:
- CPU: 2.7 GHz Intel Core i5 (双核)
- 内存: 8 GB 1867 MHz DDR3
- 硬盘: SSD
- 系统: macOS 10.12.6

测试版本:
- easyexcel: 2.1.6
- eec: 0.4.2

为了避免内存对速度的影响，测试过程中添加jvm参数`-Xmx64m -Xms1m`，限制最大堆内存为64MB，后面会有放开内存限制的测试，这里透露一下内存不是影响速度的关键因素。
