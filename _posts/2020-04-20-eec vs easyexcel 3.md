---
layout: post
title: EEC vs easyexcel(三)
categories: Excel
description: Excel操作工具EEC和alibaba开源easyexcel工具的性能对比
keywords: excel, EEC, easyexcel, xls, xlsx, 海量数据
---

[上一篇](/excel/2020/03/03/eec-vs-easyexcel-2.html)我们对比了easyexcel和EEC两款工具在innerStr模式下读写Excel文件的性能和内存，
本文是对比系列的最后一篇，主要对比两款工具在SharedString模式下的读写性能以及Excel 97~2003格式读取性能。由于easyexcel不支持SharedString模式
、EEC不支持xls格式写文件，所以本文只对比两个工具的读性能。

测试实体与上篇基本一致，只是添加了省/市/区3列文本，这3列值大概率重重，所有字符串均添加注解`@ExcelColumn(share = true)`强制使用SharedString模式。
具体测试细节可以查看[上一篇](/excel/2020/03/03/eec-vs-easyexcel-2.html)，测试代码[eec-poi-compares](https://github.com/wangguanquan/eec-poi-compares)

测试文件内容如下

nv | lv | dv | av | 省 | 市 | 区/市/县 | str4 | str5 | str6 | str7 | str8 | str9 | str10 | str11 | str12 | str13 | str14 | str15 | str16 | str17 | str18 | str19 | str20 | str21 | str22 | str23 | str24 | str25
--:|---:|---:|:--:|---|----|---------|------|------|------|------|------|------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------|-------
-836813678 | 3306314278648050000 | 0.29 | 2020-03-18 | 江苏省 | 南京市 | 栖霞区 | str4-0 | str5-0 | str6-0 | str7-0 | str8-0 | str9-0 | str10-0 | str11-0 | str12-0 | str13-0 | str14-0 | str15-0 | str16-0 | str17-0 | str18-0 | str19-0 | str20-0 | str21-0 | str22-0 | str23-0 | str24-0 | str25-0
-486185751 | -415549150152144000 | 0.04 | 2020-03-18 | 广东省 | 深圳市 | 龙岗区 | str4-1 | str5-1 | str6-1 | str7-1 | str8-1 | str9-1 | str10-1 | str11-1 | str12-1 | str13-1 | str14-1 | str15-1 | str16-1 | str17-1 | str18-1 | str19-1 | str20-1 | str21-1 | str22-1 | str23-1 | str24-1 | str25-1
-791958069 | 2872099291500680000 | 0.41 | 2020-03-18 | 湖北省 | 宜昌市 | 伍家岗区 | str4-2 | str5-2 | str6-2 | str7-2 | str8-2 | str9-2 | str10-2 | str11-2 | str12-2 | str13-2 | str14-2 | str15-2 | str16-2 | str17-2 | str18-2 | str19-2 | str20-2 | str21-2 | str22-2 | str23-2 | str24-2 | str25-2
60008236 | -3603793804803950000 | 0.54 | 2020-03-18 | 湖北省 | 黄石市 | 下陆区 | str4-3 | str5-3 | str6-3 | str7-3 | str8-3 | str9-3 | str10-3 | str11-3 | str12-3 | str13-3 | str14-3 | str15-3 | str16-3 | str17-3 | str18-3 | str19-3 | str20-3 | str21-3 | str22-3 | str23-3 | str24-3 | str25-3
-219411010 | 3595732915319120000 | 0.98 | 2020-03-18 | 江苏省 | 无锡市 | 锡山区 | str4-4 | str5-4 | str6-4 | str7-4 | str8-4 | str9-4 | str10-4 | str11-4 | str12-4 | str13-4 | str14-4 | str15-4 | str16-4 | str17-4 | str18-4 | str19-4 | str20-4 | str21-4 | str22-4 | str23-4 | str24-4 | str25-4

## 1. 测试机器配置

硬件:
- CPU: 3 GHz Intel Core i5 (6核)
- 内存: 16 GB 3000 MHz DDR4
- 硬盘: Samsung SSD 970 PRO 512GB（267G可用）
- 系统: macOS 10.14.4

测试版本:
- easyexcel: 2.1.6
- EEC: 0.4.6 
- EEC-E3-SUPPORT 0.4.6

## 2. xlsx格式读写

### 2.1. 64MB内存
*限制内存64MB测试*

先跑一个1w~100w的读写

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC(Write) | 0.586 | 2.738 | 5.394 | 26.380 | 58.51
EEC(Read) | 0.349 | 1.716 | 3.242 | 15.209 | 30.664
Easy excel(Read) | 1.109 | 5.468 | 11.35 | 55.55 | 112.187
倍数 | 318% | 319% | 350% | 365% | 366%

![64MB图表](/images/posts/eve3/hi_64chart.png)

CPU截图
![64MBCPU测试截图](/images/posts/eve3/hi_64cpu.png)

内存截图
![64MB内存测试截图](/images/posts/eve3/hi_64jvm.png)

*鼠标左边为EEC，右边为Easyexcel*

可见EEC的读取速度超过easyexcel2倍

### 2.2. 32MB内存
*限制内存32MB测试*

easyexcel并没有完成32MB内存测试，这是错误LOG

```$xslt
2020-04-26 16:41:02.569 INFO [main][org.ttzero.compares.LargeExcelTest:349] - Easy-excel start to read...
2020-04-26 16:41:02.607 INFO [main][org.ehcache.core.EhcacheManager:307] - Cache '0d5779c9-a1aa-412b-ada3-12469678e4c4' created in EhcacheManager.
2020-04-26 16:41:02.609 INFO [main][org.ehcache.core.EhcacheManager:307] - Cache '0d5779c9-a1aa-412b-ada3-12469678e4c4' created in EhcacheManager.
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x0000000103549937, pid=3772, tid=0x0000000000001d03
#
# JRE version: Java(TM) SE Runtime Environment (8.0_191-b12) (build 1.8.0_191-b12)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.191-b12 mixed mode bsd-amd64 compressed oops)
# Problematic frame:
# V  [libjvm.dylib+0x4c9937]
#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# An error report file with more information is saved as:
# /Users/guanquan.wang/workspace/eec-poi-compares/hs_err_pid3772.log
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
#

Process finished with exit code 134 (interrupted by signal 6: SIGABRT)
```

CPU截图
![32MBCPU测试截图](/images/posts/eve3/hi_32cpu.png)

内存截图
![32MB内存测试截图](/images/posts/eve3/hi_32jvm.png)

*鼠标左边为EEC，右边为Easyexcel*

easyexcel内存一直处于满载状态，且GC占用CPU很高。

### 2.3. 探底EEC最低使用内存

经多次测试我最终运行在10MB处完成1~100w测试，测试过程中我发现数据量并不能直接影响内存。

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC读 | 0.391 | 1.887 | 3.695 | 16.965 | 34.719

内存波动

![10M_RAM](/images/posts/eve3/hi_10jvm.png)

CPU波动

![10M_CPU](/images/posts/eve3/hi_10cpu.png)

Easyexcel在10MB下抛OOM错误，

```
com.alibaba.excel.exception.ExcelAnalysisException: java.lang.OutOfMemoryError: GC overhead limit exceeded

	at com.alibaba.excel.analysis.ExcelAnalyserImpl.analysis(ExcelAnalyserImpl.java:122)
	at com.alibaba.excel.ExcelReader.readAll(ExcelReader.java:160)
	at com.alibaba.excel.read.builder.ExcelReaderBuilder.doReadAll(ExcelReaderBuilder.java:275)
	...
Caused by: java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.io.ObjectInputStream$BlockDataInputStream.readUTFBody(ObjectInputStream.java:3406)
	at java.io.ObjectInputStream$BlockDataInputStream.readUTF(ObjectInputStream.java:3226)
	at java.io.ObjectInputStream.readString(ObjectInputStream.java:1905)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1564)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:431)
```

## 3. xls格式读写

测试内容与上面完全一样，测试文件首先由EEC生成xlsx格式，再转为xls格式，代码如下

```
private void eecWritePaging(final String name, final int loop) throws IOException {
    new Workbook().addSheet(new ListSheet<LargeSharedData>() {
        int n = 0;
        @Override
        public List<LargeSharedData> more() {
            return n++ < loop ? createSharedData() : null;
        }
    }).setWorkbookWriter(new XMLWorkbookWriter() {
        @Override
        protected IWorksheetWriter getWorksheetWriter(Sheet sheet) {
            return new XMLWorksheetWriter(sheet) {
                /**
                 * xls每个Worksheet最大包含65536行x256列，所以这里设置分页
                 * 参数为{@code 65536-1}(去除第一行的表头)
                 * 
                 * @return 每页最大行限制
                 */
                @Override
                public int getRowLimit() {
                    return (1 << 16) - 1;
                }
            };
        }
    }).writeTo(defaultTestPath.resolve(name + ".xlsx"));
}
```

最终的测试文件如下：

 File                | Size |
---------------------| ----:|
 eec shared 1w.xls   |  6.9M|
 eec shared 1w.xlsx  |  1.8M|
 eec shared 5w.xls   |   35M|
 eec shared 5w.xlsx  |  9.1M|
 eec shared 10w.xls  |   71M|
 eec shared 10w.xlsx |   18M|
 eec shared 50w.xls  |  487M|
 eec shared 50w.xlsx |   92M|
 eec shared 100w.xls |  978M|
 eec shared 100w.xlsx|  185M|

我们可以看到xls格式最终文件比xlsx格式文件大得多，所以强烈建议在项目中将xlsx格式设置为默认的数据导出格式，
文件大小对比xlsx格式优于xls和csv格式，对于网终传输更友好。（csv格式在相同数据量下文件大小分别为(1~100w)3.1M, 16M, 33M, 175M, 352M）

![File size](/images/posts/eve3/file-size-c.png)

进入主题吧

### 3.1 64MB内存

先跑一波64MB内存限制下1~100w的读操作

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.198 | 0.603 | 2.779 | 7.463 | 15.627
Easy excel | 1.138 | - | - | - | -
倍数 | 575% | - | - | - | -

如上图，在64MB内存限制下，easyexcel只能完成1w数据的读测试，EEC则可以完成1〜100w的读测试。

easyexcel具体的报错信息

```
org.apache.poi.util.RecordFormatException: Unable to construct record instance

	at org.apache.poi.hssf.record.RecordFactory$ReflectionConstructorRecordCreator.create(RecordFactory.java:98)
	at org.apache.poi.hssf.record.RecordFactory.createSingleRecord(RecordFactory.java:345)
	at org.apache.poi.hssf.record.RecordFactoryInputStream.readNextRecord(RecordFactoryInputStream.java:289)
	at org.apache.poi.hssf.record.RecordFactoryInputStream.nextRecord(RecordFactoryInputStream.java:255)
	at org.apache.poi.hssf.eventusermodel.HSSFEventFactory.genericProcessEvents(HSSFEventFactory.java:175)
	at org.apache.poi.hssf.eventusermodel.HSSFEventFactory.processEvents(HSSFEventFactory.java:136)
	at org.apache.poi.hssf.eventusermodel.HSSFEventFactory.processWorkbookEvents(HSSFEventFactory.java:82)
	at org.apache.poi.hssf.eventusermodel.HSSFEventFactory.processWorkbookEvents(HSSFEventFactory.java:54)
	at com.alibaba.excel.analysis.v03.XlsSaxAnalyser.execute(XlsSaxAnalyser.java:112)
	at com.alibaba.excel.analysis.ExcelAnalyserImpl.analysis(ExcelAnalyserImpl.java:105)
	at com.alibaba.excel.ExcelReader.readAll(ExcelReader.java:160)
	at com.alibaba.excel.read.builder.ExcelReaderBuilder.doReadAll(ExcelReaderBuilder.java:275)
  ...
Caused by: java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.String.<init>(String.java:166)
	at org.apache.poi.hssf.record.RecordInputStream.readStringCommon(RecordInputStream.java:405)
	at org.apache.poi.hssf.record.RecordInputStream.readUnicodeLEString(RecordInputStream.java:376)
	at org.apache.poi.hssf.record.common.UnicodeString.<init>(UnicodeString.java:464)
	at org.apache.poi.hssf.record.SSTDeserializer.manufactureStrings(SSTDeserializer.java:57)
	at org.apache.poi.hssf.record.SSTRecord.<init>(SSTRecord.java:252)
```

从错误信息上发现在解析SSTRecord时抛OOM错误。SST即Shared String Table的简称，大概率是POI将SST中的字符串全部拉到内存导致OOM。
[SharedStringTable在EEC中的处理](/excel/2020/04/15/about-SharedStringTable.html)一文中有详细介绍SST在POI中的实现，感兴趣的朋友可以移步去了解一下。

### 3.2 10MB内存

10MB限制下easyexcel全军覆没，错误同上。

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.199 | 0.763 | 1.737 | 9.224 | 18.537

内存截图：
![XLS 10MB](/images/posts/eve3/xls_10jvm.png)

### 3.3 不限制内存

由于easyexcel未完成上面的测试，所以我决定放开限制再分别跑一次

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.151 | 0.453 | 1.109 | 6.651 | 13.880
Easy excel | 0.628 | 2.810 | 4.55 | 32.116 | 118.69
倍数 | 416% | 620% | 410% | 483% | 855%

![图表](/images/posts/eve3/unlimit_test.png)

附两者测试CPU和内存截图，上面为easyexcel下面为EEC，不限制内存的情况下，前者最大分配7G堆内存，最大使用堆4.7G。EEC最大分配700MB，最大使用350MB。

![CPU & RAM](/images/posts/eve3/ram_c.png)

*我还发现一个有趣的现象，xls格式读取时间远小于xlsx，无论是EEC还是Easy excel。*

## 极限内存测试

在读取xls格式中，我分别对EEC和Easyexcel进行了极限内存测试，也就是跑完测试不抛异常时的最小内存，结果如下。

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 3M | 3M | 3M | 5M | 7M
Easy excel | 50M | 220M | 440M | 2400M | 5000M

从上图可以看出随着文件逐渐增大easyexcel所需内存也逐渐增大，读取文件所需的内存甚至超过文件本身的大小，而EEC读文件所需内存一直比较平稳，
且远低于easyexcel基本可以在10MB下完成大文件读取。

## 总结

通过前面的测试，我们对两款工具的便利性和性能有了一定的了解，与POI/easyexcel相比，EEC有着更新的设计理念，天然支持Worksheet分页、
支持数据分片、支持高亮、隔行变色、支持流式读取、懒读取等一些提高生产力的特性，且拥有更简洁、优雅的API也降低了接入成本。
无论在速度优化还是内存控制方面EEC都远优于POI，毕竟POI是十几年前的产物，就算easyexcel在此基础上有极大改善但是与EEC相比还有一定差距。

虽然EEC不支持xls的写入但瑕不掩瑜，尤其是微软已经停更xls格式多年，不支持xls格式写入也不会有太多功能损失。

