## 生产者重连

有时候由于网络波动，可能会出现客户端连接MQ失败的情况。通过配置我们可以开启连接失败后的重连机制：
```
spring:
  rabbitmq:
    connection-timeout: 1s # 设置MQ的连接超时时间
    template:
      retry:
        enabled: true # 开启超时重试机制
        initial-interval: 1000ms # 失败后的初始等待时间
        multiplier: 1 # 失败后下次的等待时长倍数，下次等待时长 = initial-interval * multiplier
        max-attempts: 3 # 最大重试次数
```
**注意：** 
当网络不稳定的时候，利用重试机制可以有效提高消息发送的成功率。不过 SpringAMQP 提供的重试机制是**阻塞式**的重试，也就是说多次重试等待的过程中，当前线程是被阻塞的，会影响业务性能。 如果对于业务性能有要求，建议**禁用**重试机制。如果一定要使用，请合理配置等待时长和重试次数，当然也可以考虑使用**异步线程**来执行发送消息的代码。

### 生产者确认

1. RabbitMQ 生产者确认包含两种机制：**Publisher Confirm**、**Publisher Return**
2. 核心作用：MQ 收到消息后，向生产者返回确认结果
3. 确认返回结果：
    - ✅ 消息抵达 MQ，但路由失败：触发 Publisher Return 返回异常原因，同时返回 ACK
    - ✅ 临时消息投递到 MQ、入队成功：返回 ACK
    - ✅ 持久消息投递到 MQ、入队并完成持久化：返回 ACK
    - ❌ 其余场景：返回 NACK，代表投递失败
4. 区分：
    - Publisher Confirm：确认消息**有没有到达交换机**
    - Publisher Return：消息到达交换机，但是**路由不到队列**时，把消息退回生产者

### 如何处理生产者的确认消息：

1.生产者确认需要额外的网络和系统资源开销，尽量不要使用
2.如果一定要使用，无需开启Publisher-Return机制，因为一般路由失败是自己业务问题
3.对于nack消息可以有限次数重试，依然失败则记录异常消息

# SpringAMQP 生产者确认
## 一、两种确认机制作用

- **Publisher Confirm**：确认消息有没有成功到达交换机。返回 ACK/NACK。
- **Publisher Return**：消息成功到达交换机，但是**路由失败（找不到队列）**时，把消息退回生产者。

> 四种回执情况：
> 
> 1. 消息抵达 MQ，路由失败：触发 ReturnCallback，同时返回 ACK
> 2. 临时消息投递入队成功：返回 ACK
> 3. 持久消息投递、入队并持久化完成：返回 ACK
> 4. 其他异常：返回 NACK，投递失败

## 二、yml 配置（生产者微服务 application.yml）

```
spring:
  rabbitmq:
    publisher-confirm-type: correlated # confirm模式
    publisher-returns: true # 开启return回调
```

publisher-confirm-type 三选项：

- `none`：关闭 confirm
- `simple`：同步阻塞等待回执
- `correlated`：**异步回调（常用）**

## 三、ReturnCallback 配置（全局，项目启动时设置）

> 重点：一个 RabbitTemplate 只能设置一个 ReturnCallback，全局统一配置

```
@Slf4j
@Configuration
public class CommonConfig implements ApplicationContextAware {
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        RabbitTemplate rabbitTemplate = applicationContext.getBean(RabbitTemplate.class);
        // return回调：路由失败时触发
        rabbitTemplate.setReturnCallback((message, replyCode, replyText, exchange, routingKey) -> {
            log.info("消息发送失败，应答码{}，原因{}，交换机{}，路由键{},消息{}",
                    replyCode, replyText, exchange, routingKey, message.toString());
        });
    }
}
```

回调参数：message 消息、replyCode 应答码、replyText 失败原因、exchange 交换机、routingKey 路由键

## 四、ConfirmCallback 使用（发送消息时绑定）

> CorrelationData：消息唯一标识，绑定消息和 confirm 回调，correlated 模式必备

```
@Test
void testPublisherConfirm() throws InterruptedException {
    // 1.创建关联对象
    CorrelationData cd = new CorrelationData();
    // 2.注册确认回调
    cd.getFuture().addCallback(new ListenableFutureCallback<CorrelationData.Confirm>() {
        @Override
        public void onFailure(Throwable ex) {
            // Future本身异常，极少触发
            log.error("handle message ack fail", ex);
        }
        @Override
        public void onSuccess(CorrelationData.Confirm result) {
            if(result.isAck()){
                log.debug("发送消息成功，收到 ack!");
            }else{
                log.error("发送消息失败，收到 nack，reason : {}", result.getReason());
            }
        }
    });
    // 3.发送消息，传入cd绑定回调
    rabbitTemplate.convertAndSend("hmall.direct", "red1", "hello", cd);
}
```

- `result.isAck() = true`：ACK，消息到达交换机
- `result.isAck() = false`：NACK，消息没到达交换机
- `result.getReason()`：NACK 失败描述

## 五、面试速记版

1. Confirm：消息到交换机的确认；Return：到交换机但路由不到队列的退回。
2. 配置 `publisher-confirm-type: correlated` 异步回调，`publisher-returns: true`开启 return。
3. ReturnCallback 全局配置，一个 RabbitTemplate 只能一个 ReturnCallback。
4. Confirm 靠 CorrelationData 绑定消息，发送消息时传入，异步回调 ACK/NACK。