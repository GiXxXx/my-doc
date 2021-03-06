## ACID

* A，Atomicity，原子性。简单理解即事务中的操作要么都成功，要么都失败。
* C，Consistency，一致性。无论事务并发量多大，事务执行前后，系统仍保持一致状态。如账户间互相转账，转账事务结束之后，账户总余额不变。
* I，Isolation，隔离性。对同一数据处理得并发事务之间互不影响，执行效果如同串行执行。
* D，Durability，持久性。事务完成之后，更新会被持久的保存。

## 事务并发的问题

* 脏读：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
* 不可重复读：事务 A 多次读取同一数据，事务 B 在事务A读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致。
* 幻读：系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。

## 事务隔离级别

* read-uncommitted
* read-committed。可解决脏读的问题。写数据会锁住行。
* repeatable-read。默认的隔离级别，可解决脏读和不可重复读的问题。
* serializable。读写会锁表，可解决脏读、不可重复读和幻读的问题。

隔离级别越高，越能保证数据的完整性和一致性，但是对并发性能的影响也越大。

### REPEATABLE READ

事务中的第一次读操作会建立快照，后续读操作结果会基于快照来保证事务中读操作的一致性。

SELECT ... LOCK IN SHARE MODE、SELECT ... FOR UPDATE、UPDATE、DELETE语句使用以下锁：

* 在唯一索引上使用结果唯一的搜索条件，InnoDB引擎只锁住查找到的index，不会锁住index前的间隙。
* 使用间隙锁或next-key lock锁住被扫描的index范围，阻塞其他事务在其中插入数据。

### READ COMMITTED

事务中每次读都设置及刷新各自快照。SELECT ... LOCK IN SHARE MODE、SELECT ... FOR UPDATE、UPDATE、DELETE语句只会锁住index记录而不是间隙。

UPDATE语句在该级别下使用“semi-consistent”读：如果行已被锁住，从而检查最新提交的版本数据是否满足WHERE条件，满足则再次读取并对这些行加锁或阻塞。

UPDATE、DELETE语句会在WHERE语句执行后立即释放不满足条件行的锁，而不是等到事务结束。

## 锁

### 共享锁（又称读锁，shared locks）、排它锁（又称写锁，exclusive locks）、意向锁（intention locks）

* 共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。
* 排他锁（X)：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。
* 意向共享锁（IS）：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的IS锁。
* 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的IX锁。

共享锁和排他锁都是行锁，意向锁都是表锁，应用中我们只会使用到共享锁和排他锁，意向锁是mysql内部使用的，不需要用户干预。
意向锁的作用就是当一个事务在需要获取资源锁定的时候，如果遇到自己需要的资源已经被排他锁占用的时候，该事务可以需要锁定行的表上面添加一个合适的意向锁。

对于普通SELECT语句，InnoDB不会加任何锁，事务可以通过以下语句显示给记录集加共享锁或排他锁：
* 共享锁（S）：SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE。
* 排他锁（X)：SELECT * FROM table_name WHERE ... FOR UPDATE。

InnoDB行锁是通过给索引上的索引项加锁来实现的，因此InnoDB这种行锁实现特点意味着：只有通过索引条件检索数据，InnoDB才使用行级锁，否则，InnoDB将使用表锁！

即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。

### 记录锁（record locks）

锁住一条索引记录。例如SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE，会阻止其他事务增删改c1 = 10的记录。

在read committed级别下，如果没有匹配的记录，则记录锁会在where执行后释放。

### 间隙锁（gap locks）

* 锁住一个范围的索引记录。例如SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE，会阻止其他事务插入c1为15的记录。
* 使用unique index查询单条记录不会使用间隙锁。如SELECT * FROM child WHERE id = 100。如果示例中id不为unique index则会使用间隙锁。
* 间隙锁的目的是阻止其他事务在范围内插入数据。
* 共享间隙锁和排他间隙锁没有差别，可同时存在于相同间隙。原因是为了使在索引中被清除的记录会被其他事务合并。
* 将事务隔离级别改为read committed后，在搜索和索引扫描时会禁用间隙锁。此时间隙锁只用来做外键约束和重复键的检查。

### next-key locks

* next-key锁是记录锁和间隙锁的组合，锁在索引记录和该记录之前的间隙，即一个事务锁住索引为R的记录后，其他事务无法在R之前的间隙插入新记录。
* 如果索引包含10, 11, 13, 20，则可能会存在以下next-key锁：
```
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```
* 在repeatable read隔离级别下，InnoDB使用在搜索以及扫描index时使用next-key锁来防止幻读。

### 插入意向锁（insert intention locks）

插入意向锁是insert操作在插入行之前设置的一种间隙锁。用来表明一种插入意向，当多个事务在同一个索引间隙不同位置插入时不需要互相等待。如两个事务在4~7索引值间隙中插入5和6，在获取排它锁之前，使用插入意向锁锁住索引间隙。

### 自增锁（AUTO-INC locks）

是一种特殊的表级锁，事务向包含AUTO_INCREMENT列的表插入时使用。

### 乐观锁、悲观锁

* 悲观锁：总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁。
* 乐观锁：顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁。可使用版本号或CAS（compare and swap）机制。

CAS问题：
* ABA问题。A变成B又变回A，如使用版本号机制，则当前的A版本为3，而期望的A版本应是1。
* CAS自旋问题。即swap失败后循环重试直到成功。资源竞争较大时，使用悲观锁可以降低CPU资源占用。
* 不支持多个共享变量的原子操作。

## MVCC

每行记录有两个隐藏字段：创建时间和删除时间，保存对应操作的事务id。

* insert时，创建时间保存当前事务id。
* delete时，删除时间保存当前事务id。
* update时，删除时间保存当前事务id，并添加新记录，其创建时间保存当前事务id。
* 查询时，返回满足以下条件的row：
  * 创建时间小于等于当前事务id
  * 删除时间未定义或大于当前事务id

## 参考资料

* https://www.jianshu.com/p/4e3edbedb9a8
* https://www.cnblogs.com/huanongying/p/7021555.html
* https://www.cnblogs.com/protected/p/6526857.html
* https://www.cnblogs.com/aipiaoborensheng/p/5767459.html
* https://www.cnblogs.com/dongqingswt/p/3460440.html
* https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html