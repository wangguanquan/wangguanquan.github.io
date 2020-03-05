---
layout: post
title: EEC vs easyexcel
categories: Excel
description: 比较EEC和alibaba开源easyexcel工具的使用便利性对比
keywords: eec, easyexcel
---

本文主要对比easyexcel和eec的使用便利性和功能对比

### 1. 开始

#### 关于easyexcel

[easyexcel](https://github.com/alibaba/easyexcel)在Apache POI基础上包装而来，主要为了降低内存使用防止OOM发生，
当然使用方法比原生的POI要简结很多，github上有12.9k Star，国内有大量用户。

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
- easyexcel最低支持JDK7，eec最低支持JDK8
- easyexcel简化了接口导致无法设置样式，eec可以方便设置任意样式
- easyexcel读取文件时忽略样式和字体也没有办法直接获取单元格的公式。
- 数据量(excel07最大行1_048_576)超过单个worksheet页时easyexcel会抛异常，eec会分为多个worksheet保存数据
- easyexcel对常用类型缺少支持（char, Timestamp, time, LocalDate, LocalDateTime, LocalTime）你必须为这些类型指定自定义Converter

相比之下eec更接近于Apache POI，而easyexcel更关注单元格的值而忽略其它不太关心的数据。

### 2. 使用方式

### 2.1 写文件

#### 2.1.1 小数据量

对于小量数据可以直接将内容放到数组中一次完成写入

以下代码中`defaultTestPath`是文件路径事先已创建好。

easyexcel写小文件

```
public void test5(List<Item> data) {
    EasyExcel.write(defaultTestPath.resolve("test5.xlsx").toString(), LargeData.class).sheet().doWrite(data);
}
```

![test5.xlsx](/images/posts/test5.png)

eec写小文件

```
public void test6(List<Item> data) throws IOException {
    new Workbook("test6").addSheet(new ListSheet<>(data)).writeTo(defaultTestPath);
}
```
![test6.xlsx](/images/posts/test6.png)

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
public void test2() {
    new Workbook("Large EEC").addSheet(new ListSheet<LargeData>() {
        int n = 0;
        @Override
        public List<LargeData> more() {
            return n++ < 100 ? data() : null;
        }
    }).writeTo(defaultTestPath);
}
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

以上是两个工具类处理大文件的不同方式，easyexcel采用push方式主动向ExcelWriter推送数据，eec采用pull方式在写完一块数据后再拉取下一块，至到返回空数组或null为止。

#### 2.1.3 写多个worksheet页

很多时候我们会分类将不同类别不同功能的数据写到不同的worksheet里，比如我们做一个表设计不会将每张表放不同的excel文件，更多的做法是将不同表写在不同的worksheet里。

easyexcel写多worksheet方式

```
public void test7() {
    ExcelWriter excelWriter = EasyExcel.write(defaultTestPath.resolve("test7.xlsx").toString()).build();
    excelWriter.write(checks(), EasyExcel.writerSheet("帐单表").build());
    excelWriter.write(customers(), EasyExcel.writerSheet("客户表").build());
    excelWriter.write(c2CS(), EasyExcel.writerSheet("用户客户关系表").build());
    excelWriter.finish();
}
```

![test7.xlsx](/images/posts/test7.png)

eec写多worksheet方式

```
public void test8() throws IOException {
    new Workbook("test8")
        .addSheet(new ListSheet<>("帐单表", checks()))
        .addSheet(new ListSheet<>("客户表", customers()))
        .addSheet(new ListSheet<>("用户客户关系表", c2CS()))
        .writeTo(defaultTestPath);
}
```

![test8.xlsx](/images/posts/test8.png)

eec相对来说要简单一些，但easyexcel也不复杂。

### 2.2 读文件

easyexcel在读文件时使用`ReadListener`来监听每行数据，这样可以做到边解析文件边做业务逻辑（插库或其它），不用把文件解析完成后再做业务逻辑，以下是解析图示：
![easyexcel解析图示](/images/posts/easyexcel解析图示.png)

eec采用迭代模式（迭代是不是一种设计模式很多人讨论），同样做到边解析文件边做业务逻辑，解决POI的高内存问题。
![eec解析图示](/images/posts/eec解析图示.png)

从两者图示大致可以看出两者的设计与写文件时正好相反。easyexcel通过监听主动把行数据推给用户，eec这边需要用户主动拉数据，用户真正需要某行数据时才再去解析它。

#### 2.2.1 easyexcel读文件

```
public void test3() {
    EasyExcel.read(defaultTestPath.resolve("Large easyexcel.xlsx").toFile(), LargeData.class,
    new AnalysisEventListener<LargeData>() {

        @Override
        public void invoke(LargeData data, AnalysisContext context) {
            // 业务处理
        }

        @Override
        public void doAfterAllAnalysed(AnalysisContext context) { }
    }).headRowNumber(1).sheet().doRead();
}
```

你需要实现一个`ReaderListener`来处理行数据。

#### 2.2.2 eec读文件

```
try (ExcelReader reader = ExcelReader.read(defaultTestPath.resolve("Large easyexcel.xlsx"))) {
    reader.sheets() // 解析所有worksheet
        .flatMap(Sheet::dataRows) // 只取数据行，跳过表头
        .map(row -> row.to(LargeData.class)) // 转为实体对象
        .forEach(o -> {
            // 业务处理
        });
} catch (IOException e) {
    e.printStackTrace();
}
```

由于eec采用迭代模式所以可以使用JDK8的Stream全部功能。数据量小的时候可以将数据全部放入内存像下面这样

```
try (ExcelReader reader = ExcelReader.read(defaultTestPath.resolve("Large easyexcel.xlsx"))) {
    List<LargeData> list = reader.sheets()
        .flatMap(Sheet::dataRows)
        .map(row -> row.to(LargeData.class))
        .collect(Collectors.toList());
    // 业务处理

} catch (IOException e) {
    e.printStackTrace();
}
```

easyexcel取出的内容必须使用`ReaderListener<T>`来接收，也就是说必须转为对象（可以是实体或者Map），eec提供与JDBC类似接口，你可以使用`row.getX(columnNumber)`获取指定列的值，对于读非规则表格或非表格这是非常有效的。

示例：

获取excel文件中所有学生的姓名并去重

```
try (ExcelReader reader = ExcelReader.read(defaultTestPath.resolve("二年级学生.xlsx"))) {
    List<String> names = reader.sheet(0) // 只取第一个worksheet页
        .dataRows()
        .map(row -> row.getString("姓名")) // 只取姓名列
        .distinct() // 去重
        .collect(Collectors.toList());
    // 业务处理

} catch (IOException e) {
    e.printStackTrace();
}
```

不要试图将数据转为Map类型，因为每个Map都需要保存表头和单元格值，这将极大的消耗内存。

简单的内容输出可以使用`row.toString()`方法，该方法使用`|`分隔单元格的值，所以你可以像这样`reader.sheets().flatMap(Sheet::rows).forEach(System.out::println);`使用一行代码轻松把内容打印到控制台或保存为`.md`格式。

eec还有一些比较亮眼的功能，如高亮，水印和code转为名称等一些实用的功能，如下代码展示如何将低于60分的学生标红且将分数显示为`不及格`

```
public void testStyleConversion() throws IOException {
    new Workbook("testStyleConversion") // 文件名
        .setCreator("奈留·智库") // 作者
        .setCompany("Copyright (c) 2020") // 公司名
        .setWaterMark(WaterMark.of("Secret")) // 水印
        .setAutoSize(true) // 自动计算列宽
        .addSheet(new ListSheet<>("期末成绩", Student.randomTestData(20)
            , new org.ttzero.excel.entity.Sheet.Column("学号", "id", int.class)
            , new org.ttzero.excel.entity.Sheet.Column("姓名", "name", String.class)
            , new org.ttzero.excel.entity.Sheet.Column("成绩", "score", int.class)
            // 低于60分显示`不及格`
            .setProcessor(n -> n < 60 ? "不及格" : n)
            // 低于60分单元格标红
            .setStyleProcessor((o, style, sst) -> {
                if ((int)o < 60) {
                    style = Styles.clearFill(style) | sst.addFill(new Fill(Color.red));
                }
                return style;
            })
        )
    )
    .writeTo(defaultTestPath);
}
```

最终生成文件如下
![](/images/posts/testStyleConversion.png)
![](/images/posts/properties.png)


### 3. 后记

easyexcel支持更低的JDK版本，eec使用了更灵活的设计模式，同时所有的低层代码均是独立实现，意味着它的依懒非常小。但是eec现阶段鲜有人使用有很多BUG也就无从发现，稳定性还有待考验。

[下一篇将对比两者在各数据量下的读写性能](/eec vs easyexcel(二).html)