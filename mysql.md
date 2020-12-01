# **DBMS-MySql**


## Sql

### DDL(Data Definition Language)

- CREATE、 ALTER、 DROP

### DML(Data Manipulation Language)

- INSERT、UPDATE、DELETE

### DQL(Data Query Language)

- SELECT-返回结果集是一张虚拟表

### DCL(Data Control Language)

- 数据控制语言，用来定义访问权限和安全级别

### 解析过程

- **语句：SELECT `name`,COUNT(`name`) AS num FROM student WHERE grade < 60 GROUP BY `name` HAVING num >= 2 ORDER BY num DESC,`name` ASC LIMIT 0,2;**

- 1.**FROM** student

	- 把数据库的表文件加载到内存中，生成虚拟表1

- 2.**WHERE** grade < 60

	- 根据WHERE条件对虚拟表1进行筛选，生成虚拟表2

	- **补充：**

		- **字段别名在此不生效，因为SELECT在WHERE之后执行**

- 3.**GROUP BY** ‘name’

	- 根据GOURP BY的字段生成多张虚拟表3s

- 4.**SELECT** `name`, COUNT(`name`) AS num

	- 4.1.有GROUP BY

		- 根据后面的字段名称对内存中的一张临时表整列读取

	- 4.2.无GROUP BY

		- 对内存中的若干临时表分别执行SELECT，而且只取各临时表中的第一条记录，然后再形成新的临时表

- 5.**HAVING** num >= 2

	- HAVING是对SELECT语句执行之后的临时表中的数据过滤，字段别名在此处可以生效

- 6.**ORDER BY** num **DESC**

- 7.**LIMIT** 0,2

	- 

### [EXPLAIN优化](https://www.toutiao.com/i6776461352522220036/)

### 慢查询：返回时间大于10s的查询

### on和where区别

1.on在生成虚拟表是使用，where在生成虚拟表之后过滤
2.在使用left join和right join的时候，on会出现左右为null的情况，但where会过滤这类记录

## 主从模式

1.三个线程
	A）master（binlog dump thread）
	B）slave（I/O thread，SQL thread）
2.binlog thread在master库数据发生更新时，将更新binlog日志，并且通知slave主库有数据更新  
3.I/O thread请求master，拿到binlog的名称，数据更新的位置，binlog文件位置的副本。随后将binlog保存在relay log种  
4.SQl thread检测大relay log有更新，就将更新内容同步到slave数据库中  
5.四种同步策略：  
	A）同步策略：master等待所有slave回应后才提交  
	B）半同步策略：master至少等待一个slave回应  
	C）异步策略：master不等待slave就可以提交  
	D）延迟策略：slave落后于master指定时间  
6.问题：  
	A）高性能方面：主从复制通过水平扩展，解决单点故障问题，主库负责读写，从库负责读  
	B）可靠性方面：主从对外提供服务时，如果主库挂了，会通过主从切换，选择其中一台slave切换成master  
	C）写性能瓶颈解决方法：分库分表  
	D）延迟问题：根据业务选择合适的同步策略

### 主服务器

### 从服务器

## 索引

1.按照索引类型来分：全文、hash、B-Tree（mysql使用B+tree(B-Tree+索引聚集）代替b-Tree)、R-tree
2.按照应用分类：普通索引、前缀索引、主键索引、唯一索引、联合索引、伪hash索引
3.在InnoDB中，主键索引是聚集索引，其他二级索引（非聚集索引。区别是聚集索引索引键和数据存储在一起，非聚集索引存放的是主键和索引列）的叶子结点包含引用行的主键列
4.回表是根据二级索引的主键，通过主键索引找到数据行，两次查找。索引覆盖是索引列是所需要的插叙的数据
5.在MyIsam中，聚集索引和非聚集索引都是叶子结点存储数据的指针

### 应用分类

- 普通索引

	- CREATE INDEX index_name ON table_name(col_name)

	- ALTER TABLE table_name ADD INDEX index_name(col_name)

- 唯一索引

	- CREATE UNIQUE INDEX index_name ON table_name(col_name)

	- 主键索引&唯一索引区别

		1. 主键一定是唯一性索引，其索引名为primary，唯一性索引并不一定是主键
		2. 一个表中可以有多个唯一性索引，但只能有一个主键
		3. 主键列不允许空值，唯一性索引列允许空值

- 联合索引

	CREATE INDEX index_name ON table_name(col_name_1,col_name_2)  
	
	CREATE UNIQUE INDEX index_name ON table_name(col_name_1,col_name_2)

	- 最左前缀原则：从左往右匹配列可以走索引，不能跳列

- 前缀索引

	KEY idx_name (name(10), birthday, phone_number)  
	
	只保留记录的前10个字符的编码，优化存储结构

- 前缀索引

	- ALTER TABLE table_name  
	  ADD KEY(column_name(length))

	- 截取column length建立索引

- 伪哈希索引

	在B+tree的基础上创建了伪哈希索引，查找的时候还是使用B+tree，但使用的是哈希值，而不是键本身
	
	就是将列计算出hash值，表中维护这个hash列并建立索引，查询的时候使用该列索引
	查询伪sql：SELECT id FROM table WHERE url_crc=CRC32(“[http://www.mysql.com](http://www.mysql.com)”) AND url=“[http://www.mysql.com](http://www.mysql.com)”;

### 索引类型

- FULLTEXT

	只有MyISAM引擎支持。其可以在CREATE TABLE ，ALTER TABLE ，CREATE INDEX 使用，不过目前只有 CHAR、VARCHAR ，TEXT 列上可以创建全文索引

- HASH

	对于每一行数据，存储引擎都会对所有的索引列计算一个哈希码，哈希码是个较小值。hash索引将所有的哈希码存储在索引中，同时在hash表中保存指向每个数据的指针(Memory存储引擎显示支持hash索引)

- B-TREE

	所有节点都存储data，范围查找没有B+tree好，

	- +聚集索引=B+TREE

		1.所有的值都是按顺序存储
		2.每一个叶子页到根的距离相同
		3.每一个叶子节点都包含指向下一个叶子节点的指针
		4.叶子节点才存储数据
		5.InnoDB的聚集索引默认是主键索引
		6.一个表只能有一个聚集索引

- R-TREE

### 索引操作

- 删除

	- 直接删除

		- DROP INDEX index_name ON table_name

	- 修改表结构

		- ALTER TABLE table_name DROP INDEX index_name

- 查看索引

	- show index from  table_name

### 磁盘拓展

- 磁盘读取数据一次读一页(一般4K)，B-TREE为了利用这特点，发挥预读，设置每个节点大小为一页，这样一页可以存储很多数据和关键字，一次I/O就能将这些数据全部读取出来

	- 读取非范围数据，B-TREE优势大

- B+TREE没有利用磁盘预读，他将数据存放在叶子节点，并且在叶子节点之间建立索引，一方面为了I/O读取更多数据，另一方面方便数据遍历

	- 读取范围数据，B+TREE先定位数据所在叶子节点位置，接下来依靠叶子节点之间的索引读取数据，非常快

## 常见问题

### You are using safe update

SET SQL_SAFE_UPDATES = 0;

### Invalid default value for ‘timestamp’ 5.5升级5.7

set GLOBAL sql_mode=‘’;

## 数据库优化

1、  减少数据访问（减少磁盘访问）

2、  返回更少数据（减少网络传输或磁盘访问）

3、  减少交互次数（减少网络传输）

4、  减少服务器CPU开销（减少CPU及内存开销）

5、  利用更多资源（增加资源）

### sql优化

2.避免以下全表扫描  
	a）对字段进行null值的判断，或者使用!=, >, <操作符会限制使用索引  
	b）慎用in和not in关键词  
	c）对字段进行函数或者表达式操作，可以将操作放到=号后面  
	d）左模糊查询“ like %...“是没法使用索引的  
	e）or多个查询条件，如果某个查询条件没有索引，也会导致全表扫描  
	f）用union all/union 代替or  
3.不要在索引列使用函数，如果想用就在等号右边用  
4.尽量不要select *，需要什么字段拿什么字段  
5.如果可以，select 加个limit 1  
6.使用explain分析sql  
7.尝试使用覆盖索引，避免回表  
8.子查询会产生临时表，尽量用join代替子查询

- in和exist

	[1.in](http://1.in)先执行子查询，并且不对null处理，如果外表大，内表小，使用in
	2.exist先执行外表，如果外表小，内表大，使用exist
	[3.in](http://3.in)是把外表和内表作hash连接，exists是对外表作loop循环，每次loop循环再对内表进行查询

- not in 和not exists

	如果查询语句使用了not in那么内外表都进行全表扫描，没有用到索引；而not extsts的子查询依然能用到表上的索引。所以无论那个表大，用not exists都比not in要快

- change buffer

	1.当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB会将这些更新操作缓存在change buffer中，这样就不需要从磁盘中读入这个数据页了
	
	2.由于唯一索引必须检查唯一性，检查唯一性要加载数据到内存，所以唯一索引不适合
	
	3.适用于写多读少的业务，页面在写完以后马上被访问到的概率比较小，此时change buffer的使用效果最好，这种业务模型常见的就是账单类、日志类的系统

### 锁优化

- 行锁优化

	mysql> show status like 'innodb_row_lock%';  
	+-------------------------------+-------+  
	| Variable_name                 | Value |  
	+-------------------------------+-------+  
	| Innodb_row_lock_current_waits | 0     |  
	| Innodb_row_lock_time          | 0     |  
	| Innodb_row_lock_time_avg      | 0     |  
	| Innodb_row_lock_time_max      | 0     |  
	| Innodb_row_lock_waits         | 0     |  
	+-------------------------------+-------+  
	复制代码  
	innodb_row_lock_current_waits: 当前正在等待锁定的数量  
	innodb_row_lock_time: 从系统启动到现在锁定总时间长度；非常重要的参数，  
	innodb_row_lock_time_avg: 每次等待所花平均时间；非常重要的参数，  
	innodb_row_lock_time_max: 从系统启动到现在等待最常的一次所花的时间；  
	innodb_row_lock_waits: 系统启动后到现在总共等待的次数；非常重要的参数。直接决定优化的方向和策略  
	
	尽可能让所有数据检索都通过索引来完成，避免无索引行或索引失效导致行锁升级为表锁。  
	尽可能避免间隙锁带来的性能下降，减少或使用合理的检索范围。  
	尽可能减少事务的粒度，比如控制事务大小，而从减少锁定资源量和时间长度，从而减少锁的竞争等，提供性能。  
	尽可能低级别事务隔离，隔离级别越高，并发的处理能力越低

- 表锁优化

	 how open tables; 1表示加锁，0表示未加锁
	mysql> show open tables where in_use > 0;  
	+----------+-------------+--------+-------------+  
	| Database | Table       | In_use | Name_locked |  
	+----------+-------------+--------+-------------+  
	| lock     | myisam_lock |      1 |           0 |  
	+----------+-------------+--------+-------------+

- 分析表锁定

	mysql> show status like 'table_locks%';  
	+----------------------------+-------+  
	| Variable_name              | Value |  
	+----------------------------+-------+  
	| Table_locks_immediate      | 104   |  
	| Table_locks_waited         | 0     |  
	+----------------------------+-------+  
	table_locks_immediate: 表示立即释放表锁数。
	table_locks_waited: 表示需要等待的表锁数。此值越高则说明存在着越严重的表级锁争用情况。

## 字段类型

### char和varchar的区别

char是固定长度的，而varchar会根据具体的长度来使用存储空间，另外varchar需要用额外的1-2个字节存储字符串长度。
1). 当字符串长度小于255时，用额外的1个字节来记录长度
2). 当字符串长度大于255时，用额外的2个字节来记录长度
比如char(255)和varchar(255)，在存储字符串"hello world"时，char会用一块255个字节的空间放那个11个字符，其余的用空格填充；而varchar就不会用255个，它先计算字符串长度为11，然后再加上一个记录字符串长度的字节，一共用12个字节存储，这样varchar在存储不确定长度的字符串时会大大减少存储空间

### tinyint

mysql数据库不提供boolean类型的数据存储，但是可以用tinyint代替，改该字段对应的javabean的那个变量定义为boolean类型即可，当存入true时，自动转换为1，false为0，取的时候也一样

### Decimal

### Numric

## 引擎

### BDB

- 页面锁，表锁

### MEMORY

- 表锁

### MyIASM

- 支持表锁，不支持事务，不支持行锁和外键，内部维护表的行数，适合读多写少，主键和二级索引都是非聚簇索引

### InnoDB

- 表锁，行锁(通过索引加载，将锁加在索引响应行)，事务，主键聚簇索引

## 约束

### 1.主键约束：数据唯一，不能为null
  2.外键约束：foreign key
  3.唯一约束：unique
  4.非空约束：not null
  5.检查约束：check(age > 18)

## [锁(Lock)](https://www.cnblogs.com/crazylqy/p/7611069.html)

### **死锁**

指两个或多个事务在同一资源上相互占用，并请求锁定对方占用的资源，从而导致恶性循环

- **避免死锁**

	- **InnoDB**

		1.以固定的顺序访问表和行。交叉访问更容易造成事务等待回路  
		2.降低隔离级别  
		3.为表添加合理的索引。防止没有索引出现表锁，出现的死锁的概率会突增  
		4.尽量避免大事务，占有的资源锁越多，越容易出现死锁。建议拆成小事务

	- **MyISAM**

		总是一次获得SQL语句所需要的全部锁，所以MyISAM表不会出现死锁

- **SHOW INNODB STATUS：分析死锁原因**

- **select * from information_schema.innodb_locks：查询事物占锁情况**

### 粒度

- 行锁

- 表锁

	- 在MyIsam，表锁随着SQL语句的执行自动添加
	  在InnoDB，没有使用索引的情况下，表锁也会自动添加

### 性质

- 悲观锁

	- 行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁

- 乐观锁

	update user set name = lisi version = version + 1 where id = #{id} and version = #{version}

- 排它锁(读写锁)

	- select math from test where math >60 for update

		1.当使用唯一索引时，行锁（共享）
		2.当使用非唯一索引时，间隙锁  
		3.当不使用索引时，表锁（共享）
		
		以上可能是RR级别  
		1.排他锁，同时为主键索引加锁

- 共享锁(读锁)

	- select  math from test where math>60 lock in share mode

		共享读锁，允许其它事务加共享读锁，但不能加写锁或者修改数据。加锁同时给覆盖索引加锁

- [GAP锁](https://zhuanlan.zhihu.com/p/48269420)

	间隙锁在InnoDB的唯一作用就是防止其它事务的插入操作，以此来达到防止幻读的发生，所以间隙锁不分什么共享锁与排它锁。 默认情况下，InnoDB工作在Repeatable Read隔离级别下，并且以Next-Key Lock的方式对数据行进行加锁，这样可以有效防止幻读的发生。Next-Key Lock是行锁与间隙锁的组合，当对数据进行条件，范围检索时，对其范围内也许并存在的值进行加锁！当查询的索引含有唯一属性（唯一索引，主键索引）时，Innodb存储引擎会对next-key lock进行优化，将其降为record lock,即仅锁住索引本身，而不是范围！若是普通辅助索引，则会使用传统的next-key lock进行范围锁定！

	- 开区间，请求共享或排他锁时，对于键值在条件范围内但并不存在的记录添加间隙锁。该锁作用于索引

		- 优化意见：并发插入比较多的应用，尽量使用相等条件来访问更新数据，避免使用范围条件查询

	- 插入意向锁

		可以看出插入意向锁是在插入的时候产生的,在多个事务同时写入不同数据至同一索引间隙的时候，并不需要等待其他事务完成，不会发生锁等待。假设有一个记录索引包含键值4和7，不同的事务分别插入5和6，每个事务都会产生一个加在4-7之间的插入意向锁，获取在插入行上的排它锁，但是不会被互相锁住，因为数据行并不冲突

		- 如果已存在间隙锁，插入意向锁会被阻塞

	- 产生条件

- Next-key锁

	这个锁本质是记录锁加上gap锁。在RR隔离级别下(InnoDB默认)，Innodb对于行的扫描锁定都是使用此算法，但是如果查询扫描中有唯一索引会退化成只使用记录锁。为什么呢?
	因为唯一索引能确定行数，而其他索引不能确定行数，有可能在其他事务中会再次添加这个索引的数据会造成幻读

	- 行锁+间隙锁（左开右闭）

- 意向共享锁

- 意向排他锁

- insert加锁过程

	加锁前：往间隙加入意向间隙锁  
	加锁：往对应索引行加入排它锁，不加gap锁  
	结果：事物并发插入时，只要插入的行不同就不会阻塞

	- 死锁

- 自增长锁

	自增长锁是一种特殊的表锁机制，提升并发插入性能。对于这个锁有几个特点:
	
	在sql执行完就释放锁，并不是事务执行完。
	对于Insert...select大数据量插入会影响插入性能，因为会阻塞另外一个事务执行。
	自增算法可以配置。

### 总结

1.普通select不加锁

2.以非主键为查询条件，给非聚集索引加锁，并且可能会给聚集索引加锁  

3.在普通索引跟唯一索引中，数据间隙的分析，数据行是优先根据普通索引排序，再根据唯一索引排序

- 使用唯一索引

	Insert/delete/update使用行锁，如果该行不存在使用间隙锁

- 不使用唯一索引

	1.Insert/delete/update如果使用索引，会是间隙锁  
	2.Insert/delete/update如果不使用索引，会是表锁

- 范围搜索

	范围搜索不管有没有唯一索引，都存在间隙锁

## 日志

### 逻辑日志：undo log

可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。  

当读取的某一行被其他事务锁定时，它可以从undo log中分析出该行记录以前的数据是什么，从而提供该行版本信息，让用户实现非锁定一致性读取（MVCC）

- MVCC多版本并发控制

	MVCC只在RR、RC级别下工作。在事务并发时取代锁机制，缺点是没行记录都需要额外的存储空间  
	
	1.InnoDB的MVCC为每行记录保存两个隐藏列，分别是创建时事务版本号，删除时事务版本号  
	2.每开始一个新的事务，系统版本号会递增，普通SELECT不会开启事务  
	
	SELECT：  
	InnoDB查找版本号<=当前事务的系统版本号，且删除版本号要么没定义，要么>当前版本号。目的是确保事务读取的行在事务开启前未被删除  
	
	INSERT：  
	为新插入的每一行保存当前系统版本号  
	
	DELETE：  
	为删除的每一行保存当前系统版本号为删除标示  
	
	UPDATE：  
	为插入一行新纪录，保存当前系统版本号为行版本号，同时保存当前系统版本号到到原来的行作为行删除标识

	- 一致性读

		背景：  
		1.快照读，普通的select语句
		
		2.事务中第一次调用SELECT语句的时候才会生成快照，在此之前事务中执行的update、insert、delete操作都不会生成快照。  
		
		
		原理：  
		1.RR：同一个事务中的所有一致性读都会读取此事务中第一次一致性读所生成的快照  
		
		2.RC：事务中每次一致性读都会设置并读取当前最新的快照

- 提供回滚

### 物理日志：redo log

第一步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝  
第二步：生成一条重做日志并写入redo log，同时写入buffer pool  
第三步：当事务commit时，将redo log buffer中的内容追加到redo log file  
第四步：定期将内存中修改的数据刷新到磁盘中  

注意：无论mysql是否正常退出，重启时都会进行恢复操作

### 二进制日志：bin log

按照事务顺序，通过BINLOG录入执行成功的INSERT、UPDATE、DELETE等更新数据的SQL语句，并由此实现数据库的恢复和主从复制

三种模式：  
（A）statement：基于sql语句的模式，某些语句含有一些函数，在复制过程可能导致数据不一致  
（B）row：基于行的模式，记录行的变化，但占用磁盘空间大。  
（C）mixed：混合模式，根据语句选择用（A）还是（B）

### Error Log

- 记录启动、运行或停止 mysqld 时出现的问题，默认开启

### General Query Log

- 通用查询日志，记录所有语句和指令，开启数据库会有 5% 左右性能损失

### Slow Query Log

- 记录所有执行时间超过 long_query_time 秒的查询或不使用索引的查询，默认关闭

### 慢查询日志

用来记录在MySQL中响应时间超过阀值的语句，具体指运行时间超过long_query_time值的SQL，则会被记录到慢查询日志中

## 事务

事务是访问和更新数据库的程序执行单元

### [ACID](https://www.cnblogs.com/kismetv/p/10331633.html)

- Atomicity(原子性)

	一个事务中所有操作要么全部成功，要么全部失败

- Consistency(一致性)

	一致性是指事务执行结束后，数据库的完整性约束没有被破坏，事务执行的前后都是合法的数据状态。数据库的完整性约束包括但不限于：实体完整性（如行的主键存在且唯一）、列完整性（如字段的类型、大小、长度要符合要求）、外键约束、用户自定义完整性（如转账前后，两个账户余额的和应该不变）

- Isolation(隔离性)

	(一个事务)写操作对(另一个事务)写操作的影响：锁机制保证隔离性  
	(一个事务)写操作对(另一个事务)读操作的影响：MVCC保证隔离性

	- 隔离级别

		- 读未提交

			SELECT语句以非锁定方式被执行，所以有可能读到脏数据，隔离级别最低。（读不锁）

		- 读已提交

			1.事务对当前被读取的数据加行级共享锁，一旦读完该行，立即释放该行级共享锁  
			
			2.事务在更新某数据的瞬间（就是发生更新的瞬间），必须先对其加行级排他锁，直到事务结束才释放

		- 可重复读

			1.事务在读取某数据的瞬间，必须先对其加行级共享锁，直到事务结束才释放  
			
			2.事务在更新某数据的瞬间，必须先对其加 行级排他锁，直到事务结束才释放

			- innoDB:防止幻读

				1.针对常规select，基于MVCC机制解决
				2.针对带锁select，当读取数据记录时，除了锁住记录本身（Record Lock），同时将符合条件的间隙锁起来（Gap Lock），预防第一次读和第二次读的期间，其他事务往间隙插入了新数据。这就是间隙锁的意义！

		- 串行化

			事务在读取数据时，必须先对其加表级共享锁 ，直到事务结束才释放  
			
			事务在更新数据时，必须先对其加表级排他锁 ，直到事务结束才释放

	- 并发事务引起的问题

		- 脏读

			读到其他事务未提交的数据

		- 不可重复读

			事务先后读取同一条记录，但两次读取的数据不同

		- 幻读

			事务按相同的查询条件重新读取以前检索过的数据，却发现其他事务插入了满足其查询条件的新数据

- [Durability(持久性)](https://www.toutiao.com/i6774657726661263884/)

	事务完成时候，事务对数据库的改动要持久的数据库中

### 总结

0.脏读源于读不加锁

1.RC和RR的区别在于，读时RC读完解锁，RR提交解锁

2.为了提供事务并发性能，mysql使用mvcc机制

(一个事务)写操作对(另一个事务)写操作的影响：锁机制保证隔离性
(一个事务)写操作对(另一个事务)读操作的影响：MVCC保证隔离性

RR级别第一次有ReadView，RU级别每次读取最新ReadView  

3.undo log保证原子性；redo log保证持久性；锁+MVCC机制保证隔离性；以上保证一致性

## 语法

### 表与表之间的关系

- 一对一

	- 一方建主表(id为主键字段), 一方建外键字段(加unique)

		CREATE TABLE man(  
		  id VARCHAR(32) PRIMARY KEY,  
		  NAME VARCHAR(30)  
		);  
		
		CREATE TABLE woman(  
		  id VARCHAR(32) PRIMARY KEY,  
		  NAME VARCHAR(30),  
		  husband VARCHAR(32) UNIQUE,  
		  CONSTRAINT wm_fk FOREIGN KEY(husband) REFERENCES man(id)  
		);  
		添加外键的另一种方法 
		ALTER TABLE woman ADD CONSTRAINT wm_fk FOREIGN KEY(husband) REFERENCES man(id);  
		注：husband这里要加unique约束，不加则是一对多关系
		参考主表的主键id，加unique)

- 一对多

	- 一方建主表(id为主键字段), 多方建外键字段(加unique)

		CREATE TABLE person2(  
		  id VARCHAR(32) PRIMARY KEY,  
		  NAME VARCHAR(30),  
		  sex CHAR(1)  
		);  
		
		DROP TABLE car2;  
		CREATE TABLE car(  
		  id VARCHAR(32) PRIMARY KEY,  
		  NAME VARCHAR(30),  
		  price NUMERIC(10,2),  
		  pid VARCHAR(32),  
		  CONSTRAINT car_fk FOREIGN KEY(pid) REFERENCES person2(id)  
		);

- 多对多(3个表= 2个实体表 + 1个关系表)

	- 

		//学生表
		CREATE TABLE stud(  
		  id VARCHAR(32) PRIMARY KEY,  
		  NAME VARCHAR(32)  
		);  
		//课程表
		CREATE TABLE ject(  
		  id VARCHAR(32) PRIMARY KEY,  
		  NAME VARCHAR(32)  
		);  
		另外补建一个关系表  
		CREATE TABLE sj(  
		  studid VARCHAR(32) NOT NULL,  
		  jectid VARCHAR(32)  
		);  
		//注意，要先建联合主键，再添加外键。顺序不能反了。
		ALTER TABLE sj ADD CONSTRAINT sj_pk PRIMARY KEY(studid,jectid);  
		ALTER TABLE sj ADD CONSTRAINT sj_fk1 FOREIGN KEY(studid) REFERENCES stud(id);  
		ALTER TABLE sj ADD CONSTRAINT sj_fk2 FOREIGN KEY(jectid) REFERENCES ject(id);

### 连表查询

- 连接查询

	- SELECT * FROM t1, t2

- 合并结果集-两个表列数，类型一致

	- UNION：去除重复记录

	- UNION ALL：不去除重复记录

- **子查询**

	1.子查询从右边开始执行

### 内连接

- SELECT * FROM emp,dept WHERE emp.deptno=dept.deptno;

- SELECT * FROM emp e   
  **[INNER]** JOIN dept d ON e.deptno=d.deptno;

### 外连接

- 左外连接

	- SELECT * FROM emp e   
	  **LEFT [OUTER]** JOIN dept d 
	  ON e.deptno=d.deptno;

		左连接是先查询出左表（即以左表为主），然后查询右表，右表中满足条件的显示出来，不满足条件的显示NULL

- 右外连接

	- SELECT * FROM emp e   
	  **RIGHT [OUTER]** JOIN dept d 
	  ON e.deptno=d.deptno

		右连接就是先把右表中所有记录都查询出来，然后左表满足条件的显示，不满足显示NULL

### 自然连接

- 两张连接的表中名称和类型完全一致的列作为条件，例如emp和dept表都存在deptno列，并且类型一致，所以会被自然连接找到

	select name1, course_id  
	from instructor, teaches  
	where instructor.ID = teaches.ID  
	等价于  
	select name1, course_id  
	from instructor natural join teaches

### 注意

- WHERE语句接在GROUP BY语句前面，HAVING语句得接在GROUP BY语句后面

