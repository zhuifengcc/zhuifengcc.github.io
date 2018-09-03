---
title: 'Spring Cloud: Spring Cloud Stream'
date: 2018-08-22 00:06:41
tags:
categories: Spring Cloud 
---
消息队列，应用在spring cloud之中大概有两个。
一个是stream作为服务之间的通信MQ，一个是bus是关于广播配置的，都基于rabbit和kafka。
那么什么是消息队列，为什么需要消息队列？

简单的来说，打个比方，一个话多的女朋友给一个木讷的男朋友发信息。
女生发送信息非常快，巴拉巴拉的发，而男生不懂女生的心思，十多分钟才能弄明白女生的一条信息。
如果没有收件箱，要等男的看完一个，消息却不停涌入，男生会崩溃。
可以把我们的消息队列当作收件箱，男生处理完一条信息，再去都下一条信息。
## 举几个消息队列的应用实例
就是用来解耦，只要保证消息格式不变，消息的发送方和接收方并不需要彼此联系，也不需要受对方的影响，作为一个中间件的存在，接收和分发消息。应用场景举个例子如下

假设用户在你的软件中注册，服务端收到用户的注册请求后，它会做这些操作：

校验用户名等信息，如果没问题会在数据库中添加一个用户记录
如果是用邮箱注册会给你发送一封注册成功的邮件，
手机注册则会发送一条短信分析用户的个人信息，以便将来向他推荐一些志同道合的人，
或向那些人推荐他发送给用户一个包含操作指南的系统通知等等……
但是对于用户来说，注册功能实际只需要第一步，只要服务端将他的账户信息存到数据库中他便可以登录上去做他想做的事情了。至于其他的事情，非要在这一次请求中全部完成么？值得用户浪费时间等你处理这些对他来说无关紧要的事情么？所以实际当第一步做完后，服务端就可以把其他的操作放入对应的消息队列中然后马上返回用户结果，由消息队列异步的进行这些操作。

实际上很多地方需要：

    上述的异步处理，提升系统响应性能
    应用解耦，接收者可以随意增加，不影响发布者，消息发送成功不影响消息接收者
    最终一致性：先写消息再操作，操作完成后再修改状态（订单系统与库存系统的一致性）
    广播：只需要关心是否发送到消息队列，下游怎么接受是下游的事情
    流量削峰：上下游处理能力有差距时，承担一个缓冲地带的功能。（秒杀抢购，请求先存放在消息队列，等待服务器处理）
    日志：利用kafaka，解决大量日志传输；（获取用户id，浏览器，ip等信息，通过flume日志收集器进行收集，传到MQ之中——进行流计算并可视化呈现，或者是进行归档，离线计算 处理后存到mysql）
    通信：可以单纯作为点对点队列或者聊天室

我们之前讲到，SpringCloudConfig通过refresh执行，刷新客户端配置，微服务的客户端越来越多的时候，每个客户端都需要执行一下refresh，这样是不是很不合适，我们现需要消息代理，构建一个公用的消息主题让所有微服务连接上来，该主题的消息会被所有实例监听和消费，所以称为消息总线。

讲到这，消息队列和消息总线有什么区别？

消息总线是基于一个已经相当成熟的消息队列或者消息系统做二次封装

    消息队列clientAPI权限太大，clientAPI信任级别太高
    消息队列clientAPI面向技术，消息总线clientAPI面向技术+业务
    消息队列无法隐藏通信细节
    消息队列无法实施实时管控
    总线的优势：统一入口，简化拦截成本

各种消息队列：ActiveMQ、RabbitMQ、Kafka、RocketMQ
spring cloud bus 支持RabbitMQ、Kafka。

## RabbitMQ实现
AMQP（高级 消息 队列 协议）的实现，面向消息的中间件，性能高，由Erlang编写

### 基本概念

    Broker：消息队列服务器实体，接受消息发送者的笑意，然后发送给接收者或者其他Broker
    Exchange：消息交换机，消息第一个到达地方，消息通过他指定的路由规则进行分发
    Queue：消息发送通过路由后最终到达地，进去队列等待消费
    Binding：将Exchange和Queue按路由规则绑定起来
    Channal：消息通道，用于连接发送接受消息的逻辑结构，每个Channal代表一个会话任务

消息队列过程：

    客户端连接到消息队列服务器，打开一个Channal
    客户端声明Exchange
    客户端声明Queue
    客户端Binding Exchange和Queue
    发送消息
    Exchange根据Binding进行消息路由，将消息投递到一个或多个Queue。echange有几种类型：Direct、topic、fanout

RabbitMQ支持消息持久化，包括Exchange持久化、Queue持久化、消息持久化

### 安装配置
首先我们需要下载RabbitMQ和Erlang，然后安装他们。 rabbitmq自带管理后台，安装后需要配置开启

    rabbitmq-plugins enable rabbitmq_management

*重启rabbitmq服务

    rabbitmq-server stop
    rabbitmq-server start

进入管理页面 http://localhost:15672 guest guest 进入Admin选项卡，创建一个springcloud用户，tags是Rabbit角色分类，给管理员角色

virtual hosts管理，相当于数据库，可以通过右侧virtual host进入进行添加，以/ 开头，可以进行用户授权

进入管理界面，可以进行set permission操作 Web 控制台会展示很多 RabbitMQ 信息，但最最重要的就一个：Unacked Message。这个数据会直接显示在登录之后的 Overview 标签中，第一眼就能看到。Unacked Message 指的是还没有被处理的消息。正常情况下，这个值应该为 0。如果这个值不是 0，并且持续增长，那你就得注意了，这意味着 RabbitMQ 出现了问题，队列开始积压，消息开始堆积，是一个严重的信号.

### quick start
Spring Boot整合RabbitMQ，实现一个最小化的发送接受消息的例子。

1) 新建rabbitmq-hello工程，pom引入spring-boot-starter-amqp

2）三个类Config、send、reciever

``` java
@Configuration
public class RabbitConfig {
    @Bean
    public Queue Queue() {
        return new Queue("hello");
    }
}
```

``` java
@Component
public class HelloSender {
    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send() {
        String context = "hello " + new Date();
        System.out.println("Sender : " + context);
        this.rabbitTemplate.convertAndSend("hello", context);
    }
}
```

``` java 
@Component
@RabbitListener(queues = "hello")
public class HelloReceiver {
    @RabbitHandler
    public void process(String hello) {
        System.out.println("Receiver  : " + hello);
    }
}
```
3）配置
``` java
spring.application.name=spirng-boot-rabbitmq
spring.rabbitmq.host=localhost
#连接rabbit的端口为5672.15672是管理页面的端口
spring.rabbitmq.port=5672
spring.rabbitmq.username=springcloud
spring.rabbitmq.password=***
```
4）启动主类，连接上

5) 编写测试类
``` java
@RunWith(SpringRunner.class)
@SpringBootTest
public class RabbitMqHelloTest {
    @Autowired
    private HelloSender helloSender;

    @Test
    public void hello() throws Exception {
        helloSender.send();
    }
}
```
运行测试类，可看到发送和接受消息

### 原生实现实例
#### 1、rabbit6大之simple queues
以下均在amqp项目中测试
P ———— queue ———— C
我们用java来实现一个简单队列，有一个发送消息和一个消费消息的，来发送一个hello world
1）建立一个maven项目，引入依赖 rabbitmq client
2）获取Mq连接工具类

``` java
public class ConnectionUtils {
public static Connection getCon() throws IOException, TimeoutException {
        //建立连接工厂及连接属性（继承项目会做配置的，这里直接写在代码里）
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        factory.setPort(5672);
        factory.setVirtualHost("/vh");
        factory.setUsername("springcloud");
        factory.setPassword("123456");
        //获取连接--创建channel
        Connection connection = factory.newConnection();
        return connection;
    }
}
```
3）Provider
这里做一个循环发送消息的测试，运行send，可以看到消息队列运行的情况；同时可以进入Queues页签，点击Get Messages，可以获取message信息

``` java
import Utils.ConnectionUtils;
import com.rabbitmq.client.Channel;
import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class Send {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws Exception {
        sendmess();
    }
    public static  void sendmess() throws IOException, TimeoutException, InterruptedException {
        Channel channel=ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        String message = "Hello CG!";
        for(int i=1;i<1000;i++){
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            Thread.sleep(500);
            System.out.println(" [x] Sent '" + message + "{}'"+i);
        }
    }
}
```
4）Consumer
``` java
public class Consumer {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws Exception {
        Channel channel = ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        //一旦又消息进入队列，就会触发消费者handleDelivery方法
        DefaultConsumer consumer=new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg=new String(body,"utf-8");
                System.out.println(msg);
            }
        };
        //监听队列，第二个参数是autoack自动应答，返回一个到达的信息，表示已经消费掉了消息，如果不加
        //消息会继续存在再消息队列中造成积压
        channel.basicConsume(QUEUE_NAME,true,consumer);
    }
}
```
#### 2、rabbit6大之Work queues
P ———— queue ————|——— C1
P ———— queue ————|——— C2
P ———— queue ————|——— C3
P ———— queue ————|——— C4

上面的只有一个消费者，且消费者生产者耦合再一起，多个消费者无法消费，如果队列名变更了，是不是P/C都要变更
1)工作队列-轮询分发
接上面一个，我们来一个发送者，两个接收者，一个接收者线程100ms中断一个200ms中断，看看消费情况
``` java
public class Send {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws Exception {
        sendmess();
    }
    public static  void sendmess() throws IOException, TimeoutException, InterruptedException {
        Channel channel=ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        for(int i=1;i<100;i++){
            String message = "Hello CG!";
            message+=i;
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println(" [x] Sent '" + message + "{}'"+i);
        }
    }
}
```
``` java
public class Consumer1 {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws Exception {
        Channel channel = ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        //一旦又消息进入队列，就会触发消费者handleDelivery方法
        DefaultConsumer consumer=new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //super.handleDelivery(consumerTag, envelope, properties, body);
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                String msg=new String(body,"utf-8");
                System.out.println(msg);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,true,consumer);
    }
}
```
``` java
public class Consumer2 {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws Exception {
        Channel channel = ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        //一旦又消息进入队列，就会触发消费者handleDelivery方法
        DefaultConsumer consumer=new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //super.handleDelivery(consumerTag, envelope, properties, body);
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                String msg=new String(body,"utf-8");
                System.out.println(msg);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,true,consumer);
    }
}
```
可以看到消费端被均匀分发，无论消费端的处理能力，消息被均分了

#### 2)工作队列-公平分发
rabbit将发送一条给消费者，等待消费者处理应答，处理完成再发下一条，按照处理能力发送了
使用此模式，acsk自动应答必须改为手动
``` java
public class Send {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws Exception {
        sendmess();
    }
    public static  void sendmess() throws IOException, TimeoutException, InterruptedException {
        Channel channel=ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        //告知队列,每次只发一条，消费者处理确认前不再发送
        channel.basicQos(1);
        for(int i=1;i<1000;i++){
            String message = "Hello CG!";
            message+=i;
            channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
            System.out.println(" [x] Sent '" + message + "{}'"+i);
        }
    }
}
```
``` java
public class Consumer1 {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws Exception {
        final Channel channel = ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.basicQos(1);
        //一旦又消息进入队列，就会触发消费者handleDelivery方法
        DefaultConsumer consumer=new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //super.handleDelivery(consumerTag, envelope, properties, body);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    String msg=new String(body,"utf-8");
                    System.out.println(msg);
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //监听队列,关闭自动应答
        channel.basicConsume(QUEUE_NAME,false,consumer);
    }
}
```
``` java
public class Consumer2 {
    private final static String QUEUE_NAME = "hello";
    public static void main(String[] args) throws Exception {
        final Channel channel = ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        channel.basicQos(1);
        //一旦又消息进入队列，就会触发消费者handleDelivery方法
        DefaultConsumer consumer=new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                //super.handleDelivery(consumerTag, envelope, properties, body);
                try {
                    Thread.sleep(20);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }finally {
                    String msg=new String(body,"utf-8");
                    System.out.println("consumer2"+msg);
                    channel.basicAck(envelope.getDeliveryTag(),false);
                }
            }
        };
        //监听队列,关闭自动应答
        channel.basicConsume(QUEUE_NAME,false,consumer);
    }
}
```
可以看到控制台打印的信息，两个消费者不再平均分配任务了

3)自动应答与持久化
上面的自动应答设置可以改写如下，清晰点。
``` java
boolean autoAsk=false;
//监听队列,关闭自动应答
channel.basicConsume(QUEUE_NAME,autoAsk,consumer);
```
（自动应答true，自动确认模式）自动应答的含义：消息投递到消费者后，消息从内存中删掉。（这种情况下可能会丢失处理中的消息！消息队列发出消息后，删除消息，但是消费者处理时发生崩溃，消息丢失）

（自动应答false，手动模式）一个消费者挂了，超时未收到回执，则将消息发送给另一个消费者，消费者处理完毕后，发送成功回执，消息队列删除消息。但是加入我们消息队列服务器挂了，我们的消息仍然丢失，这时候我们需要消息持久化

rabbitMQ消息持久化：
``` java
//是否持久化
boolean duravle=false;
channel.queueDeclare(QUEUE_NAME, duravle, false, false, null);
```
将程序中的false改为true，无法生效还会报错，因为消息队列channel只能声明一次，已经定义好了，（可以从控制台删了再次声明，或者新建一个channel）

#### 3、rabbit6大之Publish/Subscribe
之前的都是被一个消费者消费，发送一条，被消费者争抢。

现在我们需要多个消费者，发送一条消息，让所有消费者都能消费。可以采用订阅模式

    work队列：一个队列对应多个消费者
    P ———— queue ——— | ——— C1
    P ———— queue ——— | ——— C2
    P ———— queue ——— | ——— C3
    P ———— queue ——— | ——— C4
    订阅模式：引入交换机，需要将队列绑定到交换机。一个交换机对应多个队列，一个队列对应一个消费者
    P ——x—— |—queue1 ——— |— C1
    P ——x—— |—queue2 ——— |— C2
    P ——x—— |—queue3 ——— |— C3
    P ——x—— |—queue4 ——— |— C4
应用场景：注册完成—发送邮件与短信；或者修改商品—更改搜索引擎，同时修改库存，前台信息
``` java
public class Send {
private final static String EXCHANGE_NAME = "test_exchange";
    public static void main(String[] args) throws Exception {
        sendmess();
    }
    public static  void sendmess() throws IOException, TimeoutException, InterruptedException {
        Channel channel=ConnectionUtils.getChan();
        //设置分发类型为fanout
        channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
        //发送消息
        String msg="hello ps";
        //发送到exchange
        channel.basicPublish(EXCHANGE_NAME,"",null,msg.getBytes());
        channel.close();
    }
}
```
通过以上代码，我们可以进入控制台exhcange页签，看到test_exchange，但是消息已经丢失了。
因为rabbitMQ只有队列又存储能力，而没有队列绑定到exchange，所以数据丢失了。
我们需要添加消费者，通过队列绑定exchange，接受消息
``` java
public class Consumer1 {
private final static String QUEUE_NAME = "test_queue_fanout_email";
private final static String EXCHANGE_NAME = "test_exchange";

public static void main(String[] args) throws Exception {
    Channel channel = ConnectionUtils.getChan();
    channel.queueDeclare(QUEUE_NAME,false,false,false,null);
    //绑定到exhcnage
    channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"");
    //消费消息
    DefaultConsumer consumer=new DefaultConsumer(channel){
        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
            String msg=new String(body,"utf-8");
            System.out.println(msg);
        }
    };
    //监听队列
    channel.basicConsume(QUEUE_NAME,true,consumer);
    }
}
```
``` java
public class Consumer1 {
private final static String QUEUE_NAME = "test_queue_fanout_sms";
private final static String EXCHANGE_NAME = "test_exchange";

public static void main(String[] args) throws Exception {
    Channel channel = ConnectionUtils.getChan();
    channel.queueDeclare(QUEUE_NAME,false,false,false,null);
    //绑定到exhcnage
    channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,"");
    //消费消息
    DefaultConsumer consumer=new DefaultConsumer(channel){
        @Override
        public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
            String msg=new String(body,"utf-8");
            System.out.println(msg);
        }
    };
    //监听队列
    channel.basicConsume(QUEUE_NAME,true,consumer);
    }
}
```
运行程序，可以看到均受到了消息。
#### 4、rabbit6大之Routing
消息增加一个key，只把消息转到相应key的队列。如下，key=”error”,会路由到所有队列；key=”info”，只路由到queue2

    p—x(type=direct)—|—error—|—-queue1 —- c1
    p—x(type=direct)—|–info —-|↘
    p—x(type=direct)—|–error —|→ –queue2— c2
    p—x(type=direct)—|–warning–|↗
exchange：
1.匿名转发(之前演示的)
``` java
//声明交换机 fanout不处理路由键，只需要将队列绑定到交换机，和exchange绑定的队列均能收到消息
channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
channel.basicPublish(EXCHANGE_NAME,"",null,msg.getBytes());
```
2.处理路由键
``` java
//声明交换机 处理路由键 Direct，消息增加一个key，只把消息转到相应key的队列
channel.exchangeDeclare(EXCHANGE_NAME,"fanout");
channel.basicPublish(EXCHANGE_NAME,"",null,msg.getBytes());
```
``` java
public class Send {
    private final static String EXCHANGE_NAME = "test_exchange_direct";
    public static void main(String[] args) throws Exception {
        sendmess();
    }
    public static  void sendmess() throws IOException, TimeoutException, InterruptedException {
        Channel channel= ConnectionUtils.getChan();
        //设置分发类型为fanout
        channel.exchangeDeclare(EXCHANGE_NAME,"direct");
        //发送消息
        String msg="hello direct";
        String routingKey="error";
        channel.basicPublish(EXCHANGE_NAME,routingKey,null,msg.getBytes());
        channel.close();
        }
    }
```
``` java
public class Consumer1 {
    private final static String QUEUE_NAME = "test_queue_direct_1";
    private final static String EXCHANGE_NAME = "test_exchange_direct";

    public static void main(String[] args) throws Exception {
        Channel channel = ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.basicQos(1);
        //绑定到exhcnage,并设置routingKey
        String routingKey="error";
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,routingKey);
        //消费消息
        DefaultConsumer consumer=new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg=new String(body,"utf-8");
                System.out.println(""+msg);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,true,consumer);
    }
}
```
#### 5、rabbit6大之Topics
Topics:将路由和某个模式匹配，用符号来表示 #匹配一个或多个；*匹配一个

    p—x(type=topic)— |—.orange.—|—-queue1 —- c1
    p—x(type=topic)— |–..rabbit —-|↘
    p—x(type=topic)— |–lazy.# —|→ –queue2— c2
    p—x(type=topic)— |–warning–|↗

![topic模式示意图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Stream/6-3.png)

生产者发布 删除 修改 查询 商品

c1 只收某一个发布

c2 收所有发布
``` java
public class Send {
    private final static String EXCHANGE_NAME = "test_exchange_topic";

    public static void main(String[] args) throws Exception {
        sendmess();
    }

    public static  void sendmess() throws IOException, TimeoutException, InterruptedException {

        Channel channel= ConnectionUtils.getChan();
        //设置分发类型为fanout
        channel.exchangeDeclare(EXCHANGE_NAME,"topic");
        //发送消息
        String msg="添加 iphone x";
        String routingKey="goods.add";
        channel.basicPublish(EXCHANGE_NAME,routingKey,null,msg.getBytes());
        System.out.println("发送了"+msg);
        channel.close();
    }
}
```

``` java
public class Consumer1 {
    private final static String QUEUE_NAME = "test_queue_topic_1";
    private final static String EXCHANGE_NAME = "test_exchange_topic";

    public static void main(String[] args) throws Exception {
        Channel channel = ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.basicQos(1);
        //绑定到exhcnage,并设置routingKey
        //消费者1只能拿添加的消息
        String routingKey="goods.add";
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,routingKey);
        //消费消息
        DefaultConsumer consumer=new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg=new String(body,"utf-8");
                System.out.println(""+msg);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,true,consumer);
    }
}
```

``` java
public class Consumer2 {
    private final static String QUEUE_NAME = "test_queue_topic_2";
    private final static String EXCHANGE_NAME = "test_exchange_topic";

    public static void main(String[] args) throws Exception {
        Channel channel = ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.basicQos(1);
        //绑定到exhcnage,并设置routingKey
        //消费者2拿所有商品操作消息
        String routingKey="goods.#";
        channel.queueBind(QUEUE_NAME,EXCHANGE_NAME,routingKey);
        //消费消息
        DefaultConsumer consumer=new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg=new String(body,"utf-8");
                System.out.println(""+msg);
            }
        };
        //监听队列
        channel.basicConsume(QUEUE_NAME,true,consumer);
    }
}
```
#### 6、rabbit6大之Rpc
之前几个例子都是再本地执行。RPC是指远程过程调用，也就是说两台服务器A，B，一个应用部署在A服务器上，想要调用B服务器上应用提供的函数/方法，由于不在一个内存空间，不能直接调用，需要通过网络来表达调用的语义和传达调用的数据。
其实RPC的方式就是springcloud使用rabbit的方式，前面SpringCloudConfig接受git flow的refresh消息实现动态刷新即是一个示例，不过我们用的封装号的AmqpTemplate实现而已.

rabbitMQ消息确认机制
rabbitMQ可以通过持久化解决服务器异常造成的数据丢失问题。而生产者发送消息后，消息到底有没有到达服务器，默认情况是不知道的。要知道消息到达服务器一般有如下两种方式

AMQP实现了事务机制 Confirm模式

1）AMQP实现了事务机制（类似于数据库）

*txSelect    用户将当前channel设置成transation模式
*txCommit    提交
*rxRollback  回滚

示例代码
``` java
public class TxSend {
    private final static String QUEUE_NAME = "queue_tx";
    public static void main(String[] args) throws IOException, TimeoutException {
        Channel channel= ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        String msg="hello tx message";
        try{
            //开启事务
            channel.txSelect();
            channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
            //模拟抛出异常
            int x=1/0;
            channel.txCommit();
        }catch (Exception e){
            channel.txRollback();
            System.out.println("fail to send! rollback");
        }
        channel.close();
    }
}
```
``` java
public class Consumer1 {
    private final static String QUEUE_NAME = "queue_tx";
    public static void main(String[] args) throws IOException, TimeoutException {
        Channel channel= ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.basicConsume(QUEUE_NAME,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg=new String(body,"utf-8");
                System.out.println(""+msg);
            }
        });
    }
}
```
2）Confirm模式

上面的事务模式发送消息，去的消息的消息确认机制。走协议路径，其实大大降低了消息吞吐量！
confirm模式则不会。
实现原理

    生产者将channel设置未comfirm模式，所有的消息都有唯一的id，一旦消息投递到所有匹配到队列后，rabbit会回执一条消息，使得生产者知道消息发送成功

comfirm模式最大的优势就是异步处理！不像AMQP走串行模式

编程模式：
*普通 发送一条 waitForConfirms()
``` java
public class Send {
    private final static String QUEUE_NAME = "queue_confirm_1";
    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        Channel channel= ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将channel设为confirm模式
        channel.confirmSelect();
        String msg="confirm msg";
        channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
        //如果没收到confirm，说明发送失败
        if(!channel.waitForConfirms()){
            System.out.println("message send failed!");
        }else{
            System.out.println("success!");
        }
        channel.close();
    }
}
```
``` java 
public class Consumer1 {
    private final static String QUEUE_NAME = "queue_confirm_1";
    public static void main(String[] args) throws Exception {
        Channel channel= ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        channel.basicConsume(QUEUE_NAME,true,new DefaultConsumer(channel){
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String msg=new String(body,"utf-8");
                System.out.println(""+msg);
            }
        });
    }
}
```
*批量 发送一批 waitForConfirms

如果丢失一条，整批都失败，需要重发

``` java
public class Send {
    private final static String QUEUE_NAME = "queue_confirm_1";
    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        Channel channel= ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);
        //将channel设为confirm模式
        channel.confirmSelect();
        String msg="confirm msg";
        for(int i =0 ;i<100;i++){
            channel.basicPublish("",QUEUE_NAME,null,msg.getBytes());
        }
        //如果没收到confirm，说明发送失败
        if(!channel.waitForConfirms()){
            System.out.println("message send failed!");
        }else{
            System.out.println("success!");
        }
        channel.close();
    }
}
```
*异步 提供回调,进行监听

channel对象提供ConfirmListener()回调方法只包含deliverTag（消息的序列号）,我们需要为每个channel维护一个unconfirm的消息序列号集合，每publish一个数据，集合元素增加1，每会跳一次handleAck方法，unconfirm集合删掉相应的一条或多条记录。
``` java
public class AsyncSend {
    private final static String QUEUE_NAME = "queue_confirm_3";
    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        Channel channel= ConnectionUtils.getChan();
        channel.queueDeclare(QUEUE_NAME,false,false,false,null);

        //将channel设为confirm模式
        channel.confirmSelect();
        //存放未确认的消息
        final SortedSet<Long> confirmSet= Collections.synchronizedSortedSet(new TreeSet<Long>());
        channel.addConfirmListener(new ConfirmListener() {
            //handNack 回执有问题
            public void handleNack(long deliveryTag, boolean multiple) throws IOException {
                if(multiple){
                    System.out.println("---handleNack---multiple");
                    confirmSet.headSet(deliveryTag+1).clear();
                }else {
                    System.out.println("---handleNack---multiple false");
                    confirmSet.remove(deliveryTag);
                }
            }
            //没有问题的handleAck
            public void handleAck(long deliveryTag, boolean multiple) throws IOException {
                if(multiple){
                    System.out.println("---handleAack---multiple");
                    confirmSet.headSet(deliveryTag+1).clear();
                }else {
                    System.out.println("---handleAack---multiple false");
                    confirmSet.remove(deliveryTag);
                }
            }
        });
        String msg="confirm msg";
        while (true){
            long seqNo=channel.getNextPublishSeqNo();
            confirmSet.add(seqNo);
        }
    }
}
```
## 整合到Spring Cloud里的用法
之前讲了rabbit的原生用法，现在我们要结合spring cloud Stream使用
### Qucik Start
基于springboot 构建一个微服务，使用消息中间件rabbitMQ接受消息并打印到日志。

1）建立stream-hello工程

2）pom导入
``` java
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
    <version>2.0.0.RELEASE</version>
</dependency>
```
3)建立消息接受类
``` java
@EnableBinding(Sink.class)
public class SkinReceiver {
    @StreamListener(Sink.INPUT)
    public void receive(Object payload){
        System.out.println("recieved"+payload);
    }
}
```
4)主类不变，然后启动。不需要做任何配置！

可以看到控制台的Connections多了一条，我们通过Queues里的发布消息来发布一条，
可以看到程序控制台打印了

    recieved[B@6a001153

我们来看一下是如何实现的。

1.spring boot应用引入stream-rabbit的依赖，是stream对rabbitMQ的支持的封装，包含了对rabbitMQ的自动配置等内容；等价于stream-binder-rabbit

2.Stream的核心注解被定义在SinkReciever中

    1.@EnableBinding用来注解一个或多个定义了@Input @Output的接口。我们绑定了Sink接口，这个接口是spring cloud stream默认实现输入消息的绑定通道
    含义就是这个类绑定了一个input通道
    2.@StreamListener
    通过该注解将receive方法注册为input通道的监听处理器，在rabbit发布消息的时候，receive会做出相应的相应动作

#### 核心概念
Spring Cloud Stream结构图。

![Stream结构图](https://raw.githubusercontent.com/wiki/zhuifengcc/zhuifengcc.github.io/images/Spring Cloud/Spring Cloud Stream/6-4.png)
Stream构建的应用程序与消息中间件是通过Binder关联起来，实现中间件与应用程序的解耦。对于每一个Stream来说，只需要知道Binder对应用程序提供的抽象概念Channel来使用消息中间件来实现逻辑即可

##### Binder
各个消息中间件差异较大，如果没有Binder这一层，消息交互将变得很笨重，对具体中间件实现细节依赖严重。有了Binder，只需要暴露统一的Channel通道，更换消息队列也只需要更换对应的Binder即可。

而且快速入门的时候，并没有做任何rabbit配置。秉承了spring boot的理念，对rabbit做了自动化配置。当然我们也可以修改

    spring.cloud.stream.binding.input.destination=raw-sensor-data
    spring.rabbitmq.host=localhost
    spring.rabbitmq.port=5672
    spring.rabbitmq.username=cg
    spring.rabbitmq.password=123456
##### 发布订阅
stream消息通信遵循了发布订阅。一条消息投递到中间件后，会通过通向topic主题进行广播，订阅者收到它并触发自身的业务逻辑进行处理。不同的消息中间件topic对应不用的概念

rabbitmq中topic对应exchange网关

kafka中topic对应topic

1）quick start中，我们通过channel发布消息给应用程序。

2）stream应用启动时候，@EnableBinding(Skin.class)

在rabbitmq的exchange中也创建了一个名为input的exchange交换器，我们可以在控制台看到

3）我们通过不同端口号启动两个示例程序，可以看到exchange中input里绑定了两个channel

这种模式就是之前的那个rabbit6大模式之中的topic模式，消息通过exchange，exchange上绑定了多个不同的队列，一个队列对应一个接受方
##### Consumer Group
微服务架构中，实际上每一个微服务应用为了高可用和负载均衡，均会部署多个实例。

这种情况下，消息生产者发送消息给某个具体微服务实例时，只希望被消费一次，但是像刚才那种启动两个实例，消息会被消费两次。这时候我们需要用到消费组这个概念。

如果一个服务有多个实例的时候，我们可以通过配置

    spring.cloud.stream.bindings.input.group

为其制定一个组名。只有一个成员真正接收消息并处理，而不会出现多次处理的情况。

实际应用最好给服务指定一个消费组，防止消费重复处理！除非那种全局都需要处理的情况，比如刷新配置

消息分区

通过消费组，可以使消息只被一个实例处理，但是无法控制被哪个实例使用。情况就是，同一个消息，多次到达后可能被不同的机器消费处理。这种情况对与一些需要事务控制或者用于持续报告的监控服务时，会将数据分散到各个节点而导致不一致性，这种情况需要进行消息分区。

这种分区通过stream提供的通用的抽象实现，所有对于消息中间件是否支持分区不关心。

通过配置

1）消费者配置

    spring.cloud.stream.bindings.input.destination 开启分区
    spring.cloud.stream.instanceConut=2 指定当前消费组实例数量
    spring.cloud.stream.instanceInedex 设置当前实例索引号
2）生产消息也需要配置相应修改

    spring.cloud.stream.bindings.output.destination 开启分区
    spring.cloud.stream.bindings.output.producer.partitionKeyExpression=payload
    spring.cloud.stream.bindings.output.producer.partitionCount=2

### kafaka 安装配置
我们再cent 7上来进行

1需要jdk，在这使用openjdk

    rpm -qa |grep java
    rpm -qa |grep jdk
    rpm -qa |grep gcj
    如果没有输入信息表示没有安装。
    如果安装可以使用rpm -qa | grep java | xargs rpm -e --nodeps 批量卸载所有带有Java的文件  这句命令的关键字是java
    首先检索包含java的列表
    yum list java*
    检索1.8的列表
    yum list java-1.8*
    安装1.8.0的所有文件
    yum install java-1.8.0-openjdk* -y
    使用命令检查是否安装成功
    java -version
2.下载kafka

*解压

    tar -zxvf kafka_2.10-0.10.0.1.tgz
*移动到指定目录

    mv kafka_2.10-0.10.0.1 /usr/local/kafka
*启动zookeeper

    /usr/local/kafka/bin/zookeeper-server-start.sh -daemon /usr/local/kafka/config/zookeeper.properties
*启动kakfka

    /usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties
*创建topic

    /usr/local/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
*产生消息

    /usr/local/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test

    hello kafka！
*消费消息

    /usr/local/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning