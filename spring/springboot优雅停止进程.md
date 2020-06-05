springboot优雅停止进程
1. spring boot actuator
```yaml
management:
  endpoint:
    shutdown:
      enabled: true
```
> 测试结果: 不能优雅关闭
```shell
➜  ~ curl localhost:8080/test
curl: (52) Empty reply from server

➜  ~ curl -XPOST localhost:8080/manage/shutdown
{"message":"Shutting down, bye..."}%
```

2. 编码方式
```java
package com.fangmeili.testtable;

import lombok.extern.slf4j.Slf4j;
import org.apache.catalina.connector.Connector;
import org.apache.tomcat.util.threads.ThreadPoolExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.event.ContextClosedEvent;

import java.util.concurrent.Executor;
import java.util.concurrent.TimeUnit;

@Slf4j
@SpringBootApplication(scanBasePackages = {"com.fangmeili.testtable"})
public class TesttableApplication{

    public static void main(String[] args) {
        SpringApplication.run(TesttableApplication.class, args);
    }


    @Bean
    public GracefulShutdown gracefulShutdown() {
        log.warn("create gracefulShutdown");
        return new GracefulShutdown();
    }

    @Bean
    public ConfigurableServletWebServerFactory webServerFactory(final GracefulShutdown gracefulShutdown) {
        log.warn("add gracefulShutdown");
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.addConnectorCustomizers(gracefulShutdown);
        return factory;
    }

    private static class GracefulShutdown implements TomcatConnectorCustomizer,
            ApplicationListener<ContextClosedEvent> {

        private static final Logger log = LoggerFactory.getLogger(GracefulShutdown.class);

        private volatile Connector connector;

        @Override
        public void customize(Connector connector) {
            this.connector = connector;
        }

        @Override
        public void onApplicationEvent(ContextClosedEvent event) {
            log.warn("get application event");
            this.connector.pause();
            Executor executor = this.connector.getProtocolHandler().getExecutor();
            if (executor instanceof ThreadPoolExecutor) {
                try {
                    ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                    threadPoolExecutor.shutdown();
                    if (!threadPoolExecutor.awaitTermination(30, TimeUnit.SECONDS)) {
                        log.warn("Tomcat thread pool did not shut down gracefully within "
                                + "30 seconds. Proceeding with forceful shutdown");
                    }
                }
                catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
            }
            log.warn("excutors shut down");
        }
    }
}
```

> 测试结果:可以等待tomcat线程池中线程全部正常结束后 关闭进程
```shell
➜  ~ curl localhost:8080/test
{"name":"aaa","age":12}%
```