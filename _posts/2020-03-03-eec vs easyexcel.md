---
layout: post
title: EEC vs easyexcel
categories: Excel
description: 比较EEC和alibaba开源easyexcel工具的速度和内存使用
keywords: eec, easyexcel
---

本文主要对比easyexcel和eec的使用便利性和性能，以下列出测试环境

硬件:
- CPU: 2.7 GHz Intel Core i5 (双核)
- 内存: 8 GB 1867 MHz DDR3
- 硬盘: SSD
- 系统: macOS 10.12.6

测试版本:
- easyexcel: 2.1.6
- eec: 0.4.2

为了限制内存对速度的影响，测试过程中添加jvm参数`-Xmx64m -Xms16m`，限制最大堆内存为64MB

### 1. 开始

#### 关于easyexcel

[easyexcel](https://github.com/alibaba/easyexcel)在Apache POI基础上包装而来，主要为了降低内存使用防止OOM发生，
当然使用方法比原生的POI要简结很多，github上有12.9k Star，国内应该有大量用户。

引用作者总结核心原理：
1. 文件解压、文件读取通过文件形式
2. 避免将全部全部数据一次加载到内存(采用sax模式一行一行解析并使用观察者的模式通知处理)
3. 抛弃不重要的数据(忽略样式，字体，宽度等数据)

[点击这里](https://github.com/alibaba/easyexcel/blob/master/abouteasyexcel.md)查看作者原文

#### 关于eec

[eec](https://github.com/wangguanquan/eec)并没有使用Apache POI包，事实上eec仅依懒dom4j和slf4j，前者用于小文件xml读取，后者统一日志接口。

核心原理：
1. 不缓存数据或少量缓存到内存，写文件时直接往临时文件写入
2. 单元格样式仅使用一个int来保存极大缩小内存使用
3. 采用Stream方式读取excel内容，不会将整个文件读入到内存

简单总结两个工具的不同：
- 底层不同，easyexcel底层为Apache POI，eec直接使用IO/NIO
- easyexcel简化了接口导致无法设置样式，eec可以方便设置任意样式
- easyexcel读取文件时忽略样式和字体也没有办法直接获取单元格的公式。
- 数据量超过单个worksheet页时easyexcel无法自动分页，eec会分为多个worksheet保存数据

相比之下eec更接近于Apache POI，而easyexcel更关注单元格的值

### 2. 使用方式

### 2.1 写文件

#### 2.1.1 小数据量

对于小量数据可能直接将内容放到数组中一次完成写入

以下代码中`defaultTestPath`是文件路径事先已创建好。

easyexcel写小文件

```
public void test5(List<Item> data) {
    EasyExcel.write(defaultTestPath.resolve("test5.xlsx").toString(), LargeData.class).sheet().doWrite(data);
}
```

![test5.xlsx](/images/posts/test5.xlsx.png)

eec写小文件

```
public void test6(List<Item> data) throws IOException {
    new Workbook("test6").addSheet(new ListSheet<>(data)).writeTo(defaultTestPath);
}
```
![test6.xlsx](/images/posts/test6.xlsx.png)

#### 2.1.2 大数据量

数据量较大时我们无法将数据全部装载到内存，此时需要分批写文件，好在easyexcel和eec均支持分片。

easyexcel写大文件

```
public void test1() {
    ExcelWriter excelWriter = EasyExcel.write(defaultTestPath.resolve("Large easyexcel.xlsx").toFile())
        .withTemplate(defaultTestPath.resolve("temp.xlsx").toFile()).build();
    WriteSheet writeSheet = EasyExcel.writerSheet().build();
    for (int j = 0; j < 100; j++) {
        excelWriter.fill(data(), writeSheet);
    }
    excelWriter.finish();
}
```

eec写大文件

```
new Workbook("Large EEC").addSheet(new ListSheet<LargeData>() {
    int n = 0;
    @Override
    public List<LargeData> more() {
        return n++ < 100 ? data() : null;
    }
}).writeTo(defaultTestPath);
```

这里的data()方法模拟取数据过程，返回`List<LargeData>`类型。类似如下代码

```
private List<LargeData> data() {
    List<LargeData> list = new ArrayList<>();
    for (int i = 0; i < 1000; i++) {
        LargeData largeData = new LargeData();
        list.add(largeData);
        largeData.setStr1("str1-" + i);
        largeData.setStr2("str2-" + i);
        largeData.setStr3("str3-" + i);
        largeData.setStr4("str4-" + i);
        largeData.setStr5("str5-" + i);
    }
    return list
}
```

以上是两个工具类处理大文件的不同方式，easyexcel采用push方式往ExcelWriter推送数据，eec采用pull方式在写完一块数据后再拉取下一块，至到空数组或null为止。