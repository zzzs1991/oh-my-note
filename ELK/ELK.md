# ELK


### 背景
我们运行的系统每时每刻都在产生不同的日志
且基本都存在单机磁盘上
- 操作系统
- 应用服务
- 业务逻辑

闭源
    Splunk
开源
    Elasticsearch
    Logstash
    Kibana

ELK stack有如下特点
- 处理方式灵活
- 配置简易上手
- 检索性能高效
- 集群线性扩展
- 前段操作绚丽

### Logstash
logstash配置 一定要有一个input一个output
#### 插件
- 输入插件
- 编解码插件
- 过滤器插件
- 输出插件
#### 输入插件
- 1.标准输入
- 2.文件输入
```
input {
  file {
    path => ["/var/log/*.log","/var/log/message"]
    type => "system"
    start_position => "begining"
  }
}
```
利用.sincedb记录被监听文件的inode，major number，minor number
和pos
   - discover_interval
   - exclude
   - sincedb_path
   - sincedb_write_interval
   - stat_interval
   - start_position
   
  
- 3.TCP输入
用消息队列来作为 logstash的broker
```
input {
    tcp {
        port => 8888
        mode => "server"
        ssl_enable => false
    }
}
```
用 `nc 127.0.0.1 8888 < olddata`导入旧数据
- 4.syslog输入
- 5.collectd输入
可以收集服务器性能状态信息



### ElasticSearch



### Kibana



