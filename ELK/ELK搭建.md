### 0.预备


| 机器             | 软件            |
|----------------|---------------|
| logcolletcor01 | Kibana        |
| logcolletcor01 | ElasticSearch |
| logcolletcor01 | Logstash      |
| logcolletcor01 | Kafka         |
| node           | Filebeat      |

- 版本统一 7.4.2
- c2机器
```shell script
c2machine001	172.16.128.173  
c2machine002	172.16.128.178
c2machine003	172.16.128.177
c2machine004	172.16.128.174
c2machine005	172.16.128.179
c2machine006	172.16.128.175
c2machine007	172.16.128.176
c2machine008	172.16.128.180
c2machine009	172.16.128.181
c2machine010	172.16.128.182
c2machine011	172.16.128.183
c2machine012	172.16.128.184
c2machine013	172.16.128.186
c2machine014	172.16.128.185
c2machine015	172.16.128.187
c2machine016	172.16.128.188
```

| 线上机器         | ip                | 服务1            | 服务2          | 服务3           |
|--------------|-------------------|----------------|--------------|---------------|
| c2machine001 | 172\.16\.128\.173 | Discovery      | Gateway      | Bom           |
| c2machine002 | 172\.16\.128\.178 | core\-planning |              |               |
| c2machine003 | 172\.16\.128\.177 | inventory      |              |               |
| c2machine004 | 172\.16\.128\.174 | Manufactory    |              |               |
| c2machine005 | 172\.16\.128\.179 | core           |              |               |
| c2machine006 | 172\.16\.128\.175 | message        | user\-center | id\-generator |
| c2machine007 | 172\.16\.128\.176 | mongodb        | xxl\-job     |               |
| c2machine008 | 172\.16\.128\.180 | nginx          | node         |               |
| c2machine009 | 172\.16\.128\.181 | discovery      | gateway      | bom           |
| c2machine010 | 172\.16\.128\.182 | core\-planning |              |               |
| c2machine011 | 172\.16\.128\.183 | inventory      |              |               |
| c2machine012 | 172\.16\.128\.184 | manufactory    |              |               |
| c2machine013 | 172\.16\.128\.186 | core           |              |               |
| c2machine014 | 172\.16\.128\.185 | message        | user\-center |               |
| c2machine015 | 172\.16\.128\.187 | item           |              |               |
| c2machine016 | 172\.16\.128\.188 | item           |              |               |


### 1 准备

- 下载
    - elasticsearch-7.4.2
    - filebeat-7.4.2-linux-x86_64
    - kibana-7.4.2-linux-x86_64
    - logstash-7.4.2
    - kafka_2.12-2.3.1
    - jdk-11.0.5
- 解压
```shell script
tar -zxvf xxxx.7.4.2.tar.gz -C /home/newcore/
```
### 2.配置logcollector01

- elasticsearch

在【启动】bin>./elasticsearch 的时候
【报错】max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]

解决:
1)sudo vi /etc/sysctl.conf 文件最后添加一行

作者：Sam_L
链接：https://www.jianshu.com/p/402f7dfceab6
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

```shell script
vim /home/newcore/elasticsearch-7.4.2/config/elasticsearch.yml

network.host: 0.0.0.0
http.port: 9200
cluster.initial_master_nodes: ["node-1"]
```

- kibana
```shell script
vim /home/newcore/kibana-7.4.2/config/elasticsearch.yml

server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://localhost:9200"]
kibana.index: ".kibana"
```
- logstash


```shell script
input {
    kafka {
        bootstrap_servers => "192.168.0.111:9091"
        topics => ["logtest"]
        codec => json {
            charset => "UTF-8"
        }
    }
    # 如果有其他数据源，直接在下面追加
}

filter {
    # 将message转为json格式
    if [type] == "log" {
        json {
            source => "message"
            target => "message"
        }
    }
}

output {
    # 处理后的日志落到本地文件
    file {
        path => "/config-dir/test.log"
        flush_interval => 0
    }
    # 处理后的日志入es
    elasticsearch {
        hosts => "192.168.0.112:9201"
        index => "logstash-%{[type]}-%{+YYYY.MM.dd}"
  }
}

```
- kafka
```shell script
bin/zookeeper-server-start.sh config/zookeeper.properties
bin/kafka-server-start.sh config/server.properties
```
日志目录 kafka目录/ server.log

### 3.配置生产服务器
- 配置filebeat

- filebeat

- 启动


```shell script
./filebeat -e -c filebeat.yml -d 
```

```shell script
filebeat.inputs:
- type: log
  enabled: true
  tail_files: true
  paths:
    - /home/newcore/www/coreplus/logs/user-center.log
  multiline.pattern: '^\['
  multiline.negate: true
  multiline.match: after

tags: ["demo1"]

output.kafka:
  hosts: ["172.16.128.191:9092"]
  topic: "logcollector"
  partition.round_robin:
    reachable_only: true
  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000 # 10MB

logging.level: debug
logging.to_files: true
logging.files:
  path: /home/newcore/elk/
  name: filebeat.log
  rotateeverybytes: 52428800 # 50MB
  keepfiles: 5
  
```


```shell script
PID=$(ps -ef | grep elasticsearch | grep -v grep | awk '{ print $2 }')
if [ -z "$PID" ]
then
    echo elasticsearch is already stopped
else
    echo kill $PID
    kill $PID
fi
nohup 
```



```shell script
vim /usr/local/bin/start_es.sh

nohup /home/newcore/elasticsearch-7.4.2/bin/elasticsearch > /home/newcore/elk/

```


logstash

```shell script


```

