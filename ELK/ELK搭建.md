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


```
- kafka
```shell script


```
### 3.配置生产服务器
- 配置filebeat


- 分发filebeat
```shell script


```

- filebeat


- 

- 启动


