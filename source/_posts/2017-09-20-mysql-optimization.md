---
title: Mysql查询优化
date: 2017-09-20 15:45:01
description: 整理和学习了Mysql查询优化相关的知识
category: 数据库
tags: [mysql]
---

MySQL查询优化器有几个目标，但是其中**最主要的目标是尽可能地使用索引**，并且使用最严格的索引来 消除尽可能多的数据行。你的最终目标是提交SELECT语句查找 数据行，而不是排除数据行。优化器试图排除数据行的原因在 于它排除数据行的速度越快，那么找到与条件匹配的数据行也 就越快。如果能够首先进行最严格的测试，查询就可以执行地 更快。

<!--more-->

# 索引的优化

## 索引的原理

### 磁盘IO

考虑到磁盘IO是非常高昂的操作，计算机操作系统做了一些优化，当一次IO时，不光把当前磁盘地址的数据，而是把相邻的数据也都读取到内存缓冲区内，因为局部预读性原理告诉我们，当计算机访问一个地址的数据的时候，与其相邻的数据也会很快被访问到。每一次IO读取的数据我们称之为一页(page)。具体一页有多大数据跟操作系统有关，一般为4k或8k，也就是我们读取一页内的数据时候，实际上才发生了一次IO，这个理论对于索引的数据结构设计非常有帮助。

### Mysql索引结构：B+树

Mysql的索引结构，每次查找数据时把磁盘IO次数控制在一个很小的数量级，最好是常数数量级。B+树。

![innodb-tree](/images/mysql/innodb-tree.png)

浅蓝色的块我们称之为一个磁盘块，可以看到每个磁盘块包含几个数据项（深蓝色所示）和指针（黄色所示），非叶子节点只不存储真实的数据，只存储指引搜索方向的数据项。

真实的情况是，3层的b+树可以表示上百万的数据，如果上百万的数据查找只需要三次IO，性能提高将是巨大的，如果没有索引，每个数据项都要发生一次IO，那么总共需要百万次的IO，显然成本非常非常高。

IO次数取决于b+数的高度h，假设当前数据表的数据为N，每个磁盘块的数据项的数量是m，则有h=㏒(m+1)N，当数据量N一定的情况下，m越大，h越小；而m = 磁盘块的大小 / 数据项的大小，磁盘块的大小也就是一个数据页的大小，是固定的，如果数据项占的空间越小，数据项的数量越多，树的高度越低。这就是为什么每个数据项，即索引字段要尽量的小。

### 索引的最左匹配特性

当b+树的数据项是复合的数据结构，比如(name,age,sex)的时候，b+数是按照从左到右的顺序来建立搜索树的

- 比如当(张三,20,F)这样的数据来检索的时候，b+树会优先比较name来确定下一步的所搜方向，如果name相同再依次比较age和sex，最后得到检索的数据；
- 但当(20,F)这样的没有name的数据来的时候，b+树就不知道下一步该查哪个节点，因为建立搜索树的时候name就是第一个比较因子，必须要先根据name来搜索才能知道下一步去哪里查询。
- 比如当(张三,F)这样的数据来检索时，b+树可以用name来指定搜索方向，但下一个字段age的缺失，所以只能把名字等于张三的数据都找到，然后再匹配性别是F的数据了。

## 索引简历的原则

1. 最左前缀匹配原则，非常重要的原则，mysql会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如a = 1 and b = 2 and c > 3 and d = 4 如果建立(a,b,c,d)顺序的索引，d是用不到索引的，如果建立(a,b,d,c)的索引则都可以用到，a,b,d的顺序可以任意调整。
2. =和in可以乱序，比如a = 1 and b = 2 and c = 3 建立(a,b,c)索引可以任意顺序，mysql的查询优化器会帮你优化成索引可以识别的形式。
3. 尽量选择区分度高的列作为索引,区分度的公式是count(distinct col)/count(*)，表示字段不重复的比例，比例越大我们扫描的记录数越少，唯一键的区分度是1，而一些状态、性别字段可能在大数据面前区分度就是0。
4. 索引列不能参与计算，保持列“干净”，比如from_unixtime(create_time) = ’2014-05-29’就不能使用到索引，原因很简单，b+树中存的都是数据表中的字段值，但进行检索时，需要把所有元素都应用函数才能比较，显然成本太大。所以语句应该写成create_time = unix_timestamp(’2014-05-29’);
5. 尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可

# MYSQL缺省参数调整

MySQL可以通过调整其缺省设置以获得更优的性能 和更高的稳定性，需要优化的关键变量主要有：

- 索引缓冲区长度（key_buffer）
- 读缓冲区的长度（read_buffer_size）
- 打开表的数目的最大值（table_cache）
- 检索时间限制（long_query_time）

不赘述。

# SQL的优化

## 执行优化的步骤

0.先运行看看是否真的很慢，注意设置SQL_NO_CACHE
1.where条件单表查，锁定最小返回记录表。这句话的意思是把查询语句的where都应用到表中返回的记录数最小的表开始查起，单表每个字段分别查询，看哪个字段的区分度最高
2.explain查看执行计划，是否与1预期一致（从锁定记录较少的表开始查询）
3.order by limit 形式的sql语句让排序的表优先查
4.了解业务方使用场景
5.加索引时参照建索引的几大原则
6.观察结果，不符合预期继续从0分析

## explain

查询优化神器 - explain命令

需要强调rows是核心指标，绝大部分rows小的语句执行一定很快（有例外，下面会讲到）。所以优化语句基本上都是在优化rows。


分析查询性能时，考虑EXPLAIN关键字同样非常管用。 EXPLAIN命令一般放在SELECT查询语句的前面，用来描述MySQL如何执行查询 操作，以及MySQL成功返回结果所需要执行的行数，例如：

    mysql> EXPLAIN SELECT city.name, city.district FROM city, country WHERE city.countrycode = country.code AND country.code = 'IND';

    +----+-------------+---------+-------+---------------+---------+---------+-------+------+-------------+
    | id | select_type | table   | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
    +----+-------------+---------+-------+---------------+---------+---------+-------+------+-------------+
    |  1 | SIMPLE      | country | const | PRIMARY       | PRIMARY | 3       | const |    1 | Using index |
    |  1 | SIMPLE      | city    | ALL   | NULL          | NULL    | NULL    | NULL | 4079 | Using where |
    +----+-------------+---------+-------+---------------+---------+---------+-------+------+-------------+

属性解释：

**id**

显示待分析的SELECT语句的ID。如果语句不包含子查询或者union，则ID均为1。SQL执行的顺利的标识,SQL从大到小的执行。（SQL是从里向外的执行,就是从id=3 向上执行）

**select_type**

- SIMPLE：简单SELECT(不使用UNION或子查询等)
- PRIMARY：最外层的select
- UNION：UNION中的第二个或后面的SELECT语句
- DEPENDENT UNION：UNION中的第二个或后面的SELECT语句，取决于外面的查询（where id in ？）
- UNION RESULT：UNION的结果。
- SUBQUERY：子查询中的第一个SELECT.（where id =？）
- DEPENDENT SUBQUERY：子查询中的第一个SELECT，取决于外面的查询（where id in？）
- DERIVED：派生表的SELECT(FROM子句的子查询)

**table**

输出行引用的表名  （有时不是真实的表名字,看到的是derivedx(x是个数字,我的理解是第几步执行的结果)）

**type**

这列很重要,显示了连接使用了哪种类别,有无使用索引.

从最好到最差的连接类型为：system-->const-->eq_reg-->ref-->fulltext-->ref_or_null-->index_merge-->unique_subquery-->index_subquery-->range-->index-->ALL

一般来说得保证查询至少达到range级别，当然了最好能达到ref级别。

- system：这是const联接类型的一个特例。表仅有一行满足条件.如：explain select * from (select * from t3 where id=3952602) a ; 分别是system和const。
- const：表最多有一个匹配行，它将在查询开始时被读取。因为仅有一行，在这行的列值可被优化器剩余部分认为是常数。const表很快，因为它们只读取一次。const用于用常数值比较PRIMARY KEY或UNIQUE索引的所有部分时。
- eq_ref：对于每个来自于前面的表的行组合，从该表中读取一行。这可能是最好的联接类型，除了const类型。它用在一个索引的所有部分被联接使用并且索引是UNIQUE或PRIMARY KEY。
- ref：对于每个来自于前面的表的行组合，所有有匹配索引值的行将从这张表中读取。如果联接只使用键的最左边的前缀，或如果键不是UNIQUE或PRIMARY KEY（换句话说，如果联接不能基于关键字选择单个行的话），则使用ref。如果使用的键仅仅匹配少量行，该联接类型是不错的。
- ref_or_null：该联接类型如同ref，但是添加了MySQL可以专门搜索包含NULL值的行。在解决子查询中经常使用该联接类型的优化。例如：SELECT * FROM ref_table WHERE key_column=expr OR key_column IS NULL;
- index_merge：该联接类型表示使用了索引合并优化方法。在这种情况下，key列包含了使用的索引的清单，key_len包含了使用的索引的最长的关键元素。例如：select * from t4 where id=3952602 or accountid=31754306 ;
- unique_subquery：该类型替换了下面形式的IN子查询的ref：value IN (SELECT primary_key FROM single_table WHERE some_expr)。unique_subquery是一个索引查找函数，可以完全替换子查询，效率更高。
- index_subquery：该联接类型类似于unique_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引：value IN (SELECT key_column FROM single_table WHERE some_expr)
- range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引。key_len包含所使用索引的最长关键元素。在该类型中ref列为NULL。当使用=、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN或者IN操作符，用常量比较关键字列时，可以使用range
- index：该联接类型与ALL相同，除了只有索引树被扫描。这通常比ALL快，因为索引文件通常比数据文件小。
- all：对于每个来自于先前的表的行组合，进行完整的表扫描。如果表是第一个没标记const的表，这通常不好，并且通常在它情况下很差。通常可以增加更多的索引而不要使用ALL，使得行能基于前面的表中的常数值或列值被检索出。

**possible_keys**

显示可能应用在这张表中的索引。

如果为空，没有可能的索引。可以通过检查WHERE子句看是否它引用某些列或适合索引的列来提高你的查询性能。如果是这样，创造一个适当的索引并且再次用EXPLAIN检查查询

**key**

实际使用的索引。如果为NULL，则没有使用索引。

要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。

**key_len**

使用的索引的长度。

在不损失精确性的情况下，长度越短越好。

对于多重主键，该值可以看出实际使用了哪一部分

**ref**

ref列显示使用哪个列或常数与key一起从表中选择行。

**rows**

MYSQL认为必须检查的用来返回请求数据的行数

**Extra**

关于MYSQL如何解析查询的额外信息。

坏的例子包括Using temporary和Using filesort，意思MYSQL根本不能使用索引，结果是检索会很慢。如果是Using index则说明结果很理想，只需采用索引树的信息即可得到结果。

- Distinct：一旦MYSQL找到了与行相联合匹配的行，就不再搜索了
- Not exists：MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行， 就不再搜索了
- Range checked for each：Record（index map:#） 没有找到理想的索引，因此对于从前面表中来的每一个行组合，MYSQL检查使用哪个索引，并用它来从表中返回行。这是使用索引的最慢的连接之一
- Using filesort：看到这个的时候，查询就需要优化了。MYSQL需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行
- Using index：列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候
- Using temporary： 看到这个的时候，查询需要优化了。这里，MYSQL需要创建一个临时表来存储结果，这通常发生在对不同的列集进行ORDER BY上，而不是GROUP BY上
- Using where：使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。如果不想返回表中的全部行，并且连接类型ALL或index，这就会发生，或者是查询有问题

## 慢查询案例（来自[美团点评团队][2]）

下面几个例子详细解释了如何分析和优化慢查询

1.复杂语句写法

很多情况下，我们写SQL只是为了实现功能，这只是第一步，不同的语句书写方式对于效率往往有本质的差别，这要求我们对mysql的执行计划和索引原则有非常清楚的认识，请看下面的语句


    select
      distinct cert.emp_id
    from
      cm_log cl
    inner join
      (
          select
            emp.id as emp_id,
            emp_cert.id as cert_id
          from
            employee emp
          left join
            emp_certificate emp_cert
                on emp.id = emp_cert.emp_id
          where
            emp.is_deleted=0
      ) cert
          on (
            cl.ref_table='Employee'
            and cl.ref_oid= cert.emp_id
          )
          or (
            cl.ref_table='EmpCertificate'
            and cl.ref_oid= cert.cert_id
          )
    where
      cl.last_upd_date >='2013-11-07 15:03:00'
      and cl.last_upd_date<='2013-11-08 16:00:00';

先运行一下，53条记录 1.87秒，又没有用聚合语句，比较慢

    53 rows in set (1.87 sec)

explain

    +----+-------------+------------+-------+---------------------------------+-----------------------+---------+-------------------+-------+--------------------------------+
    | id | select_type | table      | type  | possible_keys                  | key                  | key_len | ref              | rows  | Extra                          |
    +----+-------------+------------+-------+---------------------------------+-----------------------+---------+-------------------+-------+--------------------------------+
    |  1 | PRIMARY    | cl        | range | cm_log_cls_id,idx_last_upd_date | idx_last_upd_date    | 8      | NULL              |  379 | Using where; Using temporary  |
    |  1 | PRIMARY    | <derived2> | ALL  | NULL                            | NULL                  | NULL    | NULL              | 63727 | Using where; Using join buffer |
    |  2 | DERIVED    | emp        | ALL  | NULL                            | NULL                  | NULL    | NULL              | 13317 | Using where                    |
    |  2 | DERIVED    | emp_cert  | ref  | emp_certificate_empid          | emp_certificate_empid | 4      | meituanorg.emp.id |    1 | Using index                    |
    +----+-------------+------------+-------+---------------------------------+-----------------------+---------+-------------------+-------+--------------------------------+


简述一下执行计划，首先mysql根据idx_last_upd_date索引扫描cm_log表获得379条记录；然后查表扫描了63727条记录，分为两部分，derived表示构造表，也就是不存在的表，可以简单理解成是一个语句形成的结果集，后面的数字表示语句的ID。derived2表示的是ID = 2的查询构造了虚拟表，并且返回了63727条记录。我们再来看看ID = 2的语句究竟做了写什么返回了这么大量的数据，首先全表扫描employee表13317条记录，然后根据索引emp_certificate_empid关联emp_certificate表，rows = 1表示，每个关联都只锁定了一条记录，效率比较高。获得后，再和cm_log的379条记录根据规则关联。从执行过程上可以看出返回了太多的数据，返回的数据绝大部分cm_log都用不到，因为cm_log只锁定了379条记录。

如何优化呢？可以看到我们在运行完后还是要和cm_log做join,那么我们能不能之前和cm_log做join呢？仔细分析语句不难发现，其基本思想是如果cm_log的ref_table是EmpCertificate就关联emp_certificate表，如果ref_table是Employee就关联employee表，我们完全可以拆成两部分，并用union连接起来，注意这里用union，而不用union all是因为原语句有“distinct”来得到唯一的记录，而union恰好具备了这种功能。如果原语句中没有distinct不需要去重，我们就可以直接使用union all了，因为使用union需要去重的动作，会影响SQL性能。

优化过的语句如下

    select
      emp.id
    from
      cm_log cl
    inner join
      employee emp
          on cl.ref_table = 'Employee'
          and cl.ref_oid = emp.id
    where
      cl.last_upd_date >='2013-11-07 15:03:00'
      and cl.last_upd_date<='2013-11-08 16:00:00'
      and emp.is_deleted = 0
    union
    select
      emp.id
    from
      cm_log cl
    inner join
      emp_certificate ec
          on cl.ref_table = 'EmpCertificate'
          and cl.ref_oid = ec.id
    inner join
      employee emp
          on emp.id = ec.emp_id
    where
      cl.last_upd_date >='2013-11-07 15:03:00'
      and cl.last_upd_date<='2013-11-08 16:00:00'
      and emp.is_deleted = 0

不需要了解业务场景，只需要改造的语句和改造之前的语句保持结果一致

现有索引可以满足，不需要建索引

用改造后的语句实验一下，只需要10ms 降低了近200倍！

    +----+--------------+------------+--------+---------------------------------+-------------------+---------+-----------------------+------+-------------+
    | id | select_type  | table      | type  | possible_keys                  | key              | key_len | ref                  | rows | Extra      |
    +----+--------------+------------+--------+---------------------------------+-------------------+---------+-----------------------+------+-------------+
    |  1 | PRIMARY      | cl        | range  | cm_log_cls_id,idx_last_upd_date | idx_last_upd_date | 8      | NULL                  |  379 | Using where |
    |  1 | PRIMARY      | emp        | eq_ref | PRIMARY                        | PRIMARY          | 4      | meituanorg.cl.ref_oid |    1 | Using where |
    |  2 | UNION        | cl        | range  | cm_log_cls_id,idx_last_upd_date | idx_last_upd_date | 8      | NULL                  |  379 | Using where |
    |  2 | UNION        | ec        | eq_ref | PRIMARY,emp_certificate_empid  | PRIMARY          | 4      | meituanorg.cl.ref_oid |    1 |            |
    |  2 | UNION        | emp        | eq_ref | PRIMARY                        | PRIMARY          | 4      | meituanorg.ec.emp_id  |    1 | Using where |
    | NULL | UNION RESULT | <union1,2> | ALL    | NULL                            | NULL              | NULL    | NULL                  | NULL |            |
    +----+--------------+------------+--------+---------------------------------+-------------------+---------+-----------------------+------+-------------+
    53 rows in set (0.01 sec)

2.明确应用场景

举这个例子的目的在于颠覆我们对列的区分度的认知，一般上我们认为区分度越高的列，越容易锁定更少的记录，但在一些特殊的情况下，这种理论是有局限性的

    select
      *
    from
      stage_poi sp
    where
      sp.accurate_result=1
      and (
          sp.sync_status=0
          or sp.sync_status=2
          or sp.sync_status=4
      );

先看看运行多长时间,951条数据6.22秒，真的很慢

    951 rows in set (6.22 sec)

先explain，rows达到了361万，type = ALL表明是全表扫描


    +----+-------------+-------+------+---------------+------+---------+------+---------+-------------+
    | id | select_type | table | type | possible_keys | key  | key_len | ref  | rows    | Extra      |
    +----+-------------+-------+------+---------------+------+---------+------+---------+-------------+
    |  1 | SIMPLE      | sp    | ALL  | NULL          | NULL | NULL    | NULL | 3613155 | Using where |
    +----+-------------+-------+------+---------------+------+---------+------+---------+-------------+


所有字段都应用查询返回记录数，因为是单表查询 0已经做过了951条

让explain的rows 尽量逼近951

看一下accurate_result = 1的记录数


    select count(*),accurate_result from stage_poi  group by accurate_result;
    +----------+-----------------+
    | count(*) | accurate_result |
    +----------+-----------------+
    |    1023 |              -1 |
    |  2114655 |              0 |
    |  972815 |              1 |
    +----------+-----------------+

我们看到accurate_result这个字段的区分度非常低，整个表只有-1,0,1三个值，加上索引也无法锁定特别少量的数据

再看一下sync_status字段的情况


    select count(*),sync_status from stage_poi  group by sync_status;
    +----------+-------------+
    | count(*) | sync_status |
    +----------+-------------+
    |    3080 |          0 |
    |  3085413 |          3 |
    +----------+-------------+

同样的区分度也很低，根据理论，也不适合建立索引

问题分析到这，好像得出了这个表无法优化的结论，两个列的区分度都很低，即便加上索引也只能适应这种情况，很难做普遍性的优化，比如当sync_status 0、3分布的很平均，那么锁定记录也是百万级别的

找业务方去沟通，看看使用场景。业务方是这么来使用这个SQL语句的，每隔五分钟会扫描符合条件的数据，处理完成后把sync_status这个字段变成1,五分钟符合条件的记录数并不会太多，1000个左右。了解了业务方的使用场景后，优化这个SQL就变得简单了，因为业务方保证了数据的不平衡，如果加上索引可以过滤掉绝大部分不需要的数据

根据建立索引规则，使用如下语句建立索引

    alter table stage_poi add index idx_acc_status(accurate_result,sync_status);

观察预期结果,发现只需要200ms，快了30多倍。

    952 rows in set (0.20 sec)

我们再来回顾一下分析问题的过程，单表查询相对来说比较好优化，大部分时候只需要把where条件里面的字段依照规则加上索引就好，如果只是这种“无脑”优化的话，显然一些区分度非常低的列，不应该加索引的列也会被加上索引，这样会对插入、更新性能造成严重的影响，同时也有可能影响其它的查询语句。所以我们第4步调差SQL的使用场景非常关键，我们只有知道这个业务场景，才能更好地辅助我们更好的分析和优化查询语句。

3.无法优化的语句

    select
      c.id,
      c.name,
      c.position,
      c.sex,
      c.phone,
      c.office_phone,
      c.feature_info,
      c.birthday,
      c.creator_id,
      c.is_keyperson,
      c.giveup_reason,
      c.status,
      c.data_source,
      from_unixtime(c.created_time) as created_time,
      from_unixtime(c.last_modified) as last_modified,
      c.last_modified_user_id
    from
      contact c
    inner join
      contact_branch cb
          on  c.id = cb.contact_id
    inner join
      branch_user bu
          on  cb.branch_id = bu.branch_id
          and bu.status in (
            1,
          2)
      inner join
          org_emp_info oei
            on  oei.data_id = bu.user_id
            and oei.node_left >= 2875
            and oei.node_right <= 10802
            and oei.org_category = - 1
      order by
          c.created_time desc  limit 0 ,
          10;

还是几个步骤

先看语句运行多长时间，10条记录用了13秒，已经不可忍受

    10 rows in set (13.06 sec)

explain

    +----+-------------+-------+--------+-------------------------------------+-------------------------+---------+--------------------------+------+----------------------------------------------+
    | id | select_type | table | type  | possible_keys                      | key                    | key_len | ref                      | rows | Extra                                        |
    +----+-------------+-------+--------+-------------------------------------+-------------------------+---------+--------------------------+------+----------------------------------------------+
    |  1 | SIMPLE      | oei  | ref    | idx_category_left_right,idx_data_id | idx_category_left_right | 5      | const                    | 8849 | Using where; Using temporary; Using filesort |
    |  1 | SIMPLE      | bu    | ref    | PRIMARY,idx_userid_status          | idx_userid_status      | 4      | meituancrm.oei.data_id  |  76 | Using where; Using index                    |
    |  1 | SIMPLE      | cb    | ref    | idx_branch_id,idx_contact_branch_id | idx_branch_id          | 4      | meituancrm.bu.branch_id  |    1 |                                              |
    |  1 | SIMPLE      | c    | eq_ref | PRIMARY                            | PRIMARY                | 108    | meituancrm.cb.contact_id |    1 |                                              |
    +----+-------------+-------+--------+-------------------------------------+-------------------------+---------+--------------------------+------+----------------------------------------------+

从执行计划上看，mysql先查org_emp_info表扫描8849记录，再用索引idx_userid_status关联branch_user表，再用索引idx_branch_id关联contact_branch表，最后主键关联contact表。

rows返回的都非常少，看不到有什么异常情况。我们在看一下语句，发现后面有order by + limit组合，会不会是排序量太大搞的？于是我们简化SQL，去掉后面的order by 和 limit，看看到底用了多少记录来排序

    select
      count(*)
    from
      contact c
    inner join
      contact_branch cb
          on  c.id = cb.contact_id
    inner join
      branch_user bu
          on  cb.branch_id = bu.branch_id
          and bu.status in (
            1,
          2)
      inner join
          org_emp_info oei
            on  oei.data_id = bu.user_id
            and oei.node_left >= 2875
            and oei.node_right <= 10802
            and oei.org_category = - 1
    +----------+
    | count(*) |
    +----------+
    |  778878 |
    +----------+
    1 row in set (5.19 sec)

发现排序之前居然锁定了778878条记录，如果针对70万的结果集排序，将是灾难性的，怪不得这么慢，那我们能不能换个思路，先根据contact的created_time排序，再来join会不会比较快呢？

于是改造成下面的语句，也可以用straight_join来优化

    select
    c.id,
    c.name,
    c.position,
    c.sex,
    c.phone,
    c.office_phone,
    c.feature_info,
    c.birthday,
    c.creator_id,
    c.is_keyperson,
    c.giveup_reason,
    c.status,
    c.data_source,
    from_unixtime(c.created_time) as created_time,
    from_unixtime(c.last_modified) as last_modified,
    c.last_modified_user_id
    from
    contact c
    where
    exists (
    select
    1
    from
    contact_branch cb
    inner join
    branch_user bu
    on cb.branch_id = bu.branch_id
    and bu.status in (
    1,
    2)
    inner join
    org_emp_info oei
    on oei.data_id = bu.user_id
    and oei.node_left >= 2875
    and oei.node_right <= 10802
    and oei.org_category = - 1
    where
    c.id = cb.contact_id
    )
    order by
    c.created_time desc limit 0 ,
    10;

验证一下效果 预计在1ms内，提升了13000多倍！

    ```sql
    10 rows in set (0.00 sec)

本以为至此大工告成，但我们在前面的分析中漏了一个细节，先排序再join和先join再排序理论上开销是一样的，为何提升这么多是因为有一个limit！大致执行过程是：mysql先按索引排序得到前10条记录，然后再去join过滤，当发现不够10条的时候，再次去10条，再次join，这显然在内层join过滤的数据非常多的时候，将是灾难的，极端情况，内层一条数据都找不到，mysql还傻乎乎的每次取10条，几乎遍历了这个数据表！
用不同参数的SQL试验下

    select
      sql_no_cache  c.id,
      c.name,
      c.position,
      c.sex,
      c.phone,
      c.office_phone,
      c.feature_info,
      c.birthday,
      c.creator_id,
      c.is_keyperson,
      c.giveup_reason,
      c.status,
      c.data_source,
      from_unixtime(c.created_time) as created_time,
      from_unixtime(c.last_modified) as last_modified,
      c.last_modified_user_id
    from
      contact c
    where
      exists (
          select
            1
          from
            contact_branch cb
          inner join
            branch_user bu
                on  cb.branch_id = bu.branch_id
                and bu.status in (
                  1,
                2)
            inner join
                org_emp_info oei
                  on  oei.data_id = bu.user_id
                  and oei.node_left >= 2875
                  and oei.node_right <= 2875
                  and oei.org_category = - 1
            where
                c.id = cb.contact_id
          )
      order by
          c.created_time desc  limit 0 ,
          10;
    Empty set (2 min 18.99 sec)

2 min 18.99 sec！比之前的情况还糟糕很多。由于mysql的nested loop机制，遇到这种情况，基本是无法优化的。这条语句最终也只能交给应用系统去优化自己的逻辑了。
通过这个例子我们可以看到，并不是所有语句都能优化，而往往我们优化时，由于SQL用例回归时落掉一些极端情况，会造成比原来还严重的后果。所以，第一：不要指望所有语句都能通过SQL优化，第二：不要过于自信，只针对具体case来优化，而忽略了更复杂的情况。

# 总结

**关于Where条件：**

1：对查询进行优化，应尽量避免全表扫描，首先应考虑在where及order by 涉及的列上创建索引。

因为：索引对查询的速度有着至关重要的影响。

2：尽量避免在where字句中对字段进行null值的判断。否则将会导致引擎放弃使用索引而进行全表扫描。

3：应尽量避免在where子句中使用!=或者是<>操作符号。否则引擎将放弃使用索引，进而进行全表扫描。

4：应尽量避免在where子句中使用or来连接条件，否则导致放弃使用索引而进行全表扫描。可以使用 union 或者是 union all代替。

5：in 和 not in 也要慎用，否则将会导致全表扫描。

in 对于连续的数组，可以使用between ...and.来代替。

6：like使用需注意

下面这个查询也将导致全表查询：

    select id from user where name like '%三'；

而下面这个查询却使用到了索引：

    select id from user where name like '张%'；

7：where子句参数使用时候需注意

如果在where子句中使用参数，也会导致全表扫描。因为sql只会在运行时才会解析局部变量。但优化程序不能将访问计划的选择推迟到运行时；必须在编译时候进行选择。然而，如果在编译时建立访问计划，变量的值还是未知大，因而无法作为索引选择输入项。

8：尽量避免在where子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。

例如：

    select id from user where num/2=100

应修改为：

    select id from user where num = 100*2;

9：尽量避免爱where子句中对字段进行函数操作，这将导致引擎放弃索引，而进行全表扫描。

例如：

    select id from user substring(name,1,3) = 'abc' ，这句sql的含义其实就是，查询name以abc开头的用户id

(注：substring(字段，start,end)这个是mysql的截取函数)

应修改为：

    select id from user where name like 'abc%';

10：不要在where子句中的"="左边进行函数、算术运算或是使用其他表达式运算，否则系统可能无法正确使用索引

**关于索引：**
11：复合索引查询注意（最左前缀匹配原则）

在使用索引字段作为条件时候，如果该索引是复合索引，那么必须使用该索引中的第一个字段作为条件时候才能保证系统使用该所以，否则该索引将不会被使用，并且应尽可能的让字段顺序和索引顺序一致。

12：并不是所有索引对查询都有效，sql是根据表中数据进行查询优化的，当索引lie(索引字段)有大量重复数据的时候，sql查询可能不会去利用索引。如一表中字段 sex、male、female 几乎各一半。那么即使在sex上创建了索引对查询效率也起不了多大作用。

13：索引创建需注意

并非索引创建越多越好。索引固然可以提高相应的查询效率，但是同样会降低insert以及update的效率。因为在insert或是update的时候有可能会重建索引或是修改索引。所以索引怎样创建需要慎重考虑，视情况而定。一个表中所以数量最好不要超过6个。若太多，则需要考虑一些不常用的列上创建索引是否有必要。

14：索引字段要尽量的小

15：尽量比较数据类型相同的数据列。当你在比 较操作中使用索引数据列的时候，请使用数据类型相同的列。相同 的数据类型比不同类型的性能要高一些。

尽可能地让索引列在比较表达式中独立。如果 你在函数调用或者更复杂的算术表达式条件中使用了某个数据列， MySQL就不会使用索引，因为它必须计算出每个数据行的表达式值。

16：包含NULL值的列不能 添加索引

17：尽量选择区分度高的列作为索引

18：尽量的扩展索引，不要新建索引。比如表中已经有a的索引，现在要加(a,b)的索引，那么只需要修改原来的索引即可

19：MYSQL分页limit速度太慢优化方法

- 在mysql中limit可以实现快速分页，但是如果数据到了几百万时我们的limit必须优化才能有效的合理的实现分页了，否则可能卡死你的服务器哦。
- 当一个数据库表过于庞大，LIMIT offset, length中的offset值过大，则SQL查询语句会非常缓慢，你需增加order by，并且order by字段需要建立索引。
- 第一页会很快。先找出第一条数据，然后大于等于这条数据的id就是要获取的数据

**其他：**

20：不要写一些没意义的查询。

例如：需要生成一个空表结构和user表结构一样(注：生成的新 new table的表结构和 老表 old table 结构一致)

    select col1,col2,col3.....into newTable from user where 1=0

上面这行sql执行后不会返回任何的结果集，但是会消耗系统资源的。

21：很多时候用exists 代替 in是一个很好的选择。

比如：

    select num from user where num in(select num from newTable);

可以使用下面语句代替：

    select num from user a where exists(select num from newTable b where b.num = a.num );


# 其他优化手段

- 如果MySQL版本小于5.5，那么升级版本到5.5以后，最好是最新版本，5.5对in的操作有了飞跃性的提高。
- 增加内存，开大innodb_buffer_pool,增加pool可以可以缓存page的空间，让尽可能多的数据都缓存。
- 改善磁盘配置，用ssd或者flash卡存储，提高磁盘扫描速度

# 参考文献

* [关系数据库原理-MySQL查询优化][1]
* [MySQL索引原理及慢查询优化][2]
* [MySQL 5.5 Reference Manual][3]

[1]: http://cbb.sjtu.edu.cn/course/database/lab6.htm
[2]: https://tech.meituan.com/mysql-index.html
[3]: https://dev.mysql.com/doc/refman/5.5/en/explain-output.html#explain_type
