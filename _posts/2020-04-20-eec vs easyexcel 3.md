---
layout: post
title: EEC vs easyexcel(三)
categories: Excel
description: Excel操作工具EEC和alibaba开源easyexcel工具的性能对比
keywords: excel, EEC, easyexcel, xls, xlsx
---

[上一篇](/excel/2020/03/03/eec-vs-easyexcel-2.html)我们对比了easyexcel和EEC两款工具在InnerStr模式下读写Excel文件的性能和内存，
本文是对比系列的最后一篇，主要对比两款工具在SharedString模式下的读写性能以及Excel 97~2003格式读取性能。由于easyexcel不支持SharedString模式
写文件，EEC不支持xls格式写文件，所以本文只对比两个工具的读性能。

测试实体与上篇基本一致，只是添加了省/市/区3列文本，这3列值大概率重重。具体测试细节可以查看[上一篇](/excel/2020/03/03/eec-vs-easyexcel-2.html)

最终生成的文件内容如下

![Excel内容](/images/posts/eve3/sst-content.png)

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
![64MB测试截图](/images/posts/eve3/hi_64test.png)

CPU截图
![64MBCPU测试截图](/images/posts/eve3/hi_64cpu.png)

内存截图
![64MB内存测试截图](/images/posts/eve3/hi_64jvm.png)

*鼠标左边为EEC，右边为Easyexcel*

使用表格来直观表示

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC(Write) | 0.586 | 2.738 | 5.394 | 26.380 | 58.51
EEC(Read) | 0.349 | 1.716 | 3.242 | 15.209 | 30.664
Easy excel(Read) | 1.109 | 5.468 | 11.35 | 55.55 | 112.187
倍数 | 318% | 319% | 350% | 365% | 366%

![64MB图表](/images/posts/eve3/hi_64chart.png)

EEC的读取速度超过easyexcel，且在一个稳定范围内。

### 2.2. 32MB内存
*限制内存32MB测试*

![32MB测试截图](/images/posts/eve3/hi_32test.png)

CPU截图
![32MBCPU测试截图](/images/posts/eve3/hi_32cpu.png)

内存截图
![32MB内存测试截图](/images/posts/eve3/hi_32jvm.png)

*鼠标左边为EEC，右边为Easyexcel*，easyexcel并没有完成32MB内存测试，这是错误LOG

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

### 2.3. 探底EEC最低使用内存

经多次测试我最终运行在10MB处。

10MB限制下内存波动

![10M_RAM](/images/posts/eve3/hi_10jvm.png)

10MB限制下CPU波动

![10M_CPU](/images/posts/eve3/hi_10cpu.png)

10MB限制下读写时间

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC读 | 0.391 | 1.887 | 3.695 | 16.965 | 34.719

与64MB相比，10MB内存限制并没有影响太多速度，同时占用内存受数据量影响较小。

Easyexcel在10MB下抛OOM
![10M_Easyexcel OOM](/images/posts/eve3/hi_10easy_error.png)

## 3. xls格式读写

EEC在v0.4.6版本只支持读取操作，所以这里只测两款工具的读性能和内存使用。

测试文件由EEC先生成xlsx格式，再转为xls格式，代码如下

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
        protected IWorksheetWriter getWorksheetWriter(org.ttzero.excel.entity.Sheet sheet) {
            return new XMLWorksheetWriter(sheet) {
                /**
                 * xls每个Worksheet最大包含65536行x256列，所以这里设置分页参数为{@code 65536-1}(去除第一行的表头)
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

Size | File
----:|-------
 6.9M| eec shared 1w.xls
 1.8M| eec shared 1w.xlsx
  35M| eec shared 5w.xls
 9.1M| eec shared 5w.xlsx
  71M| eec shared 10w.xls
  18M| eec shared 10w.xlsx
 487M| eec shared 50w.xls
  92M| eec shared 50w.xlsx
 978M| eec shared 100w.xls
 185M| eec shared 100w.xlsx

我们可以看到xls格式最终文件比xlsx格式文件大得多，所以强烈建议在项目中将xlsx格式设置为默认的数据导出格式，最终文件大小xlsx格式优于xls和csv格式(附录有这三种文件大小)，对于网终传输更友好。

进入主题吧

### 3.1 64MB内存

![XLS Test](/images/posts/eve3/xls_64test.png)

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.198 | 0.603 | 2.779 | 7.463 | 15.627
Easy excel | 1.138 | - | - | - | -
倍数 | 575% | - | - | - | -

如上图，在64MB内存限制下，easyexcel只能完成1w数据的读测试，EEC可以完成1〜100w的读测试。

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
	at org.ttzero.compares.LargeExcelTest.easyRead(LargeExcelTest.java:400)
	at org.ttzero.compares.LargeExcelTest.easySharedRead(LargeExcelTest.java:376)
	at org.ttzero.compares.XlsTest.testEas100w(XlsTest.java:92)
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
在[SharedStringTable在EEC中的处理](/excel/2020/04/15/about-SharedStringTable.html)一文中有详细介绍SST在POI中的实现。

### 3.2 10MB内存

10MB限制下POI全军覆没，错误同上。

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.199 | 0.763 | 1.737 | 9.224 | 18.537

内存截图：
![XLS 10MB](/images/posts/eve3/xls_10jvm.png)

### 3.3 不限制内存

由于easyexcel未完成上面的测试，所以我决定放开限制分别再跑一次

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC | 0.151 | 0.453 | 1.109 | 6.651 | 13.880
Easy excel | 0.628 | 2.810 | 4.55 | 32.116 | 118.69
倍数 | 416% | 620% | 410% | 483% | 855%

![图表](/images/posts/eve3/unlimit_test.png)

附两者测试CPU和内存截图，上面为easyexcel下面为EEC，不限制内存的情况下，前者最大分配7G堆内存，最大使用堆4.7G。EEC最大分配700MB，最大使用350MB。
xls格式读取时间远小于xlsx，不管是EEC还是Easy excel。

![CPU & RAM](/images/posts/eve3/ram_c.png)

极限内存和时间对比(内存单位MB，时间单位秒)

描述 | 1w | 5w | 10w | 50w | 100w
----|---:|---:|----:|----:|-----:|
EEC(内存) | 3 | 3 | 3 | 5 | 7
EEC(时间) | 1.450 | 4.673 | 6.231 | 12.391 | 23.623
Easy excel(内存) | 50 | 220 | 440 | 2400 | 5000
Easy excel(时间) | 4.196 | 53.834 | 97.854 | 118.996 | 228.368

*xls格式下无论在速度还是内存，EEC都远优于Easyexcel*

## 附:

对比xlsx、xls以及csv格式在相同数据量下的最终文件大小：

格式 | 1w | 5w | 10w | 50w | 100w|
--|---:|---:|----:|----:|----:|
xlsx | 1.8 | 9.1 | 18 | 92 | 185|
xls | 6.9 | 35 | 71 | 487 | 978|
csv | 3.1 | 16 | 33 | 175 | 352|

![File size](/images/posts/eve3/file-size-c.png)
