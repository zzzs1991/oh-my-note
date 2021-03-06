### day 16

```
day 16
查询一行语句也这么慢的原因有
1.等待锁
- 等待MDL锁 waiting for table metadata lock
  lock table
- 等待表锁 waiting for table flush
  flush table T with read lock
  flush tables with read lock 
- 等待行锁 

2.一致性视图和redo log导致的查询缓慢

查询
show processlist
select blocking_pid from sys.schema_table_lock_waits

```
### day 17
```
day 17
幻读
- 可重复读级别下 当前读 才会产生幻读
- 专指两次查询中新出现的行

幻读的问题
- 打破了锁的语义 该锁的行没锁上
- 导致数据一致性问题

加锁级别
- 给指定行加锁
- 给所有扫描的行加锁
- 给间隙加上锁 能解决幻读问题

两个间隙锁不互斥 会导致死锁 影响并发度

行锁和间隙锁 组成了 左闭右开的 next key lock
```
### day 18
```
day 18
加锁范围规则
两个原则，两个优化，一个bug
- 加锁的单位是next key lock
- 查找过程中访问到的对象才会加锁
   加在索引上
- 索引上的等值查询，给唯一索引加锁，nextkey退化为行锁
- 索引上的等值查询，向右遍历到不满足等值条件的，退化为间隙锁
- 唯一索引上的范围查找会访问到第一个不满足条件的值为止
```
### day 19
```
day 19
count(*)的实现方式
- myisam 将总行数记录在磁盘里
- innodb把数据逐行读出来，计数

为什么innodb要逐行读出来 因为mvvc

优化 选最小的树进行读取

show table status 基于采样，不准确

自己计数
- redis
  无法保证原子性
- 数据库
  可以
  
count(字段)<count(主键id)<count(1)≈count(*)
```
### day 22
```
day22
普通索引和唯一索引的区别
在业务满足唯一性要求的时候推荐使用普通索引来优化更新的性能

在写多读少的场景中，例如日志表，账单表
在使用普通索引时，可以利用change buffer来减少磁盘随机io的次数，提高性能。
而唯一索引，需要进行唯一性校验，必须将数据读到内存中，所以无法利用change buffer来提高性能。

change buffer 和 redo log
changebuffer是利用内存来减少读磁盘的次数，而redo log是减少写磁盘的次数。
```
### day 23
```
day 23
为什么mysql会偶尔选错索引
优化器选择索引的目的是为了找到一个最优的执行方案，用最小的代价去执行。
影响到执行代价的有以下几个方面
- 扫描行数
- 是否临时表
- 是否排序

优化器根据区分度来估计扫描行数，区分度在mysql上称为基数
基数是通过采样统计得来的，innodb默认选取N个数据页，计算平均值，乘以总页数得到基数。在更新行数超过1/M时，会更新采样数据。

索引统计数据可以通过 innodb_stats_persistent来选择
- on 持久化存储 N=20 M=10
- off 保留在内存中 N=8 M=16

优化器根据索引扫描行数和回表代价综合考量，会出现选错索引的现象。

统计信息不对，可以修正
analyze table t;

优化器误判索引可以通过以下方式解决
- 用force index
- 修改语句诱导优化器
- 增加或删除索引
```

```sql
CREATE TABLE `t` (
    `id` int(11) NOT null,
    `a` int(11) DEFAULT null,
    `b` int(11) DEFAULT null,
    PRIMARY KEY (`id`),
    KEY `a` (`a`),
    KEY `b` (`b`)
) ENGINE=INNODB;
```

```sql
DELIMITER ;;
CREATE PROCEDURE idata()
BEGIN
	DECLARE i int;
    set i=1;
    WHILE(i<=100000)DO
    	INSERT INTO t VALUES(i,i,i);
        set i=i+1;
    END WHILE;
END;;
DELIMITER ;
CALL idata();
```

```sql
select * from t where a between 10000 and 20000;
```

```sql
start transaction with consistent snapshot;



commit;
```

```sql
delete from t;
call idata();

set long_query_time=0;
select * from t where a between 10000 and 20000;
select * from t force index(a) where a between 10000 and 20000;
```

```sql
select * from t where a between 1 and 1000 and b between 50000 and 100000 order by b limit 1;

select * from t force index(a) where a between 1 and 1000 and b between 50000 and 100000 order by b limit 1;
```
### day 24
```
day 24
怎么给字符串类型添加索引
1.直接创建完整索引，这样可能比较占用空间，可以使用覆盖索引
2.创建前缀索引，节省空间，但是会增加扫描次数，并且不能使用覆盖索引
3.倒序存储，再创建前缀索引，用于字符串本身区分度不够的情况，不支持范围查询
4.创建hash字段索引，查询性能稳定，有额外的存储和计算消耗，不支持范围查询


```
### day 25
```
day 25
mysql不时会变慢，基本上是由于刷脏页导致的。
引发刷脏页的4中情况：
- redo log 记录满了，需要停下所有的更新操作，来将redolog中 的一段操作对应的脏页都刷到磁盘中。
- 读取的数据太多，内存装不下了，需要淘汰掉一些内存也来存放新的数据，如果这些内存页是脏页，需要刷到内存中。
- 系统空闲时会进行刷脏页，有机会就刷
- mysql正常关闭时，也会刷脏页。

innodb使用内存的策略是尽量使用内存，空闲页面很少，淘汰最久不使用的页面，可能是干净页，也可能是脏页。
- 一个query要淘汰的脏页太多。
- redo log满了，会阻塞更新操作

mysql宿主机的io能力决定了刷脏页速度，需要正确评估io能力，将io能力iops应用到 innodb_io_capacity中，表示mysql全力刷脏页的能力。

可以用 fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 来测试磁盘随机读写能力

innodb如何控制刷脏页的速度
- 脏页比例
- redo log写盘速度

用两个参数表示 
- 目前脏页百分比M/innodb_max_dirty_pages_pct 表示脏页比例 记为F(M)
- redo log当前序号 与 checkpoint对应的序号 的差值 N 算出一个百分比 F(N)
取两者大值 为R，按照innodb_io_capacity * R%来控制刷脏页的速度

平时需多关注脏页比例
SELECT variable_value INTO @a from GLOBAL_STATUS where variable_name = 'Innodb_buffer_pool_pages_dirty';
SELECT variable_value INTO @b from GLOBAL_STATUS where variable_name = 'Innodb_buffer_pool_pages_total';
SELECT @a/@b;

innodb_flush_neighbors 为1时，刷脏页时，相邻脏页会被一同刷掉，iops大时不需要设置为1.

```

```sql
SELECT variable_value INTO @a from GLOBAL_STATUS where variable_name = 'Innodb_buffer_pool_pages_dirty';
SELECT variable_value INTO @b from GLOBAL_STATUS where variable_name = 'Innodb_buffer_pool_pages_total';
SELECT @a/@b;
```

```sh
fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest
```

```shell
 fio -filename=abc -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest
 -------------
mytest: (g=0): rw=randrw, bs=(R) 16.0KiB-16.0KiB, (W) 16.0KiB-16.0KiB, (T) 16.0KiB-16.0KiB, ioengine=psync, iodepth=1
...
fio-3.7
Starting 10 threads
mytest: Laying out IO file (1 file / 500MiB)
Jobs: 10 (f=10): [m(10)][100.0%][r=16.6MiB/s,w=16.8MiB/s][r=1064,w=1078 IOPS][eta 00m:00s]
mytest: (groupid=0, jobs=10): err= 0: pid=25598: Sat Feb 15 15:35:28 2020
   read: IOPS=1054, BW=16.5MiB/s (17.3MB/s)(165MiB/10017msec)
    clat (usec): min=256, max=51684, avg=4664.75, stdev=8417.92
     lat (usec): min=257, max=51685, avg=4665.42, stdev=8417.92
    clat percentiles (usec):
     |  1.00th=[  351],  5.00th=[  383], 10.00th=[  408], 20.00th=[  469],
     | 30.00th=[  906], 40.00th=[ 2540], 50.00th=[ 3294], 60.00th=[ 3785],
     | 70.00th=[ 4228], 80.00th=[ 4752], 90.00th=[ 5735], 95.00th=[10290],
     | 99.00th=[44827], 99.50th=[46400], 99.90th=[48497], 99.95th=[49021],
     | 99.99th=[51643]
   bw (  KiB/s): min= 1344, max= 2080, per=10.01%, avg=1688.61, stdev=139.10, samples=200
   iops        : min=   84, max=  130, avg=105.50, stdev= 8.71, samples=200
  write: IOPS=1086, BW=16.0MiB/s (17.8MB/s)(170MiB/10017msec)
    clat (usec): min=494, max=52202, avg=4667.47, stdev=8167.94
     lat (usec): min=495, max=52203, avg=4668.87, stdev=8167.97
    clat percentiles (usec):
     |  1.00th=[  502],  5.00th=[  515], 10.00th=[  537], 20.00th=[  562],
     | 30.00th=[ 1336], 40.00th=[ 2835], 50.00th=[ 3425], 60.00th=[ 3884],
     | 70.00th=[ 4359], 80.00th=[ 4883], 90.00th=[ 5735], 95.00th=[ 7963],
     | 99.00th=[44827], 99.50th=[46400], 99.90th=[48497], 99.95th=[49546],
     | 99.99th=[51119]
   bw (  KiB/s): min= 1085, max= 2528, per=10.01%, avg=1739.80, stdev=247.71, samples=200
   iops        : min=   67, max=  158, avg=108.70, stdev=15.49, samples=200
  lat (usec)   : 500=11.88%, 750=17.36%, 1000=0.63%
  lat (msec)   : 2=3.97%, 4=30.92%, 10=30.42%, 20=0.15%, 50=4.63%
  lat (msec)   : 100=0.04%
  cpu          : usr=0.14%, sys=0.65%, ctx=41083, majf=0, minf=8
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=10561,10879,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=16.5MiB/s (17.3MB/s), 16.5MiB/s-16.5MiB/s (17.3MB/s-17.3MB/s), io=165MiB (173MB), run=10017-10017msec
  WRITE: bw=16.0MiB/s (17.8MB/s), 16.0MiB/s-16.0MiB/s (17.8MB/s-17.8MB/s), io=170MiB (178MB), run=10017-10017msec

Disk stats (read/write):
  vda: ios=10556/10882, merge=0/7, ticks=9244/9557, in_queue=17153, util=87.35%

```

### day 26

```
day 26
答疑 关于日志和索引问题
1. mysql怎么知道binlog时完整的
- statement格式的binlog，最后会有commit。
- row格式的binlog，最后会有XID event
- binlog-checksum参数
2. redolog和binlog是怎么关联起来的
他们有共同的数据字段XID
3. 处于prepare阶段的redolog加上完整的binlog，重启就能恢复，为什么这么设计？
binlog已经写入，就会被从库用来同步数据，所以需要在主库上有这样的恢复策略。
4. 为什么还需要两阶段提交？
为了分布式系统
5. 可以只使用binlog吗？
binlog无法恢复内存中的场景，因为有WAL的存在，内存中场景存放在redolog中，binlog无法恢复，不能只使用binlog。
6. 只要redolog，不要binlog可以吗？
理论上可以，但是binlog是用来归档和下游一些软件进行通信的组件，无法丢弃。
7. redolog一般设置成多大？
硬盘足够大，4个文件，每个1gb
8. 正常运行的实例，数据写入后的最终落盘，是从redolog更新还是从buffer pool更新？
是从buffer pool。redolog并没有记录数据页的完整信息，没有能力去更新磁盘数据页。
9. redo log buffer是什么，是先修改内存还是先写redo log文件？
一次事务中，日志可能要多次写入，先将redo log记录在内存中，在事务commit的时候再写入redo log文件中，文件名 ib_logfile+数字
10. 业务场景问题
insert 。。。 on duplicate key 。。。 不存在就插入，存在就更新
insert ignore into 。。 表示插入时如果存在相同的记录就忽略。
```

```shell script

GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;

GRANT ALL PRIVILEGES ON *.* TO 'zzzs1991'@'%' WITH GRANT OPTION;

docker run --name mysql -v /home/data/mysql:/var/lib/mysql -v /home/conf/mysql:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=123456 -p 3306:3306 -d mysql:5.7
```

### Day29

```
day 29
mysql是怎么保证数据不丢失的
只要保证bin log 和 redo log持久化到磁盘，就能确保mysql异常重启后，恢复数据
1. bin log的持久化过程
- 事务执行过程中，先把日志写到bin log cache中
- 事务提交的时候，将binlog cache写到binlog文件中

  - 事务的binlog不能被拆开
  - 每个线程一个binlog cache，通过binlog_cache_size控制
  - binlog cache满了 会暂存到磁盘
  - 事务提交时，才将binlog cache中完整的事务写入到binlog中
  - 所有线程共用一个binlog文件
    - write 是将日志写到文件系统的page cache，并不是持久化
    - fsync，才是持久化到硬盘的操作，才占系统的IOPS
    - write和fsync的时机是由sync_binlog控制
    	- 0 每次提交事务只write 不fsync
    	- 1 每次提交事务都fsync
    	- N 每次提交事务都write，但是N个事务累计才fsync
    	sync_binlog值调大可以提升性能，但有丢失日志的风险，可设置为100-1000，风险较为可控
2. redo log的写入机制
- redo log在事务执行过程中，先存到redo log buffer中
- redo log buffer的内容不需要每次生成都持久化到硬盘中
	- 事务执行期间，mysql异常重启，redo log丢失，但是事务并没有提交，所以日志丢失了也无所谓
- 事务还没提交的时候，redo log会不会已经持久化到磁盘了呢
	- 有可能
		- redo log可能处于三个位置
			- redo log buffer
			- 已经write到page cache中
			- 已经fsync到磁盘中
		- 用innodb_flush_log_at_trx_commit控制
			- 0 每次事务提交时只将redo log保留在redo log buffer中
			- 1 每次事务提交时都将redo log持久化到磁盘
			- 2 每次事务提交时都将redo log write到page cache
		- innodb后台有一个线程，每一秒钟将redo log buffer中的日志，write到page cache中，在fsync到磁盘中，所以虽然事务执行中，redo log是记录在redo log buffer中的，但是有后台线程工作，是有可能被持久化到硬盘中的
		- 除此之外还有两个场景会让一个没有提交的事务持久化到磁盘中
			- redo log buffer占用的空间即将达到innodb_log_buffer_size的一半时，后台会主动写盘，write到page cache
			- 并行事务提交的时候，顺带将这个事务的redo log一起持久化到磁盘
		- 如果innodb_flush_log_at_trx_commit = 1 那么redo log在prepare阶段就要持久化一次，因为有个崩溃恢复的逻辑需要依赖于prepare的redo log。
	- 每秒一次的刷盘，加上崩溃恢复的逻辑，innodb认为提交事务的时候只要write，不需要fsync
	
3	mysql双一配置
		- sync_binlog
		- innodb_flush_log_at_trx_commit
		即事务提交前进行两次刷盘，一次binlog 一次 redo log
4 组提交，来减少双一配置的写盘
	- redo log 利用LSN这个单调递增的数字来保证写入顺序，可以进行组提交，即一次fsync写入多个事务
	- binlog 通过将redo log的fsync阶段拖后到binlog write之后，实现fsync可以组提交，但是效果没那么好
		可以通过两个参数来提升
			- binlog_group_commit_sync_delay
			延迟多少后进行调用
			- binlog_group_commit_sync_no_delay_count
			提交次数达到即可调用
			这两个参数是或的关系
			
5 WAL得益于两方面
	- redo log 和 binlog顺序写 提升写入性能
	- 组提交机制 减少iops消耗

6 如果mysql性能瓶颈卡在io上 可以从以下几个方面切入
	- 设置binlog_group_commit_sync_delay和binlog_group_commit_sync_no_delay_count来提升组提交性能，但会增加相应时间，加大数据丢失风险
	- sync_binlog设置为大于1的值，也会有丢失数据的风险
	- innodb_flush_log_at_trx_commit 设置为2比设置为0的风险小
		- 设置为2，掉电时才会丢数据
		- 设置为0，异常重启也会掉数据
```

### Day 30

```
day 30 
> 我查这么多数据会不会把数据库内存打爆
100G内存的机器上对200G的表进行全表扫描，会不会把主机内存用光？
不会，逻辑备份也不见OOM啊
- 全表扫描对server层的影响
  - 对表进行全表扫描然后结果保留在客户端
  	mysql -h$host -P$port -u$user -p$passwd -e "select * from db1.t" > $target.file
  	- 结果虽然存在“结果集”里，但服务端并不需要一个完整的结果集。
  		- 获取一行，放入net_buffer中，默认16K，有net_buffer_length控制
  		- 重复获取行，知道net_buffer写满，调用网络接口发出去
  		- 发送成功，则清空net_buffer，重复上述步骤
  		- 如果发送函数调用返回EAGAIN或者WSAEWOULDBLOCK，就表示本地网络栈(socket send buffer)写满了，进入等待，等网络栈重新可写，再继续发送
  	- 如果客户端接收的慢，会导致事务执行边长，处于sending to client
  		- 减少处于sending to client线程的数量，可以调大net_buffer_size大小
  		- mysql_use_result接口
  		- mysql_store_result接口
  	- sending to client 是在等待网络发送
  	- sending data 表示正在执行
- 全表扫描对innodb的影响
	- WAL提升了 读写性能，buffer pool加速查询
		- 读性能体现在内存命中率上
			show engine innodb status
	- innodb buffer pool是由innodb_buffer_pool_size控制，一般可控制在物理机的6成到8成
	- innodb内存管理使用改进版的LRU，对LRU链表划分成了两个区域 5:3 young 5 old 3
		- 访问young区的数据页，会放在young区头部
		- 访问内存中没有的数据，淘汰链表尾部的数据页，将其放在old区头部
		- 访问old区数据页
			- 如果它在内存中存在的时间大于1秒，移动到链表头部
			- 如果小于1秒，保持不变
	- 保证了在做冷数据处理时，热点数据不会完全从内存中淘汰，保证了命中率不会下降的很厉害
  		
```

### Day 31

```
day 31
为什么表数据删掉一半，表文件大小不变
- 对innodb来说，包含两部分
	- 表结构定义
	- 数据
- 删除表
  - innodb_file_per_table
    - OFF 数据放在系统共享表空间里
      - drop table 空间不会回收
    - ON 每个表放在一个单独的.ibd文件中
      - drop table 直接删除.ibd文件
- 删除行
	- 在数据页上将该行标记为可复用
		- 满足范围限制才真可用
	- 整个数据页也可以被标记为可复用
		- 随时可用
		- 两个数据页合并也会使其中一页可复用
	- 增删改都会造成空洞
- 重建表可以收缩表空间
	alter table A engine = InnoDB
		- 5.5 之前 mysql自动完成 server层创建临时文件 转存数据，交换表名，删除旧表
			但不是online的，DDL执行期间不能有数据写入，会丢失
		- 5.6 之后，优化成了online DDL
			- 可以由数据写入，由rowlog记录
			- innodb创建临时文件，对于server无感知，是inplace的
			- alter开始时获取MDL写锁，在copy数据时就退化成了读锁，退化时为了实现Online，不解锁是为了不让其他线程同时做DDL
		- 5.5 等效于 alter table t engine = innodb,ALGROITHM=copy
		- 5.6 等效于 alter table t engine = innodb,ALGROITHM=inplace
		
- optimize , analyze ,alter table的区别
	- 5.6 开始 alter table t engine = innodb 就是recreate的方式
	- analyze table t 只是对索引信息进行重新统计，加了MDL读锁
	- optimize table t 等于 recreate + analyze
```

### Day 32

```
day 32
怎样快速的复制一张表
- 如果可以控制对源表的扫描行数和加锁范围，可以使用insert ... select 语句实现
- 为了避免加读锁，更稳妥的方式是先将数据写到外部文本文件，再写回目标表
	- mysqldump
		- mysqldump -h$host -P$port -u$user --add-locks=0 --no-create-info --single-transaction --set-gtid-purged=OFF db1 t --where="a>900" --result-file=/client_tmp/t.sql
		1. -single-transaction,在导出数据的时候不要对表加读锁，而使用start transaction with consistent snapshot
		2. -add-locks=0,在输出的文件结果中，不要增加LOCK TABLES t WRITE
		3. -no-create-inof,不需要导出表结构
		4. -set-gtid-purged=off,不输出gtid相关信息
		5. -result-file制定了输出文件的路径，client表示在客户端机器上生成
		6. -skip-extended-insert 一条语句只插入一条记录
		- mysql -h$host -p$passwd -u$user db2 -e "source /client_tmp/t.sql"
			- source不是SQL语句，而是客户端命令。
			- mysql 客户端这样执行命令的
				- 打开文件，默认以分号结尾读取一条sql语句
				- 将sql语句发送到服务端执行
	- 导出csv文件
		- select * from db1.t where a>900 into outfile '/server_tmp/t.csv'
			- 文件保存在server端
			- 受secure_file_priv控制
				- empty 不限制文件位置
				- 表示路径的字符串 只能存放在指定文件夹
				- null 禁止操作
			- 不会覆盖文件，保证没有同名文件
		- load data infile '/server_tmp/t.csv' into table db2.t
			- 打开文件 /server_tmp/t.csv,以制表符为字段间分隔符，以换行符为记录之间的分隔符，进行数据读取
			- 启动事务
			- 判断每一行字段数和db2.t的是否相同
				- 相同，构成一行，调用innodb接口写表
				- 不相同，报错，事务回滚
			- 重复上述步骤，直到整个文件读取完成，提交事务
		- 如果binlog_format=statment
			- 主库执行完后，将/server_tmp/t.csv文件内容直接写到binlog中
			- 在binlog文件中写入Load data local infile '/tmp/SQL_LOAD_MB-1-0' into table `db2`.`t`
			- 把这个binlog日志传到备库
			- 备库的apply线程在执行这个事务日志时
				1. 先将binlog中的t.csv文件读出来，写入本地临时目录
				2. 在执行load data语句
			- 备库执行多了个local
				- 不加local是读取服务端文件，受服务端限制
				- 加local是读客户端文件，mysql客户端有权限即可，先把本地文件上传给服务端，再执行load data操作
		- select ... into outfile不会生成表结构文件
			- mysqldump -h$host -P$port -u$user --add-locks=0 --no-create-info --single-transaction --set-gtid-purged=OFF db1 t --where="a>900" --tab=$secure_file_priv
			- 这条命令会在指定目录下创建一个t.sql文件保存建表语句，一个t.txt文件保存csv文件
	- 物理拷贝
		- 直接将db1.t的.frm和.ibd文件拷贝到db2目录下，不可行，因为还需要在数据字典中注册
		- 5.6引入了transportable tablespace可传输表空间，通过导出+导入表空间的方式，实现物理拷贝的功能
			1. create table r like t
			2. alter table r discard tablespace,r.ibd会被删除
			3. flush table t for export,在db1目录下会生成t.cfg文件
			4. cp t.cfg r.cfg ; cp t.ibd r.ibd;
			5. unlock tables,此时t.cfg文件会被删除。flush 之后 表处于只读状态，需要解锁
			6. alter table r import tablespace
- 对比
	- 物料拷贝最快，限制也最多，必须时全量拷贝，在服务器上，必须时innodb
	- mysqldump 可以使用where，但不能使用复杂的写法
	- select ... into outfile 最灵活，但一次只能一张表，表结构也需另外单独备份
```

### Day 33

```markdown
day 33 
如何正确的显示随机消息
在一个10000个单词的表中随机选取三个
- 内存临时表
	- select word from words order by rand() limit 3
    - explain 结果显示 Extra字段为 Using temporary,Using filesort，需要临时表，并且在临时表上排序
    - 有全字段排序和rowid排序，对innodb来说，全字段排序会减少磁盘访问，优先选择
	- 语句执行的流程
		1. 创建临时表，用的是memory引擎，两个字段，一个double，记为R，一个varchar，记为W
		2. 从words表中按主键顺序取出所有word值，调用rand()生成一个[0,1)之间的随机小数，分别存入W，R字段-到此扫描行数为10000
		3. 临时表有10000行数据了，按照R排序
		4. 初始化sort_buffer。sort_buffer中有两个字段，一个double，一个整型
		5. 从内存临时表中R值和位置信息，存入sort_buffer，此时扫描20000行
		6. 在sort_buffer按R排序
		7. 排序完成后，取3个word值给客户端，20003
	- 地址信息的意思rowid，每个引擎来唯一标识数据行信息
		- 对于innodb表来说
			- 有主键，rowid就是主键
			- 没有主键，rowid由系统生成
		- memory引擎，rowid就是数组下表
	- order by rand() 用到了内存临时表，内存临时表排序的时候用到了rowid方法
- 磁盘临时表
	- 超过tmp_table_size默认16M大小，就会变为磁盘临时表
	- sort_buffer_size小于内容大小，不能使用归并排序，使用优先队列排序
- 随机排序方法
	- 按值范围随机，不是真正的随机
	- 按行数随机，真随机
	- 在业务代码里随机，mysql只负责取值
	
```

```sql
USE
    test;
DELIMITER
    ;;
CREATE PROCEDURE idata()
BEGIN
    DECLARE
        i INT ;
    SET
        i = 0 ; 
    WHILE i < 10000
    DO
    INSERT INTO words(word)
        VALUES(
            CONCAT(
                CHAR(97 +(i DIV 1000)),
                CHAR(97 +(i % 1000 DIV 1000)),
                CHAR(97 +(i % 100 DIV 10)),
                CHAR(97 +(i % 10))
            )
        ) ;
        SET
            i = i + 1 ;
        END WHILE ;
	END ;;
DELIMITER
    ;
CALL
    idata();
```

```sql
use test;
/* 默认为 128M */
set tmp_table_size=1024;
/* 默认为 8M */
set sort_buffer_size=32768;
/* 默认为 1024*/
set max_length_for_sort_data=16;
/* 默认为 */
set optimizer_trace='enabled=on';

select word from words order by rand() limit 3;

select * from `information_schema`.`OPTIMIZER_TRACE`\G
```



```json
*************************** 1. row ***************************
                            QUERY: select word from words order by rand() limit 3
                            TRACE: {
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `words`.`word` AS `word` from `words` order by rand() limit 3"
          }
        ]
      }
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "substitute_generated_columns": {
            }
          },
          {
            "table_dependencies": [
              {
                "table": "`words`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          {
            "rows_estimation": [
              {
                "table": "`words`",
                "table_scan": {
                  "rows": 9980,
                  "cost": 21
                }
              }
            ]
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ],
                "table": "`words`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 9980,
                      "access_type": "scan",
                      "resulting_rows": 9980,
                      "cost": 2017,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 9980,
                "cost_for_plan": 2017,
                "chosen": true
              }
            ]
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": null,
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`words`",
                  "attached": null
                }
              ]
            }
          },
          {
            "clause_processing": {
              "clause": "ORDER BY",
              "original_clause": "rand()",
              "items": [
                {
                  "item": "rand()"
                }
              ],
              "resulting_clause_is_simple": false,
              "resulting_clause": "rand()"
            }
          },
          {
            "refine_plan": [
              {
                "table": "`words`"
              }
            ]
          }
        ]
      }
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "creating_tmp_table": {
              "tmp_table_info": {
                "table": "intermediate_tmp_table",
                "row_length": 268,
                "key_length": 0,
                "unique_constraint": false,
                "location": "memory (heap)",
                "row_limit_estimate": 3
              }
            }
          },
          {
            "converting_tmp_table_to_ondisk": {
              "cause": "memory_table_size_exceeded",
              "tmp_table_info": {
                "table": "intermediate_tmp_table",
                "row_length": 268,
                "key_length": 0,
                "unique_constraint": false,
                "location": "disk (InnoDB)",
                "record_format": "packed"
              }
            }
          },
          {
            "filesort_information": [
              {
                "direction": "asc",
                "table": "intermediate_tmp_table",
                "field": "tmp_field_0"
              }
            ],
            "filesort_priority_queue_optimization": {
              "limit": 3,
              "rows_estimate": 1170,
              "row_size": 14,
              "memory_available": 32768,
              "chosen": true
            },
            "filesort_execution": [
            ],
            "filesort_summary": {
              "rows": 4,
              "examined_rows": 10000,
              "number_of_tmp_files": 0,
              "sort_buffer_size": 88,
              "sort_mode": "<sort_key, rowid>"
            }
          }
        ]
      }
    }
  ]
}
MISSING_BYTES_BEYOND_MAX_MEM_SIZE: 0
          INSUFFICIENT_PRIVILEGES: 0
1 row in set (0.00 sec)

```

```sql
use test;
select max(id), min(id) into @M,@N from words;
set @X = floor((@M-@N+1)*rand()+@N);
select * from words where id >= @X limit 1;
```

```sql
use test;
select count(*) into @C from words;
set @Y = floor(@C * rand());
set @sql = concat("select * from words limit ",@Y, ",1");
prepare stmt from @sql;
execute stmt;
deallocate prepare stmt;
```

```sql
use test;
select count(*) into @C from words;
set @X = floor(@C * rand());
set @Y = floor(@C * rand());
set @Z = floor(@C * rand());
select * from words limit @X, 1;
select * from words limit @Y, 1;
select * from words limit @Z, 1;
```

### Day 36

```markdown
MySQL是如何保证主备一致的
主A和备B，虽然B没有备直接访问，但还是建议设置为readonly的。
	- 运营类的查询可能会放备库做
	- 防止切换逻辑有bug，导致主备不一致
	- 可用readonly状态来判断节点的角色
	- readonly对super用户是无效的，同步更新的线程拥有超级权限

备库B和主库A之间维持了一个长连接，主库A有一个线程，专门服务于这个长连接
	1. 在备库A上通过change master命令，设置主库A的IP、端口
	、用户名、密码、从哪个位置开始请求binlog（包含文件名和偏移量）
	2. 在备库上执行start slave命令，这时备库会启动两个线程，io_thread sql_thread
	3. 主库校验完后，从本地读取binlog发送到B
	4. 备库拿到binlog后，写到本地文件，称谓relay log
	5. sql_thread读取中转日志，解析出日志里的命令，执行
	
bin log 有三种格式
	- statement
	- row
	- mixed
	
	1. statement
		记录的事命令的原文
		show warnings 展示的 某些操作可能是unsafe的，如delete 。。。limit 在主备上可能使用的索引不同
	2. row
		记录的是对行的操作
	3. mixed
		安全就用statement，不安全就用row
	
循环复制的问题
	1. 规定两个库的server id 不同
	2. binlog重放的过程中，生成server id 与原来的bin log中的server id相同
	3. 收到日志先判断server id 跟自己相同则直接丢弃


```

### Day 37

```markdown
day 37
mysql怎么保证高可用的

主备延迟
    - 主库A执行完一个事务，写入binlog，T1
    - 传给备库B，B接收完，T2
    - B执行完这个事务，T3
    - 主备延迟 = T3-T1
  show slave status命令会展示seconds_behind_master
  
主备延迟最直接的表现，备库消费中转日志(relay log)的速度，比主库生产binlog的速度要慢。

主备延迟产生的原因
	1. 备库比主库配置低，现在少见
	2. 备库压力大
		- 一主多从
		- 将binlog输出到外部系统，让其提供统计类查询能力
	3. 大事务
		主库上必须执行完事务才会写入binlog，再传给备库
			- delete大量的数据
			- 大表ddl
	4. 备库的并行复制能力
	
主备切换的策略
	- 可靠性优先 建议
	- 可用性优先

```

### Day 38
```markdown
day 38
备库为什么会延迟好几个小时
MySQL 5.6之前都是单线程执行relay log，因此在主库并发高，TPS高时，会出现严重的主备延迟
5.7之后开始多线程复制机制，sql_thread变为coordinator线程，负责读取日志和分发事务，真正执行日志的是worker线程
- 由slave_parallel_workers决定
- 8-16最好(32核虚拟机情况)


1. 可以按照轮训的方式分发事务吗
   不行，事务先后顺序可能会不同，导致主备不一致
2. 同一个事务的更新语句能发给不同的worker来执行吗
   不行，破坏了事务的隔离性

所以，coordinator在分发事务的时候，要遵循一下两点
1. 不能造成更新覆盖，更新同一行的事务必须在一个worker中
2. 同一个事务不能被拆开。

5.5版本丁奇自己实现的分发逻辑
1. 按表分发
每个worker对应一个hash table，用于表示当前这个worker正在执行的事务队列所涉及到的表
key database+table
value 修改这个表的事务数量
- 如果新事务跟所有worker都不冲突，分配一个空闲的worker
- 如果跟多于一个的worker冲突，等待
- 如果仅和一个worker冲突，分配到该worker

2. 按行分发
需要考虑唯一键
在hash table中存
- 库 表 主键 行
- 库 表 唯一键 行

消耗更多计算资源
binlog必须时row格式的
必须有主键
不能有外键

3. MySQL 5.6 并行复制策略
按库分发，简单高效，但适用场景少

4. mariaDB的并行复制策略
参考redo log组提交优化
- 能够在同一组提交的事务，一定不会修改同一行
- 在主库可以并发的，在备库也可以并发

但是还是有区别的，在主库上，一组事务commit之后，另一组事务处于running，很快就会commit，而在从库上，做完一组commit，才会取下一组，吞吐量有差别。

5. 5.7的策略
通过slave_parallel_type区分
- DATABASE 按库分发
- LOGICAL_CLOCK maria方案的优化

处于prepare状态的事务，在备库是可以并行的
处于prepare和commit的事务是可以并行的
处于running状态的事务是不可以并行的。

利用binlog_group_commit_sync_delay和binlog_group_commit_sync_no_delay_count来制造更多同时处于prepare状态的事务

6. 5.7.22的策略
基于writeset的并行复制
增加了binlog-transaction-dependency-tracking来控制是否启用
  - COMMIT_ORDER 5.7策略
  - WRITESET 按行分发
  - WRITESET_SESSION 保证顺序

writeset是主库直接写入到binlog里的，不用读取计算
```

### Day 39
```markdown
day 39
主库出问题了，从库怎么办
AA'互为主备BCD从指向A，A出问题，切换到A'
1. 基于位点的主备切换
通过change master命令
有六个参数host,port,username,password,log_file,log_pos
需要找AA'之间的同步位点，很难精确取到，只能取个范围。
- 等A'将relay log执行完
- show master status，看A'的file和positon
- 取A的故障时刻T
- 用mysqlbinlog工具分析A'的file，得到T时刻的位点

会有一些重复的事务，在从库上需要跳过。
- 主动跳过一个事务 set global sql_slave_skip_counter=n;start slave;跳过n个事务
- slave_skip_errors跳过指定错误，1062插入数据唯一键错误，1032删除数据找不到行

2. GTID
global transaction identifier即全局事务id，是在事务提交时生成的。
GTID=server_uuid:gno
而官方文档里是GTID=source_id:transaction_id
但是这容易产生误解，因为transaction_id在回滚时也会增加，而gno在提交事务时才生成，时单调递增的。
通过在启动时设置gtid_mode=on enforce_gtid_consistency=on

3. 基于GTID的主备切换
change master to
master_host=
master_port=
master_user=
master-password=
master_auto_position=1
即使用GTID来进行主备切换
```

### Day 40
```markdown
读写分离有哪些坑
一主多从就是读写分离的基本结构
读写分离主要为了分摊主库的压力
会选择在中间做proxy来做负载均衡和路由
proxy也是趋势
1. 过期读问题
存在主从延迟，客户端更新事务之后立马查询，会读到没有更新的数据。
2. 过期读的解决方案

- 强制走主库
  必须读新数据的，强制走主库，其他走从库
- sleep
  直接sleep或通过直接将更新内容返回前端，第一时间不做查询
- 判断主备有无延迟方案
  通过seconds_behind_master是否为0
  通过对比位点
  通过对比GTID
  但是不精确，会有binlog还没传到从库的时候
- 配合semi-sync
  半同步复制
  - 事务提交时，主库把binlog发给从库
  - 从库收到binlog后，返回给主库一个ack
  - 主库收到ack后，才返回给客户端事务完成
  在一主多从的时候，会出现过期读
  在持续延迟的情况，会出现过度等待
- 等主库位点
  select master_pos_wait(file,pos[,timeout])
  在从库上执行，等待主库file，pos上的事务timeout秒，返回从执行命令开始，到执行到此为止，一共多少事务M
  - 异常返回null
  - 超时 -1
  - 已经执行过 0
  1. 在trx1执行后，立刻show master status 得到主库file，pos
  2. 选定一个从库执行查询
  3. 在从库上执行select_master_pos_wait(file,position,1)
  4. 如果返回值是>=0的正整数，在这个从库上执行查询
  5. 否则到主库上查询
- 等GTID
  select wait_for_executed_gtid_set(gtid_set,1)
  - 等待，知道这个库执行的事务包含传入的GTID_set,返回0
  - 超时返回1
  1. trx1事务执行完成后，从返回中获取GITD，即gtid1
  2. 选定一个从库执行查询
  3. 在从库上执行select wait_for_executed_gtid_set(gtid1,1)
  4. 返回值是0，在此库执行查询
  5. 否则到主库上查
  将session_track_gtids设为OWN_GTID
  通过mysql_session_track_get_first从返回解析出GTID即可

```

### Day 43
```markdown
day 43
为什么这些sql逻辑相同，性能却差异巨大？
1. 条件字段函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索的功能
2. 隐式类型转换，字符串和数字做比较，是将字符串转化为数字
3. 隐式字符编码转换
对索引字段做函数操作，可能会放弃走搜索树导致查询变慢

```

### Day 44

```markdown
day 44
mysql有哪些饮鸩止渴提高性能的方法？
业务高峰期，生产环境的mysql压力太大，无法正常响应，需要短期内提升一下性能
1. 短连接风暴
数据库建立连接的开销是很大的
	- 三次握手
	- 权限判断
	- 获取这个连接的数据读写权限
短连接只要数据库处理的稍微慢些就会连接数暴涨，由max_connections控制
 1. 先处理掉占用连接但是不工作的线程
 		- kill connection
 		- 设置wait_timeout
 	通过show processlist和select * from information_schema.innodb_trx表的trx_mysql_thread_id
 2. 减少连接过程的损耗
 	跳过权限验证阶段
 	重启 并加上 -skip-grant-tables
2. 慢查询
	1. 索引没有设计好
		online ddl 或者主备切换 进行alter table
	2. SQL没有写好
		query rewrite
	3. MySQL没有选对索引
		force index
	4. 上线前通过slow_log long_query_time=0来检验每条语句
3. QPS突增
	1. 白名单中去掉
	2. 删除掉单独的用户
	3. 重写为select 1
	
```

### Day 45

```markdown
day45
如何判断一个数据库是不是出问题了
1. select 1
  select 1成功返回只能说明数据库的进程还在，无法说明库没有问题。
  并发线程数达到了innodb_thread_concurrency的值时，select 1可以返回，但查询表会堵住。
  innodb_thread_concurrency默认为0，但这样会导致cpu切换上下文浪费资源，建议设置为64-128.
	- 并发连接数
    show processlist
	- 并发查询数
    即现在innodb_thread_concurrency
  线程进入锁等待时，并发线程计数会减一，避免整个系统锁死。
2. 查表判断
  比如建立一个表health_check
  定期执行 select * from mysql.health_check;
  在磁盘空间占用100%时，更新语句和事务提交语句都会被堵住，但是查询是可以通过的。
3. 更新判断
  update mysql.health_check set t_modified=now();
  但是外部查询都有一定的随机性。
4. 内部统计
  磁盘利用率这个问题，mysql可以告诉我们，内部每一次IO请求的时间。
  在performance_schema库，在file_summary_by_event_name表中统计了每次IO的时间。
  需要打开
  `update setup_instruments set ENABLED='YES', Timed='YES' where name like '%wait/io/file/innodb/innodb_log_file%';`
  打开监控后，通过`select event_name,MAX_TIME_WAIT FROM performance_schema.file_summary_by_event_name in ('wait/io/file/innodb/innodb_log_file','wait/io/file/sql/binlog') and MAX_TIMER_WAIT>200*1000000000;`查询
  发现异常后，用`truncate table performance.shcema.file_summary_by_event_name;`清空之前的统计信息。
  
```

### Day 46 

```markdown
day46
误删库后除了跑路，还能怎么办
误删操作分类
 	- delete误删某行数据
	- drop table 或者 truncate table误删数据表
	- drop database误删数据库
	- rm 误删整个数据库实例
1. 误删行
   误删行可以使用Flashback工具恢复，它是将binlog的内容修改后拿回原库回放
   前提
   	- binlog_format=row
   	- binlog_row_image=FULL
   不建议在主库上直接回放，有关联事务会造成二次伤害
   提前预防 
   	1. sql_safe_updates设置为on，这要求delete或者update语句必须有where语句且包含索引字段。
   	2. 代码上线前必须经过SQL审计
2. 误删库、表
  全量备份+增量日志
  要求:定期全量备份+实时备份binlog
3. 延迟复制备库
  通过change master to MASTER_DELAY= N设置这个备库与主库有N秒的延迟
  预防误删表、库的方法
  1. 账号分离
  2. 指定操作规范
    - 删表之前，先改名，无影响后删除
    - 改名加固定的后缀，只允许删指定后缀的表
4. rm删除数据
  删除节点对集群来说并不可怕
```

### Day 47

```markdown
day47
为什么有kill不掉的语句
	- kill query + 线程id
	  终止这个线程中正在执行的语句
	- kill [connection] + 线程id
	  断开这个线程的连接，如果有语句执行，先停止这个语句
收到kill命令，线程做了些什么
1. 将变量killed复制为THD:KILL_QUERY
2. 通知线程

隐藏着三个意思
	- 语句执行中有多处埋点，执行到埋点时才会判断线程状态，若发现killed状态为THD:KILL_QUERY，进入语句终止逻辑
	- 若处于等待状态，须可唤醒，否则无法进入埋点
	- 终止逻辑需要执行时间
等行锁时，pthread_cond_timedwait函数，每10毫秒判定一下是否可以进入innodb执行，不行，继续sleep，并没有判断状态
kill connection
	- 将线程状态设置为KILL_CONNECTION
	- 关闭网络连接
	- KILL_CONNECTION状态在show processlist中显示为killed
	
总结
	1. 线程没有执行到判断线程状态的逻辑
	2. 终止逻辑耗时较长
		- 超大事务，回滚时间长，需要将事物期间生成的数据版本进行回收
		- 大查询回滚，生成临时文件，加上系统此时的IO压力较大，导致耗时较长
		- DDL命令执行到最后阶段被kill。删除中间过程中生成的临时文件，受IO资源影响也可能耗时较长
		
Ctl+C只能终止客户端线程，并启动另外一个连接，发送kill query命令

关于客户端的两个误解
1. 如果库里面表很多，连接就会变慢
 受自动补全功能影响，是客户端慢，可以加上-A来禁用自动补全
2. -quick -q 参数误解
 提升客户端性能，会影响服务端性能。
 mysql客户端接收服务端返回结果又两种方式
 	- 本地缓存，mysql_store_result（默认）
 	- 不缓存，读一个处理一个,mysql_use_result方法
 -quick 会达到以下三个效果
 1. 跳过自动补全
 2. 不缓存，不影响客户端本地机器性能
 3. 不会把执行命令记录到本地命令历史文件
```



### Day 50

```
day50
到底可不可以用join && join语句怎么优化
1. 如果可以使用被驱动表的索引，join语句还是有其优势的
2. 不能使用被驱动的表的索引时，只能使用Block Nested-Loop Join算法，这样的语句就尽量不要使用。
3. 在使用join的时候，尽量使用小表作为驱动表
```

```sql
CREATE TABLE `t2` (
    `id` int(11) NOT NULL,
    `a` int(11) DEFAULT NULL,
    `b` int(11) DEFAULT NULL, 
    PRIMARY KEY (`id`),
    KEY `a` (`a`)
) ENGINE=innodb;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
    declare i int;
    set i=1;
    while(i<=1000)do
        insert into t2 values(i, i, i);
        set i=i+1;
    end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100);
```
1. Index Nested-Loop Join
```sql
select * from t1 straight_join t2 on (t1.a=t2.a);

-- 直接使用join语句时，mysql优化器可能会使用t1或t2来作为驱动表，使用straight join来让
-- mysql使用固定的连接方式来执行
```
```
mysql> explain select * from t1 straight_join t2 on (t1.a=t2.a);
+----+-------------+-------+------------+------+---------------+------+---------+-----------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref       | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+-----------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | a             | NULL | NULL    | NULL      |  100 |   100.00 | Using where |
|  1 | SIMPLE      | t2    | NULL       | ref  | a             | a    | 5       | test.t1.a |    1 |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+------+---------+-----------+------+----------+-------------+
2 rows in set, 1 warning (0.00 sec)

1. 从表t1中取出一个数据行R
2. 从数据行R中拿出a字段，取表t2中查询
3. 取出t2中满足条件的行，跟R组成一行，作为结果集的一部分
4. 重复执行1-3，直到t1的末尾循环结束
分析时间复杂度 
前提: 驱动表N行 被驱动表M行 且在被驱动表上查询时可以走索引
1. t1做了全表扫描 N
2. t2可以走索引 2*logM
3. 总的时间复杂度为N*(1+2logM)
用小表做驱动表，大表还能走索引是最理想的情况。
```

2. Simple Nested-Loop Join
3. Block Nested-Loop Join
```sql
select * from t1 straight_join t2 on (t1.a=t2.b);
```
```
mysql> explain select * from t1 straight_join t2 on (t1.a=t2.b);
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                                              |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | a             | NULL | NULL    | NULL |  100 |   100.00 | NULL                                               |
|  1 | SIMPLE      | t2    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1000 |    10.00 | Using where; Using join buffer (Block Nested Loop) |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+----------------------------------------------------+
2 rows in set, 1 warning (0.00 sec)

执行流程
1. 把表t1的数据读入线程内存join_buffer中，
2. 扫描表t2，把t2中的每一行取出来，跟join_buffer中的数据做对比，满足join条件的作为结果集返回

Block Nested-Loop Join使用的内存中的Join buffer，对比操作是内存中的操作，比Simple Nested-Loop Join快
Join Buffer 由参数 join_buffer_size 控制， 默认256k
假设要分K段来放入join_buffer, N = K * jbs, K = N/jbs， K = x * N
扫描行数 N + x *N * M
内存判断次数 N * M
小表作为驱动表，join_buffer_size 越大，分段次数越少
```
能不能用join
1. 如果使用到index nested-loop join, 即用上了被驱动表的索引，是可以的
2. 如果使用block nested-loop join, 扫描行数就会过多，尤其是大表，尽量不要用
判断标准，explain结果中，Extra字段出现了Block Nested-Loop join

如果使用join, 应该使用大表还是小表作为驱动表
1. 如果是index nested-loop join，选择小表作为驱动表
2. 如果是block nested-loop join
    1. 若join buffer size足够大 ，都一样
    2. 若join buffer size不够大，应选择小表

什么叫做小表？
```sql
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id <= 50;
select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id <= 50;
```
t2的前50行是小表
```sql
select t1.b,t2.* from t1 straight_join t2 on (t1.b=t2.b) where t2.id <=100;
select t1.b,t2.* from t2 straight_join t1 on (t1.b=t2.b) where t2.id <=100;
```
t1.b和t2.*比 t1.b是小表

join语句怎么优化
```sql
create database `test1`;
use test1;
create table `t1` (
    `id` int primary key,
    `a` int,
    `b` int,
    index(`a`)
)engine=innodb;
create table `t2` like `t1`;
delimiter ;;
create procedure idata()
begin
    declare i int;
    set i=1;
    while(i<=1000)do
        insert into t1 values(i, 1001-i, i);
        set i=i+1;
    end while;

    set i=1;
    while(i<=1000000)do
        insert into t2 values(i, i, i);
        set i=i+1;
    end while;

end ;;
delimiter ;

-- 直接call idata() 很慢 每次都要提交事务 一个多小时
call idata();
-- 可以通过以下提升速度 43.594s
set autocommit=0;
call idata();
commit;
```

### Day 51

```markdown
day51 什么时候会使用内部临时表
1. union执行流程
union的语义是取两个子查询的并集，两个集合加起来，相同的行只保留一次
在这个过程中使用到了内存临时表，并且利用了内存临时表的主键唯一性约束
unionall 就没有去重的语义
2. group by的执行流程
内存临时表大小有参数tmp_table_size控制，默认为16M
3. group by的优化方法 利用索引
group by之所以用临时表，是因为数据是无序的，需要进行统计排序
利用索引
4. group by的优化方法 直接排序
利用SQL_BIG_RESULT来暗示优化器直接使用磁盘临时表

总结：
1.语句执行过程可以一边读数据，一边得到结果，就不需要临时表
2.join_buffer是无序数组，sort_buffer是有序数组，临时表是二维表结构
3.如果执行逻辑需要用到二维表的特性，会有限使用临时表。
    - union用到了唯一索引约束
    - group by需要另外一个字段来保存累计计数

指导原则：
1. group by 如果没有排序需求，可以加上order by null
2. 尽量让group by 走索引，确认方法是explain结果没有using temporary 和 using filesort
3. group by 统计数量不大，走内存临时表
4. group by 统计量大，用SQL_BIG_RESULT来让优化器直接走磁盘临时表
```

### Day 52

```
day52
insert语句的锁为什么这么多
并不是所有的insert语句都是轻量级操作，有些在执行过程中需要给其他资源加锁，
或者无法在申请到自增id以后就立马释放自增锁
1.
insert ... select 是很常见的在两个表之间拷贝数据的方法。
在可重复读隔离级别下，这个语句会给select的表里扫描到的记录和间隙加读锁
2.
insert语句和select的对象是同一个表，可能会导致循环写，需要引入用户临时表来做优化
3.
insert语句如果出现唯一键冲突，会在冲突的唯一值上加共享的next-key lock。
因此在碰到唯一键约束导致报错后，要尽快提交或回滚事务，避免加锁时间过长。
```

### Day 53

```
day53
grant之后要跟着flush privileges吗
grant语句会同时修改数据表和内存，判断权限的时候使用的是内存数据。
因此，规范的使用grant和revoke语句，是不需要随后加上flush privileges语句的

```

### Day 54

```
day54
为什么临时表可以重名
在实际应用中，临时表一般用于处理比较复杂的计算逻辑。由于临时表是每个线程自己可见的，
所以不需要考虑多个线程执行同一个处理逻辑时，临时表的重名问题。在线程退出的时候，临时表能
自动删除，省去了收尾和异常处理工作。
```

### Day 57

```
day57都说innodb好，那还要不要使用memory引擎
1. 内存表的数据组织结构
    - innodb引擎把数据放在主键索引上，其他索引上保存的是主键id. 即索引组织表
    - memory引擎采用的是把数据单独存放，索引上保存数据位置. 堆组织表
2. innodb与memory表的区别
    - innodb表的数据总是有序存放的。而内存数据表是按照写入顺序存放的
    - 数据文件有空洞时，在插入数据时，innodb为了保证有序，只能在固定的位置上写入新值
        而内存表找到空位就可以插入新值
    - 数据位置发生变化时，innodb只需要修改主键索引，而内存表需要修改所有索引
    - innodb用主键索引时只需要走一次表，而其他索引需要走两次表
        memory表所有索引都是相同的
    - innodb支持定长的数据类型，不同记录的长度可能不同。内存表不支持Blob和Text字段
        并且即使定义了varchar(N)，也是当作char(N)，内存表每条记录长度都相同
3. 内存表的优势是快
    - hash
    - 内存读写快
4. 不支持在生产环境使用memory表
    - 锁粒度问题
    - 数据持久化问题
5. 锁粒度问题
    内存表不支持行锁，支持表锁。
6. 数据持久性问题
    - 内存表在断电或重启后都会消失
    - 主备架构下，内存表断电后会在binlog中插入delete from t1；传到备库后都消失了。
7. innodb与memory对比
    - 表更新量大，innodb的并发度更高
    - 表量不大，且考虑的是读性能，innodb的数据也都会在buffer pool 中的，读数据的速度也很快
8. memory的特殊应用
   用户临时表
    - 内存临时表正好可以无视内存表的不足
        1. 临时表不会被其他线程访问，没有并发性的问题
        2. 临时表重启后也是要删除的
        3. 备库的临时表也不会影响主库的用户线程
```
```sql
create table t1(id int primary key, c int) engine=memory;
create table t2(id int primary key, c int) engine=innodb;
insert into t1 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
insert into t2 values(1,1),(2,2),(3,3),(4,4),(5,5),(6,6),(7,7),(8,8),(9,9),(0,0);
```
```sql
-- 内存表删除5再插入10，10会出现在原来5所在的位置
delete from t1 where id =5;
insert into t1 values(10,10);
select * from t1;
-- 在内存表上作范围查询，走不了主键索引，只能全表查询
select * from t1 where id<5;
mysql> explain select * from t1 where id<5;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |   10 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

-- 内存表也支持B-Tree索引
alter table t1 add index a_btree_index using btree (id);

mysql> select * from t1 where id <5;
+----+------+
| id | c    |
+----+------+
|  0 |    0 |
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
+----+------+
5 rows in set (0.00 sec)

mysql> explain select * from t1 where id <5;
+----+-------------+-------+------------+-------+-----------------------+---------------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys         | key           | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+-----------------------+---------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | range | PRIMARY,a_btree_index | a_btree_index | 4       | NULL |    6 |   100.00 | Using where |
+----+-------------+-------+------------+-------+-----------------------+---------------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)


mysql> select * from t1 force index(primary) where id < 5;
+----+------+
| id | c    |
+----+------+
|  1 |    1 |
|  2 |    2 |
|  3 |    3 |
|  4 |    4 |
|  0 |    0 |
+----+------+
5 rows in set (0.00 sec)

mysql> explain select * from t1 force index(primary) where id < 5;
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | PRIMARY       | NULL | NULL    | NULL |   10 |    33.33 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.01 sec)

```
```sql
-- session 1
update t1 set id=sleep(50) where id=1;
-- session 2
select * from t1 where id=2;
-- session 3 
show processlist;

mysql> show processlist;
+----+------+-----------+-------+---------+------+------------------------------+-----------------------------------------+
| Id | User | Host      | db    | Command | Time | State                        | Info                                    |
+----+------+-----------+-------+---------+------+------------------------------+-----------------------------------------+
|  2 | root | localhost | test2 | Query   |    0 | starting                     | show processlist                        |
|  3 | root | localhost | test2 | Query   |    5 | User sleep                   | update t1 set id=sleep(50) where id = 1 |
|  4 | root | localhost | test2 | Query   |    2 | Waiting for table level lock | select * from t1 where id = 2           |
+----+------+-----------+-------+---------+------+------------------------------+-----------------------------------------+
3 rows in set (0.00 sec)
```

### Day 58

```
day58 要不要使用分区表

```
```sql
create database test;
use test;
create table `t` (
    `ftime` datetime not null,
    `c` int(11) default null,
    key (`ftime`)
)engine=innodb default charset=latin1
partition by range (year(ftime))(
    partition p_2017 values less than (2017) engine=innodb,
    partition p_2018 values less than (2018) engine=innodb,
    partition p_2019 values less than (2019) engine=innodb,
    partition p_others values less than maxvalue engine=innodb
);
insert into t values ('2017-4-1',1),('2018-4-1',1);

root@80b89c9b7dd4:/var/lib/mysql/test# ls
db.opt  t1#P#p_2017.ibd  t1#P#p_2018.ibd  t1#P#p_2019.ibd  t1#P#p_others.ibd  t1.frm
```
```
1.分区表的存储形式
对于引擎层，是一张表
对于server层是四张表
2.分区表的引擎层行为
```
```sql
-- 对innodb来说
-- session A

use test;
begin;
select * from t where ftime='2017-5-1' for update;

-- session B

use test;
insert into t values('2018-2-1',1);
insert into t values('2017-12-1',1);

-- session 3

show engine innodb status;
```

```
-- 重点
------- TRX HAS BEEN WAITING 26 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 25 page no 4 n bits 72 index ftime of table `test`.`t1` /*
 Partition `p_2018` */ trx id 1818 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;


-- 详情
mysql> show engine innodb status\G
*************************** 1. row ***************************
  Type: InnoDB
  Name:
Status:
=====================================
2020-03-18 05:32:49 0x7f9a04100700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 6 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 4 srv_active, 0 srv_shutdown, 2481 srv_idle
srv_master_thread log flush and writes: 2485
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 14
OS WAIT ARRAY INFO: signal count 14
RW-shared spins 0, rounds 23, OS waits 10
RW-excl spins 0, rounds 0, OS waits 0
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 23.00 RW-shared, 0.00 RW-excl, 0.00 RW-sx
------------
TRANSACTIONS
------------
Trx id counter 1819
Purge done for trx's n:o < 1818 undo n:o < 0 state: running but idle
History list length 2
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421774760934136, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 1818, ACTIVE 26 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 2, OS thread handle 140299470391040, query id 29 localhost root
update
insert into t1 values('2017-12-1',1)
------- TRX HAS BEEN WAITING 26 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 25 page no 4 n bits 72 index ftime of table `test`.`t1` /*
 Partition `p_2018` */ trx id 1818 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

------------------
---TRANSACTION 1812, ACTIVE 40 sec
2 lock struct(s), heap size 1136, 1 row lock(s)
MySQL thread id 3, OS thread handle 140299470120704, query id 25 localhost root
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
429 OS file reads, 169 OS file writes, 75 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log sequence number 12394303
Log flushed up to   12394303
Pages flushed up to 12394303
Last checkpoint at  12394294
0 pending log flushes, 0 pending chkp writes
42 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 128868
Buffer pool size   8192
Free buffers       7731
Database pages     461
Old database pages 0
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 404, created 57, written 100
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 461, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=1, Main thread ID=140299484112640, state: sleeping
Number of rows inserted 3, updated 0, deleted 0, read 8
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================

1 row in set (0.00 sec)

```
```sql
-- 对myisam来说
-- myisam引擎只支持表锁，所以这条update语句会锁住整个表t1上的读
-- session A
alter table t engine=myisam;
update t set c=sleep(100) where ftime='2017-4-1';

-- session B
use test;
select * from t where ftime='2018-4-1';
select * from t where ftime='2017-5-1';-- blocked

```
```
分区策略
1. 通用分区策略 myisam 由server层控制，把所有分区表都打开
2. 本地分区策略 引擎层控制
```

```sql
-- session A
use test;
begin;
select * from t where ftime='2018-4-1';

-- session B
use test;
alter table t truncate partition p_2017;

mysql> show processlist;
+----+------+-----------+------+---------+------+---------------------------------+-----------------------------------------+
| Id | User | Host      | db   | Command | Time | State                           | Info                                    |
+----+------+-----------+------+---------+------+---------------------------------+-----------------------------------------+
|  2 | root | localhost | test | Query   |    0 | starting                        | show processlist                        |
|  3 | root | localhost | test | Query   |   25 | Waiting for table metadata lock | alter table t truncate partition p_2017 |
+----+------+-----------+------+---------+------+---------------------------------+-----------------------------------------+
2 rows in set (0.00 sec)
```
```
1 MySQL在第一次打开分区表的时候要访问所有的分区
2 在server层，认为这是一张表，共用一个MDL锁
3 在引擎层，是不同的表，MDL锁之后的流程，只访问必要的分区

分区表的应用场景
1 对业务透明
2 删除历史数据快 alter table t drop partition ...
```








### Day 59

```markdown
day59-1 自增主键为什么不是连续的
1. 自增主键的存储
    - myISAM 是存在数据文件上的
    - InnoDB 自增值是存储在内存中的 直到8.0版本才给InnoDB表的自增加上了持久化的能力
2. 自增主键在MySQL回滚时不能回收
3. 自增锁
    - 语句级别 5.0起
    - 5.1.22引入了新策略 innodb_automic_lock_mode 默认为1
        1. 0 语句级别
        2. 1 insert语句 在申请之后马上释放
             insert ... select 这样批量插入的数据，还是等到语句结束后释放
        3. 2 所有的都是申请完后释放
    innodb_automic_lock_mode 为 2时，binlog_format=row 即能提高并发度，又不会出现数据一致性问题
    批量插入的语句 不知道插入多少条数据
        - insert ... select 
        - replace ... select
        - load data
    还有一个优化
    批量插入时申请锁的数量每次都会乘以2 这也是自增主键不连续的原因之一

day 59-2 自增id用完了怎么办
自增id达到上限后的行为
1. 表的自增id达到上限后，再申请时它不会变，进而导致继续插入数据时报主键冲突
2. row_id达到上限后，会回归0重新递增，若出现相同的row_id，会覆盖
3. Xid只需要不在同一个binlog文件中出现重复值即可。可能性极小
4. Innodb的max_trx_id递增值每次MySQL重启都会被保存起来，所以我们文章中提到的脏读就是一个必现的bug
5. thread_id
6. table_id
7. binlog文件序号
```

### Day 60

```
day60
```