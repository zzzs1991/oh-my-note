### mysql相关知识记录

#### 1.`@@`与`@`的区别
- `@@`代表系统配置
- `@`代表用户配置

#### 2.设置当前会话的隔离级别
```sql
set session tranaction isolation level read uncommitted;
set session tranaction isolation level read committed;
set session tranaction isolation level repeatable read;
set session tranaction isolation level serializable;
select @@tx_isolation;
show variables like ''
```

#### 3.docker启动的mysql如何将表dump下来
```shell script
#dump指定数据库
docker exec mysqltest sh -c 'exec mysqldump -uroot -p123456 test' > ./test.sql
#dump所有数据库
docker exec some-mysql sh -c 'exec mysqldump --all-databases -uroot -p"$MYSQL_ROOT_PASSWORD"' > /some/path/on/your/host/all-databases.sql
```

#### 4.docker启动的mysql如何load数据
```shell script
docker exec -i some-mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < /some/path/on/your/host/all-databases.sql
```