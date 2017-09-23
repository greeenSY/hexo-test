---
title: Pig浅谈——小心pig给你挖的坑
date: 2016-12-01 10:15:25
description: 之前在组里关于pig的分享，把ppt整理一下
category: 大数据
tags: [分布式系统, pig]
---
# Pig是什么

一个基于Hadoop的并行执行数据流处理的引擎，被设计为实现已知将执行的一系列的数据操作

<!--more-->

## HIVE vs Pig vs MapReduce

- 相对于MapReduce，PIG提供实现一些标准数据操作，早起的数据错误检查和优化，更容易编写和维护
- 相对于Hive，PIG没有数据约束和标准化，不需要事先导入表中，直接操作存放在HDFS中的数据。可以在无模式、模式不全或者模式不一致的情况下进行操作。

## Pig和HIVE

Hive更适合于数据仓库的任务，主要用于静态的结构以及需要经常分析的工作
Hive和SQL语法类似，促使其成为Hadoop与其他BI工具结合的理想交集
Pig赋予开发人员在大数据集领域更多的灵活性，并允许开发简洁的脚本用于转换数据流以便嵌入到较大的应用程序
Pig相比Hive更轻量，它主要的优势是灵活性，同事相比于直接使用Hadoop Java APIs可以大幅削减代码量

## Pig的应用场景

- 传统的抽取转换加载（ETL）数据流
- 原生数据研究
- 迭代处理

典型案例：

收集日志，进行数据清洗和简单聚合预计算，导入数据仓库。

     Pig是家畜，喜欢按照用户的命令做事

Pig做脏活累活，需要教化，如果不注意也会挖坑。

## 一个典型的Pig脚本

     DEFINE flowAction `$streaming_file` SHIP(‘$streaming_file');//定义stream处理的脚本
     log = LOAD '$input_path' USING PigStorage('\t')
     AS (type:chararray,vendor:chararray,...,last_login:chararray); //载入数据，并制定数据模式
     log_filter = filter log by type != ‘’; //过滤脏数据
      group_log = GROUP log_filter BY (deviceid,app); //按照设备id和来源分组
      order_log = FOREACH group_log {
          ORDERED = ORDER log BY type; //分组后按照type排序
          GENERATE FLATTEN(ORDERED); //打平分组的bag并输出
      }
      data = STREAM order_log THROUGH flowAction; //利用stream脚本处理每一行数据
     STORE data INTO ‘$store_path'; // 输出数据到目标路径

执行操作命令：

     pig -p date=2016-12-20 -p input_path={{path_of_data}}/dt=2016-12-20/* -p store_path={{path_of_data_output}}/ 2016-12-20 -p streaming_file=dwTutorDeviceDi.py dwTutorDeviceDi.pig

# Pig的数据类型

*  基本： int, long, float, double, chararray, bytearray

*  复杂：map, tuple, bag

*  操作符：+ - * / %    == ......

*  投射映射符：

     *  散列表：#

     bat: map[], bat#’batting_average'

     *  点操作符：.

    t: tuple(x:int, y:int), t.x, t.$1

## 类型转换

bytearray是默认的数据类型，当pig没有发现数据模式时，都被认为是bytearray

基本类型int，long，float，double，chararray可以转换，但是不允许转换成bytearray

加载函数，有责任指定数据的模式。必要时需要重写加载函数。

隐式转换

     int -> long
     int, long -> float
     int, long, float -> double

## Pig是强类型吗？

Pig比较特殊，是强类型，但是又弱一点。

Pig会对数据类型进行猜测：

     $7 / 1000 long
     $3 * 100.0 double
     substring($0,0,1) chararray
     &6-$3 double
     &6 > $3 bytearray

坑：自动认定类型时采用较为安全的方式，变慢和损失精度，应该进行类型转换或者声明
坑：当不需要系统类型时可以不指定，例如：统计行数，bytearray转换类型时需要消耗资源

## 特殊的NULL

null表示缺失
null对于所有的算术操作符是抵消的：null + x = null
x==null返回null，而不是true/false，应该写作 x is null
order, group, join等操作需要注意，字段中是否有null？

# Pig常用操作

## 只需要MAP操作

1. LOAD：

     divs = load ’NYSE_divdends’ using HBaseStorage() as (exchange, symbol, date, dividends);

加载数据：using PigStorage(“\t”)，默认制表符分割

模式: as (x:chararray, y:int, z:int)

指定文件，指定文件夹则会遍历所有文件

2. STORE：

     store process into ‘processed’ using HBaseStorage();

processed是一个文件夹，文件数由并行数决定

3. DUMP：

     dump processed;

打印到屏幕

4. FILTER：

     filter divs by symbol matches ‘cm.*’;
     filter divs by not symbol matches ‘cm.*’;
     filter log by type == ‘down'

过滤操作，支持正则

5. FOREACH：

     B = foreach A generate user,id;
     B1 = foreach A generate close-open;
     B2 = foreach A generate $6-$3;

     beginning = foreach A generate ..open;
     end = foreach A generate volume..;
     middle = foreach A generate open..close;

数据管道中把表达式应用到每一条记录

注意：以上操作不需要产生reduce过程，都可以在map中完成。引起Reduce的操作有：

GROUP ORDER DISTINCT JOIN LIMIT  COGROUP CROSS

Reduce往往带来的是性能的问题，也是调整和调优的重点战场。比如并行度，Map和Reduce的拆分，数据倾斜问题……等等。

## 引起REDUCE的操作

6. Group

     group daily by stock;
     group daily by (x,y);
     group daily by (x,x);
     group daily all;
          (all,{(a,1,2),(d,2,2),(a,0,1)})

一个键group+一个包含聚集记录的bag

     b1: {group: chararray,a: {(x: chararray,y: int,z: int)}}

注意：null的key会被分到一个组中。

group出发reduce引发数据倾斜的问题：

     input = load ‘data’ as (x,y);
     grpd = group input by x;
     cnt = foreach grpd generate group,COUNT(input);
     store cnt into ‘result'

pig会通过combiner来应对数据倾斜

     map:
           load
     reduce:
          foreach(group,count),store

优化后：

     map:
          load
          foreach(group,count.Initial)
     combine:
          foreach(group,count.Intermediate)
     reduce:
          foreach(group,count.final),store

和combiner结合的函数：

* 可代数计算：可以分割为初始、中间、最终三个阶段。例如：count(load,sum,sum)

* 可分布计算：特殊的可代数计算（三个阶段是相同的），如sum

坑：UDF中使用Algebraic接口来声明可代数计算的函数，三个函数。

7. UDF：

* Pig UDF：

ABS（绝对值），ACOS（反余弦），ASIN， ATAN，CBRT（立方根），CEIL（向上取整），COS（余弦），COSH（双曲余弦），EXP（e的幂次方），

AVG（平均值，忽略null），COUNT（计数，忽略null），COUNT_STAR（所有记录个数），MAX（最大值），MIN（最小值），SUM（求和），

CONCAT（连接c1和c2），INDEXOF（查找字符串），LAST_INDEX_OF，LCFIRST（第一个字符转小写），LOWER（转小写），REGEX_EXTRACT（正则匹配），REPLACE（替换），SIZE（长度），STRSPLIT（分割），SUBSTRING（子串），TRIM（去除空格），UPPER（大写），

TOP（n,field,bag），IsEmpty， RANDOM（0,1直接的随机数）

* 用户自定义 UDF：java，python

1. 注册UDF： register ''

2. 指定完整路径，或者使用-Dudf.import.list=...

3. define命令指定UDF的完整路径

坑：PIg的大小写敏感：关键词不敏感，关系名敏感，UDF敏感

8. ORDER：

     b = order daily by date;
     b = order daily by date, symbol;
     b = order daily by date desc, symbol desc;

map，tuple，bag不能进行排序，语义不明

null在排序中是最小的

坑：desc只用于一个字段

order中对于数据倾斜的优化：

对于输出先取样（轻量级的mapreduce，读每个块的第一条）获得key分布的预算，然后根据取样来创建分割器，进行排序。例如：a,b,e,e,e,e,e,e,m,q,r,z会被分到三个分割器中：a~e，e，m~z

order、skew join重写了MapReduce的分割器

9. DISTINCT：

     uniq = distinct daily

类似于sql中的select distinct daily

10. JOIN：

     jnd = join daily by symbol, divs by symbol
      jnd = join daily by (symbol,date), divs by (symbol,date)
     jnd = join daily by (symbol,date) left, divs by (symbol,date)
      jnd = join daily by a dailyB by b dailyC by c //只能内连接

歧义的字段：daily:a, daily::b

     outer join(left,right,full)

key为null的不会保留

坑：

* 自连接，需要load两次，以区分语义：

     a1 = load ‘postition’ as (x,y,z)
     a2 = load ‘postition’ as (x,y,z)
     a3 = join a1 by x, a2 by x;

* 连接的时候左边的记录会先到。会把n-1的记录放到内存，n次记录来的时候依次做叉乘，直接通过。因此把记录多的数据放在右边可以节省内存。

* 几种特殊的join，处理特殊的情况：

1. 小数据join大数据——分片-重复-join：

     jnd = join daily by symbol, divs by symbol using ‘replicated'

将小文件分发到每台机器并加载到内存，然后join大文件。如果不能加载到内存则执行失败。只支持inner join，left join。多个表join时，最左的表读入内存。利用MapReduce的分布式缓存机制。不会触发reduce进行文件排序，防止多个机器访问同一个文件造成的namenode负担，多个任务在一个物理机器上时可以减少文件被复制的次数。

2. 倾斜数据的join——skew join：

     jnd = join daily by symbol, divs by symbol using ‘skewed'

第一个MapReduce中对输入数据进行抽样，找到数据量较多的key，把它分发到多个Reduce中（按照输入数据大小保证每块都能放入内存）

在第二个MapReduce中进行join，其他key在第一个MapReduce中进行标准的join

如果两边都数据倾斜的话，执行仍然很慢，但是可以保证最后的时间后能够完成。

只能接受两个连接，多个连接可以分解成一组join

分割器：和order一样，打破了同一个key进入同一个reducer的MapReduce规则

3. 对与排序的数据join——merge join：

     jnd = join daily by symbol, divs by symbol using ’merge‘

不需要reduce过程，因此更为高效，需要两个join的数据预先排序好

先对第二个输入进行抽样，建立一个索引。然后接受第一个输入，查找索引，来找到等值的数据块来放入一个map中进行计算

支持两两join

11. LIMIT：

     first10 = limit divs 10;

产生reduce： 先map输出n条，reduce输出n条，总会读取全部数据

Pig的优化：limit和order在一起使用，可以优化到一个MapReduce过程中

12. COGROUP：

基于一个key接受多个输入：

     cogroup A by id, B by id;

类似于join的前一半，把key收集到一起，但是不进行叉乘

13. CROSS：

     笛卡尔乘积：n,m=>n*m

14. 内嵌FOREACH：

     grpd = group daily by exchange;
     uniqcnt = foreach grpg {
           sym = daily.symbol;
          uniq_sym = distinct sym;
          generate group, count(uniq_sym);
     };

支持：distinct，filter，limit，order，flattern

## Pig的其他操作

15. FLATTERN：

     foreach players generate name, flatten(position) as position

把bag，tuple打平，嵌套级别降低

常在内嵌foreach中使用

多个bag则需要多个flattern，产生n*m行

坑：如果bag为空，则不会产生任何记录。避免这种情况需要预处理：

     foreach players generate name,((position is null or isEmpty(position)) ? {(‘unknown’)} : position)

16. PARALLEL：

     group daily by symbol parallel 10;

控制reduce的并行度

可以接在：group，order，distinct，join，cogroup，cross

默认reduce并行度修改：

     set default_parallel 10;

优化：pig会在没有设置并行度时估算一个值，比如1GB设置一个reducer

MAP并行度：由输入文件决定，可以通过加载函数来重新InputFormat方法，控制MapReduce中的map任务数量。

17. UNION：

     a = union a1,a2;

两个数据合并

与SQL不同：Pig不要求两个输入具有相同的模式（隐式转换，否则无模式）

不会省略重复记录（不需要Reduce）

union onschema A,B：强制生成一个通用的模式，通过名称进行匹配

18. STREAM：

     d = stream divs through ‘parse.py’ as (a,b,c)

在数据流中插入自定义的可执行任务

配套：ship命令来将可执行文件加载到集群

     define hd ‘highdiv.pl’ ship (‘highdiv.pl’);
     define hd ‘highdiv.pl’ ship (‘highdiv.pl’,’Financial.pm');

脚本中通过：标准输入/输出，或者指定 -i -o
脚本类型：python，php，sh，awk，……

19. SAMPLE：

     some = sample divs 0.1

等价于filter A by random() <= 0.1

20. SPLIT：

     split wlogs into apr03 if timestamp < ‘...’,apr02 if timestamp >’’ and timestamp <‘';

等价于filter的组合

21. MAPDEDUCE：

在数据流中直接添加MapReduce任务

## 预处理器

参数传入：

     pig -p DATE=2016-12-12 daily.pig
     pig -para_file daily.params daily.pig

宏： 定义一段代码
import： 可以配合宏来使用

# Pig的优化

## 影响pig性能的因素

* 输入数据的大小（map数量）
* shuffle数量的大小（map ->reduce的数量）
* 输出数据的大小（reduce数量）
* 中间结果数据量大小
* 内存大小

## 尽早并经常进行过滤

减少shuffle和存储的数据量

pig优化器会主动把过滤前置，比如distinct之后是filter，但是如果distinct，stream，filter则不会过滤。用户可以根据情况手动把filter前置

inner join之前过滤掉null数据会增强join效率

## 尽早并经常使用映射：

pig优化器会对于不需要的字段进行移除

有时候它判断不出来是否不需要，例如：COUNT(整行)，UDF出入整行

避免出入整行可以提高性能

## 增加并行数 vs 数据倾斜

在到达槽位后增加并行度并不会提高效率反而会因为网络和调度而降低

增加并行度并不解决数据倾斜的问题

## 其他

修改MapReduce/pig的配置可以调节执行效率

对于计算的中间结果可以压缩

优化数据层：对于大文件的操作效率更高？

# 总结

之前提到的那些坑：

* 自动认定类型时采用较为安全的方式，变慢和损失精度，应该进行类型转换或者声明
* 当不需要系统类型时可以不指定，例如：统计行数，bytearray转换类型时需要消耗资源
* UDF中使用Algebraic接口来声明可代数计算的函数，三个函数。
* PIg的大小写敏感：关键词不敏感，关系名敏感，UDF敏感
* desc只用于一个字段
* 自连接，需要load两次，以区分语义
* 标准join中把记录多的数据放在右边可以节省内存和加速
* 对几种特殊情况采用特殊的join可以提高效率：
     1. 小数据join大数据——分片-重复-join：
     2. 倾斜数据的join——skew join：
     3. 对与排序的数据join——merge join：
* 如果bag为空，则不会产生任何记录。避免这种情况需要预处理：

本文大部分观点来自于：

![pigbook](/images/pig/pig.png)
