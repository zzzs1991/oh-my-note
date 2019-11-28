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
        bootstrap_servers => "localhost:9092"
        topics => ["logcollector"]
        codec => json {
            charset => "UTF-8"
        }
    }
    # 如果有其他数据源，直接在下面追加
}

filter {
  grok {
    match => { "message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}" }
  }
    # 将message转为json格式
#    if [type] == "log" {
#        json {
#            source => "message"
#            target => "message"
#        }
#    }
}

output {
    # 处理后的日志落到本地文件
    file {
        path => "/config-dir/test.log"
        flush_interval => 0
    }
    # 处理后的日志入es
    elasticsearch {
        hosts => "localhost:9200"
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

# tags: ["demo1"]

output.kafka:
  hosts: ["172.16.128.191:9092"]
  topic: "logcollector"
  partition.round_robin:
    reachable_only: true
  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000 # 10MB
# debug
logging.level: debug
logging.to_files: true
logging.files:
  path: /home/newcore/elk/
  name: filebeat.log
  rotateeverybytes: 52428800 # 50MB
  keepfiles: 5
  
```
```shell script
{
  "@timestamp": "2019-11-28T08:32:58.597Z",
  "@metadata": {
    "beat": "filebeat",
    "type": "_doc",
    "version": "7.4.2"
  },
  "message": "[ERROR] [inventory] 2019-11-28 16:32:56 [com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient-DiscoveryClient-InstanceInfoReplicator-0] - Request execution error \n com.sun.jersey.api.client.ClientHandlerException: java.net.ConnectException: Connection refused: connect\n\tat com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:187)\n\tat com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123)\n\tat com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27)\n\tat com.sun.jersey.api.client.Client.handle(Client.java:652)\n\tat com.sun.jersey.api.client.WebResource.handle(WebResource.java:682)\n\tat com.sun.jersey.api.client.WebResource.access$200(WebResource.java:74)\n\tat com.sun.jersey.api.client.WebResource$Builder.post(WebResource.java:570)\n\tat com.netflix.discovery.shared.transport.jersey.AbstractJerseyEurekaHttpClient.register(AbstractJerseyEurekaHttpClient.java:56)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat com.netflix.discovery.shared.transport.decorator.MetricsCollectingEurekaHttpClient.execute(MetricsCollectingEurekaHttpClient.java:73)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient.executeOnNewServer(RedirectingEurekaHttpClient.java:118)\n\tat com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient.execute(RedirectingEurekaHttpClient.java:79)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat com.netflix.discovery.shared.transport.decorator.RetryableEurekaHttpClient.execute(RetryableEurekaHttpClient.java:119)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat com.netflix.discovery.shared.transport.decorator.SessionedEurekaHttpClient.execute(SessionedEurekaHttpClient.java:77)\nCaused by: java.net.ConnectException: Connection refused: connect\n\tat java.net.DualStackPlainSocketImpl.waitForConnect(Native Method)\n\tat java.net.DualStackPlainSocketImpl.socketConnect(DualStackPlainSocketImpl.java:85)\n\tat java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)\n\tat java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)\n\tat java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)\n\tat java.net.PlainSocketImpl.connect(PlainSocketImpl.java:172)\n\tat java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)\n\tat java.net.Socket.connect(Socket.java:589)\n\tat org.apache.http.conn.scheme.PlainSocketFactory.connectSocket(PlainSocketFactory.java:121)\n\tat org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:180)\n\tat org.apache.http.impl.conn.AbstractPoolEntry.open(AbstractPoolEntry.java:144)\n\tat org.apache.http.impl.conn.AbstractPooledConnAdapter.open(AbstractPooledConnAdapter.java:134)\n\tat org.apache.http.impl.client.DefaultRequestDirector.tryConnect(DefaultRequestDirector.java:610)\n\tat org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java:445)\n\tat org.apache.http.impl.client.AbstractHttpClient.doExecute(AbstractHttpClient.java:835)\n\tat org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:118)\n\tat org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56)\n\tat com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:173)\n\tat com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123)\n\tat com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27)\n ",
  "input": {
    "type": "log"
  },
  "ecs": {
    "version": "1.1.0"
  },
  "host": {
    "name": "WINDOWS-TFRODU8"
  },
  "agent": {
    "ephemeral_id": "f273ae84-552a-49b2-86fc-a042ffdd4769",
    "hostname": "WINDOWS-TFRODU8",
    "id": "a71ede78-e1fa-48e0-916e-4953775bba49",
    "version": "7.4.2",
    "type": "filebeat"
  },
  "log": {
    "offset": 1359162,
    "file": {
      "path": "D:\\elklog\\newcore-error.log"
    },
    "flags": [
      "multiline"
    ]
  }
}


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

