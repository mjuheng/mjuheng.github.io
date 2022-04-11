---
layout: post
title: SpringBoot集成RabbitMQ
categories: [SpringBoot]
description: SpringBoot集成RabbitMQ
keywords: SpringBoot, RabbitMQ
---

SpringBoot集成RabbitMQ

# 概念
![](/images/posts/springboot/rabbitmq-base.png)
常用的交换机有以下三种，因为消费者是从队列获取信息的，队列是绑定交换机的（一般），所以对应的消息推送/接收模式也会有以下几种：

Direct Exchange 

直连型交换机，根据消息携带的路由键将消息投递给对应队列。

大致流程，有一个队列绑定到一个直连交换机上，同时赋予一个路由键 routing key 。
然后当一个消息携带着路由值为X，这个消息通过生产者发送给交换机时，交换机就会根据这个路由值X去寻找绑定值也是X的队列。

Fanout Exchange

扇型交换机，这个交换机没有路由键概念，就算你绑了路由键也是无视的。 这个交换机在接收到消息后，会直接转发到绑定到它上面的所有队列。

Topic Exchange

主题交换机，这个交换机其实跟直连交换机流程差不多，但是它的特点就是在它的路由键和绑定键之间是有规则的。
简单地介绍下规则：

\*  (星号) 用来表示一个单词 (必须出现的)

\#  (井号) 用来表示任意数量（零个或多个）单词
通配的绑定键是跟队列进行绑定的，举个小例子
队列Q1 绑定键为 *.TT.*          队列Q2绑定键为  TT.#
如果一条消息携带的路由键为 A.TT.B，那么队列Q1将会收到；
如果一条消息携带的路由键为TT.AA.BB，那么队列Q2将会收到；

# 环境配置搭建
### 引入jar包
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 编写application.yml文件
```yaml
spring:
  #配置rabbitMq 服务器
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: root
    password: root
```

# 交换机模型使用样例
## 直连交换机
### 生产者
创建DirectRabbitConfig.java类
```java
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DirectRabbitConfig {

    // 创建队列TestDirectQueue
    @Bean
    public Queue testDirectQueue() {
        // durable:是否持久化,默认是false,持久化队列：会被存储在磁盘上，当消息代理重启时仍然存在，暂存队列：当前连接有效
        // exclusive:默认也是false，只能被当前创建的连接使用，而且当连接关闭后队列即被删除。此参考优先级高于durable
        // autoDelete:是否自动删除，当没有生产者或者消费者使用此队列，该队列会自动删除。
        // return new Queue("TestDirectQueue",true,true,false);

        // 一般设置一下队列的持久化就好,其余两个就是默认false
        return new Queue("TestDirectQueue", true);
    }

    // 创建Direct交换机TestDirectExchange
    @Bean
    DirectExchange testDirectExchange() {
        //  return new DirectExchange("TestDirectExchange",true,true);
        return new DirectExchange("TestDirectExchange", true, false);
    }

    // 将队列和交换机绑定, 并设置用于匹配键：TestDirectRouting
    @Bean
    Binding bindingDirect() {
        return BindingBuilder.bind(testDirectQueue()).to(testDirectExchange()).with("TestDirectRouting");
    }
}
```
创建消息发送接口
```java
// 使用RabbitTemplate,这提供了接收/发送等等方法
@Autowired
RabbitTemplate rabbitTemplate;

/**
 * 向直连型交换机发送消息
 *
 * @return 处理结果
 */
@GetMapping("/sendDirectMessage")
public String sendDirectMessage() {
    String messageId = String.valueOf(UUID.randomUUID());
    String messageData = "test message, hello!";
    String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    Map<String, Object> map = new HashMap<>();
    map.put("messageId", messageId);
    map.put("messageData", messageData);
    map.put("createTime", createTime);
    //将消息携带绑定键值：TestDirectRouting 发送到交换机TestDirectExchange
    rabbitTemplate.convertAndSend("TestDirectExchange", "TestDirectRouting", JSON.toJSONString(map));
    return "ok";
}
```
### 消费者
消费方不在意交换机的类型，只监听指定队列，所以后续的其它交换机模型，都可以使用该消费者方法
```java
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

@Component
public class DirectReceiver {

    @RabbitListener(queues = "TestDirectQueue")
    public void process(String testMessage, Message message, Channel channel) {
        System.out.println("DirectReceiver消费者收到消息  : " + testMessage);
    }
}
```
## 主题交换机
```java
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class TopicRabbitConfig {
    // 绑定键
    public final static String part = "topic.part";
    public final static String all = "topic.all";

    @Bean
    public Queue firstQueue() {
        return new Queue(TopicRabbitConfig.part);
    }

    @Bean
    public Queue secondQueue() {
        return new Queue(TopicRabbitConfig.all);
    }

    @Bean
    TopicExchange exchange() {
        return new TopicExchange("topicExchange");
    }


    // 将firstQueue和topicExchange绑定,而且绑定的键值为topic.man
    // 这样只要是消息携带的路由键是topic.man,才会分发到该队列
    @Bean
    Binding bindingExchangeMessage() {
        return BindingBuilder.bind(firstQueue()).to(exchange()).with(part);
    }

    // 将secondQueue和topicExchange绑定,而且绑定的键值为用上通配路由键规则topic.#
    // 这样只要是消息携带的路由键是以topic.开头,都会分发到该队列
    @Bean
    Binding bindingExchangeMessage2() {
        return BindingBuilder.bind(secondQueue()).to(exchange()).with("topic.#");
    }

}
```
消息发送接口，routingKey选择topic.part时两个队列都会接收到数据，topic.all时只有topic.all队列接收数据
```java
/**
 * 发送topic part消息
 *
 * @return
 */
@GetMapping("/sendTopicMessagePart")
public String sendTopicMessagePart() {
    String messageId = String.valueOf(UUID.randomUUID());
    String messageData = "message: M A N ";
    String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    Map<String, Object> manMap = new HashMap<>();
    manMap.put("messageId", messageId);
    manMap.put("messageData", messageData);
    manMap.put("createTime", createTime);
    rabbitTemplate.convertAndSend("topicExchange", "topic.part", JSON.toJSONString(manMap));
    return "ok";
}
```

## 扇型交换机
```java
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.FanoutExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class FanoutRabbitConfig {

    /**
     *  创建三个队列 ：fanout.A   fanout.B  fanout.C
     *  将三个队列都绑定在交换机 fanoutExchange 上
     *  因为是扇型交换机, 路由键无需配置,配置也不起作用
     */
    @Bean
    public Queue queueA() {
        return new Queue("fanout.A");
    }

    @Bean
    public Queue queueB() {
        return new Queue("fanout.B");
    }

    @Bean
    public Queue queueC() {
        return new Queue("fanout.C");
    }

    @Bean
    FanoutExchange fanoutExchange() {
        return new FanoutExchange("fanoutExchange");
    }

    @Bean
    Binding bindingExchangeA() {
        return BindingBuilder.bind(queueA()).to(fanoutExchange());
    }

    @Bean
    Binding bindingExchangeB() {
        return BindingBuilder.bind(queueB()).to(fanoutExchange());
    }

    @Bean
    Binding bindingExchangeC() {
        return BindingBuilder.bind(queueC()).to(fanoutExchange());
    }
}
```
生产方（routingKey无需指定，该交换机会将数据发送至绑定的全部队列）
```java
/**
 * 发送扇型交换机消息
 *
 * @return
 */
@GetMapping("/sendFanoutMessage")
public String sendFanoutMessage() {
    String messageId = String.valueOf(UUID.randomUUID());
    String messageData = "message: testFanoutMessage ";
    String createTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
    Map<String, Object> map = new HashMap<>();
    map.put("messageId", messageId);
    map.put("messageData", messageData);
    map.put("createTime", createTime);
    rabbitTemplate.convertAndSend("fanoutExchange", null, JSON.toJSONString(map));
    return "ok";
}
```

# 消息回调
在消息发送至交换机和队列时，添加结果回调
### 添加配置信息
```yaml
spring:
  rabbitmq:
    # 开启发送确认
    publisher-confirm-type: correlated
    # 开启发送失败退回
    publisher-returns: true
```
### 添加配置类，回调执行方法
```java

import org.springframework.amqp.rabbit.connection.ConnectionFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitConfig {


    @Bean
    public RabbitTemplate createRabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate();
        rabbitTemplate.setConnectionFactory(connectionFactory);
        // 设置开启Mandatory,才能触发回调函数,无论消息推送结果怎么样都强制调用回调函数
        rabbitTemplate.setMandatory(true);

        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            System.out.println("ConfirmCallback:     " + "相关数据：" + correlationData);
            System.out.println("ConfirmCallback:     " + "确认情况：" + ack);
            System.out.println("ConfirmCallback:     " + "原因：" + cause);
        });

        rabbitTemplate.setReturnsCallback(returnedMessage -> {
            System.out.println("ReturnCallback:     " + "消息：" + returnedMessage.getMessage());
            System.out.println("ReturnCallback:     " + "回应码：" + returnedMessage.getReplyCode());
            System.out.println("ReturnCallback:     " + "回应信息：" + returnedMessage.getReplyText());
            System.out.println("ReturnCallback:     " + "交换机：" + returnedMessage.getExchange());
            System.out.println("ReturnCallback:     " + "路由键：" + returnedMessage.getRoutingKey());
        });
        return rabbitTemplate;
    }

}
```
从总体的情况分析，推送消息存在四种情况：
①消息推送到server，但是在server里找不到交换机
②消息推送到server，找到交换机了，但是没找到队列
③消息推送到sever，交换机和队列啥都没找到
④消息推送成功

# 消息确认机制
和生产者的消息确认机制不同，因为消息接收本来就是在监听消息，符合条件的消息就会消费下来。
所以，消息接收的确认机制主要存在三种模式：

①根据情况确认， AcknowledgeMode.AUTO，这也是默认的情况，
如果消费方方法正常结束，没有抛出异常，则表示消息被处理成功。
如果抛出异常，则将消息重新放回队列中继续消费。
②自动确认，  AcknowledgeMode.NONE
RabbitMQ成功将消息发出（即将消息成功写入TCP Socket）中立即认为本次投递已经被正确处理，不管消费者端是否成功处理本次投递。
所以这种情况如果消费端消费逻辑抛出异常，也就是消费端没有处理成功这条消息，那么就相当于丢失了消息。
一般这种情况我们都是使用try catch捕捉异常后，打印日志用于追踪数据，这样找出对应数据再做后续处理。


③ 手动确认 ， AcknowledgeMode.MANUAL 这个比较关键，也是我们配置接收消息确认机制时，多数选择的模式。
消费者收到消息后，手动调用basic.ack/basic.nack/basic.reject后，RabbitMQ收到这些消息后，才认为本次投递成功。
basic.ack用于肯定确认 
basic.nack用于否定确认（注意：这是AMQP 0-9-1的RabbitMQ扩展） 
basic.reject用于否定确认，但与basic.nack相比有一个限制:一次只能拒绝单条消息 

### 手动确认样例（推荐）
添加配置文件
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        # 消息确认模式
        acknowledge-mode: manual
```
```java
@RabbitListener(queues = "TestDirectQueue")
public void process(String testMessage, Message message, Channel channel) throws IOException {
    try {
        System.out.println("DirectReceiver消费者收到消息  : " + testMessage);
        channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
    } catch (Exception e) {
        // 第二个参数，false只拒绝当前消息包，true拒绝小于当前包id的所有消息
        // 第三个参数，是否重新放回队列顶部，不建议返回可能会因为消息无法消费导致系统循环报错，打满CPU占用，建议做日志记录，手动修复数据
        channel.basicNack(message.getMessageProperties().getDeliveryTag(), false, false);
    }
}
```
### 手动确认样例（不推荐）
```java
import com.rabbitmq.client.Channel;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.listener.api.ChannelAwareMessageListener;
import org.springframework.stereotype.Component;

@Component
public class MyAckReceiver implements ChannelAwareMessageListener {

    @Override
    public void onMessage(Message message, Channel channel) throws Exception {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            // 因为传递消息的时候用的map传递,所以将Map从Message内取出需要做些处理
            String messageId = message.getMessageProperties().getMessageId();
            String messageData = new String(message.getBody());
            System.out.println("=============MyAckReceiver Message Start=================");
            System.out.println("MyAckReceiver  messageId:" + messageId + "  messageData:" + messageData);
            System.out.println("消费的主题消息来自：" + message.getMessageProperties().getConsumerQueue());
            System.out.println("=============MyAckReceiver Message End=================");
            // 第二个参数，手动确认可以被批处理，当该参数为 true 时，则可以一次性确认 delivery_tag 小于等于传入值的所有消息
            channel.basicAck(deliveryTag, true);
        } catch (Exception e) {
            channel.basicReject(deliveryTag, false);
            e.printStackTrace();
        }
    }
}
```
```java
import org.springframework.amqp.core.AcknowledgeMode;
import org.springframework.amqp.rabbit.connection.CachingConnectionFactory;
import org.springframework.amqp.rabbit.listener.SimpleMessageListenerContainer;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MessageListenerConfig {



    @Autowired
    private CachingConnectionFactory connectionFactory;
    @Autowired
    private MyAckReceiver myAckReceiver;//消息接收处理类

    @Bean
    public SimpleMessageListenerContainer simpleMessageListenerContainer() {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setConcurrentConsumers(1);
        container.setMaxConcurrentConsumers(1);
        // RabbitMQ默认是自动确认，这里改为手动确认消息
        container.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        // 设置一个队列
        container.setQueueNames("TestDirectQueue");
        // 如果同时设置多个如下： 前提是队列都是必须已经创建存在的
        // container.setQueueNames("TestDirectQueue","TestDirectQueue2","TestDirectQueue3");


        // 另一种设置队列的方法,如果使用这种情况,那么要设置多个,就使用addQueues
        //container.setQueues(new Queue("TestDirectQueue",true));
        //container.addQueues(new Queue("TestDirectQueue2",true));
        //container.addQueues(new Queue("TestDirectQueue3",true));
        container.setMessageListener(myAckReceiver);

        return container;
    }

}
```

# 消息顺序性
其实队列本身是有顺序的，但是生产环境服务实例一般都是集群，当消费者是多个实例时，队列中的消息会分发到所有实例进行消费（同一个消息只能发给一个消费者实例），这样就不能保证消息顺序的消费，因为你不能确保哪台机器执行消费端业务代码的速度快
所以对于需要保证顺序消费的业务，我们可以只部署一个消费者实例，然后设置 RabbitMQ 每次只推送一个消息，再开启手动 ack 即可，配置如下
```yaml
spring:
  rabbitmq:
    listener:
      simple:
        # 每次推送消息个数
        prefetch: 1 
```
