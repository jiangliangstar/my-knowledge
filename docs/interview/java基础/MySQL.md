
# MySQL 深分页如何优化
## 为什么会慢
```
select p.* from tb_user p  
```
server 层
语法解析 ：sql 是否正确
	⬇️
预处理：表是否存在，字段是否存在
	⬇️
优化SQL：

引擎层，找数据：从第 0 ~ 50100 条
也就是慢的原因，页分的越深，数据越多，越慢

拿到这 50100 条数据，会返回 server 层，然后根据 limit 以及条件、分组等等筛选100的数据

如果涉及到二级索引（回表），并且查询字段没有用到索引覆盖再结合深分页，会更慢
```
select p.* from tb_user p where age > 1 limit 50000,100

create index idx_age on tb_user(age)
创建二级索引
```

二级索引的 B+ 树上的叶子节点就会存储 age 和 主键 ID 这两个数据
<div align="center"> <img src="" width=""/> </div>
![[Pasted image 20250714151941.png]]
当查询的字段，涉及到 ID 和 age 之外的，此时就会进行回表

```
select p.* from tb_user p where age > 1 limit 50000,100
```
从引擎层拿到 age > 1 第 0 条到 50100 条数据，返回到 server 层，由于查询字段涉及到 ID 和 age 之外的，需要来到 1 级索引的 B+ 树，也就是主键索引的 B+ 树，因为所有的数据都在主键索引的 B+ 树上，这个过程称之为**回表**，涉及到回表的性能会进一步下降。

![[Pasted image 20250714152935.png]]


![[Pasted image 20250714151806.png]]
## 如何优化呢

### 1 将字段改成索引覆盖
```
SELECT id,age from tb_user p where age > 1 limit 50000,100
```

### 2 子查询
如果非要查询二级索引以外的数据 name，这个时候就会涉及到回表
```
SELECT id,age,  name  from tb_user p where age > 1 limit 50000,100;

可以用子查询代替 ( select id from tb_user where age > 1 LIMIT 50000,100)，子查询进行索引覆盖，再将这 100 条索引覆盖的数据去一级索引中进行查询，

SELECT id,age,name from tb_user p join ( select id from tb_user where age > 1 LIMIT 50000,100) temp on p.id = temp.id;
```

但是，深分页仍然没有解决 *LIMIT 50000,100*，这个时候，我们可以将id 作为页码进行定位，可以直接通过 B+ 树的索引检索，定位到 50000 条，再拿到 100 条数据，
- 前提是 id 自增，不能用uuid 等
- 中间不能删掉数据

```
SELECT id,age,name from tb_user p join ( select id from tb_user where id > 50000 and age > 1 LIMIT 100) temp on p.id = temp.id;
```
### 3 游标分页
```
SELECT id,age,name from tb_user p join ( select id from tb_user where id > 上一页maxID and age > 1 LIMIT 100) temp on p.id = temp.id;
```
上一页maxID，就可以定位到第二页起始值
- 分页功能需要禁止掉跨页，只能一页一页的翻。
- id 自增，不能用uuid 等
### 4 分库分表
按照数据范围分库表发，ES 大数据等
### 5 修改业务
业务允许的话，限制深分页，最多 100 页

# 动态查询条件，如何进行优化

![[Pasted image 20250714154529.png]]
请假查询
```
CREATE INDEX idx_column1_column2 ON table_name (
column1,
column2,
column3,
...,
)
```

不建议这么建索引，既浪费资源，又没有将索引优化发挥到最大。
分析查询条件，哪些字段用的最多，用最常用的字段建立联合索引。顺序也是有有讲究的，要遵循最左前缀原则
## 最左前缀原则
### 工作原理：
1. 从左到右匹配：查询必须从索引定义的第一个列开始使用
2. 连续使用：不能跳过中间的列（但可以不是用后面的列）
3. 范围查询后的列失效：如果某一列使用了范围查询（>，<，LIKE 等），其后的列无法使用索引

当建立了联合索引之后，mysql 底层会生成一个 B+ 树，将联合索引的数据存在叶子节点，会根据最左边的字段排序，再第一个字段的基础上给第二个字段排序，以此类推，如果再查询的时候，条件没有最左边的字段，就没有办法进行有序的数据检索，导致索引失效。
根据查询字段常用性，来设置联合索引的字段顺序。最常用的放左边。在业务允许的条件下，保证最左边的一直有值.


![[Pasted image 20250714155223.png]]
## 优化建议
### 1. 最常用的字段建立联合索引
### 2. 根据常用性设置索引顺序
### 3. 最左边字段设置默认值
### 4. 分库分表
根据最常用的字段进行水平拆分，如果数据量还大，根据时间，按照每个月再拆，再加上索引优化
### 5. ES，或者大数据

## SQL 执行过程
![[Pasted image 20250715104154.png]]

当客户端执行一个 sql
1. 跟远程数据建立链接，用户名密码正确的话，就会来到 MySQL 的服务层，
2. 缓存：通过服务层的缓存来进行查询（8以后已经废弃），key：sql，value：数据
3. 解析器：解析sql 语法
4. 预处理器：判断表名，字段名是否存在
5. 优化器，优化 sql ，选择最佳执行路径，生成执行计划。比如查询条件中使用联合索引，最左边的没有放到第一个，优化器会帮忙放到第一个，满足最左前缀原则。
6. 执行器：调用引擎层执行查询结果。
7. 引擎层：不同的引擎，**InnoDB**，去 BufferPool（缓冲区） 中查找，存在的话就不去磁盘中查找。没有还就回去磁盘查找，放到 BufferPool 中，并记录日志。

## 单表最多数据量需要分表
### 推荐
单表数据超过 500 万或者单表容量超过 2GB，才推荐进行分库分表。（如果预计三年后的数据达不到这个量级，不要再创建的时候就分表）
## 三层 B+ 树
### 
一颗 B+ 树是有一个一个磁盘页说组成的，每一页的大小，可以通过sql语句进行查询，默认 16KB
`show GLOBAL STATUS like 'Innodb_page_size'`
![[Pasted image 20250715110206.png]]
### InnoDB 数据页结构
每一页的结构有页头，页尾等一些结构，真正存储数据，只有中间的部分（Infimum + supremum，user records）
有效数据需要减掉页头页尾，大概200字节， 16184字节
（ 16184字节/一行的字节数=一页可以存储的数据 ）* 所有磁盘页的数据 = 一个三层 B+ 树容纳的总数据量

![[Pasted image 20250715110248.png]]

假设有一个表
```
create table t1 (
	a int(11) primay key,
	b int(11),
	c int(11),
	d int(11),
	e varchar(20) character set utf8mb4
) engine = InnoDB ;
```

一个 int 占 4个字节，一个 varchar占 4*20+2 =82 （字符类型 utf8mb4） ≈ 100 字节。-- +2 是长度前缀，占 1~2 字节
16184 / 100 ≈ 161 条记录 = 叶子节点当中一个节点的记录。
每个页指针占用 6 个字节，每个键 int 占用 4 个字节，共 10 字节。

第一层：一页里面有 16184 / 10 ≈ 1618 个指针。
第二层： 1618 * 1618 ≈ 2,617,924 个指针
第三层： 2,617,924 * 161 ≈ 421,485,764 ，4 亿多条 100 字节的数据。
图如下
![[Pasted image 20250715112221.png]]
## B 树和 B+ 树的区别
### 共同点：
1. 小的在左，大的在右
### 不同点：
B 树 
	每个节点都存储了索引和数据 -- 树更高
	无叶子节点链表结构，范围查找慢
B+ 
	所有数据都在叶子节点，非叶子节点只存储索引 -- 树更矮
	叶子节点通过双向链表连接，范围查找性能更好

## MySQL 引擎层是如何工作的
MySQL 两大内存 BufferPoll，RedoLogBuffer 和三大日志 binlog，redolog和undolog

![[Pasted image 20250715144127.png]]
关闭自动提交，执行一条修改语句
```
set autocommit = 0 ;
update employees set name = 'a' where id = 1;
```
### InnoDB

### BufferPoll
客户端连接服务层，由服务层的执行器调用 InnoDB 引擎，首先去 BufferPool 缓存当中，看 Id = 1 的数据有没有再缓存中，有的话直接更新 BufferPool 中的缓存，没有的话，去磁盘 idb 文件中加载 id = 1 的数据，根据索引找到 id = 1 的数据页，找到之后会把整页缓存到 BufferPool 中，然后对 id = 1 的数据进行修改，然后把修改之前的数据放到 Undolog 中进行备份，为了后续的数据回滚以及事务隔离的一些功能，比如修改失败了，就可以将备份的数据还原。修改之后，BufferPool 中的数据是脏页，因为 BufferPool 中的数据和磁盘中的数据不一致，这个时候需要将脏页中的数据进行提交`commit`，把数据同步到磁盘中，BufferPool 中恢复到正常页。
### Redolog，RedologBuffer
利用 Redolog 进行崩溃恢复。
当数据进入 BufferPool 之后，修改数据后，进行 commit 的时候，MySQL 挂掉了，是不是这个数据就丢失了呢，因为 BufferPool 的数据还没有同步到磁盘页中。
MySQL 中，但我们把数据从磁盘中找到，同步到 BufferPool 中，然后进行修改，这个时候，数据也会同步到 Redolog 中，当 MySQL 崩溃了，就会把 Redolog 中的数据，进行恢复到 BufferPool ，保证数据的不丢失。 
问：为什么不直接写入磁盘 idb 中呢
答：性能快。从一定概率上保证数据丢失的风险降低。Redolog 写是顺序写，开批一块磁盘空间，所有的数据按照顺序，依次写入，满了之后回到第一个继续写。
进一步提高写入 Redolog 的效率，引入了 RedologBuffer，修改了数据之后，会把新数据放到 RedologBuffer 中，然后根据刷盘的策略`innodb_flush_log_at_trx_comit`，决定怎么把 RedologBuffer 中的数据同步到 Redolog 中。
三个策略
	0：每隔 1 秒吸入，延迟写，延迟刷
	1：提交时写入，实时写，实时刷，默认参数
	2：写入系统缓存，延迟写，延迟刷。操作系统级别的 pageCatche 中，由操作系统调度 fsync 函数，持久化到 Redolog 磁盘
### binlog

![[Pasted image 20250715152645.png]]
由 MySQL server 层提供的一种日志记录方式，
	0：操作系统决定，实时写，延迟刷，5.7.7 以前版本的默认值
	1：提交时写入，实时写，实时刷，5.7.7 以后版本的默认值
	n：n 次事务提交，实时写，延迟刷

Redolog 和 binlog 区别
Redolog：
	InnoDB 存储引擎
	恢复事务数据持久性，自动恢复
	针对BufferPool
binlog
	整个MySQL
	恢复磁盘数据
	针对磁盘
	记录所有日志
	主从同步，数据误删恢复
## 聚簇索引和非聚簇索引（聚集）
![[Pasted image 20250715154143.png]]

这两种索引不是索引类型，而是存储方式。
### 聚集索引（主键索引）
比如说我们创建了一个主键索引，在 innodb 引擎下，默认的数据结构是一个 B+ 树，在叶子节点中，除了存储索引以外，还存储了这一整行数据，所以她的索引和数据是聚集在一起的，所以叫聚集索引。
### 非聚集索引（非主键索引、二级索引）
所有的非主键索引，统称为二级索引，他们都是非聚集索引，叶子节点除了存储索引以外，还存储这个索引对应的主键，如果说你当前查询需要查到 Alice 这个名字索引以外的其他数据，这个时候就涉及到回表了，需要回到主键索引这棵树上，根据 18 主键，查到其他的数据（77），当我们使用非聚集索引，查询索引以外的数据，需要回表，所以性能会差

![[Pasted image 20250715154631.png]]

## count(1)，count(\*)，count(name) 
功能：
	count(1)
	count(\*)
	count(name)  排除null
性能：
	count(1) 没有区别
	count(\*) -> 底层优化成count(0)  没有区别
	count(name) 
## 前模糊索引优化
```
name like '%aaa'
```
	反向索引
	限制范围
	索引覆盖 select * -> selcet name
	ES

## 分库分表后 Id 冲突
	步长，起始值，步长，缺点，无法扩容
		set @@auto increment offset = 1; set @@auto increment increment = 2;  1,3,5,...
		set @@auto increment offset = 2; set @@auto increment increment = 2;  2,4,6,...
	号段表，单点故障，号段表挂了
	UUID， 字符串插入，查询性能低
	redis 生成 id，缺点：redis 挂了，会丢失。办法，进行数据持久化：1.RDB，每隔一段时间，生成数据快照，不可取，2.AOF，恢复慢。
	雪花ID
雪花ID:
由 64 个二进制 bit 位组成的，转换成十进制后，就是一串有序数值，不会影响查询和插入性能，
保证唯一性：
由三部分组成：
1. 41bit 时间戳，毫秒级别
2. 10bit 工作进程位，最多存储 1024 个节点
3. 12bit 序列号位
缺点，加入服务时间由于不可抗力，倒退了几秒或几个小时，这个就有可能重复。时间回拨。