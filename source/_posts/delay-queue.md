---
date: 2019-01-01 
title: 延迟队列
categories: PHP
tag: 延迟队列 
---

比如要实现30分钟未支付订单取消,量少的时候可以用数据库轮训的方式,但是数据量大的话，轮训的并发和准确性就不可靠,这个时候可以用延迟队列来解决这个问题

### 延迟队列的实现
* [RabbitMQ] RabbitMQ通过[RabbitMQ Delayed Message Plugin](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange)可支持延迟队列
* [Redis] Redis的Sorted Set可被用于实现简单的延迟队列。利用Redis的Lua支持我们也可以将基建设成一个功能全面的延迟队列服务。
* [Redis + RabbitMQ] redis用来消费任务不可靠，没有ack机制支持,如果消费的脚本出现异常,那这个任务就有可能丢失,redis可以仅仅用来做延迟,取到消息后直接推到  RabbitMQ 或者 Kafka 进行消费

### Redis 实现延迟队列

Sorted Set是一个有序的集合，集合内元素的排序基于其加入集合时指定的score。通过ZRANGEBYSCORE命令，我们可以取得score在指定区间内的元素。将集合中的元素做为消息，score视为延迟的时间，这便是一个延迟队列的模型。

生产者通过ZADD将消息发送到队列中：
``` php
ZADD delay-queue 1520985600 "publish article"
```

消费者通过ZRANGEBYSCORE获取消息。如果时间未到，将得不到消息；当时间已到或已超时，都可以得到消息：
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count] 
min 和 max 可以是 -inf 和 +inf ，这样一来，你就可以在不知道有序集的最低和最高 score 值的情况下，使用 ZRANGEBYSCORE 这类命令。
``` php
127.0.0.1:6379> ZRANGEBYSCORE delay-queue -inf 1520985599
(empty list or set)
127.0.0.1:6379> ZRANGEBYSCORE delay-queue -inf 1520985600 WITHSCORES
1) "publish article"
2) "1520985600"
127.0.0.1:6379> ZRANGEBYSCORE delay-queue -inf 1520985601 WITHSCORES
1) "publish article"
2) "1520985600"
```

使用ZRANGEBYSCORE取得消息后，消息并没有从集合中删出。需要调用ZREM删除消息：
``` php
127.0.0.1:6379> ZREM delay-queue "publish article"
```

美中不足的是，消费者组合使用ZRANGEBYSCORE和ZREM的过程不是原子的，当有多个消费者时会存在竞争，可能使得一条消息被消费多次。此时需要使用Lua脚本保证消费操作的原子性：
``` php
local message = redis.call('ZRANGEBYSCORE', KEYS[1], '-inf', ARGV[1], 'WITHSCORES', 'LIMIT', 0, 1);
if #message > 0 then
  redis.call('ZREM', KEYS[1], message[1]);
  return message;
else
  return {};
end
```

