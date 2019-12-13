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

### 一.Logstash

logstash配置 一定要有一个input一个output
#### 1插件
- 输入插件
- 编解码插件
- 过滤器插件
- 输出插件
#### 1.1输入插件
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

#### 1.2编解码配置
Codec = coder/decoder

logstash = input｜filter｜output
｜
logstash = input|decode|filter|encode|output

- json编解码
- 多行事件编码
    
    codec/multiline
- 网络流编码
    
    需要单独配置elasticsearch索引模版

#### 1.3过滤器配置
丰富的过滤器插件是logstash如此强大的原因

- date时间处理
    logstash-filter-date插件
    
    1.ISO8601
    
    2.UNIX
    
    3.UNIX_MS
    
    4.TAI64N
    
    5.Joda-Time
    
- grok正则捕获

  logstash中最重要的插件
```
filter {
    grok {
        patterns_dir =>“/path/to/your/own/patterns”
        match => {
            “message” =>“%{SYSLOGBASE} %{DATA:message}”
        }
        overwrite => [“message”]
    }
}
```

如果你把“message”里所有的信息都grok到不同的字段了，数据实质上就相当于是重复存储了两份。所以你可以用remove_field参数来删除掉message字段，或者用overwrite参数来重写默认的message字段，只保留最重要的部分。

使用Grok Debugger（http://grokdebug.herokuapp.com/）来调试自己的grok表达式。

多项选择
```
match => [“message”, “（？<request_time>\d+（？：\.\d+）？）”,“message”, “%{SYSLOGBASE} %{DATA:message}”,“message”, “（？m）%{WORD}”
]
```

- geoIp查询

- json编解码

- key-value切分

- metrics数值统计

- mutate数据修改
    
    - 类型转换
        ```
            filter {
                mutate {
                    convert => [“request_time”, “float”]
                }
            }
        ```
    - 字符串处理
        
        - gsub
        
        - split 分割字符串
        
        - join 连接
        
        - merge 合并
        
        - strip 去掉空格
        
        - lowercase
        
        - uppercase
        
    - 字段处理
        
        - rename
        
        - update
        
        - replace
        
    - 执行次序
    
    - 随心所欲的ruby处理
        
        使用好ruby脚本可以减少cpu消耗
     
    - split 拆分事件
    
        split插件中使用的是yield功能，其结果是split出来的新事件，会直接结束其在filter阶段的历程，也就是说写在split后面的其他filter插件都不起作用，进入到output阶段。所以，一定要保证split配置写在全部filter配置的最后。
        
#### 1.4输出插件

- 输出到elasticsearch

```
output {
  elasticsearch {
    host =>“192.168.0.2”
    protocol =>“http”
    index =>“logstash-%{type}-%{+YYYY.MM.dd}”
    index_type =>“%{type}”
    workers => 5
    template_overwrite => true
  }
}
```

- 发送email

    logstash-output-email
    
- 调用系统命令执行

    logstash-output-exec
    
- 保存成文件

- 报警发送到nagios

- statsd
    
    统计信息
    
- stdout
    
    codec 和 worker
    用来调试
- TCP 发送数据
    
    不建议使用，建议使用消息队列
    
- 输出到HDFS

### 2.场景示例


#### 2.1Nginx访问日志


#### 2.2Java日志

    


        
    
    
    


### 二.ElasticSearch



### 三.Kibana



