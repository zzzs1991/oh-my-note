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

```mysql
CREATE TABLE `t` (
    `id` int(11) NOT null,
    `a` int(11) DEFAULT null,
    `b` int(11) DEFAULT null,
    PRIMARY KEY (`id`),
    KEY `a` (`a`),
    KEY `b` (`b`)
) ENGINE=INNODB;
```

```mysql
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

```mysql
select * from t where a between 10000 and 20000;
```

```mysql
start transaction with consistent snapshot;



commit;
```

```mysql
delete from t;
call idata();

set long_query_time=0;
select * from t where a between 10000 and 20000;
select * from t force index(a) where a between 10000 and 20000;
```

```mysql
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

```mysql
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

```shell

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

```
day46

```

### Day 47

```
day47
```



