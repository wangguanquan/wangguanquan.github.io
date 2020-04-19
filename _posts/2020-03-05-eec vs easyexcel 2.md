---
layout: post
title: EEC vs easyexcel(二)
categories: Excel
description: Excel操作工具EEC和alibaba开源easyexcel工具的性能对比
keywords: excel, EEC, easyexcel
---

[上一篇](/excel/2020/03/03/eec-vs-easyexcel.html)我们对比了java操作Excel的两款工具easyexcel和EEC的使用便利性和功能对比，本篇主要比较两款工具读写时的性能和内存。

本次测试在高配和低配两台机器上进行

## 1. 低配机测试

硬件:
- CPU: 2.7 GHz Intel Core i5 (双核)
- 内存: 8 GB 1867 MHz DDR3
- 硬盘: SATA 128G(35G可用)
- 系统: macOS 10.12.6

测试版本:
- easyexcel: 2.1.6
- EEC: 0.4.2

为了避免内存对速度的影响，测试过程中添加jvm参数`-Xmx64m -Xms1m`，限制最大堆内存为64MB，后面会有放开内存限制的测试，这里透露一下内存不是影响速度的关键因素。

从BIFF5以后Office就使用SharedString方式保存字符串，使用共享字符串可以达到压缩文件的目的，但是POI使用的是`inlineStr`方式写字符串，easyexcel底层是POI所以自然的继承了这一方式，
EEC默认也是使用inlineStr方式，可以使用注解`@ExcelColumn(share = true)`来使用SharedString方式。

测试实体由29个字段组成，分别由int,long,Date,double各一个，加上25个字符串，按1000条记录进行一次分片。
下面对比两个工具分别对1w，5w, 10w, 50w, 100w数据的读写，所有测试代码已上传到github，地址[eec-poi-compares](https://github.com/wangguanquan/eec-poi-compares)

### 1.1 读写性能对比

测试之前先进行1w~100w数据空测，模拟取数据的时间以此为基准来测试Excel读写时间。

这是测试结果截图

![](/images/posts/testcut.png)

为了抓取jvm参数我在test100w方法里休眠了20秒钟，从测试数据上看100w空转时间为3秒。

下面我们用图表来直观展示:

1〜100w行数据写文件（时间为秒）

![写文件对比](/images/posts/chat1.png)

读取1~100w行数据（时间为秒）

![读文件对比](/images/posts/chat2.png)

通过上图可以简单总结: 写文件EEC平均比easyexcel快1倍以上，读文件EEC平均比easyexcel快约2倍左右。*注意: 这个结论只适用于配置较低的机器，并不适用于高主频高性能SSD的高配机*

### 1.2 堆内存对比

我们限制了jvm堆大小为64M，以下是运行过程中堆的波动情况，鼠标标识的位置(21:44:11)左边是easyexcel使用堆内存情况，右边是EEC使用堆内存情况。
两个工具均能在64MB的限制下完成测试，easyexcel最高占用57.7MB，EEC最高49.3MB略低。

![堆使用情况](/images/posts/heap.png)

*下面详细日志有准确的时间序列*

### 1.3 文件大小对比(单位为MB)

文件大小影响不太重要，某些情况下如网络传输会有一些影响。我们测试数据中有Date类型，EEC使用int值或者double值保存时间数据，这与Office Excel的处理方式相同。最终两个文件大小非常接近，如果全部是字符串可能会得到另一种结论

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 1.7 | 8.6 | 17.2 | 86.6 | 173.4
Easy excel | 1.7 | 8.7 | 17.4 | 87.7 | 175.7


### 1.4 更细节的分析

下面贴出测试日志进一步分析，为了减少篇幅我对日志进行了删减仅保留主要信息

```
2020-03-05 21:34:44.412 [LargeExcelTest:200] - Easy-excel start to write...
2020-03-05 21:34:49.922 [LargeExcelTest:206] - 0 fill success.
2020-03-05 21:35:33.660 [LargeExcelTest:206] - 500 fill success.
2020-03-05 21:36:17.902 [LargeExcelTest:206] - 999 fill success.
2020-03-05 21:37:09.910 [LargeExcelTest:209] - Easy-excel write finished. used: 145481
2020-03-05 21:37:09.911 [LargeExcelTest:226] - Easy-excel start to read...
2020-03-05 21:37:35.376 [LargeDataListener:19] - Already read:100000
2020-03-05 21:39:02.767 [LargeDataListener:19] - Already read:500000
2020-03-05 21:40:35.470 [LargeDataListener:19] - Already read:1000000
2020-03-05 21:40:35.471 [LargeDataListener:25] - Large row count:1000000
2020-03-05 21:40:35.477 [LargeExcelTest:230] - Easy-excel read finished. used: 205566

2020-03-05 21:44:11.408 [LargeExcelTest:213] - EEC start to write...
2020-03-05 21:44:11.765 [LargeExcelTest$1:218] - 0 fill success.
2020-03-05 21:44:25.623 [LargeExcelTest$1:218] - 500 fill success.
2020-03-05 21:44:38.838 [LargeExcelTest$1:218] - 1000 fill success.
2020-03-05 21:45:18.211 [LargeExcelTest:222] - EEC write finished. used: 66803
2020-03-05 21:45:18.212 [LargeExcelTest:234] - EEC start to read...
2020-03-05 21:45:27.765 [LargeExcelTest:238] - Worksheet [Sheet1] dimension: A1:AC1000001
2020-03-05 21:45:32.812 [LargeExcelTest:242] - Reading 100000 rows
2020-03-05 21:46:19.416 [LargeExcelTest:242] - Reading 1000000 rows
2020-03-05 21:46:19.417 [LargeExcelTest:246] - Data rows: 1000000
2020-03-05 21:46:19.556 [LargeExcelTest:250] - EEC read finished. used: 61343
```

第一段日志是easyexcel导出100w数据，我们看到从21:34:44.412开始到21:36:17.902写完100w数据，再到21:37:09.910完成数据压缩，写数据用时93.49秒，压缩用时52.008秒。
而EEC写100w用时27.43秒(实际比这个大一点，因为EEC使用pull方式拉数据，我们无法知道最后1000条数据完成的实际时间)，压缩用了39.373秒，写数据的速度远超easyexcel，我们进一步列出所有写数据和压缩时间

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC写数据 | 0.252 | 1.118 | 2.144 | 10.867 | 27.43
EEC压缩数据 | 0.364 | 1.777 | 3.399 | 16.816 | 39.373
Easy excel写数据 | 0.935 | 4.172 | 7.944 | 37.38 | 93.49
Easy excel压缩数据 | 0.7 | 2.586 | 4.409 | 23.174 | 52.008

上图可以发现EEC写数据的速度远大于easyexcel，压缩速度略快于后者(EEC的压缩等级为5，文件可能较easyexcel大)。

## 2. 高配机测试

硬件:
- CPU: 3 GHz Intel Core i5 (6核)
- 内存: 16 GB 3000 MHz DDR4
- 硬盘: Samsung SSD 970 PRO 512GB（267G可用）
- 系统: macOS 10.14.4

测试代码与上面的完全一样，所以这里不写细节直接上最终的对比图。

### 2.1 限制内存64MB测试

![64MB内存测试截图](/images/posts/hi_64test.png)

写文件测试

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.306 | 1.530 | 3.680 | 14.807 | 32.939
Easy excel | 0.682 | 3.443 | 6.996 | 34.415 | 78.789
倍数 | 223% | 225% | 190% | 232% | 239%

读文件测试

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.452 | 2.201 | 4.359 | 20.917 | 41.929
Easy excel | 1.100 | 4.953 | 9.875 | 49.848 | 98.390
倍数 | 243% | 225% | 227% | 238% | 235%

内存和CPU使用情况，同样鼠标位置左边为easyexcel右边为EEC

![](/images/posts/hi_64jvm.png)

![](/images/posts/hi_64cpu.png)

### 2.2 不限制内存测试

![不限制内存测试截图](/images/posts/hi_gt64test.png)

写文件测试

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.320 | 1.506 | 3.118 | 14.922 | 37.280
Easy excel | 0.879 | 3.304 | 6.834 | 33.375 | 76.182
倍数 | 275% | 219% | 219% | 224% | 204%

读文件测试

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.451 | 2.211 | 4.377 | 20.766 | 43.117
Easy excel | 1.194 | 5.593 | 10.494 | 55.605 | 99.653
倍数 | 265% | 253% | 240% | 268% | 231%

内存和CPU使用情况，同样鼠标位置左边为easyexcel右边为EEC

![](/images/posts/hi_gt64_jvm.png)

![](/images/posts/hi_gt64_cpu.png)

可以看出无论在高配机和低配机上，EEC的读写速度均超过easyexcel，且在一个稳定范围内。


## 3. 其它测试

最后我在低配机上做了32MB和16MB的内存极限测试，其中测试16MB时将分批数量改为100条，32MB未做修改，所有文件均为inlineStr方式写字符串。

### 3.1 32MB内存

![32MB内存](/images/posts/lo_32jvm.png)

写文件测试

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.578 | 2.334 | 4.586 | 25.918 | 83.500
Easy excel | 1.331 | 6.336 | 13.671 | 65.490 | 143.368
倍数 | 230% | 271% | 298% | 253% | 172%

读文件测试

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.658 | 2.948 | 6.280 | 25.277 | 53.847
Easy excel | 1.699 | 9.202 | 16.215 | 78.261 | 163.702
倍数 | 258% | 312% | 258% | 310% | 304%

32MB完全没问题，与64MB相比也没有明显变慢，所以内存大小并不能明显影响两个工具的性能

### 3.2 16MB内存

easyexcel并没有完成16MB的写文件测试，确切的说未完成10~100W数据写Excel文件测试，后面会贴出部分日志。所以这里我先测试EEC的读写然后再测试easyexcel在16MB限制下的读文件测试。

#### 3.2.1 EEC读写测试

![16MB内存](/images/posts/lo_16eec_jvm.png)

![16MBCPU使用](/images/posts/lo_16eec_cpu.png)

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC写 | 0.576 | 2.196 | 4.824 | 22.360 | 46.903
EEC读 | 0.672 | 2.867 | 6.286 | 25.261 | 51.137


#### 3.2.2 easyexcel测试

上面已经说了easyexcel在写到7万数据左右时就抛错误"Exception in thread "RMI TCP Connection(idle)" java.lang.OutOfMemoryError: GC overhead limit exceeded"，同时CPU掉到0终止运行

下面是测试截图

![](/images/posts/lo_16m_easy_test.png)

CPU截图

![](/images/posts/lo_16easy_cpu.png)

内存截图

![](/images/posts/lo_16easy_jvm.png)

CPU波动很大且多次爆满，内存回收活动一直占用较高CPU，有时爆到100%。相比下EEC的CPU使用就非常平稳。

可以看到在16MB限制下Easyexcel内存一直保持在12~13MB的高位，说明一次GC仅回收少量内存，导致程序不停GC使CPU的占用爆满。回看上面EEC的内存截图它一次回收可以释放较多的内存，也就不用一直GC占CPU了。

Easyexcel部分测试日志

```
2020-03-08 10:56:20.463 INFO [main][LargeExcelTest:200] - Easy-excel start to write...
2020-03-08 10:56:23.301 INFO [main][LargeExcelTest:206] - 0 fill success.
2020-03-08 10:56:23.341 INFO [main][LargeExcelTest:206] - 1 fill success.
2020-03-08 10:56:23.382 INFO [main][LargeExcelTest:206] - 2 fill success.

2020-03-08 10:57:17.462 INFO [main][LargeExcelTest:206] - 690 fill success.
2020-03-08 10:57:17.832 INFO [main][LargeExcelTest:206] - 691 fill success.
2020-03-08 10:57:18.087 INFO [main][LargeExcelTest:206] - 692 fill success.
2020-03-08 10:58:55.359 INFO [main][LargeExcelTest:206] - 693 fill success. <-
2020-03-08 10:58:56.282 INFO [main][LargeExcelTest:206] - 694 fill success.
2020-03-08 10:58:56.982 INFO [main][LargeExcelTest:206] - 695 fill success.
2020-03-08 10:58:57.337 INFO [main][LargeExcelTest:206] - 696 fill success.
2020-03-08 11:00:34.404 INFO [main][LargeExcelTest:206] - 697 fill success. <-
2020-03-08 11:00:35.214 INFO [main][LargeExcelTest:206] - 698 fill success.
2020-03-08 11:00:35.642 INFO [main][LargeExcelTest:206] - 699 fill success.
2020-03-08 11:01:36.603 INFO [main][LargeExcelTest:206] - 700 fill success. <-
2020-03-08 11:01:40.382 INFO [main][LargeExcelTest:206] - 701 fill success.
2020-03-08 11:01:40.830 INFO [main][LargeExcelTest:206] - 702 fill success.
2020-03-08 11:03:21.814 INFO [main][LargeExcelTest:206] - 703 fill success. <-
2020-03-08 11:03:54.896 INFO [main][LargeExcelTest:206] - 704 fill success. <-
2020-03-08 11:04:27.861 INFO [main][LargeExcelTest:206] - 705 fill success. <-
2020-03-08 11:05:02.006 INFO [main][LargeExcelTest:206] - 706 fill success. <-
2020-03-08 11:05:23.921 INFO [main][LargeExcelTest:206] - 707 fill success. <-
2020-03-08 11:05:49.776 INFO [main][LargeExcelTest:206] - 708 fill success. <-
2020-03-08 11:07:17.917 INFO [main][LargeExcelTest:206] - 709 fill success. <-
2020-03-08 11:08:17.004 INFO [main][LargeExcelTest:206] - 710 fill success. <-
Exception in thread "RMI TCP Connection(idle)" java.lang.OutOfMemoryError: GC overhead limit exceeded
Exception in thread "RMI TCP Connection(idle)" java.lang.OutOfMemoryError: GC overhead limit exceeded
```

我们可以发现刚开始速度还很快，每100条记录只需要40ms，随着数据不断增加速度也就不断减少，应该是CPU一直被GC占用的原因。

Easyexcel读文件是正常的，与EEC的性能差别与32MB，64MB比较中的差别相当，这里就不贴图了。

## 4. 探底EEC最低使用内存

为了探底EEC最低使用内存，我将分片调到10，也就是说每10条写一次数据(EEC内部写文件的最小单位是32，这里调到10条并没有意义)，多次调整后最终的下限为6M。

6MB限制下内存波动

![6M_RAM](/images/posts/lo_6m_eec_jvm.png)

6MB限制下CPU波动

![6M_CPU](/images/posts/lo_6m_eec_cpu.png)

6MB限制下读写时间

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC写 | 0.735 | 3.3 | 6.68 | 29.724 | 58.3
EEC读 | 0.741 | 3.145 | 6.27 | 27.17 | 62.354


*注意: 上面所有测试均使用inlineStr方式写字符串，SharedString方式在EEC 0.4.2版上有BUG，Issue编号[#110](https://github.com/wangguanquan/eec/issues/110)，后续版本更新后再列出性能测试*

## 5. 后记

1. 两个工具均为单线程、高IO设计，多核心不会提高速度，高主频和一块好SSD能显著提升速度。
2. 本次测试均在macOS上完成，缺少windows系统测试
3. 希望有更多人在实际项目上使用，以帮助EEC改进未知BUG