# MQ 消息队列

## 1、RabbitMQ 如何保证消息不丢失

使用 RabbitMQ 来确保 MySQL 和 Redis 之间的数据双写一致性，要求我们实现消息的高可用性，具体措施包括：

1. **开启生产者确认机制：**确保消息能被送达队列，如有错误则记录日志并修复数据。
2. **启用持久化功能：**保证消息在未消费前不会在队列中丢失，需要对交换机、队列和消息本身都进行持久化。
3. **对消费者开启自动确认机制为 false**，根据实际情况手动进行确认
4. **开启消费者失败重试机制**，多次重试后将消息投递到异常交换机，交由人工处理。

**其他补充措施**

- **设置合理的队列长度限制：**避免队列溢出导致消息丢失
- **启用死信队列：**处理无法正常消费的消息，避免消息被丢弃
- **定期备份：**对 RabbitMQ 的元数据和消息进行定期备份
- **监控警告：**实时监控 RabbitMQ 运行状态，出现异常及时告警。

---



## 2、RabbitMQ 消息的重复消费问题如何解决

RabbitMQ 消息的重复消费是分布式系统中常见的问题，通常由网络波动、消费者处理超时、重试机制等原因导致。解决重复消费的核心是实现**消息消费的幂等性**，即多次消费同一消息不会对业务状态产生负面影响。

1. **业务层实现幂等性处理**

   - **基于唯一标识去重**
     - 为每条消息生成唯一 ID（如 UUID）消费时先判断该 ID 是否已处理
     - 可将 ID 存入 **Redis** 或数据库，处理前检查存在性

   ```java
   // 伪代码示例
   String messageId = message.getUniqueId();
   // 检查是否已处理
   if (redisTemplate.opsForValue().setIfAbsent(messageId, "processed", 24, TimeUnit.HOURS)) {
       // 未处理过，执行业务逻辑
       processMessage(message);
   } else {
       // 已处理过，直接确认消息
       channel.basicAck(deliveryTag, false);
   }
   ```

   - **基于业务唯一键去重**
     - 利用业务天然的唯一标识（如订单号、用户 ID+操作类型）
     - 数据库层面可通过唯一索引约束防止重复插入

   ```sql
   -- 示例：为订单号创建唯一索引
   CREATE UNIQUE INDEX idx_order_no ON orders(order_no);
   ```

   - **乐观锁机制**
     - 适用于更新操作，通过版本号控制避免重复更新

   ```sql
   -- 示例：更新时检查版本号
   UPDATE products 
   SET stock = stock - 1, version = version + 1 
   WHERE id = 100 AND version = 3;
   ```

2. **消息属性优化**

   - **设置消息过期时间：**避免无效消息长期滞留队列
   - **添加消息序号：**在消息中包括递增序号，便于追踪和去重

3. **消费者端控制**

   - **合理设置重试策略**：避免无限制重试导致的重复消费

     ```java
     // 示例：限制最大重试次数
     if (retryCount >= MAX_RETRY_COUNT) {
         // 超过最大次数，发送到死信队列
         channel.basicNack(deliveryTag, false, false);
     }
     ```

   - **确保消费逻辑的原子性**：将消息处理和结果记录放在同一事务中

---

## 3、死信

### 死信队列

死信队列（Dead Letter Queue，DLQ）其实就是一个 “失败消息的垃圾桶”，但它的作用比垃圾桶大得多。

简单来说，当一条消息在正常的消息队列中**无法被成功消费**时（比如处理报错、超时、超过重试次数等），它会被 “丢” 到另一个专门存放这些失败消息的队列里，这个专门的队列就是**死信队列**。

**常见触发死信的原因**

- **消费失败**：消费者处理消息时抛出异常，并且达到了最大重试次数。
- **消息超时**：消息在队列中超过了预设的存活时间（TTL）。
- **队列满了**：消息队列达到容量上限，新消息被挤掉。
- **消息被拒绝**：消费者明确拒绝处理该消息（requeue=false）。

**死信队列的好处**

1. **不阻塞正常业务**

   失败的消息不会一直在正常队列里占位置或反复重试，影响正常消息处理

2. **方便排查问题**

   可以转吗监控和分析死信队列里的消息，找出失败原因（比如数据错误、业务逻辑缺陷）

3. **可恢复性**

   解决问题后，可以从死信队列把消息重新发回正常队列处理。

---

### 死信交换机

在 RabbitMQ 中，死信交换机（Dead-Letter Exchange，DLX）是一种特殊的交换机，用于处理 “死信”—— 即无法被正常消费的消息。当消息满足特定条件时，会被路由到死信交换机，再由死信交换机转发到绑定的死信队列，便于后续处理（如人工干预、日志分析等）。

**死信交换机的三种场景**

1. **消息被拒绝（basicNack/basicReject）** 且 `requeue=false`（不重新入队）；
2. **消息过期（TTL，Time-To-Live）**：消息在队列中超过预设的存活时间；
3. **队列达到最大长度**：队列满后，新消息无法入队，旧消息被挤入死信队列。

**死信交换机的工作流程**

1. 声明一个普通交换机和队列（业务队列），并为队列指定死信交换机（通过 `x-dead-letter-exchange` 参数）
2. 声明死信交换机（类型通常为 direct 和 topic）和死信队列，绑定两者
3. 当业务队列中的消息成为死信时，RabbitMQ 自动将其路由到死信交换机，再转发到死信队列
4. 消费者监听死信队列，处理死信消息（如重试、记录日志、人工处理等）

```java
// 1. 声明死信交换机和死信队列
channel.exchangeDeclare("dlx.exchange", "direct", true);
channel.queueDeclare("dlx.queue", true, false, false, null);
channel.queueBind("dlx.queue", "dlx.exchange", "dlx.routingKey");

// 2. 声明业务队列，并指定死信交换机和路由键
Map<String, Object> args = new HashMap<>();
// 指定死信交换机
args.put("x-dead-letter-exchange", "dlx.exchange");
// 指定死信路由键（可选，默认使用原消息的路由键）
args.put("x-dead-letter-routing-key", "dlx.routingKey");
// 设置消息过期时间（可选，单位：毫秒）
args.put("x-message-ttl", 10000);
// 设置队列最大长度（可选）
args.put("x-max-length", 100);

channel.exchangeDeclare("business.exchange", "direct", true);
channel.queueDeclare("business.queue", true, false, false, args);
channel.queueBind("business.queue", "business.exchange", "business.routingKey");

// 3. 发送消息到业务队列
channel.basicPublish("business.exchange", "business.routingKey", 
                   MessageProperties.PERSISTENT_TEXT_PLAIN, "test".getBytes());

// 4. 消费者拒绝消息（导致消息成为死信）
channel.basicConsume("business.queue", false, (consumerTag, delivery) -> {
    try {
        // 处理消息失败，拒绝并设置不重新入队
        channel.basicReject(delivery.getEnvelope().getDeliveryTag(), false);
    } catch (Exception e) {
        // 异常处理
    }
}, consumerTag -> {});
```

---

### 延迟队列和死信交换机的关系

延迟队列用于实现 “消息延迟一段时间后再被消费” 的场景（如订单超时未支付自动取消）。RabbitMQ 本身没有专门的延迟队列，但可以通过 **“死信交换机 + TTL”** 实现：

1. 声明一个“延迟队列”（实际是普通队列），设置 `x-message-ttl`（消息过期时间）和死信交换机；
2. 消息发送到该队列后，不会被消费者立即消费，而是等待 TTL 过期
3. 过期后，消息成为死信，被路由到死信交换机，最终进入真正的业务队列
4. 消费者监听业务队列，实现“延迟消费”效果

**优势**：通过 TTL 和死信机制，无需额外组件即可实现延迟功能，适用于对延迟精度要求不高的场景（毫秒级误差可接受）。



---

## 4、如果有 100 万消息堆积在 MQ，如何解决

若出现消息堆积，可采取以下措施：

1. 提高消费者消费能力，如使用多线程。
2. 增加消费者数量，采用工作队列模式，让多个消费者并行消费同一队列。
3. 扩大队列容量，使用RabbitMQ的惰性队列，支持数百万条消息存储，直接存盘而非内存。



---

## 5、RabbitMQ 的高可用机制

RabbitMQ 的高可用机制是指通过一系列设计和配置，确保在节点故障、网络异常等情况下，消息队列服务仍能正常运行，避免服务中断和消息丢失。其核心目标是实现 **服务连续性** 和 **数据可靠性**。



## 

