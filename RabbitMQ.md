# RabbitMQ

> **RabbitMQ**是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件）。RabbitMQ服务器是用Erlang语言编写的

<a name="bSvlP"></a>
# 为什么使用MQ

- 解耦
- 异步
- 削峰
<a name="H0Kls"></a>
# 相关名词
<a name="R5qRW"></a>
## 1、ConnectionFactory、Connection、Channel
ConnectionFactory、Connection、Channel都是RabbitMQ对外提供的API中最基本的对象。<br />Connection是RabbitMQ的socket链接，它封装了socket协议相关部分逻辑。<br />ConnectionFactory为Connection的制造工厂。<br />Channel是我们与RabbitMQ打交道的最重要的一个接口，我们大部分的业务操作是在Channel这个接口中完成的，包括定义Queue、定义Exchange、绑定Queue与Exchange、发布消息等。
<a name="iswmL"></a>
## 2、Queue
Queue（队列）是RabbitMQ的内部对象，用于存储消息。<br />RabbitMQ中的消息都只能存储在Queue中。<br />生产者生产消息并最终投递到Queue中，消费者从Queue中获取消息并消费。
<a name="dixTl"></a>
## 3、Message acknowledgment
（一种保险机制）要求消费者在消费完消息后发送一个回执给RabbitMQ，RabbitMQ收到消息回执（Message acknowledgment）后才将该消息从Queue中移除。
<a name="MBIyQ"></a>
## 4、Message durability
如果我们希望即使在RabbitMQ服务重启的情况下，也不会丢失消息，我们可以将Queue与Message都设置为可持久化的（durable），这样可以保证绝大部分情况下我们的RabbitMQ消息不会丢失。
<a name="3znSG"></a>
## 5、Prefetch count
我们可以通过设置prefetchCount来限制Queue每次发送给每个消费者的消息数，比如我们设置prefetchCount=1，则Queue每次给每个消费者发送一条消息；消费者处理完这条消息后Queue会再给该消费者发送一条消息。
<a name="bgG2v"></a>
## 6、Exchange
实际的情况是，生产者将消息发送到Exchange（交换器，下图中的X），由Exchange将消息路由到一个或多个Queue中（或者丢弃）。<br />![](https://cdn.nlark.com/yuque/0/2020/png/1262675/1593412266505-1beed9f0-7b3a-47c6-ab83-237766478d76.png#align=left&display=inline&height=110&margin=%5Bobject%20Object%5D&originHeight=110&originWidth=332&size=0&status=done&style=none&width=332)
<a name="9279Q"></a>
## 7、routing key
生产者在将消息发送给Exchange的时候，一般会指定一个routing key，来指定这个消息的路由规则。
<a name="3pWAd"></a>
## 8、Binding
RabbitMQ中通过Binding将Exchange与Queue关联起来，这样RabbitMQ就知道如何正确地将消息路由到指定的Queue了。<br />![](https://cdn.nlark.com/yuque/0/2020/png/1262675/1593412448029-fa1fc87f-10fd-4aac-8d23-2dcfffa8e81d.png#align=left&display=inline&height=90&margin=%5Bobject%20Object%5D&originHeight=90&originWidth=322&size=0&status=done&style=none&width=322)
<a name="ZLDcn"></a>
## 9、Binding key
当binding key与routing key相匹配时，消息将会被路由到对应的Queue中。
<a name="zUdvS"></a>
## 10、Exchange Types
RabbitMQ常用的Exchange Type有fanout、direct、topic这3种。

- **fanout（不用匹配）**

fanout类型的Exchange路由规则非常简单，它会把所有发送到该Exchange的消息路由到所有与它绑定的Queue中。

- **direct（完全匹配）**

direct类型的Exchange路由规则也很简单，它会把消息路由到那些binding key与routing key完全匹配的Queue中。

- **topic（模糊匹配）**

binding key与routing key一样也是句点号“. ”分隔的字符串。<br />binding key中可以存在两种特殊字符“*”与“#”，用于做模糊匹配，其中“*”用于匹配一个单词，“#”用于匹配多个单词（可以是零个）。
<a name="wLdk2"></a>
# 快速实验
_本次实例需要创建2个springboot项目，一个 rabbitmq-provider （生产者），一个rabbitmq-consumer（消费者）。_
<a name="YKk0n"></a>
## 0、在本地启动RabbitMQ
![image.png](https://cdn.nlark.com/yuque/0/2020/png/1262675/1593415923950-c332ec03-a65b-4907-8c0c-b29b65f8cea1.png#align=left&display=inline&height=167&margin=%5Bobject%20Object%5D&name=image.png&originHeight=256&originWidth=785&size=21570&status=done&style=shadow&width=511)<br />命令：**rabbitmq-server.bat**
<a name="A7fo5"></a>
## 1、编写生产者
需要依赖
```xml
<!--消息队列模块-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
编写配置文件
```yaml
server:
  port: 8021
spring:
  application:
    name: rabbitmq-provider #项目名字
    rabbitmq: #配置rabbitMq 服务器
      host: 127.0.0.1
      port: 5672
      username: guest
      password: guest
```


**此处使用直连交互机**<br />创建DirectRabbitConfig.java
```java
package com.gx.rabbitmq.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DirectRabbitConfig {

    //队列，起名：TestDirectQueue
    @Bean
    public Queue TestDirectQueue(){
        return new Queue("TestDirectQueue",true);
    }

    //Direct交换机，起名：TestDirectExchange
    @Bean
    DirectExchange TestDirectExchange(){
        return new DirectExchange("TestDirectExchange",true,false);
    }

    //绑定，将队列和交换机绑定，并设置匹配键：TestDirectRouting
    @Bean
    Binding bindingDirect(){
        return BindingBuilder.bind(TestDirectQueue()).to(TestDirectExchange()).with("TestDirectRouting");
    }

    @Bean
    DirectExchange lonelyDirectExchange(){
        return new DirectExchange("lonelyDirectExchange");
    }

}
```
**消息推送**<br />SendMessageController.java
```java
package com.gx.rabbitmq.controller;

import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

@RestController
public class SendMessageController {

    //使用RabbitTemplate,这提供了接收/发送等等方法
    @Autowired
    private RabbitTemplate rabbitTemplate;

    @RequestMapping("/sendDirectMessage")
    public String sendDirectMessage(){
        String messageId = String.valueOf(UUID.randomUUID());
        String messageData = "test message, hello!";
        String creatTime = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        Map<String,Object> map = new HashMap<>();
        map.put("messageId",messageId);
        map.put("messageData",messageData);
        map.put("createTime",creatTime);
        //将消息携带绑定键值：TestDirectRouting 发送到交换机TestDirectExchange
        rabbitTemplate.convertAndSend("TestDirectExchange","TestDirectRouting",map);
        return "ok";
    }
}
```
启动rabbitmq-provider项目，调用下接口:<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/1262675/1593416912377-459e0313-bc71-46a5-b0c4-94affe230625.png#align=left&display=inline&height=194&margin=%5Bobject%20Object%5D&name=image.png&originHeight=388&originWidth=939&size=35228&status=done&style=shadow&width=469.5)<br />（这个工具是Postman）

因为目前还没弄消费者 rabbitmq-consumer，消息没有被消费的，可以去rabbitMq管理页面看看，是否推送成功：<br />http://localhost:15672/#/<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/1262675/1593417023117-ea4f1236-79c2-42e1-97c2-01ac131ee3d5.png#align=left&display=inline&height=338&margin=%5Bobject%20Object%5D&name=image.png&originHeight=676&originWidth=959&size=55443&status=done&style=shadow&width=479.5)<br />很好，消息已经推送到rabbitMq服务器上面了。

<a name="5f5s2"></a>
## 2、编写消费者
导入依赖
```xml
<!--rabbitmq-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
编写配置文件
```yaml
server:
  port: 8022
spring:
  #给项目来个名字
  application:
    name: rabbitmq-consumer
  #配置rabbitMq 服务器
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: guest
    password: guest
```
创建DirectRabbitConfig.java（**消费者单纯的使用，其实****可以不用添加这个配置****，直接建后面的监听就好，使用注解来让监听器监听对应的队列即可。配置上了的话，其实消费者也是生成者的身份，也能推送该消息。**）：
```java
package com.gx.rabbitmqconsumer.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class DirectRabbitConfig {
    //队列 起名：TestDirectQueue
    @Bean
    public Queue TestDirectQueue() {
        return new Queue("TestDirectQueue",true);
    }

    //Direct交换机 起名：TestDirectExchange
    @Bean
    DirectExchange TestDirectExchange() {
        return new DirectExchange("TestDirectExchange");
    }

    //绑定  将队列和交换机绑定, 并设置用于匹配键：TestDirectRouting
    @Bean
    Binding bindingDirect() {
        return BindingBuilder.bind(TestDirectQueue()).to(TestDirectExchange()).with("TestDirectRouting");
    }
}
```
创建消息接收监听类，DirectReceiver.java
```java
package com.gx.rabbitmqconsumer.receiver;

import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;
import java.util.Map;

@Component
@RabbitListener(queues = "TestDirectQueue")//监听的队列名称 TestDirectQueue
public class DirectReceiver {
    @RabbitHandler
    public void process(Map testMessage){
        System.out.println("DirectReceiver消费者收到消息  : "+testMessage.toString());
    }
}
```
启动rabbitmq-consumer项目<br />可以看到把之前推送的那条消息消费下来了。<br />那么直连交换机既然是一对一，那如果咱们配置多台监听绑定到同一个直连交互的同一个队列，会怎么样？<br />![image.png](https://cdn.nlark.com/yuque/0/2020/png/1262675/1593420762682-e9072008-f125-4a26-aff8-b5b0217815f9.png#align=left&display=inline&height=480&margin=%5Bobject%20Object%5D&name=image.png&originHeight=681&originWidth=1058&size=123021&status=done&style=shadow&width=746)<br />可以看到是实现了轮询的方式对消息进行消费，而且不存在重复消费。

基本的内容就是这些，其他的相关操作详见：<br />[Springboot 整合RabbitMq ，用心看完这一篇就够了](https://blog.csdn.net/qq_35387940/article/details/100514134)

