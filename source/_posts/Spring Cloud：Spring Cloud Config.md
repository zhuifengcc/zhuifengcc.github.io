---
title: 'Spring Cloud: Spring Cloud Config'
date: 2018-08-17 23:40:18
tags:
categories: Spring Cloud 
---
Spring Cloud Config是用来为分布式系统中基础设施和微服务提供集中化外部配置的支持，分为客户端和服务端。  
*客户端：微服务的各个微服务应用

    通过指定的配置中心来管理应用资源与业务的配置内容，启动时从配置中心获取加载配置信息

*服务端：分布式配置中心

    独立的微服务应用，连接配置仓库，为客户端获取配置信息。默认采用Git

## Quick Start
构建一个Git分布式配置中心，并在客户端中通过配置中心获取配置
### 配置中心构建
1）springboot pom引入config-server即可
``` java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.demo</groupId>
    <artifactId>config-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>config-server</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.M9</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
</project>
```
### 主类及配置
主类加入@EnableConfigServer、同时配置congfig中心
``` java
spring.application.name=config-server
server.port=7001

spring.cloud.config.server.git.uri=https://github.com/zhuifengcc/SpringCloud-Learn/
spring.cloud.config.server.git.searchPaths=config-repo
spring.cloud.config.server.git.username=*****
spring.cloud.config.server.git.password=*****
```
然后我们在git上新建仓库SpringCloud-Learn，仓库下建立文件夹config-repo,文件夹下放入配置文件

*config.properties

    form=git-default-1.0

*config-dev.properties

    form=git-dev-1.0

*config-test.properties

    form=git-test-1.0

### 启动应用看结果
可以通过如下几种方式获取配置信息，其中master为主分支名，如果有其它分支，只需要改为相应的分支名

http://localhost:7001/config/dev/master http://localhost:7001/config/master

http://localhost:7001/config/test/master http://localhost:7001/master/config.properties
可以看到
``` java
{
    name: "config",
    profiles: [
        "test"
    ],
    label: "master",
    version: "03994b44d95897ae55dcf6f240e71c8d6e26c064",
    state: null,
    propertySources: [
        {
            name: "https://github.com/nijigenCG/SpringCloud-Learn//config-repo/config-test.properties",
            source: {
            from: "git-test-1.0"
        }
        },
        {
            name: "https://github.com/nijigenCG/SpringCloud-Learn//config-repo/config.properties",
            source: {
            from: "git-default-1.0"
            }
        }
    ]
}
```
config-server从Git获取后，会在本质相应地储存一份文件，实际上通过git clone命令复制了一个副本，然后返回给微服务进行加载。这样可以防止git崩坏掉无法获取配置。
## 客户端配置映射
有了提供配置的配置中心，如何让客户端去获取配置
### 新建spring boot 项目
命名为config-client，引入web和config-client
### 新建测试类

建立common包，建立接口类
``` java
@RefreshScope
@RestController
//可以使用@Value绑定配置服务中心的from属性
//也可以使用Environment获取配置属性
public class TestController {
    @Autowired
    private Environment env;

    @RequestMapping("/from")
    public String from(){
        return env.getProperty("from");
    }
    @Value("${from}")
    private String from;
    @RequestMapping("/from2")
    public String getFrom(){
        return "版本信息"+from;
    }
}
```
### bootstrap.properties 一定新建一个新的

    #spring.application.name必须为配置中的xx-aa.properties的xx，否则会value注入会报错。
    spring.application.name=config
    #spring.cloud.config.profile必须为配置中的xx-aa.properties的aa
    spring.cloud.config.profile=test
    spring.cloud.config.label=master
    spring.cloud.config.uri=http://localhost:7001/

    server.port=7002
### 启动看结果

    http://localhost:7002/from2
    版本信息git-test-1.0 http://localhost:7002/from1
    git-test-1.0
### 服务端详细

### 基础架构

![架构图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Config/4-1.png)

包含如下几个要素：

远程Git仓库 Config Server
本地Git仓库：存放获取的Git配置本地 Server A、Server B
获取配置流程：

启动应用，根据bootstrap.properties中的{application应用名}、{profile环境名}、{label分支名}，向config Server发起请求 Config Server维护自己的Git仓库，查找相应配置信息，通过git clone下载到文件系统
Config Server创建ApplicationContext实例，从Git本地仓库加载配置文件，返回给用户端 客户端获取外部配置文件后加载到客户端的ApplicationContext实例中，优先加载外部，外部配置加载完毕后再加载本地配置

### Git仓库配置

Git的版本控制很好的与config服务进行融合，一个应用不同的部署实例可以轻松获取springcloudconfig的不同版本配置。
Git配置可以参照前面，同样也可以脱离Git比如；

    #使用本地仓库，脱离Git服务端，快速调试开发
    spring.cloud.config.server.git.url=file://${user.home}/config-repo

### 本地文件系统
不使用Git，需要设置

    spring.profiles.active=native
    
Config server会默认从应用的src/main/resource下搜索配置文件。
### 安全性
配置中心内容敏感，结合Spring Security可以实现
pom中引入security 在配置文件中指定用户名密码

    security.user.name=user
    security.user.password=f4s8f1w9fs
启动config-server，可以访问http://localhost:7001/config/test/master

会跳转到登陆页面

*启动客户端，客户端访问不到配置中心的配置，启动时就会报错

    spring.cloud.config.username=user
    spring.cloud.config.password=f4s8f1w9fs
做了用户名密码校验后可以
### 高可用配置中心
有两种模式可以实现

传统模式：

多个config服务器，将指向同一个Git仓库，客户端通过config服务器的负载均衡器获取配置 服务模式：
将config作为微服务纳入eureka
### config server改造
接下来，我们来看如何将config加入到eureka中
改造开始：

config server pom增加eureka application.properties配置添加

    eureka.client.serviceUrl.defaultZone=http://localhost:8765/eureka/
主类添加@EnableDiscoveryClient 启动eureka和config server，可以看到注册上了

### client改造
pom引入eureka bootstrap.properties中增加配置

    spring.application.name=config
    server.port=7002

    spring.cloud.config.discovery.enabled=true
    spring.cloud.config.discovery.service-id=CONFIG-SERVER
    spring.cloud.config.profile=test

    eureka.client.serviceUrl.defaultZone=http://localhost:8765/eureka/
主类添加@EnableDiscoveryClient 启动，按之前的访问/from、/from2接口，可以看到实现了
### 动态刷新配置
spring cloud config可以实现实时更新配置，我们接着来改造。我们刚才启动的项目，访问/from
可以获取到

    git-test-1.0

修改配种后Git推送出去，发现访问/from还是没有发生变化。

需要修改配置。在clien端开启动态刷新

*client中新增actuator监控模块
``` java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
重启即可！

我们通过对client Post请求localhost:7002/refresh然后再访问可看到更新完成

改功能可以同Git仓库的WebHook进行关联，当Git提交变化时，就向相应的阻止发送/refresh post请求。
问题来了，系统壮大后，维护配置刷新也会造成系统负担，而且容易犯错，如何解决复杂度？
我们需要SpringCloudBus来实现以消息总线的反思进行配置变更通知！
