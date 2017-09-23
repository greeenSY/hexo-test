---
title: Mysql引擎、索引结构、事务级别和锁
date: 2017-09-12 10:25:15
description: 整理和学习了Mysql引擎相关的知识
category: 数据库
tags: [mysql]
---

# MySQL存储引擎

## 概述

MySQL有多种存储引擎，每种存储引擎有各自的优缺点，可以择优选择使用：

    MyISAM、InnoDB、MERGE、MEMORY(HEAP)、BDB(BerkeleyDB)、EXAMPLE、FEDERATED、ARCHIVE、CSV、BLACKHOLE。

<!--more-->

比较常用的是MyISAM和InnoBD

- 两种类型最主要的差别就是Innodb 支持事务处理与外键和行级锁。
- MySQL5.5版本开始Innodb已经成为Mysql的默认引擎(之前是MyISAM)，说明其优势是有目共睹的，如果你不知道用什么，那就用InnoDB，至少不会差。
 基本上可以考虑使用InnoDB来替代MyISAM引擎了，原因是InnoDB自身很多良好的特点，比如事务支持、存储 过程、视图、行级锁定等等，在并发很多的情况下，相信InnoDB的表现肯定要比MyISAM强很多。另外，任何一种表都不是万能的，只用恰当的针对业务类型来选择合适的表类型，才能最大的发挥MySQL的性能优势。如果不是很复杂的Web应用，非关键应用，还是可以继续考虑MyISAM的

## MyISAM和InnoDB的对比

|         | MyISAM           | InnoDB  |
| ------------- |:-------------:| -----:|
| 外键：      | 不支持 | 支持 |
| 全文索引:      | 支持 FULLTEXT类型的全文索引      |   不支持FULLTEXT类型的全文索引，但是innodb可以使用sphinx插件支持全文索引，并且效果更好. |
| 表主键： | 允许没有任何索引和主键的表存在，索引都是保存行的地址。      |    如果没有设定主键或者非空唯一索引，就会自动生成一个6字节的主键(用户不可见)，数据是主索引的一部分，附加索引保存的是主索引的值。 |
| 表的具体行数： |保存有表的总行数，如果select count() from table;会直接取出出该值。 | 没有保存表的总行数，如果使用select count() from table；就会遍历整个表，消耗相当大，但是在加了wehre条件后，myisam和innodb处理的方式都一样。 |
| 存储空间： | 被压缩，存储空间较小。支持三种不同的存储格式：静态表(默认，但是注意数据末尾不能有空格，会被去掉)、动态表、压缩表。 | 需要更多的内存和存储，它会在主内存中建立其专用的缓冲池用于高速缓冲数据和索引。 |
| 可移植性、备份及恢复： | 数据是以文件的形式存储，所以在跨平台的数据转移中会很方便。在备份和恢复时可单独针对某个表进行操作。 | 免费的方案可以是拷贝数据文件、备份 binlog，或者用 mysqldump，在数据量达到几十G的时候就相对痛苦了。 |
| 构成上的区别： | 每个MyISAM在磁盘上存储成三个文件。第一个文件的名字以表的名字开始，扩展名指出文件类型。<br>.frm文件存储表定义。<br>数据文件的扩展名为.MYD (MYData)。<br> 索引文件的扩展名是.MYI (MYIndex)。 | 所有的表都保存在同一个数据文件中（也可能是多个文件，或者是独立的表空间文件）。<br> 基于磁盘的资源是InnoDB表空间数据文件和它的日志文件，InnoDB 表的大小只受限于操作系统文件的大小，一般为 2GB |
| 事务处理上方面: | MyISAM类型的表强调的是性能，其执行数度比InnoDB类型更快，但是不提供事务支持 | InnoDB提供事务支持事务，外部键（foreign key）等高级数据库功能 <br> 具有事务(commit)、回滚(rollback)和崩溃修复能力(crash recovery capabilities)的事务安全(transaction-safe (ACID compliant))型表。 |
| CRUD： | 如果执行大量的SELECT，MyISAM是更好的选择 | 1.如果你的数据执行大量的INSERT或UPDATE，出于性能方面的考虑，应该使用InnoDB表 <br> 2.DELETE   FROM table时，InnoDB不会重新建立表，而是一行一行的删除。在innodb上如果要清空保存有大量数据的表，最好使用truncate table这个命令。<br> 3.LOAD   TABLE FROM MASTER操作对InnoDB是不起作用的，解决方法是首先把InnoDB表改成MyISAM表，导入数据后再改成InnoDB表，但是对于使用的额外的InnoDB特性（例如外键）的表不适用 |
| 对AUTO_INCREMENT的操作: | 每表一个AUTO_INCREMEN列的内部处理。<br> MyISAM为INSERT和UPDATE操作自动更新这一列。这使得AUTO_INCREMENT列更快（至少10%）。在序列顶的值被删除之后就不能再利用。(当AUTO_INCREMENT列被定义为多列索引的最后一列，可以出现重使用从序列顶部删除的值的情况）。<br>AUTO_INCREMENT值可用ALTER TABLE或myisamch来重置 <br> 对于AUTO_INCREMENT类型的字段，InnoDB中必须包含只有该字段的索引，但是在MyISAM表中，可以和其他字段一起建立联合索引 <br> 更好和更快的auto_increment处理 | 如果你为一个表指定AUTO_INCREMENT列，在数据词典里的InnoDB表句柄包含一个名为自动增长计数器的计数器，它被用在为该列赋新值。 <br> 自动增长计数器仅被存储在主内存中，而不是存在磁盘上 <br> InnoDB中必须包含只有该字段的索引。引擎的自动增长列必须是索引，如果是组合索引也必须是组合索引的第一列。 |
| 表的具体行数: | select count(*) from table,MyISAM只要简单的读出保存好的行数，注意的是，当count(*)语句包含   where条件时，两种表的操作是一样的 | InnoDB 中不保存表的具体行数，也就是说，执行select count(*) from table时，InnoDB要扫描一遍整个表来计算有多少行 |
| 锁: | 表锁 <br> 用户在操作myisam表时，select，update，delete，insert语句都会给表自动加锁，如果加锁以后的表满足insert并发的情况下，可以在表的尾部插入新的数据。 | 提供行锁(locking on row level)，提供与 Oracle 类型一致的不加锁读取(non-locking read in SELECTs). <br> 另外，InnoDB表的行锁也不是绝对的，InnoDB的行锁，只是在WHERE的主键是有效的，非主键的WHERE都会锁全表的。 <br> 如果在执行一个SQL语句时MySQL不能确定要扫描的范围，InnoDB表同样会锁全表， 例如update table set num=1 where name like “%aaa%” |

## TRUNCATE TABLE

若要删除表中的所有行，则 TRUNCATE TABLE 语句是一种快速、有效的方法。TRUNCATE TABLE 与不含 WHERE 子句的 DELETE 语句类似。但是，TRUNCATE TABLE 速度更快，并且使用更少的系统资源和事务日志资源。

与 DELETE 语句相比，TRUNCATE TABLE 具有以下优点：

- 所用的事务日志空间较少。DELETE 语句每次删除一行，并在事务日志中为所删除的每行记录一个项。TRUNCATE TABLE 通过释放用于存储表数据的数据页来删除数据，并且在事务日志中只记录页释放。
- 使用的锁通常较少。当使用行锁执行 DELETE 语句时，将锁定表中各行以便删除。TRUNCATE TABLE 始终锁定表和页，而不是锁定各行。
- 如无例外，在表中不会留有任何页。执行 DELETE 语句后，表仍会包含空页。例如，必须至少使用一个排他 (LCK_M_X) 表锁，才能释放堆中的空表。如果执行删除操作时没有使用表锁，表（堆）中将包含许多空页。对于索引，删除操作会留下一些空页，尽管这些页会通过后台清除进程迅速释放。

与 DELETE 语句相同，使用 TRUNCATE TABLE 清空的表的定义与其索引和其他关联对象一起保留在数据库中。如果表中包含标识列，该列的计数器将重置为该列定义的种子值。如果未定义种子，则使用默认值 1。若要保留标识计数器，请使用 DELETE。

# MyISAM和Innodb的索引结构

## MyISAM引擎

MyISAM引擎使用B+Tree作为索引结构，叶节点的data域存放的是数据记录的地址。可以看出MyISAM的索引文件仅仅保存数据记录的地址。在MyISAM中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复。下图是MyISAM索引的原理图：

![myisam-index](/images/mysql/myisam-index.png)

## Innodb引擎

InnoDB的数据文件本身就是索引文件。从上文知道，MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。而在InnoDB中，表数据文件本身就是按B+Tree组织的一个索引结构，这棵树的叶节点data域保存了完整的数据记录。这个索引的key是数据表的主键，因此InnoDB表数据文件本身就是主索引。叶节点包含了完整的数据记录。这种索引叫做聚集索引。因为InnoDB的数据文件本身要按主键聚集，所以InnoDB要求表必须有主键（MyISAM可以没有）。

![innodb-index](/images/mysql/innodb-index.png)

第二个与MyISAM索引的不同是InnoDB的辅助索引data域存储相应记录主键的值而不是地址。换句话说，InnoDB的所有辅助索引都引用主键作为data域。

聚集索引这种实现方式使得按主键的搜索十分高效，但是辅助索引搜索需要检索两遍索引：首先检索辅助索引获得主键，然后用主键到主索引中检索获得记录。

## InnoDB的主键选择与插入优化

在使用InnoDB存储引擎时，如果没有特别的需要，请永远使用一个与业务无关的自增字段作为主键。

# Innodb的事务性级别

## ACID

ACID，是指数据库管理系统（DBMS）在写入或更新资料的过程中，为保证事务（transaction）是正确可靠的，所必须具备的四个特性：**原子性（atomicity，或称不可分割性）、一致性（consistency）、隔离性（isolation，又称独立性）、持久性（durability）**。

mysql中，innodb所提供的事务符合ACID的要求，而事务通过事务日志中的redo log和undo log满足了原子性、一致性、持久性，事务还会通过锁机制满足隔离性，在innodb存储引擎中，有不同的隔离级别，它们有着不同的隔离性。

## 四种隔离级别

我们先列出innodb中事务的所有隔离级别，然后再逐个了解它们，事务的隔离级别一共有如下4种。

- READ-UNCOMMITTED : 此隔离级别翻译为 "读未提交"。
- READ-COMMITTED : 此隔离级别翻译为 "读已提交" 或者 "读提交"。
- REPEATABLE-READ : 此隔离级别翻译为 "可重复读" 或者 "可重读"。
- SERIALIZABLE : 此隔离级别翻译为"串行化"。

而mysql默认设置的隔离级别为REPEATABLE-READ，即 "可重读"。使用如下语句可以查看当前设置的隔离级别：show variables like 'tx_isolation';可通过如下参数配置mysql的事务隔离级别，注意，不是使用tx_isolation，而是使用transaction_isolation

## 不可重复读和幻读的区别

不可重复读重点在于update和delete，而幻读的重点在于insert。

如果使用锁机制来实现这两种隔离级别，在可重复读中，该sql第一次读取到数据后，就将这些数据加锁，其它事务无法修改这些数据，就可以实现可重复读了。

但这种方法却无法锁住insert的数据，所以当事务A先前读取了数据，或者修改了全部数据，事务B还是可以insert数据提交，这时事务A就会发现莫名其妙多了一条之前没有的数据，这就是幻读，不能通过行锁来避免。

需要Serializable隔离级别 ，读用读锁，写用写锁，读锁和写锁互斥，这么做可以有效的避免幻读、不可重复读、脏读等问题，但会极大的降低数据库的并发能力。

使用悲观锁机制来处理这两种问题，但是MySQL、ORACLE、PostgreSQL等成熟的数据库，出于性能考虑，都是使用了**以乐观锁为理论基础的MVCC（多版本并发控制）**来避免这两种问题。

# Innodb的锁和MVCC

## 两段锁协议

数据库遵循的是两段锁协议，将事务分成两个阶段，加锁阶段和解锁阶段（所以叫两段锁）

- 加锁阶段：在该阶段可以进行加锁操作。在对任何数据进行读操作之前要申请并获得S锁（共享锁，其它事务可以继续加共享锁，但不能加排它锁），在进行写操作之前要申请并获得X锁（排它锁，其它事务不能再获得任何锁）。加锁不成功，则事务进入等待状态，直到加锁成功才继续执行。

- 解锁阶段：当事务释放了一个封锁以后，事务进入解锁阶段，在该阶段只能进行解锁操作不能再进行加锁操作。

## 事务隔离级别和锁

- 未提交读(Read Uncommitted)：允许脏读，也就是可能读取到其他会话中未提交事务修改的数据。不加锁
- 提交读(Read Committed)：只能读取到已经提交的数据。Oracle等多数数据库默认都是该级别 (不重复读)。数据的读取都是不加锁的，但是数据的写入、修改和删除是需要加锁的。
- 可重复读(Repeated Read)：可重复读。在同一个事务内的查询都是事务开始时刻一致的，InnoDB默认级别。在SQL标准中，该隔离级别消除了不可重复读，但是还存在幻象读
- 串行读(Serializable)：完全串行化的读，每次读都需要获得表级共享锁，读写相互都会阻塞。读加共享锁，写加排他锁，读写互斥。

## 共享锁和排他锁

共享锁又称读锁，是读取操作创建的锁。其他用户可以并发读取数据，但任何事务都不能对数据进行修改（获取数据上的排他锁），直到已释放所有共享锁。如果事务T对数据A加上共享锁后，则其他事务只能对A再加共享锁，不能加排他锁。获准共享锁的事务只能读数据，不能修改数据。

    SELECT ... LOCK IN SHARE MODE;

排他锁又称写锁，如果事务T对数据A加上排他锁后，则其他事务不能再对A加任任何类型的封锁。获准排他锁的事务既能读数据，又能修改数据。

    SELECT ... FOR UPDATE;

## 乐观锁，悲观锁和MVCC

在悲观锁的情况下，为了保证事务的隔离性，就需要一致性锁定读。读取数据时给加锁，其它事务无法修改这些数据。修改删除数据时也要加锁，其它事务无法读取这些数据。

乐观锁，大多是基于数据版本（ Version ）记录机制实现。何谓数据版本？即为数据增加一个版本标识，在基于数据库表的版本解决方案中，一般是通过为数据库表增加一个 “version” 字段来实现。读取出数据时，将此版本号一同读出，之后更新时，对此版本号加一。此时，将提交数据的版本数据与数据库表对应记录的当前版本信息进行比对，如果提交的数据版本号大于数据库表当前版本号，则予以更新，否则认为是过期数据。

## MVCC在MySQL的InnoDB中的实现：

在InnoDB中，会在每行数据后添加两个额外的隐藏的值来实现MVCC，这两个值一个记录这行数据何时被创建，另外一个记录这行数据何时过期（或者被删除）。 在实际操作中，存储的并不是时间，而是事务的版本号，每开启一个新事务，事务的版本号就会递增。 在可重读Repeatable reads事务隔离级别下：

- SELECT时，读取创建版本号<=当前事务版本号，删除版本号为空或>当前事务版本号。
- INSERT时，保存当前事务版本号为行的创建版本号
- DELETE时，保存当前事务版本号为行的删除版本号
- UPDATE时，插入一条新纪录，保存当前事务版本号为行创建版本号，同时保存当前事务版本号到原来删除的行

![innodb-mvcc](/images/mysql/innodb-mvcc.png)

通过MVCC，虽然每行记录都需要额外的存储空间，更多的行检查工作以及一些额外的维护工作，但可以减少锁的使用，大多数读操作都不用加锁，读数据操作很简单，性能很好，并且也能保证只会读取到符合标准的行，也只锁住必要行。在MySQL的RR级别中，是解决了幻读的读问题的。

在RR级别中，通过MVCC机制，虽然让数据变得可重复读，但我们读到的数据可能是历史数据，是不及时的数据，不是数据库当前的数据。对于这种读取历史数据的方式，我们叫它快照读 (snapshot read)，而读取数据库当前版本数据的方式，叫当前读 (current read)

当前读：

- select * from table where ? lock in share mode;
- select * from table where ? for update;
- insert;
- update ;
- delete;

为了解决当前读中的幻读问题，MySQL事务使用了Next-Key锁。Next-Key锁是行锁和GAP（间隙锁）的合并。

行锁可以防止不同事务版本的数据修改提交时造成数据冲突的情况。

但如何避免别的事务插入数据就成了问题。Innodb将这段数据分成几个个区间

- (negative infinity, 5],
- (5,30],
- (30,positive infinity)；


update class_teacher set class_name='初三四班' where teacher_id=30;不仅用行锁，锁住了相应的数据行；同时也在两边的区间，（5,30]和（30，positive infinity），都加入了gap锁。

这样事务B就无法在这个两个区间insert进新数据。受限于这种实现方式，Innodb很多时候会锁住不需要锁的区间。

行锁防止别的事务修改或删除，GAP锁防止别的事务新增，行锁和GAP锁结合形成的的Next-Key锁共同解决了RR级别在写数据时的幻读问题。

# 参考文献

* [MySQL存储引擎InnoDB与Myisam的六大区别][1]
* [MySQL存储引擎－－MyISAM与InnoDB区别][2]
* [MySQL存储引擎－－MyISAM与InnoDB区别][3]
* [使用 TRUNCATE TABLE 删除所有行][4]
* [事务隔离级别(事务总结之三)][5]
* [Innodb中的事务隔离级别和锁的关系][6]
* [ACID的Wiki][7]
* [MySQL索引背后的数据结构及算法原理][8]
* [MySQL中的共享锁与排他锁][9]


[1]: https://my.oschina.net/junn/blog/183341
[2]: http://blog.csdn.net/xifeijian/article/details/20316775
[3]: http://www.jianshu.com/p/a957b18ba40d
[4]: https://technet.microsoft.com/zh-cn/library/ms188249(v=sql.105).aspx
[5]: http://www.zsythink.net/archives/1233
[6]: https://tech.meituan.com/innodb-lock.html
[7]: https://zh.wikipedia.org/zh-hans/ACID
[8]: http://blog.codinglabs.org/articles/theory-of-mysql-index.html
[9]: http://www.hollischuang.com/archives/923