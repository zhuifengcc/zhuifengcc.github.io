---
title: 'Spring Cloud: Spring Cloud Sleuth'
date: 2018-09-02 00:07:02
tags:
categories: Spring Cloud 
---
随着微服务业务壮大，各种服务调用非常复杂，几乎每一个前端请求都会形成一条负载的分布式服务调用链路，途中任何一个服务失败都可能造成最后失败，所以整个链路追踪非常重要。Sleuth则时这样一个人分布式服务最终的解决方法。

## Quick Start
首先我们先搭建一个环境，需要的服务有：
eureka-server
微服务1，实现一个/trace-1接口，调用这个接口将触发对trace-2应用的调用
pom引入ribbon、web、eureka、sleuth
``` java 
@RestController
@EnableDiscoveryClient
@SpringBootApplication
public class SleuthApplication {
    private static Logger logger = LoggerFactory.getLogger(SleuthApplication.class);
    public static void main(String[] args) {
        SpringApplication.run(SleuthApplication.class, args);
    }
    @Bean
    @LoadBalanced
    RestTemplate restTemplate(){
        return new RestTemplate();
    }
    @RequestMapping(value = "/trace-1", method = RequestMethod.GET)
    public String trace(){
        logger.info("------call trace1--------------");
        return restTemplate().getForEntity("http://trace2/trace-2",String.class).getBody();
    }
}
```
配置
spring.application.name=trace1
server.port=9101
eureka.client.serviceUrl.defaultZone=http://localhost:8765/eureka

微服务2
``` java
@RestController
@EnableDiscoveryClient
@SpringBootApplication
public class Sleuth2Application {
    private static Logger logger = LoggerFactory.getLogger(Sleuth2Application.class);
    public static void main(String[] args) {
        SpringApplication.run(Sleuth2Application.class, args);
    }
    @RequestMapping(value = "/trace-2", method = RequestMethod.GET)
    public String trace(){
        logger.info("------call trace2--------------");
        return "Traced";
    }
}
```
我们启动，访问trace-1.可以看到控制台info的消息
INFO [trace1,e6032bd804035fe4,e6032bd804035fe4,false] 19800 — [nio-9101-exec-1] com.cg.sleuth.SleuthApplication : ——call trace1————–

INFO [trace2,e6032bd804035fe4,7af75df6c4f9693c,false] 11272 — [nio-9102-exec-1] com.cg.sleuth2.Sleuth2Application : ——call trace2————–

上述info种trace1为应用名，第二个值为Sleuth生成的ID，标记请求链路，第三个值为sleuth生成的另一个id，表示一个基本的工作单元，false表示是否输出到ziplin收集展示

## 跟踪原理
请求发送到分布式系统入口端点时，sleuth为该请求创建一个唯一标识，分布式系统内部流转时，始终传递该唯一表示，直到返回给请求方。通过traceID，可以将请求过程的日志关联起来
为了统计各个单元处理时间延迟，当请求到达服务时，通过唯一标识来标记他的开始及结束。
引入sleuth后，可以自动为当前应用构建起各种通信通道的最终机制：如rabbitMQ传递请求、zuul代理传递请求、resttemplate请求

上述示例通过resttemplate实现，sleuth会对该请求追踪\
## 抽样收集
追踪通过traceId和span Id实现了，但是分布式系统加入对所有进行追踪，将产生海量日志数据，对性能造成影响，日志储存开销也很大。之前的第四个值就是用来代表该信息是否要被后续的跟踪信息收集器获取和储存。
sleuth通过Sampler接口实现
``` java 
public interface Sampler{
    boolean isSampled(Span span);
}
```
通过该接口，sleuth会再产生跟踪信息的时候生成是否要被收集的标志。
默认情况下，sleuth会使用PercentageBasedSampler来实现抽样策略，可以在applicatino.properties中配置，默认10%收集

    spring.sleuth.sampler.percentage=0.1

不过开发测试阶段一般设置为1

## Logstash整合
之前实现了通过日志添加跟踪信息的功能，但是，在海量的日志文件中去排查分析是基本不可能的。
一般的日志分析平台有ELK，可以轻松搜索出我么想要的明细日志。
ELK 平台由ElesticSearch、Logstash、Kibana组成

    ElesticSearch:开源分布式搜索引擎。分布式、零配置，自动发现。。。
    Logstash：对日志进行收集过滤，并将其储存
    Kibana ： 结合上面两个，提供日志分析web界面

 spring cloud 与ELK整合时候，只需要和Logstash对接数据即可，为Logstaash提供json格式的日志输出。
 ## 整合过程

1.pom引入logstash-logback-encoder依赖

2./resource下创建bootstrap.properties配置文件，将spring.application.name=trace-1配置移动到该文件里，因为logback-spring.xml在application.properties之前加载

3./resource下创建logback-spring.xml.
``` java
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <springProperty scope="context" name="springAppName" source="spring.application.name"/>
    <!-- 日志在工程中的输出位置 -->
    <property name="LOG_FILE" value="${BUILD_FOLDER:-build}/${springAppName}"/>
    <!-- 控制台的日志输出样式 -->
    <property name="CONSOLE_LOG_PATTERN"
            value="%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr([${springAppName:-},%X{X-B3-TraceId:-},%X{X-B3-SpanId:-},%X{X-Span-Export:-}]){yellow} %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}"/>
    <!-- 控制台Appender -->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>utf8</charset>
    </encoder>
    </appender>

    <!-- 为logstash输出的json格式的Appender -->
    <appender name="logstash" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_FILE}.json</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE}.json.%d{yyyy-MM-dd}.gz</fileNamePattern>
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>UTC</timeZone>
                </timestamp>
                <pattern>
                    <pattern>
                        {
                        "severity": "%level",
                        "service": "${springAppName:-}",
                        "trace": "%X{X-B3-TraceId:-}",
                        "span": "%X{X-B3-SpanId:-}",
                        "exportable": "%X{X-Span-Export:-}",
                        "pid": "${PID:-}",
                        "thread": "%thread",
                        "class": "%logger{40}",
                        "rest": "%message"
                        }
                    </pattern>
                </pattern>
            </providers>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="console"/>
        <appender-ref ref="logstash"/>
    </root>
</configuration>
```
启动完成，访问/trace-1接口，可以看到工程目录下的build文件夹下的json记录
## Zipkin整合
ELK缺少对请求链路中各个阶段时间延迟的关注，使用zipkin可以解决。
zippkin基于google Dapper，可以使用它来收集各个服务器上请求链路的跟踪数据，并通过它提供的REST API接口来辅助对分布式系统的监控程序。

## Zuul网关
![架构图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Sleuth/7-1.png)

Collector：收集器组件，处理从外部系统发送过来的跟踪信息，转换为Zipkin内部处理的Span格式
Storage：存储组件，它主要对处理收集器接收到的跟踪信息，默认会将这些信息存储在内存中，也可以使用其他存储组件将跟踪信息存储到数据库中。
RESTful API：API组件，它主要用来提供外部访问接口。比如给客户端展示跟踪信息，或是外接系统访问以实现监控等。
Web UI：UI组件，基于API组件实现的上层应用。用户可以方便而有直观地查询和分析跟踪信息。
## 整合流程

1） 直接下载zipkin jar 直接启动，是一个spring boot 应用
2） trace-1 trace-2添加spring-cloud-sleuth-zipkin依赖
3） 在application.properties中增加zipkinserver配置信息

    spring.zipkin.base-url=http://localhost:9411

可以在zipkin页面看到 http://localhost:9411
## 整合消息中间件，收集消息，ELK处理储存Hbase；或者流计算，实时呈现
就是这一套技术。
1）
trace-1 trace-2添加spring-cloud-sleuth-stream
application.properties去除zipkin url参数，并配置rabbitmq信息
2）
zipkin-server 引入zipkin-stream依赖，

最后，我们使用之前的验证方法，通过向trace-1的接口发送几个请求：http://localhost:9101/trace-1 ,当有被抽样收集的跟踪信息时（调试时我们可以设置AlwaysSampler抽样机制来让每个跟踪信息都被收集），我们可以在RabbitMQ的控制页面中发现有消息被发送到了sleuth交换器中，同时我们再到zipkin服务端的Web页面中也能够搜索到相应的跟踪信息，那么我们使用消息中间件来收集跟踪信息的任务到这里就完成了。