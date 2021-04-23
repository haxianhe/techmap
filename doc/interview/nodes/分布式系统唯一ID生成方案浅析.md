# 分布式系统唯一ID生成方案浅析

在复杂分布式系统中，往往需要对大量的数据和消息进行唯一标识。业务ID需要满足的要求如下

- 全局唯一性：不能出现重复的ID号，既然是唯一标识，这是最基本的要求。
- 趋势递增：在MySQL InnoDB引擎中使用的是聚集索引，由于多数RDBMS使用B-tree的数据结构来存储索引数据，在主键的选择上面我们应该尽量使用有序的主键保证写入性能。
- 单调递增：保证下一个ID一定大于上一个ID，例如事务版本号、IM增量消息、排序等特殊需求。
- 信息安全：如果ID是连续的，恶意用户的扒取工作就非常容易做了，直接按照顺序下载指定URL即可；如果是订单号就更危险了，竞对可以直接知道我们一天的单量。所以在一些应用场景下，会需要ID无规则、不规则。

## UUID

UUID(Universally Unique Identifier)的标准型式包含32个16进制数字，以连字号分为五段，形式为8-4-4-4-12的36个字符，示例：550e8400-e29b-41d4-a716-446655440000，到目前为止业界一共有5种方式生成UUID，详情见IETF发布的UUID规范[A Universally Unique IDentifier (UUID) URN Namespace](https://www.ietf.org/rfc/rfc4122.txt)

优点:

- 性能非常高：本地生成，没有网络消耗。

缺点:

- 不易于存储：UUID太长，16字节128位，通常以36长度的字符串表示，很多场景不适用。

- 信息不安全：基于MAC地址生成UUID的算法可能会造成MAC地址泄露，这个漏洞曾被用于寻找梅丽莎病毒的制作者位置。

UUID作为主键时在特定的环境会存在一些问题，比如做DB主键的场景下，UUID就非常不适用：

- MySQL官方有明确的建议主键要尽量越短越好，36个字符长度的UUID不符合要求。

>All indexes other than the clustered index are known as secondary indexes. In InnoDB, each record in a secondary index contains the primary key columns for the row, as well as the columns specified for the secondary index. InnoDB uses this primary key value to search for the row in the clustered index. **If the primary key is long, the secondary indexes use more space, so it is advantageous to have a short primary key**.

- 对MySQL索引不利：如果作为数据库主键，在InnoDB引擎下，UUID的无序性可能会引起数据位置频繁变动，严重影响性能。

```php
public static function v4() {
        return sprintf('%04x%04x-%04x-%04x-%04x-%04x%04x%04x',

            // 32 bits for "time_low"
            mt_rand(0, 0xffff), mt_rand(0, 0xffff),

            // 16 bits for "time_mid"
            mt_rand(0, 0xffff),

            // 16 bits for "time_hi_and_version",
            // four most significant bits holds version number 4
            mt_rand(0, 0x0fff) | 0x4000,

            // 16 bits, 8 bits for "clk_seq_hi_res",
            // 8 bits for "clk_seq_low",
            // two most significant bits holds zero and one for variant DCE1.1
            mt_rand(0, 0x3fff) | 0x8000,

            // 48 bits for "node"
            mt_rand(0, 0xffff), mt_rand(0, 0xffff), mt_rand(0, 0xffff)
        );
    }
```

## 类snowlack

这种方案大致来说是一种以划分命名空间（UUID也算，由于比较常见，所以单独分析）来生成ID的一种算法，这种方案把64-bit分别划分成多段，分开来标示机器、时间等，比如在snowflake中的64-bit分别表示如下图（图片来自网络）所示：

![snowlack](https://raw.githubusercontent.com/haxianhe/pic/master/image/20191011213141.png)

41-bit的时间可以表示（1L<<41）/(1000L*3600*24*365)=69年的时间，10-bit机器可以分别表示1024台机器。如果我们对IDC划分有需求，还可以将10-bit分5-bit给IDC，分5-bit给工作机器。这样就可以表示32个IDC，每个IDC下可以有32台机器，可以根据自身需求定义。12个自增序列号可以表示2^12个ID，理论上snowflake方案的QPS约为409.6w/s，这种分配方式可以保证在任何一个IDC的任何一台机器在任意毫秒内生成的ID都是不同的。

这种方式的优缺点是：

**优点：**

毫秒数在高位，自增序列在低位，整个ID都是趋势递增的。

不依赖数据库等第三方系统，以服务的方式部署，稳定性更高，生成ID的性能也是非常高的。

可以根据自身业务特性分配bit位，非常灵活。

**缺点：**

强依赖机器时钟，如果机器上时钟回拨，会导致发号重复或者服务会处于不可用状态。

**应用举例Mongdb objectID：**

[MongoDB官方文档](https://docs.mongodb.com/manual/reference/method/ObjectId/#description) ObjectID可以算作是和snowflake类似方法，通过“时间+机器码+pid+inc”共12个字节，通过4+3+2+3的方式最终标识成一个24长度的十六进制字

## 数据库生成

以MySQL举例，利用给字段设置auto_increment_increment和auto_increment_offset来保证ID自增，每次业务使用下列SQL读写MySQL得到ID号。

这种方案的优缺点如下：

**优点：**

- 非常简单，利用现有数据库系统的功能实现，成本小，有DBA专业维护。
ID号单调自增，可以实现一些对ID有特殊要求的业务。

**缺点：**

- 强依赖DB，当DB异常时整个系统不可用，属于致命问题。配置主从复制可以尽可能的增加可用性，但是数据一致性在特殊情况下难以保证。主从切换时的不一致可能会导致重复发号。
- ID发号性能瓶颈限制在单台MySQL的读写性能。

## 微信 seqsvr

不考虑 seqsvr 的具体架构的话，它应该是一个巨大的 64 位数组，而我们每一个微信用户，都在这个大数组里独占一格 8bytes 的空间，这个格子就放着用户已经分配出去的最后一个 sequence：cur_seq。每个用户来申请 sequence 的时候，只需要将用户的 cur_seq+=1，保存回数组，并返回给用户。

![图 1. 小明申请了一个 sequence，返回 101](https://raw.githubusercontent.com/haxianhe/pic/master/image/20191012163308.png)

图 1. 小明申请了一个 sequence，返回 101

**预分配中间层:**

任何一件看起来很简单的事，在海量的访问量下都会变得不简单。前文提到，seqsvr 需要保证分配出去的 sequence 递增（数据可靠），还需要满足海量的访问量（每天接近万亿级别的访问）。满足数据可靠的话，我们很容易想到把数据持久化到硬盘，但是按照目前每秒千万级的访问量（~10^7 QPS），基本没有任何硬盘系统能扛住。

后台架构设计很多时候是一门关于权衡的哲学，针对不同的场景去考虑能不能降低某方面的要求，以换取其它方面的提升。仔细考虑我们的需求，我们只要求递增，并没有要求连续，也就是说出现一大段跳跃是允许的（例如分配出的 sequence 序列：1,2,3,10,100,101）。于是我们实现了一个简单优雅的策略：

1. 内存中储存最近一个分配出去的 sequence：cur_seq，以及分配上限：max_seq
2. 分配 sequence 时，将 cur_seq++，同时与分配上限 max_seq 比较：如果 cur_seq > max_seq，将分配上限提升一个步长 max_seq += step，并持久化 max_seq
3. 重启时，读出持久化的 max_seq，赋值给 cur_seq

![图 2. 小明、小红、小白都各自申请了一个 sequence，但只有小白的 max_seq 增加了步长 100](https://raw.githubusercontent.com/haxianhe/pic/master/image/20191012163521.png)

图 2. 小明、小红、小白都各自申请了一个 sequence，但只有小白的 max_seq 增加了步长 100

这样通过增加一个预分配 sequence 的中间层，在保证 sequence 不回退的前提下，大幅地提升了分配 sequence 的性能。实际应用中每次提升的步长为 10000，那么持久化的硬盘 IO 次数从之前~10^7 QPS 降低到~10^3 QPS，处于可接受范围。在正常运作时分配出去的 sequence 是顺序递增的，只有在机器重启后，第一次分配的 sequence 会产生一个比较大的跳跃，跳跃大小取决于步长大小。

**分号段共享存储:**

请求带来的硬盘 IO 问题解决了，可以支持服务平稳运行，但该模型还是存在一个问题：重启时要读取大量的 max_seq 数据加载到内存中。

我们可以简单计算下，以目前 uid（用户唯一 ID）上限 2^32 个、一个 max_seq 8bytes 的空间，数据大小一共为 32GB，从硬盘加载需要不少时间。另一方面，出于数据可靠性的考虑，必然需要一个可靠存储系统来保存 max_seq 数据，重启时通过网络从该可靠存储系统加载数据。如果 max_seq 数据过大的话，会导致重启时在数据传输花费大量时间，造成一段时间不可服务。

为了解决这个问题，我们引入号段 Section 的概念，uid 相邻的一段用户属于一个号段，而同个号段内的用户共享一个 max_seq，这样大幅减少了 max_seq 数据的大小，同时也降低了 IO 次数。

![图 3. 小明、小红、小白属于同个 Section，他们共用一个 max_seq。在每个人都申请一个 sequence 的时候，只有小白突破了 max_seq 上限，需要更新 max_seq 并持久化](https://raw.githubusercontent.com/haxianhe/pic/master/image/20191012163622.png)

图 3. 小明、小红、小白属于同个 Section，他们共用一个 max_seq。在每个人都申请一个 sequence 的时候，只有小白突破了 max_seq 上限，需要更新 max_seq 并持久化

目前 seqsvr 一个 Section 包含 10 万个 uid，max_seq 数据只有 300+KB，为我们实现从可靠存储系统读取 max_seq 数据重启打下基础。

## 参考资料

**文章：**

- [高并发分布式系统唯一ID生成](https://juejin.im/entry/59eb02806fb9a0451049a0ab)
- [Leaf——美团点评分布式ID生成系统](https://tech.meituan.com/2017/04/21/mt-leaf.html)
- [Tinyid 原理介绍](https://github.com/didi/tinyid/wiki/tinyid%E5%8E%9F%E7%90%86%E4%BB%8B%E7%BB%8D)
- [微信序列号生成器架构设计及演变](https://www.infoq.cn/article/wechat-serial-number-generator-architecture/)

**各大公司的开源项目：**

- [twitter snowflake](https://github.com/twitter-archive/snowflake)
- [baidu uid-generator](https://github.com/baidu/uid-generator)
- [meituan leaf](https://github.com/Meituan-Dianping/Leaf)
- [didi tinyid](https://github.com/didi/tinyid)
