---
layout: post
title: EEC vs easyexcel
categories: Excel
description: 比较EEC和alibaba开源easyexcel工具的使用便利性对比
keywords: EEC, easyexcel
---

本文主要对比easyexcel和EEC的使用便利性和功能对比

### 1. 开始

#### 关于easyexcel

[easyexcel](https://github.com/alibaba/easyexcel)是alibaba开发的快速、简单、且避免OOM的java处理Excel工具，于2018.2在github上开源。它是在Apache POI基础上包装而来，主要解决Apache POI高内存且API臃肿的诟病，
easyexcel提供了比原生的POI简结很多的接口，读写Excel文件均可以一行代码完成，目前(2020.3)github上有13k个Star和3.4K个Fork。

引用作者总结核心原理：
1. 文件解压、读取通过文件形式
2. 避免将全部数据一次加载到内存(采用sax模式一行一行解析并使用观察者的模式通知处理)
3. 抛弃不重要的数据(忽略样式，字体，宽度等数据)

[点击这里](https://github.com/alibaba/easyexcel/blob/master/abouteasyexcel.md)查看作者原文

#### 关于EEC

[EEC](https://github.com/wangguanquan/eec)是国内一个个人开发者开发并于2017.10月在github开源，EEC的底层并没有使用Apache POI包，所有的底层读写代码均由作者实现，事实上EEC仅依懒dom4j和slf4j，前者用于小文件xml读取，后者统一日志接口。

核心原理：
1. 不缓存数据或少量缓存
2. 使用分片来处理较大的数据
3. 单元格样式仅使用一个int值来保存，极大缩小内存使用
4. 使用迭代模式读取行内容，不会将整个文件读入到内存

简单总结两个工具的不同：
- 底层不同，easyexcel底层使用Apache POI，EEC使用IO/NIO
- easyexcel最低支持JDK7，EEC最低支持JDK8
- easyexcel简化了接口使得像设置样式这种基本功能非常困难，EEC默认带有便于阅读的样式也提供方法设置其它样式
- easyexcel读取文件时忽略样式和字体也没有办法直接获取单元格的公式。
- easyexcel对常用类型缺少支持（char, Timestamp, Time, LocalDate, LocalDateTime, LocalTime)，如果实体类中有这些类型就必须为这些类型编写自定义Converter

相比之下EEC更接近于Apache POI，而easyexcel更关注单元格的值而忽略其它不太关心的数据。

### 2. 写文件

#### 2.1 少量数据

对于少量数据可以直接将内容放到数组/集合中一次写入，两个工具都能做到一行代码完成数据写入。下面展示两者的实现方式，代码中出现的`defaultTestPath`是文件路径事先已创建好。

easyexcel可以将文件直接写入`OutputStream`或磁盘

```
public void test5(List<Item> data) {
    EasyExcel.write(defaultTestPath.resolve("test5.xlsx").toString(), LargeData.class).sheet().doWrite(data);
}
```

![test5.xlsx](/images/posts/test5.png)

EEC同样可以写入`OutputStream`或磁盘，使用`writeTo`方法指定输出位置

```
public void test6(List<Item> data) throws IOException {
    new Workbook("test6").addSheet(new ListSheet<>(data)).writeTo(defaultTestPath);
}
```
![test6.xlsx](/images/posts/test6.png)



#### 2.2 大数据量

数据量较大时我们无法将数据全部装载到内存，此时需要分批写文件，好在easyexcel和EEC均支持分片处理做到边读数据边写文件。

easyexcel写大文件需要指定一个模板文件，然后调用`fill`方法循环写数据，需要注意如果数据量超出excel单页上限会抛异常

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

EEC分片写大文件时需要继承`ListSheet<T>`或`ListMapSheet`然后重写`more`方法并返回批量数据

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

以上是两个工具类处理大文件的不同方式，easyexcel采用push方式主动向ExcelWriter推送数据，EEC采用pull方式由工具决定何时拉取下一块数据，返回空数组或null时表明没有更多数据，所以这里要注意控制分页参数防止出现死循环。

对于数据量巨大且使用关系型数据库的场景，EEC提供另一种方案，用户可以使用`StatementSheet`和`ResultSetSheet`两种方式，它们的工作方式是将SQL和参数交给EEC，EEC内部去查询并使用游标做到取一个值写一个值，省掉了将表数据转为Java实体的过程。

#### 2.3 写多个worksheet页

两个工具都提供便利的方法实现多worksheet页写入，基本可以使用一行代码搞定。

easyexcel通过创建多个WriteSheet来实现

```
public void test7() {
    EasyExcel.write(defaultTestPath.resolve("test7.xlsx").toString()).build()
        .write(checks(), EasyExcel.writerSheet("帐单表").build())
        .write(customers(), EasyExcel.writerSheet("客户表").build())
        .write(c2CS(), EasyExcel.writerSheet("用户客户关系表").build())
        .finish();
}
```

![test7.xlsx](/images/posts/test7.png)

EEC通过`addSheet`方法添加多个worksheet，看上去更直观更容易理解。

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


### 3. 读文件

easyexcel在读文件时使用`ReadListener`来监听每行数据，这样可以做到边解析文件边做业务逻辑（插库或其它），不用把文件解析完成后再做业务逻辑，以下是解析图示：
![easyexcel解析图示](/images/posts/easyexcel解析图示.png)

EEC采用迭代模式，同样做到边解析文件边做业务逻辑，解决POI的高内存问题。
![EEC解析图示](/images/posts/eec解析图示.png)

从两者图示大致可以看出两者的设计与写文件时正好相反。easyexcel通过监听主动把行数据推给用户，EEC这边需要用户主动拉数据，用户真正需要某行数据时才去解析它们。

#### 3.1 easyexcel读文件

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

#### 3.2 EEC读文件

```
try (ExcelReader reader = ExcelReader.read(defaultTestPath.resolve("Large easyexcel.xlsx"))) {
    reader.sheet("帐单表") // 解析指定worksheet
        .flatMap(Sheet::dataRows) // 只取数据行，跳过表头
        .map(row -> row.to(LargeData.class)) // 转为实体对象
        .forEach(o -> {
            // 业务处理
        });
} catch (IOException e) {
    e.printStackTrace();
}
```

### 3.3 读取多个worksheet页

两个工具都提供方便的多worksheet读取，可以看示例

easyexcel示例

```
public void test9() {
    ExcelReader excelReader = EasyExcel.read(defaultTestPath.resolve("test7.xlsx").toFile(), simpleListener).headRowNumber(0).build();
    List<ReadSheet> sheets = excelReader.excelExecutor().sheetList();
    sheets.forEach(sheet -> {
        System.out.println("----------" + sheet.getSheetName() + "-----------");
        excelReader.read(sheet);
    });
}

// 输出内容
----------帐单表-----------
{0=1.0, 1=100.8}
{0=2.0, 1=34.2}
{0=3.0, 1=983.0}
----------客户表-----------
{0=1001.0, 1=张三}
{0=1002.0, 1=李四}
----------用户客户关系表-----------
{0=1.0, 1=1001.0}
{0=2.0, 1=1002.0}
{0=3.0, 1=1002.0}
```

EEC示例

```
public void test10() {
    try (ExcelReader reader = ExcelReader.read(defaultTestPath.resolve("test8.xlsx"))) {
        reader.sheets().flatMap(sheet -> {
            System.out.println("----------" + sheet.getName() + "-----------");
            return sheet.rows();
        }).forEach(System.out::println);
    } catch (IOException e) {
        e.printStackTrace();
    }
}

// 输出内容
----------帐单表-----------
id | total
1 | 100.8
2 | 34.2
3 | 983
----------客户表-----------
id | name
1001 | 张三
1002 | 李四
----------用户客户关系表-----------
ch_id | cu_id
1 | 1001
2 | 1002
3 | 1002
```

操作都还算方便，相较easyexcel来说EEC要更简单一点，如果不输出worksheet名那么一行命令就可以完成输出`reader.sheets().flatMap(Sheet::rows).forEach(System.out::println);`

### 4. EEC更多使用方式

由于EEC采用迭代模式因此可以使用JDK8的Stream全部功能，下面展示一些常用功能。

#### 4.1 将内容转为集合

数据量小的时候可以将数据全部放入内存像下面这样

```
try (ExcelReader reader = ExcelReader.read(defaultTestPath.resolve("Large easyexcel.xlsx"))) {
    List<LargeData> list = reader.sheets().flatMap(Sheet::dataRows)
        .map(row -> row.to(LargeData.class)).collect(Collectors.toList());
    // 业务处理

} catch (IOException e) {
    e.printStackTrace();
}
```
#### 4.2 取单列数据

EEC提供与JDBC类似的接口，用户可以使用`row.getX(columnNumber)`获取指定位置的值，对于读非规则表格或非表格时这是非常有效的。

示例：获取"二年级学生.xlsx"中所有学生的姓名并去重

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
#### 4.3 过滤某些行

我相信很多时候都会遇到这样的需求，我们仅需要处理满足某些要求的数据而过滤掉检查失败的数据，这时候JDK8的`filter`就派上用场了

比如我们需要打印帐单页金额大于100的记录

```
public void test9() {
    try (ExcelReader reader = ExcelReader.read(defaultTestPath.resolve("test8.xlsx"))) {
        reader.sheet("帐单表")
            .dataRows()
            .filter(row -> row.getDouble("total") > 100.0)
            .forEach(System.out::println);
    } catch (IOException e) {
        e.printStackTrace();
    }
}

// 输出结果
1 | 100.8
3 | 983.0
```

#### 4.4 其它一些亮眼功能

EEC还有一些比较亮眼的功能如高亮，水印等一些实用的功能，下面代码展示如何将低于60分的学生标红且将分数显示为`不及格`

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


### 5. 后记

读excel时不要试图将数据转为Map类型，因为每个Map都需要保存表头和单元格值，这将极大的消耗内存。

简单总结: easyexcel和EEC两个工具都极大的简化了java操作excel，从原本Apache POI繁锁的API和高内存中解脱出来。其中easyexcel支持更低的JDK版本，而EEC使用了更灵活的设计模式，同时所有的底层代码均是独立实现，意味着它的依懒非常小。但是EEC现阶段鲜有人使用有很多BUG也就无从发现，稳定性还有待考验。

[下一篇将对比两者在各数据量下的读写性能](/excel/2020/03/05/eec-vs-easyexcel-2.html)