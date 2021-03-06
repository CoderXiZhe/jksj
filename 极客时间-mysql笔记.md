# 4. 索引

>索引的出现其实就是为了提高数据查询的效率，就像书的目录一样

## 4.1索引的常见模型

### 4.1.1 哈希表（K-V）

​	哈希表这种结构适用于**只有等值查询**的场景，比如Memcached及其他一些NoSQL引 擎。

### 4.1.2 有序数组

​	有序数组在等值查询和范围查询场景中的性能就都非常优秀。但是，在需要更新数据的时候就麻烦了。所以，有序数组索引**只适用于静态存储引擎**

### 4.1.3 二叉树

​	二叉树是搜索效率最高的，但是实际上大多数的数据库存储却并不使用二叉树。其原因 是，**索引不止存在内存中，还要写到磁盘上。**

### 4.1.4 N叉树

​	“N叉”树中的“N”取决于数据块的大小。N叉树由于在读写上的性能优点，以及适配磁盘的访问模式，已经被广泛应用在数据库引擎中 了。

## 4.2 InnoDB 的索引模型

> InnoDB使用了B+树索引模型，所以数据都是存储在B+树中的。
>
> B+树能够很好地配合磁盘的读写特性，减少单次查询的磁盘访问次数。

### 4.2.1 主键索引

​	主键索引也被称为聚簇索引

​	一般情况下**建议你创建一个自增主键**，这样非主键索引占用的空间最小

### 4.2.2 非主键索引

​	非主键索引也被称为二级索引

### 4.2.3 区别

+ 如果语句是select * from T where ID=500，即主键查询方式，则只需要搜索ID这棵B+树
+ 如果语句是select * from T where k=5，即普通索引查询方式，则需要先搜索k索引树，得到ID 的值为500，再到ID索引树搜索一次。这个过程称为回表。

也就是说，基于非主键索引的查询需要多扫描一棵索引树。因此，我们在应用中应该尽量使用主键查询。

## 4.3 索引维护

>B+树为了维护索引有序性，在插入新值的时候需要做必要的维护。

### 4.3.1 页分裂

​	某个记录所在的数据页满了，根据B+树的算法，这时候需要申请一个新的数据页，然后挪动部分数据过去。这个过程称为页分裂。

页分裂操作影响数据页的性能和利用率

### 4.3.2 合并

​	当相邻两个页由于删除了数据，利用率很低之后，会将数据页做合并。合并的过程，可以认为是分裂过程的逆过程。

### 4.3.3 自增主键的应用场景

#### 4.3.3.1 性能

+ 自增主键的插入数据模式，每次插入一条新记录，都是追加操作，都不涉及到挪动其他记录，也不会触发叶子节点的分裂。
+ 而有业务逻辑的字段做主键，则往往不容易保证有序插入，这样写数据成本相对较高。

#### 4.3.3.2 存储空间

+ 身份证号做主键，那么每个二级索引的叶子节点占用约20个字节，

+ 如果用整型做主键，则只要4个字节，

+ 如果是长整型（bigint）则是 8个字节。

显然，主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。

所以，从性能和存储空间方面考量，**自增主键往往是更合理的选择。**

### 4.3.4 业务字段直接做主键

1. 只有一个索引
2. 该索引必须是唯一索引

由于没有其他索引，所以也就不用考虑其他索引的叶子节点大小的问题。

直接将这个索引设置为主键， 可以避免每次查询需要搜索两棵树。

## 4.4 覆盖索引

>覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段

对比如下两个语句:

+ select * from T where k between 3 and 5
+ select ID from T where k between 3 and 5

第一种情况可能会回表，而第二种ID值已经在已经在k索引树上了，因此可以直接提供查询结果，不需要回表。

索引k已经“覆盖了”我们的查询需求，我们称为覆盖索引。



讨论问题：在一个市民信息表上，是否有必要将身份证号和名字建立联合索引？

+ 如果有根据身份证号查询市民信息的需求， 我们只要在身份证号字段上建立索引就够了。
+ 如果现在有一个高频请求，要根据市民的身份证号查询他的姓名，这个**联合索引**就有意义了。可以在这个高频请求上用到**覆盖索引**，**不再需要回表**查整行记录，减少语句的执行时间。当然，索引字段的维护总是有代价的。因此，在建立冗余索引来支持覆盖索引时就需要权衡考虑 了

## 4.5 最左前缀原则

> B+树这种索引结构，可以利用索引的“最左前缀”，来定位记录。只要满足最左前缀，就可以利用索引来加速检索。
>
> 这个最左前缀可以是联合索引的最左N个字段，也可以是字符串索引的最左M个字符

用（name，age）这个**联合索引**

![image-20220410122144755.png](https://github.com/CoderXiZhe/jksj/blob/main/images/image-20220410122144755.png?raw=true)

可以看到，索引项是按照索引定义里面出现的字段顺序排序的。

当你的逻辑需求是查到所有名字是“张三”的人时，可以快速定位到ID4，然后向后遍历得到所有 需要的结果。

如果你要查的是所有名字第一个字是“张”的人，你的SQL语句的条件是"where name like ‘张%’"。这时，你也能够用上这个索引，查找到第一个符合条件的记录是ID3，然后向后遍历， 直到不满足条件为止。



讨论问题： 在建立联合索引的时候，如何安排索引内的字段顺序。

1. 评估标准：**索引的复用能力**

所以当已经有了(a,b)这个联合索引后，一般就不需要单独在a上建立索引了。因此，第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。

2. 考虑的原则：**空间**

如果既有联合查询，又有基于a、b各自的查询呢？

查询条件里面只有b的语句，是无法使 用(a,b)这个联合索引的，这时候你不得不维护另外一个索引，也就是说你需要同时维护(a,b)、 (b) 这两个索引。

这时候，我们要考虑的原则就是空间了。比如上面这个市民表的情况，name字段是比age字段 大的 ，那我就建议你创建一个（name,age)的联合索引和一个(age)的单字段索引。

## 4.6 索引下推

> 不符合最左前缀的部分，会怎么样呢？

以市民表的联合索引（name, age）为例。如果现在有一个需求：检索出表中“名字第一 个字是张，而且年龄是10岁的所有男孩”。那么，SQL语句是这么写的：

```sql
mysql> select * from tuser where name like '张%' and age=10 and ismale=1;
```

这个语句在搜索索引树的时候，只能用 “张”，找到第一个满足 条件的记录ID3。当然，这还不错，总比全表扫描要好。

然后呢？ 当然是判断其他条件是否满足。

+ 在MySQL 5.6之前，只能从ID3开始一个个回表。到主键索引上找出数据行，再对比字段值
+ 而MySQL 5.6 引入的索引下推优化（index condition pushdown)， 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。

<img src="https://github.com/CoderXiZhe/jksj/blob/main/images/image-20220410124015813.png?raw=true" alt="image-20220410124015813.png" style="zoom: 67%;" /><img src="https://github.com/CoderXiZhe/jksj/blob/main/images/image-20220410124029046.png?raw=true" alt="image-20220410124029046.png" style="zoom:67%;" />

 	图3 无索引下推执行流程				图4 索引下执行流程



+ 图3中，这个过程InnoDB并不会去看age的值，需要回表4次。
+ 图4中，InnoDB在(name,age)索引内部就判断了age是否等于10，需要回表2次



# 5.全局锁和表级锁

> 根据加锁的范围，MySQL里面的锁大致可以分成全局锁、表级锁和行锁三类

## 5.1 全局锁

> 全局锁就是对整个数据库实例加锁
>
> 全局锁主要用在逻辑备份过程中。对于全部是InnoDB引擎的库，我建议你选择使用–singletransaction参数，对应用会更友好。

1. MySQL提供了一个加**全局读锁**的方法，命令是 Flush tables with read lock (FTWRL)。

2. 全局锁的典型使用场景是，做全库逻辑备份

3. 备份为什么要加锁呢？不加锁的话，备份系统备份的得到的库不是一个逻辑时间点，这个视图是逻辑不一致 的。
4. 可重复读隔离级别也能够拿到一致性视图，但前提是引擎要支持这个隔离级别。当mysqldump使用参数–single-transaction的时候，导 数据之前就会启动一个事务，来确保拿到一致性视图。single-transaction方法只适用于所有的表使用事务引擎的库

## 5.2 表级锁

> MySQL里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。



### 5.2.1 表锁

+ 表锁的语法是 lock tables …read/write。
+ 而对于InnoDB这种支持行锁的引擎，一般不使用lock tables命令来控制并发，毕竟锁住整个表的影响面还是太大。
+ 表锁一般是在数据库引擎不支持行锁的时候才会被用到的



### 5.2.2 MDL锁

1. MDL不需要显式使用，在访问一个表的时候会被自动加上。MDL的作用是，保证读写的正确性。

2. 在MySQL 5.5版本中引入了MDL，当对一个表做增删改查操作的时候，加MDL读锁；当要对表做结构变更操作的时候，加MDL写锁。

   + 读锁之间不互斥，因此你可以有多个线程同时对一张表增删改查。
   + 读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。因此，如果有两个线 程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。

3. 例子：

   给一个小表加个字段，导致整个库挂了。（这里的实验环境是MySQL 5.6）

   <img src="https://github.com/CoderXiZhe/jksj/blob/main/images/image-20220411122149289.png?raw=true" alt="image-20220411122149289.png" style="zoom:80%;" />

我们可以看到session A先启动，这时候会对表t加一个MDL读锁。由于session B需要的也是 MDL读锁，因此可以正常执行。

之后session C会被blocked，是因为session A的MDL读锁还没有释放，而session C需要MDL写 锁，因此只能被阻塞。

如果只有session C自己被阻塞还没什么关系，但是之后所有要在表t上新申请MDL读锁的请求也 会被session C阻塞

所有对表的增删改查操作都需要先申请MDL读锁，就都被锁住，等于这个表现在完全不可读写了。 

如果某个表上的查询语句频繁，而且客户端有重试机制，也就是说超时后会再起一个新session 再请求的话，这个库的线程很快就会爆满。

**事务中的MDL锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。**



4. 如何安全地给小表加字段？
   1. 解决长事务，事务不提交，就会一直占着MDL锁
   2. 如果你要做DDL变更的表刚好有长事务 在执行，要考虑先暂停DDL，或者kill掉这个长事务
   3. 如果你要变更的表是一个热点表，虽然数据量不大，但是上面的请求很频繁，这时候kill可能未必管用，比较理想的机制是，在alter table语句里面 设定等待时间，如果在这个指定的等待时间里面能够拿到MDL写锁最好，拿不到也不要阻塞后 面的业务语句，先放弃。之后开发人员或者DBA再通过重试命令重复这个过程。![image-20220411122711929.png](https://github.com/CoderXiZhe/jksj/blob/main/images/image-20220411122711929.png?raw=true)

## 5.3 行锁

> 行锁就是针对数据表中行记录的锁
>
> 比如事务A更新了一行，而这时候 事务B也要更新同一行，则必须等事务A的操作完成后才能进行更新
>
> 并不是所有的引擎都支持行锁, InnoDB是支持行锁的， 这也是MyISAM被InnoDB替代的重要原因之一。

### 5.3.1 两阶段锁

<img src="https://github.com/CoderXiZhe/jksj/blob/main/images/image-20220412103821653.png?raw=true" alt="image-20220412103821653.png" style="zoom:50%;" />

实际上事务B的update语句会被阻塞，直到事务A执行commit之后，事务B才 能继续执行。

两阶段锁协议：在InnoDB事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。

因此，如果你的事务中需要锁多个行，**要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。**



例子：顾客A要在影院B购买电影票，涉及到以下操作

1. 从顾客A账户余额中扣除电影票价； 
2. 给影院B的账户余额增加这张电影票价； 
3. 记录一条交易日志。

要完成这个交易，我们需要update两条记录，并insert一条记录。

为了保证交易的原子性，我们要把这三个操作放在一个事务中。那么，你会怎样安排这三个语句在事务中的顺序呢？

试想如果同时有另外一个顾客C要在影院B买票，那么这两个事务冲突的部分就是语句2了。因为 它们要更新同一个影院账户的余额，需要修改同一行数据。

根据两阶段锁协议，不论你怎样安排语句顺序，所有的操作需要的行锁都是在事务提交的时候才释放的。所以，如果你把语句2安排在最后，比如按照3、1、2这样的顺序，那么影院账户余额这一行的锁时间就最少。这就最大程度地减少了事务之间的锁等待，提升了并发度。

但是，调整语句顺序并不能完全避免死锁。所以我们引入了死锁和死锁检测的概念

### 5.3.2 死锁和死锁检测

> 当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致 这几个线程都进入无限等待的状态，称为死锁

<img src="https://github.com/CoderXiZhe/jksj/blob/main/images/image-20220412110553627.png?raw=true" alt="image-20220412110553627.png" style="zoom:50%;" />

这时候，事务A在等待事务B释放id=2的行锁，而事务B在等待事务A释放id=1的行锁。 

事务A和 事务B在互相等待对方的资源释放，就是进入了死锁状态。

解锁策略：

+ 直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout（InnoDB默认50S）来设置。
+ 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其他事 务得以继续执行。将参数innodb_deadlock_detect设置为on，表示开启这个逻辑

正常情况下采取第二种：主动死锁检测，但是它也是有额外负担的。

​	负担如：每当一个事务被锁的时候，就要看看它所依赖的线程有没有被别人锁 住，如此循环，最后判断是否出现了循环等待，也就是死锁

**怎么解决由这种热点行更新导致的性能问题呢？**

+ 一种头痛医头的方法，就是如果你能确保这个业务一定不会出现死锁，可以临时把死锁检测关掉(有一定的风险)
+ 另一个思路是控制并发度

