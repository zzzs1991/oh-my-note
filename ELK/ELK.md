## elk日志收集架构

---

#### 一.架构介绍

##### 1.logback

> logback 继承自 log4j，它建立在有十年工业经验的日志系统之上。它比其它所有的日志系统更快并且更小，包含了许多独特并且有用的特性。

日志的生产者，我们利用它的一些配置，讲error级别日志按照一定的格式单独打印出来

日志级别：

`TRACE < DEBUG < INFO < WARN < ERROR < FATAL`

##### 2.filebeat

> ## 轻量型日志采集器
>
> 当您要面对成百上千、甚至成千上万的服务器、虚拟机和容器生成的日志时，请告别 SSH 吧。Filebeat 将为您提供一种轻量型方法，用于转发和汇总日志与文件，让简单的事情不再繁杂。

特点：

- 读取日志文件，但不做数据解析处理
- 保证数据至少被读取一次，数据不丢失
- 其他能力
  - 处理多行数据
  - json格式解析
  - 简单过滤功能

##### 3.kafka

> 一个分布式流处理平台
>
> 它可以用于两大类别的应用:
>
> 1. 构造实时流数据管道，它可以在系统或应用之间可靠地获取数据。 (相当于message queue)
> 2. 构建实时流式应用程序，对这些流数据进行转换或者影响。 (就是流处理，通过kafka stream topic和topic之间内部进行变化)

我们将它作为简单的消息队列使用

- 利用不同`topic`输送不同种类日志
- 日志量大的时候`削峰填谷`
- 将`filebeat`和`logstash`进行`解耦`

##### 4.logstash

> ## 集中、转换和存储数据
>
> Logstash 是开源的服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到您最喜欢的“存储库”中。

核心

`输入-过滤-输出`

- 采集各种样式、大小和来源的数据
- 实时解析和转换数据
    - 利用 Grok 从非结构化数据中派生出结构
    - 从 IP 地址破译出地理坐标
    - 将 PII 数据匿名化，完全排除敏感字段
    - 简化整体处理，不受数据源、格式或架构的影响
- 选择不同存储库，导出数据

我们利用它，在kafka上消费数据，并将一条一条日志数据，解析过滤成为结构化的数据，存入elasticsearch中

##### 5.elasticsearch

> Elastic Stack 的核心
>
> Elasticsearch 是一个分布式、RESTful 风格的搜索和数据分析引擎，能够解决不断涌现出的各种用例。 作为 Elastic Stack 的核心，它集中存储您的数据，帮助您发现意料之中以及意料之外的情况。

- 倒排索引
- 全文检索

我们利用它来存储logstash过滤处理好的结构化数据，还可实时检索

##### 6.kibana

> 您了解 Elastic Stack 的窗口
>
> 通过 Kibana，您可以对自己的 Elasticsearch 进行可视化，还可以在 Elastic Stack 中进行导航，这样您便可以进行各种操作了，从跟踪查询负载，到理解请求如何流经您的整个应用，都能轻松完成。

功能

- 基本内容
- 位置分析
- 时序分析
- 机器学习
- 图表、网络

我们利用它来进行，日志数据时序展示、和检索的简单功能。


#### 二.日志收集搭建过程

##### 1.logback

在`logback-spring.xml`文件中配置
```xml
<configuration>
    <!--错误日志收集start-->
    <property name="LOG_HOME" value="/home/newcore/www/coreplus/logs/"/>

    <property name="moduleName" value="inventory"/>

    <appender name="errorCollect" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>error</level>
        </filter>
        <encoder charset="UTF-8">
            <pattern>
                <!-- 日志级别 模块名 日期 类名 日志信息 异常栈信息（20行） -->
                [%-5level] [${moduleName}] %d{yy/MM/dd HH:mm:ss} [%c] - %msg %n %ex{20} %n
            </pattern>
        </encoder>
        <file>${LOG_HOME}/${moduleName}-error.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/${moduleName}-error.%d{yyyy-MM-dd}.log</fileNamePattern>
            <MaxHistory>14</MaxHistory>
        </rollingPolicy>
    </appender>
    <!--错误日志收集end-->

    <!-- 使用INFO级别的日志输出 -->
    <root level="INFO">
        <appender-ref ref="errorCollect"/>
    </root>
</configuration>
```

配置完成后，产生的日志如下：

```shell script
[ERROR] [manufacture] 19/12/13 13:39:10 [com.coreplus.machine.internet.model.bo.MachineInternetCreator] - 设备对接 -> 存储实时数据失败：网关id对应设备不存在 -> {"actualQty":0,"gatewayFailureCode":100,"gatewayId":"JoyrryIotBox-C0002-caf04098-76cd-48dc-b702-77438f0a2b0e","gatewayStatus":1,"machineFailureCode":100,"machineStatus":0,"planQty":0,"runSpeed":0,"runtime":1576215559000} 
[ERROR] [usercenter] 19/12/13 13:39:08 [com.xinheyun.userCenter.common.BaseController] - tenantId : 945dffdd-1073-403e-bca9-042b8b2432ff; request : GET /api/user-center/auth/i/openapi/app {"tenantId":["c607f27e-c97a-445f-8977-172ab267e9a4"]}  
 java.lang.IllegalArgumentException: Source must not be null
	at org.springframework.util.Assert.notNull(Assert.java:134)
	at org.springframework.beans.BeanUtils.copyProperties(BeanUtils.java:591)
	at org.springframework.beans.BeanUtils.copyProperties(BeanUtils.java:537)
	at com.xinheyun.userCenter.auth.controller.AuthInnerController.getOpenApiApp(AuthInnerController.java:72)
	at sun.reflect.GeneratedMethodAccessor419.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:205)
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:133)
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:97)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:827)
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:738)
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:85)
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:967)
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:901)
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:970)
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:861)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:635)
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:846)
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:742)
```

##### 2.filebeat
- 安装

`wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.5.0-x86_64.rpm`
`rpm -ivh filebeat-7.5.0-x86_64.rpm`
centos系统推荐使用rpm安装，
使用tar.gz文件解压安装的进程会无故停止

- 配置

备份配置文件

`sudo cp /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.bak`

添加配置

`sudo vim /etc/filebeat/filebeat.yml`

```yaml
# 配置扫描那些日志文件
filebeat.inputs:
- type: log
	# 开关 可以配置多个type，只能有一个有作用
  enabled: true
  tail_files: true
  paths:
    - /home/newcore/www/coreplus/logs/*-error.log
  multiline.pattern: '^\['
  multiline.negate: true
  multiline.match: after

# 配置输出到kafka
output.kafka:
  enabled: true
  hosts: ["172.16.128.191:9092"]
  topic: newcore-error
  partition.round_robin:
    reachable_only: true
  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000
```

- 启动

修改hosts
filebeat通过host与kafka相连，必须配置hosts，否则消息丢不进去
```shell script
vim /etc/hosts

172.16.128.191  logcollector01  logcollector01
```

`sudo systemctl start filebeat.service`

```json5
// filebeat 输出信息

{
  "@timestamp": "2019-11-29T03:12:33.313Z",
  "@metadata": {
    "beat": "filebeat",
    "type": "_doc",
    "version": "7.4.2"
  },
  "agent": {
    "id": "a71ede78-e1fa-48e0-916e-4953775bba49",
    "version": "7.4.2",
    "type": "filebeat",
    "ephemeral_id": "fcaee862-75b1-45d3-a855-2beb4b6eae13",
    "hostname": "WINDOWS-TFRODU8"
  },
  "log": {
    "offset": 7035,
    "file": {
      "path": "D:\\elklog\\newcore-error.log"
    },
    "flags": [
      "multiline"
    ]
  },
  "message": "[ERROR] [inventory] 19/11/29 11:12:26 [com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient] - Request execution error \n com.sun.jersey.api.client.ClientHandlerException: java.net.ConnectException: Connection refused: connect\n\tat com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:187)\n\tat com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123)\n\tat com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27)\n\tat com.sun.jersey.api.client.Client.handle(Client.java:652)\n\tat com.sun.jersey.api.client.WebResource.handle(WebResource.java:682)\n\tat com.sun.jersey.api.client.WebResource.access$200(WebResource.java:74)\n\tat com.sun.jersey.api.client.WebResource$Builder.post(WebResource.java:570)\n\tat com.netflix.discovery.shared.transport.jersey.AbstractJerseyEurekaHttpClient.register(AbstractJerseyEurekaHttpClient.java:56)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat com.netflix.discovery.shared.transport.decorator.MetricsCollectingEurekaHttpClient.execute(MetricsCollectingEurekaHttpClient.java:73)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient.executeOnNewServer(RedirectingEurekaHttpClient.java:118)\n\tat com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient.execute(RedirectingEurekaHttpClient.java:79)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat com.netflix.discovery.shared.transport.decorator.RetryableEurekaHttpClient.execute(RetryableEurekaHttpClient.java:119)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat com.netflix.discovery.shared.transport.decorator.SessionedEurekaHttpClient.execute(SessionedEurekaHttpClient.java:77)\nCaused by: java.net.ConnectException: Connection refused: connect\n\tat java.net.DualStackPlainSocketImpl.waitForConnect(Native Method)\n\tat java.net.DualStackPlainSocketImpl.socketConnect(DualStackPlainSocketImpl.java:85)\n\tat java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)\n\tat java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)\n\tat java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)\n\tat java.net.PlainSocketImpl.connect(PlainSocketImpl.java:172)\n\tat java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)\n\tat java.net.Socket.connect(Socket.java:589)\n\tat org.apache.http.conn.scheme.PlainSocketFactory.connectSocket(PlainSocketFactory.java:121)\n\tat org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:180)\n\tat org.apache.http.impl.conn.AbstractPoolEntry.open(AbstractPoolEntry.java:144)\n\tat org.apache.http.impl.conn.AbstractPooledConnAdapter.open(AbstractPooledConnAdapter.java:134)\n\tat org.apache.http.impl.client.DefaultRequestDirector.tryConnect(DefaultRequestDirector.java:610)\n\tat org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java:445)\n\tat org.apache.http.impl.client.AbstractHttpClient.doExecute(AbstractHttpClient.java:835)\n\tat org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:118)\n\tat org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56)\n\tat com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:173)\n\tat com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123)\n\tat com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27)\n ",
  "input": {
    "type": "log"
  },
  "ecs": {
    "version": "1.1.0"
  },
  "host": {
    "name": "WINDOWS-TFRODU8"
  }
}
```

##### 3.kafka
下载
```shell script
cd /usr/local

wget https://www.apache.org/dyn/closer.cgi?path=/kafka/2.3.1/kafka_2.12-2.3.1.tgz

```

解压
```shell script
tar -zxvf kafka_2.12-2.3.1.tgz

sudo mv kafka_2.12-2.3.1/ kafka/

```

运行

```shell script
#先启动zookeeper
/usr/local/kafka/bin/zookeeper-server-start.sh /usr/local/kafka/config/zookeeper.properties
#再启动kafka
/usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties
```

可以不用创建队列，kafka会自动创建topic

##### 4.logstash

下载

`wget https://artifacts.elastic.co/downloads/logstash/logstash-7.5.0.tar.gz`

解压

```shell script
cd /usr/local
tar -zxvf logstash-7.5.0.tar.gz
mv logstash-7.5.0/ logstash/
```

配置

```shell script
cd /usr/local/logstash/conf
cp .conf test.conf
vim test.conf
```
```ruby
# ----
input {
    # 从kafka消费信息
    kafka {
        # elasticsearch ip port
        bootstrap_servers => "172.16.128.191:9092"
        # 从那些主题消费
        topics => ["newcore-error"]
        # 配置字符集
        codec => json {
            charset => "UTF-8"
        }
    }
}

filter{
    # 配置grok 根据将message字段 处理为结构化数据
    grok {
      match => ['message',"\[%{LOGLEVEL:level}\]\s\[%{HOSTNAME:project}\]\s%{DATESTAMP:date}\s\[%{URIHOST:class}\]\s\-\s%{GREEDYDATA:exception}"]
    }
    # 也可一通过mutate将一些不需要的字段删除
}

output {
    # 配置elasticsearch ip 和 port
    # 配置数据存在es中的哪张表里
    elasticsearch {
      hosts => "172.16.128.191:9200"
      index => "logstash-error-log-%{+YYYY.MM.dd}"
    }
}
```

运行

编写简单启动脚本并启动

```shell script
cd \
&& echo 'cd /home/newcore/logstash
nohup ./bin/logstash -f ./conf/test-wangc.conf &' > .start_logstash.sh \
&& chmod +x .start_logstash.sh

cd
sh .start_logstash.sh
```

logstash输出的内容 rubydebug输出
```ruby
{
       "class" => "com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient",
       "project" => "inventory",
       "message" => "[ERROR] [inventory] 19/11/29 12:06:17
[com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient]
- Request execution error \n
com.sun.jersey.api.client.ClientHandlerException:
java.net.ConnectException: Connection refused: connect\n\tat
com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:187)\n\tat
com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123)\n\tat
com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27)\n\tat
com.sun.jersey.api.client.Client.handle(Client.java:652)\n\tat
com.sun.jersey.api.client.WebResource.handle(WebResource.java:682)\n\tat
com.sun.jersey.api.client.WebResource.access$200(WebResource.java:74)\n\tat
com.sun.jersey.api.client.WebResource$Builder.post(WebResource.java:570)\n\tat
com.netflix.discovery.shared.transport.jersey.AbstractJerseyEurekaHttpClient.register(AbstractJerseyEurekaHttpClient.java:56)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat
com.netflix.discovery.shared.transport.decorator.MetricsCollectingEurekaHttpClient.execute(MetricsCollectingEurekaHttpClient.java:73)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat
com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient.executeOnNewServer(RedirectingEurekaHttpClient.java:118)\n\tat
com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient.execute(RedirectingEurekaHttpClient.java:79)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat
com.netflix.discovery.shared.transport.decorator.RetryableEurekaHttpClient.execute(RetryableEurekaHttpClient.java:119)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat
com.netflix.discovery.shared.transport.decorator.SessionedEurekaHttpClient.execute(SessionedEurekaHttpClient.java:77)\nCaused
by: java.net.ConnectException: Connection refused: connect\n\tat
java.net.DualStackPlainSocketImpl.waitForConnect(Native Method)\n\tat
java.net.DualStackPlainSocketImpl.socketConnect(DualStackPlainSocketImpl.java:85)\n\tat
java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)\n\tat
java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)\n\tat
java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)\n\tat
java.net.PlainSocketImpl.connect(PlainSocketImpl.java:172)\n\tat
java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)\n\tat
java.net.Socket.connect(Socket.java:589)\n\tat
org.apache.http.conn.scheme.PlainSocketFactory.connectSocket(PlainSocketFactory.java:121)\n\tat
org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:180)\n\tat
org.apache.http.impl.conn.AbstractPoolEntry.open(AbstractPoolEntry.java:144)\n\tat
org.apache.http.impl.conn.AbstractPooledConnAdapter.open(AbstractPooledConnAdapter.java:134)\n\tat
org.apache.http.impl.client.DefaultRequestDirector.tryConnect(DefaultRequestDirector.java:610)\n\tat
org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java:445)\n\tat
org.apache.http.impl.client.AbstractHttpClient.doExecute(AbstractHttpClient.java:835)\n\tat
org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:118)\n\tat
org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56)\n\tat
com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:173)\n\tat
com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123)\n\tat
com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27)\n
",
         "agent" => {
                "type" => "filebeat",
        "ephemeral_id" => "ea1c710d-d956-4b8f-9709-38051e8229ac",
            "hostname" => "WINDOWS-TFRODU8",
                  "id" => "a71ede78-e1fa-48e0-916e-4953775bba49",
             "version" => "7.4.2"
    },
     "exception" => "Request execution error \n
com.sun.jersey.api.client.ClientHandlerException:
java.net.ConnectException: Connection refused: connect\n\tat
com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:187)\n\tat
com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123)\n\tat
com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27)\n\tat
com.sun.jersey.api.client.Client.handle(Client.java:652)\n\tat
com.sun.jersey.api.client.WebResource.handle(WebResource.java:682)\n\tat
com.sun.jersey.api.client.WebResource.access$200(WebResource.java:74)\n\tat
com.sun.jersey.api.client.WebResource$Builder.post(WebResource.java:570)\n\tat
com.netflix.discovery.shared.transport.jersey.AbstractJerseyEurekaHttpClient.register(AbstractJerseyEurekaHttpClient.java:56)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat
com.netflix.discovery.shared.transport.decorator.MetricsCollectingEurekaHttpClient.execute(MetricsCollectingEurekaHttpClient.java:73)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat
com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient.executeOnNewServer(RedirectingEurekaHttpClient.java:118)\n\tat
com.netflix.discovery.shared.transport.decorator.RedirectingEurekaHttpClient.execute(RedirectingEurekaHttpClient.java:79)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat
com.netflix.discovery.shared.transport.decorator.RetryableEurekaHttpClient.execute(RetryableEurekaHttpClient.java:119)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator.register(EurekaHttpClientDecorator.java:56)\n\tat
com.netflix.discovery.shared.transport.decorator.EurekaHttpClientDecorator$1.execute(EurekaHttpClientDecorator.java:59)\n\tat
com.netflix.discovery.shared.transport.decorator.SessionedEurekaHttpClient.execute(SessionedEurekaHttpClient.java:77)\nCaused
by: java.net.ConnectException: Connection refused: connect\n\tat
java.net.DualStackPlainSocketImpl.waitForConnect(Native Method)\n\tat
java.net.DualStackPlainSocketImpl.socketConnect(DualStackPlainSocketImpl.java:85)\n\tat
java.net.AbstractPlainSocketImpl.doConnect(AbstractPlainSocketImpl.java:350)\n\tat
java.net.AbstractPlainSocketImpl.connectToAddress(AbstractPlainSocketImpl.java:206)\n\tat
java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:188)\n\tat
java.net.PlainSocketImpl.connect(PlainSocketImpl.java:172)\n\tat
java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392)\n\tat
java.net.Socket.connect(Socket.java:589)\n\tat
org.apache.http.conn.scheme.PlainSocketFactory.connectSocket(PlainSocketFactory.java:121)\n\tat
org.apache.http.impl.conn.DefaultClientConnectionOperator.openConnection(DefaultClientConnectionOperator.java:180)\n\tat
org.apache.http.impl.conn.AbstractPoolEntry.open(AbstractPoolEntry.java:144)\n\tat
org.apache.http.impl.conn.AbstractPooledConnAdapter.open(AbstractPooledConnAdapter.java:134)\n\tat
org.apache.http.impl.client.DefaultRequestDirector.tryConnect(DefaultRequestDirector.java:610)\n\tat
org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java:445)\n\tat
org.apache.http.impl.client.AbstractHttpClient.doExecute(AbstractHttpClient.java:835)\n\tat
org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:118)\n\tat
org.apache.http.impl.client.CloseableHttpClient.execute(CloseableHttpClient.java:56)\n\tat
com.sun.jersey.client.apache4.ApacheHttpClient4Handler.handle(ApacheHttpClient4Handler.java:173)\n\tat
com.sun.jersey.api.client.filter.GZIPContentEncodingFilter.handle(GZIPContentEncodingFilter.java:123)\n\tat
com.netflix.discovery.EurekaIdentityHeaderFilter.handle(EurekaIdentityHeaderFilter.java:27)\n
",
          "date" => "19/11/29 12:06:17",
         "level" => "ERROR",
         "input" => {
        "type" => "log"
    },
      "@version" => "1",
          "host" => {
        "name" => "WINDOWS-TFRODU8"
    },
    "@timestamp" => 2019-11-29T04:06:36.681Z,
           "log" => {
         "flags" => [
            [0] "multiline"
        ],
        "offset" => 1687115,
          "file" => {
            "path" => "D:\\elklog\\newcore-error.log"
        }
    },
           "ecs" => {
        "version" => "1.1.0"
    }
}
```

##### 5.elasticsearch

直接下载、解压、采用默认配置后台启动即可
```shell script
cd /usr/local

wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.5.0-linux-x86_64.tar.gz

tar -zxvg https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.5.0-linux-x86_64.tar.gz

mv elasticsearch-7.5.0-linux-x86/ elasticsearch/

nohup /usr/local/elasticsearch/bin/elasticsearch &
```

##### 6.kibana

直接下载、解压、采用默认配置后台启动即可
```shell script
cd /usr/local

wget https://artifacts.elastic.co/downloads/kibana/kibana-7.5.0-linux-x86_64.tar.gz

tar -zxvg https://artifacts.elastic.co/downloads/kibana/kibana-7.5.0-linux-x86_64.tar.gz

mv kibana-7.5.0-linux-x86_64/ kibana/

nohup /usr/local/kibana/bin/kibana &
```

##### 7.kibana 网页操作













