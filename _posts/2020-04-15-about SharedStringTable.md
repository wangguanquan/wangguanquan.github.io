---
layout: post
title: SharedStringTable在EEC中的处理
categories: Excel
description: 描述EEC如何处理共享字符串，以达到高效低内存的目标
keywords: EEC, excel, SharedStringTable
---

## 概念引入

SharedStringTable是微软在BIFF8版本中引入的概念，其目地是整个Workbook共享一个字符串区域，
所有的Worksheet只需要保存字符串下标即可，读取文档时根据下标值从SharedStringTable中读取相应的值。
>A BIFF8 workbook collects the strings of all text cells in a global list,
 the Shared String Table. This table is located in the record SST in the 
 Workbook Globals Substream
 
SharedStringTable在Excel07及以后版本保存在`sharedStrings.xls`文件中。内容大致像下面这样，
其中count表示内容被引用次数，uniqueCount表示整个文件中保存的字符串个数。

```$xml
<sst count="5" uniqueCount="3">
<si><t>Shared</t></si>
<si><t>String</t></si>
<si><t>Table</t></si>
</sst>
```

简单的理解可以将SharedStringTable看成一个字符串数组，写Excel的时候先从数组中查找字符串，如果找到则将下标
当做值写入文件，如果未找到则将字符串追加到数组最后，再将当前下标写入文件。

*为了方便描述，以下将SharedStringTable简写为SST*

图示: 假设SST已存在3个值，状态如下

```$xslt
+--------+--------+-------+
| Shared | String | Table |->tail
+--------+--------+-------+
```

现在要写入字符串"Table"，步骤如下：
1. 在SST中查找字符串"Table"，此处返回2
2. 将下标2写入文件`<v>2</v>`

如果要写入字符串"Workbook"，步骤如下：
1. 在SST中查找字符串"Workbook"，查无
2. 将"Workbook"追加到SST尾部
3. 返回当前下标3
4. 将下标3写入文件`<v>3</v>`

现在的SST状态如下:

```$xslt
+--------+--------+-------+----------+
| Shared | String | Table | Workbook |->TAIL
+--------+--------+-------+----------+
```

读取字符串时只需要使用val值从SST取相应下标即可。

下图描述Excel文件内部各Worksheet与SST的关联关系

```$xslt
                       +--------+--------+-------+----------+
                       | Shared | String | Table | Workbook |->TAIL
                       +--------+--------+-------+----------+
                           |            \    |         |
     +---------------------+   /---------\---+         |             
     |          /--------------|----------|------------+-\
+---------------------+   +---------------------+   +----------+
| <v>0</v>...<v>3</v> |	  | <v>2</v>...<v>1</v> |   | <v>3</v> |
+-------Sheet1--------+	  +-------Sheet2--------+   +--Sheet3--+
```

## 如何实现SharedStringTable

通过上面的描述我们对SST有了大致的了解，理论上要实现一个SST很简单，只需要简单封装一个字符串数组即可，我们先实现一个简单的SST。

// TODO

但实际应用中将面临下面两个问题
1. 当SST长度很大时我们无法将所有值放到数组中，此时应该如何处理（读/写均会出现）
2. 从数组中查找字符串需要遍历整个SST时间复杂度是(On)，应该如何提升查找速度（仅写操作）

先从第一个问题开始吧，如果字符串太多时如何处理？

首先想到的是分片，按一定长度将大文件分为若干个小片，内存中只保存一片的数量。

```$xslt
<sst>
<si><t>Table</t></si>    ------ +----------+ offset: 0, size: 100
         .  0                   |  Table   |
         .  |                   |    .     |
         .  99                  |    .     | 内存中的一个片对应SST一段值
<si><t>Workbook</t></si> \_     |    .     |
<si><t>Hello</t></si>      \_   | Workbook |
         . 100               \_ +----------+
         .  |
         . 199
<si><t>World</t></si>
</sst>
```

