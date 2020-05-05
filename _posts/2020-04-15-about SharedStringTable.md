---
layout: post
title: SharedStringTable在EEC中的处理
categories: Excel
description: 描述EEC如何处理共享字符串，以达到高效低内存的目标
keywords: EEC, excel, SharedStringTable
---

## 概念引入

SharedStringTable是微软在BIFF8版本中引入的概念，其目地是使整个Workbook共享一个字符串区域，所有单元格使用其在共享区域中的位置引用即可，读取文档时根据这个引用找到相对应的值。
>A BIFF8 workbook collects the strings of all text cells in a global list,
 the Shared String Table. This table is located in the record SST in the 
 Workbook Globals Substream
 
SharedStringTable在Excel07及以后版本保存在`sharedStrings.xls`文件中。内容大致像下面这样，其中count表示内容被引用次数，uniqueCount表示整个`sharedStrings.xls`文件中保存的字符串个数。

```
<sst count="5" uniqueCount="3">
  <si><t>Shared</t></si>
  <si><t>String</t></si>
  <si><t>Table</t></si>
</sst>
```

SharedStringTable的操作顺序像这样: 单元格写字符串时先从SharedStringTable中查找需要写入的值，如果存在则将其下标当做值写入单元格，如果未找到则将字符串追加到SharedStringTable末尾，再将当前下标写入单元格，读取时通过单元格中的整数值从SharedStringTable中取回相应的字符串，所以单元格并没有真正保存字符串字面值而是保存引用。简单的理解可以将SharedStringTable看成是超市门口的储物柜，进超市前你需要将背包保存到储物柜，储物柜会给你一个条形码或手环，离开超市时通过条形码或手环取回你的背包。

*为了方便描述，以下将SharedStringTable简写为SST*

下面我们使用案例描述SST是如何运作的。

图示: 假设SST已存在3个值，状态如下

```
+--------+--------+-------+
| Shared | String | Table |->TAIL
+--------+--------+-------+
```

案例1:

现在要写入字符串"Table"，步骤如下：
1. 在SST中查找字符串"Table"，此处返回2
2. 将下标2写入文件`<v>2</v>`
3. count+1

案例2:

如果要写入字符串"Workbook"，步骤如下：
1. 在SST中查找字符串"Workbook"，查无
2. 将"Workbook"追加到SST尾部
3. 返回当前下标3
4. 将下标3写入文件`<v>3</v>`
5. count+1，uniqueCount+1

现在的SST状态如下:

```
+--------+--------+-------+----------+
| Shared | String | Table | Workbook |->TAIL
+--------+--------+-------+----------+
```

sharedStrings.xls文件内容如下:

```
<sst count="7" uniqueCount="4">
  <si><t>Shared</t></si>
  <si><t>String</t></si>
  <si><t>Table</t></si>
  <si><t>Workbook</t></si>
</sst>
```

这里count等于7,因为两个案例各加了1次，uniqueCount在第二个案例中加1次所以等于4。读取字符串时只需要使用val值从SST取相应下标的字符串即可。

下图描述Excel文件内部各Worksheet与SST的关联关系

```
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

图示中Sheet1和Sheet3都含有"Workbook"字符串，字符串"Workbook"在SST中下标为3，所以两个Sheet通过`<v>3</v>`来指向此共享字符串。

与SharedString相对应的方式是`inlineStr`，从字面意思不难理解，它将字符串值直接写到每个单元格中，所以被称为inline模式。在各Worksheet.xls中大概像下面这样

```
<c t="inlineStr">
  <is>
    <t>Shared</t>
  </is>
</c>
<c t="inlineStr">
  <is>
    <t>Workbook</t>
  </is>
</c>
```

`is`是`inlineStr`的简写，此标签与sharedStrings.xml中的`si`标签对应。

## 如何实现SharedStringTable

实现SST是完成Excel读写的必经之路，通过上面的描述我们对SST也有了大致的了解，它大致有以下几个特点

1. 写操作时通过字符串换取一个数字引用
2. 读操作时通过数字引用换回字符串
3. SST中的字符串不重复(不是强制的)
4. 如果有扩容那么扩容操作不能打乱原有顺序

前两点应该很好理解正如我们的案例描述一样，第三点正是SST的精髓它保证了共享和压缩两大特性，第四点应该也不难理解顺序打乱后会导致我们取值混乱错位。

下面我来逐步实现吧，实现之前我们先定一个接口来统一规范

```
public interface SST {
    /**
     * 保存字符中，换取一个整形凭证
     */
    int put(String str);

    /**
     * 通过凭证换回原始字符串
     */
    String get(int index);

    /**
     * 所有字符串被引用的次数
     */
    int count();

    /**
     * 所有字符串的个数
     */
    int uniqueCount();
}
```

### 原始实现

首先想到的是使用Array实现，因为它是顺序的且扩容不影响原有顺序，理论上要实现一个ArraySST很简单，只需要简单封装一个字符串数组即可，下面我们先使用Array实现一个SST。

```
public class ArraySST implements SST {
    // 保存所有共享字符串
    private List<String> table;
    private int count;

    public ArraySST() {
        table = new ArrayList<>();
    }

    @Override
    public int put(String str) {
        // 查找Table是否包含str
        int index = table.indexOf(str);
        // 如果不包含则添加到末尾并返回当前下标
        if (index < 0) {
            index = table.size();
            table.add(str);
        }
        // 引用次数+1
        count++;
        // 如果包含则返回str在数组中的下标
        return index;
    }

    @Override
    public String get(int index) {
        return table.get(index);
    }

    @Override
    public int count() {
        return count;
    }

    @Override
    public int uniqueCount() {
        return table.size();
    }
}
```

好了，现在我们有了最简单的SST实现，它由一个ArrayList来存储字符串，并提供一个put和一个get方法分别用于读和写。在某些情况(比如少量数据)它能很好的工作，但是它有一个明显的短板，我想你应该注意到了吧，查找数组中是否存在某个值的时间复杂度是O(n)，对于我们的应用场景这个时间复杂度是不合适的，我们需要对它时间一些改造。

### 改造第1版

使用Array实现SST出现的瓶颈是put方法时间复杂度为O(n)，那么使用什么数据结构能降低查找时的时间复杂度呢？是的，首选Hash算法，查找时它的时间复杂度接近O(1)。很多人认为Hash的时间复杂度就是O(1)，其实并不是这样，当发生碰撞后它会被弱化为链表或树，JDK8中当链表长度超过8时会升为一棵红黑树，红黑树长度小于6时降为链表，链表的时间复杂度是O(n)，红黑树的时间复杂度是O(log n)。

扯远了，我们使用HashMap来改造一个ArraySST吧。引入HashMap的目的是为了减少查找时间复杂度，所以我们可以继承ArraySST并覆写put方法代码如下:

```
public class HashSST extends ArraySST {
    // 用于字符串查找
    private Map<String, Integer> searcher;

    public HashSST() {
        super();
        searcher = new HashMap<>();
    }

    @Override
    public int put(String str) {
        // 查找HashMap中是否包含str
        Integer index = searcher.get(str);
        // 如果不包含则添加到末尾并返回当前下标
        if (index == null) {
            index = table.size();
            table.add(str);
            searcher.put(str, index);
        }
        // 引用次数+1
        count++;
        // 如果包含则返回str在数组中的下标
        return index;
    }
}
```

我们从HashMap中查找字符串替换父类直接从Array中查找，升级之后似乎能很好的工作，put和get方法时间复杂度均趋近于O(1)。

事实上POI就是这样实现SST的，下面贴出部分代码

```
public class SharedStringsTable extends POIXMLDocumentPart {

    private final List<CTRst> strings = new ArrayList<CTRst>();

    private final Map<String, Integer> stmap = new HashMap<String, Integer>();

    private int count;

    private int uniqueCount;
    
    public CTRst getEntryAt(int idx) {
        return strings.get(idx);
    }

    public int getCount(){
        return count;
    }

    public int getUniqueCount(){
        return uniqueCount;
    }

    public int addEntry(CTRst st) {
        String s = getKey(st);
        count++;
        if (stmap.containsKey(s)) {
            return stmap.get(s);
        }

        uniqueCount++;
        CTRst newSt = _sstDoc.getSst().addNewSi();
        newSt.set(st);
        int idx = strings.size();
        stmap.put(s, idx);
        strings.add(newSt);
        return idx;
    }
}
```

这是一段POI实现的SST，addEntry方法相当于我们的put方法，getEntryAt相对于我们的get方法。查找时先从stmap查找是否包含目标字符串，如果包含则直接返回下标，如果不存在则添加。感兴趣的朋友可以自行前去查看原代码。xls格式实现[SSTRecord](https://github.com/apache/poi/blob/trunk/src/java/org/apache/poi/hssf/record/SSTRecord.java)，xlsx格式实现[SharedStringsTable](https://github.com/apache/poi/blob/trunk/src/ooxml/java/org/apache/poi/xssf/model/SharedStringsTable.java)

*这种实现有没有弱点？它真的是终级实现吗？有没有更好的实现方法？*

既然我们解决了时间复杂度问题，似乎应该从空间复杂度来重新审视这种实现。它似乎比较占内存...不用怀疑它真的非常耗内存，在[EEC vs easyexcel(三)](https://www.ttzero.org/excel/2020/04/20/eec-vs-easyexcel-3.html#3-xls%E6%A0%BC%E5%BC%8F%E8%AF%BB%E5%86%99)性能测试中，读取一个978MB大小的xls文件POI需要用到超过5G的内存，读文件使用的内存是文件本体的5倍...我们似乎可以从内存优化入手继续改造。

### 改造第2版

这一版我们要解决字符串数量超大(超过百万，千万)，无法将所有字符串放入Array和HashMap时应该如何处理？Trie树？B+树？

// TODO 未完待续

首先想到的是分片，按一定长度将大文件分为若干个小片，内存中只保存一片的数量。

```
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

