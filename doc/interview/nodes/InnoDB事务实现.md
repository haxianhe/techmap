>我们都知道事务的几种性质，数据库为了维护这些性质，尤其是一致性和隔离性，一般使用加锁这种方式。
>
>同时数据库又是个高并发的应用，同一时间会有大量的并发访问，如果加锁过度，会极大的降低并发处理能力。
>
>所以对于加锁的处理，可以说就是数据库对于事务处理的精髓所在。
>
>这里通过分析MySQL中InnoDB引擎的事务处理来理解事务、锁、隔离级别、MVCC、Next-Key Locks等概念。

# 事务

## 概念

事务是指满足ACID特性的一组操作，可以通过 Commit 提交一个事务，也可以使用 Rollback 进行回滚。

![](https://raw.githubusercontent.com/haxianhe/pic/master/image/20190812172411.png)

## ACID

**原子性（Atomicity）**

事务被视为不可分割的最小单元，事务的所有操作要么全部提交成功，要么全部失败回滚。

回滚可以用回滚日志来实现，回滚日志记录着事务所执行的修改操作，在回滚时反向执行这些修改操作即可。

**一致性（Consistency）**

数据库在事务执行前后都保持一致性状态。在一致性状态下，所有事务对一个数据的读取结果都是相同的。

**隔离性（Isolation）**

一个事务所做的修改在最终提交以前，对其他事务是不可见的。

**持久性（Durability）**

一旦事务提交，则其所做的修改将会永远保存到数据库中。即使系统发生崩溃，事务执行的结果也不能丢。

使用重做日志来保证持久性。

## ACID 之间的关系

事务的 ACID 特性概念简单，但不是很好理解，主要是因为这几个特性不是一种平级关系：

- 只有满足一致性，事务的执行结果才是正确的。 
- 在无并发的情况下，事务串行执行，隔离性一定能够满足。此时只要能满足原子性，就
一定能满足一致性。 
- 在并发的情况下，多个事务并行执行，事务不仅要满足原子性，还需要满足隔离性，才能满足一致性。
- 事务满足持久化是为了能应对数据库崩溃的情况。

![](https://raw.githubusercontent.com/haxianhe/pic/master/image/20190812193340.png)

# 隔离级别

**未提交读（READ UNCOMMITTED）**

事务中的修改，即使没有提交，对其他事务也是可见的。

**提交读（READ COMMITTED）**

一个事务只能读取已经提交的事务所做的修改。换句话说，一个事务所做的修改在提交之前对其他事务是不可见的。

**可重复读（REPEATABLE READ）**

保证在同一个事务中多次读取同样数据的结果是一样的。

**可串行化（SERIALIZABLE）**

强制事务串行执行。

需要加锁实现，而其它隔离级别通常不需要。


| 隔离级别 | 脏读 | 不可重复读 | 幻影读 |
| :---: | :---: | :---:| :---: |
| 未提交读 | √ | √ | √ |
| 提交读 | × | √ | √ |
| 可重复读 | × | × | √ |
| 可串行化 | × | × | × |


# InnoDB 可重复读的实现

**在 InnoDB 中，SELECT、UPDATE、DELETE 操作的不可重复读问题可以通过 MVCC 来解决，但是 INSERT 操作的不可重复读问题（幻读问题）需要通过 MVCC + Next-Key Locks 来解决。**

## 不可重复读和幻读

**不可重复读**

T2 读取一个数据，T1 对该数据做了修改。如果 T2 再次读取这个数据，此时读取的结果和第一次读取的结果不同。

![](https://raw.githubusercontent.com/haxianhe/pic/master/image/20190812200910.png)

**幻读**

T1 读取某个范围的数据，T2 在这个范围内插入新的数据，T1 再次读取这个范围的数据，此时读取的结果和和第一次读取的结果不同。

![](https://raw.githubusercontent.com/haxianhe/pic/master/image/20190812201057.png)

可以看出，幻读问题是不可重复读问题的子集。

## 不可重复读和幻读的区别

很多人容易搞混不可重复读和幻读，确实这两者有些相似。但不可重复读重点在于update和delete，而幻读的重点在于insert。

如果使用锁机制来实现这两种隔离级别，在可重复读中，该sql第一次读取到数据后，就将这些数据加锁，其它事务无法修改这些数据，就可以实现可重复读了。但这种方法却无法锁住insert的数据，所以当事务A先前读取了数据，或者修改了全部数据，事务B还是可以insert数据提交，这时事务A就会发现莫名其妙多了一条之前没有的数据，这就是幻读，不能通过行锁来避免。需要Serializable隔离级别 ，读用读锁，写用写锁，读锁和写锁互斥，这么做可以有效的避免幻读、不可重复读、脏读等问题，但会极大的降低数据库的并发能力。

所以说不可重复读和幻读最大的区别，就在于如何通过锁机制来解决他们产生的问题。

上文说的，是使用悲观锁机制，但是**MySQL、ORACLE、PostgreSQL等成熟的数据库，出于性能考虑，都是使用了以乐观锁为理论基础的MVCC（多版本并发控制）。**

**在 InnoDB 中，SELECT、UPDATE、DELETE 操作的不可重复读问题可以通过 MVCC 来解决，但是 INSERT 操作的不可重复读问题（幻读问题）需要通过 MVCC + Next-Key Locks 来解决。**

# 多版本并发控制（MVCC）

多版本并发控制（Multi-Version Concurrency Control, MVCC）是 MySQL 的 InnoDB 存储引擎实现隔离级别的一种具体方式，用于实现提交读和可重复读这两种隔离级别。而未提交读隔离级别总是读取最新的数据行，无需使用 MVCC。可串行化隔离级别需要对所有读取的行都加锁，单纯使用 MVCC 无法实现。

## 基础概念

**版本号**

- 系统版本号：是一个递增的数字，每开始一个新的事务，系统版本号就会自动递增。
- 事务版本号：事务开始时的系统版本号。

**隐藏的列**

MVCC 在每行记录后面都保存着两个隐藏的列，用来存储两个版本号：

- 创建版本号：指示创建一个数据行的快照时的系统版本号；
- 删除版本号：如果该快照的删除版本号大于当前事务版本号表示该快照有效，否则表示该快照已经被删除了。

**Undo 日志**

MVCC 使用到的快照存储在 Undo 日志中，该日志通过回滚指针把一个数据行（Record）的所有快照连接起来。

![](https://raw.githubusercontent.com/haxianhe/pic/master/image/20190812205848.png)

## 实现过程

以下实现过程针对可重复读隔离级别。

当开始一个事务时，该事务的版本号肯定大于当前所有数据行快照的创建版本号，理解这一点很关键。数据行快照的创建版本号是创建数据行快照时的系统版本号，系统版本号随着创建事务而递增，因此新创建一个事务时，这个事务的系统版本号比之前的系统版本号都大，也就是比所有数据行快照的创建版本号都大。

**SELECT**

多个事务必须读取到同一个数据行的快照，并且这个快照是距离现在最近的一个有效快照。但是也有例外，如果有一个事务正在修改该数据行，那么它可以读取事务本身所做的修改，而不用和其它事务的读取结果一致。

把没有对一个数据行做修改的事务称为 T，T 所要读取的数据行快照的创建版本号必须小于等于 T 的版本号，因为如果大于 T 的版本号，那么表示该数据行快照是其它事务的最新修改，因此不能去读取它。除此之外，T 所要读取的数据行快照的删除版本号必须是未定义或者大于 T 的版本号，因为如果小于等于 T 的版本号，那么表示该数据行快照是已经被删除的，不应该去读取它。

**INSERT**

将当前系统版本号作为数据行快照的创建版本号。

**DELETE**

将当前系统版本号作为数据行快照的删除版本号。

**UPDATE**

将当前系统版本号作为更新前的数据行快照的删除版本号，并将当前系统版本号作为更新后的数据行快照的创建版本号。可以理解为先执行 DELETE 后执行 INSERT。

## 快照读与当前读

在可重复读级别中，通过MVCC机制，虽然让数据变得可重复读，但我们读到的数据可能是历史数据，是不及时的数据，不是数据库当前的数据！这在一些对于数据的时效特别敏感的业务中，就很可能出问题。

对于这种读取历史数据的方式，我们叫它快照读 (snapshot read)，而读取数据库当前版本数据的方式，叫当前读 (current read)。很显然，在MVCC中：

快照读：就是select。

- select * from table ….;

当前读：特殊的读操作，插入/更新/删除操作，属于当前读，处理的都是当前的数据，需要加锁。

- select * from table where ? lock in share mode;
- select * from table where ? for update;
- insert;
- update ;
- delete;

事务的隔离级别实际上都是定义了当前读的级别，MySQL为了减少锁处理（包括等待其它锁）的时间，提升并发能力，引入了快照读的概念，使得select不用加锁。而update、insert这些“当前读”，就需要另外的模块来解决了。

# Next-Key Locks

Next-Key Locks 是 MySQL 的 InnoDB 存储引擎的一种锁实现。

MVCC 不能解决幻影读问题，Next-Key Locks 就是为了解决这个问题而存在的。在可重复读（REPEATABLE READ）隔离级别下，使用 MVCC + Next-Key Locks 可以解决幻读问题。

## Record Locks

锁定一个记录上的索引，而不是记录本身。

如果表没有设置索引，InnoDB 会自动在主键上创建隐藏的聚簇索引，因此 Record Locks 依然可以使用。

## Gap Locks

锁定索引之间的间隙，但是不包含索引本身。例如当一个事务执行以下语句，其它事务就不能在 t.c 中插入 15。

```sql
SELECT c FROM t WHERE c BETWEEN 10 and 20 FOR UPDATE;
```

## Next-Key Locks

它是 Record Locks 和 Gap Locks 的结合，不仅锁定一个记录上的索引，也锁定索引之间的间隙。例如一个索引包含以下值：10, 11, 13, and 20，那么就需要锁定以下区间：

```sql
(-∞, 10]
(10, 11]
(11, 13]
(13, 20]
(20, +∞)
```

## 示例讲解

MySQL是这么实现的：

在class_teacher这张表中，teacher_id是个索引，那么它就会维护一套B+树的数据关系，为了简化，我们用链表结构来表达（实际上是个树形结构，但原理相同）

![](https://raw.githubusercontent.com/haxianhe/pic/master/image/20190812211402.png)

如图所示，InnoDB使用的是聚集索引，teacher_id身为二级索引，就要维护一个索引字段和主键id的树状结构（这里用链表形式表现），并保持顺序排列。

Innodb将这段数据分成几个个区间

- (negative infinity, 5],
- (5,30],
- (30,positive infinity)；

update class_teacher set class_name=‘初三四班’ where teacher_id=30;不仅用行锁，锁住了相应的数据行；同时也在两边的区间，（5,30]和（30，positive infinity），都加入了gap锁。这样事务B就无法在这个两个区间insert进新数据。

受限于这种实现方式，Innodb很多时候会锁住不需要锁的区间。如下所示：

|事务A|	事务B|	事务C|
|-----|------|-------|
|begin;|begin;|	begin;|
|select id,class_name,teacher_id from class_teacher;|||
|表字段：id class_name teacher_id|||
|update class_teacher set class_name='初一一班' where teacher_id=20;|||	 	
||insert into class_teacher values (null,'初三五班',10);waiting .....|insert into class_teacher values (null,'初三五班',40);|
|commit;|事务A commit之后，这条语句才插入成功|commit;|
||commit;||

update的teacher_id=20是在(5，30]区间，即使没有修改任何数据，Innodb也会在这个区间加gap锁，而其它区间不会影响，事务C正常插入。

如果使用的是没有索引的字段，比如update class_teacher set teacher_id=7 where class_name=‘初三八班（即使没有匹配到任何数据）’,那么**会给全表加入gap锁**。同时，它不能像上文中行锁一样经过MySQL Server过滤自动解除不满足条件的锁，因为没有索引，则这些字段也就没有排序，也就没有区间。除非该事务提交，否则其它事务无法插入任何数据。

**行锁防止别的事务修改或删除，GAP锁防止别的事务新增，行锁和GAP锁结合形成的的Next-Key锁共同解决了RR级别在写数据时的幻读问题。**

欢迎关注我的公众号：荒古传说        

<div align="center">
    <br>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://raw.githubusercontent.com/haxianhe/pic/master/image/20190806153414.png">
</div>

>本文作者： 荒古        
本文链接： https://haxianhe.com/2019/08/12/MySQL学习笔记之InnoDB事务处理/       
版权声明： 本博客所有文章除特别声明外，均采用 CC BY-NC-SA 3.0 许可协议。转载请注明出处！