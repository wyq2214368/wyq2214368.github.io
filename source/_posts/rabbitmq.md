---
title: RabbitMQ总结
date: 2019-01-25
categories: PHP
tag: rabbitMQ
---
RabbitMQ是实现了高级消息队列协议（AMQP）的开源消息代理软件（亦称面向消息的中间件）

## Rabbitmq特性

* 可靠性：持久化存储、ACK消息确认、发布confirm、事务支持。
* 灵活的路由：交换机功能。交换机类型：direct,topic,headers,fanout。
* ha镜像，master-slave 多协议支持,[集群节点](https://blog.csdn.net/qq_35246620/article/details/72473098)
* 多语言客户端支持：java、c#、ruby、Python、php、c、scale、nodejs、go、erlang…
* 管理界面功能丰富、命令行rabbitmqctl、RPC远程调度

## AMQP高级消息协议

AMQP(高级消息队列协议) 是一个异步消息传递所使用的应用层协议规范，作为线路层协议，AMQP 客户端能够无视消息的来源任意发送和接受信息。
AMQP四个重要组成部分：
- virtual host，虚拟主机
- exchange，交换机
- queue，队列
- binding，绑定

一个虚拟主机持有一组交换机、队列和绑定。每台rabbitmq服务器可以有多个虚拟主机，默认为/。虚拟主机主要用于用户权限控制。因为RabbitMQ当中，用户只能在虚拟主机的粒度进行权限控制。因此，如果需要禁止A组访问B组的交换机/队列/绑定，必须为A和B分别创建一个虚拟主机。

队列（Queues）是你的消息（messages）的终点，可以理解成装消息的容器。队列是由消费者（Consumer）通过程序建立的，如果一个消费者试图创建一个已经存在的队列，RabbitMQ会直接忽略这个请求。

交换机（Exchange）可以理解成具有路由表的路由程序。每个消息都有一个称为路由键（routingkey）的属性，就是一个简单的字符串。每个交换机都是一个独立的进程，合理利用服务器多核CPU使得rabbitmq性能得到最佳。交换机有多种类型，不同的交换机类型CPU开销是不一样的，一般来说CPU开销顺序是:

`TOPIC > DIRECT > FANOUT > NAMELESS`

## 交换机类型
消息不直接发送到queue中，中间有一个exchange做消息分发，producer甚至不知道消息发送到那个队列中去。因此，当exchange收到message时，必须准确知道该如何分发。是推送到一定规则的queue，还是推送到多个queue中，还是被丢弃。这些规则都是通过exchange去定义的。

- 匿名交换机，工作队列模式 
- 扇形交换机（fanout），发布订阅/广播模式 
- 直连交换机（direct），路由绑定/广播精确匹配 
- 主题交换机（topic），路由规则/广播模糊匹配 
- 头交换机（header），定义AMQP头部属性


## 相关特性

### 持久化

如果你没有特意告诉RabbitMQ，那么在它退出或者崩溃的时候，将会丢失所有队列和消息。已经定义过非持久化的队列不能再定义为持久化队列，我们得重新命名一个新的队列。必须把“队列”和“消息”都设为持久化。
``` php
//交换机持久化
$channel->exchange_declare('exchange', 'fanout', false, true, false);
//队列持久化
$channel->queue_declare('task_queue', false, true, false, false);
//消息持久化
$msg = new AMQPMessage($body, ['delivery_mode' => 2]);
```
### Ack消息确认

当消息被RabbitMQ发送给消费者（consumers）之后，马上就会在内存中移除。这种情况，你只要把一个工作者（worker）停止，正在处理的消息就会丢失。同时，所有发送到这个工作者的还没有处理的消息都会丢失。我们不想丢失任何任务消息。如果一个工作者（worker）挂掉了，我们希望任务会重新发送给其他的工作者（worker）。为了防止消息丢失，RabbitMQ提供了消息响应（acknowledgments）。消费者会通过一个ack（响应），告诉RabbitMQ已经收到并处理了某条消息，然后RabbitMQ就会释放并删除这条消息。
``` php
$channel->basic_consume('task_queue', '', false, false, false, false, $callback);
//在回调函数中发送ack消息
$msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
```
### Qos公平调度

如果多个worker进程中，某个worker处理比较慢，另一个worker比较快，默认RabbitMQ只管分发进入队列的消息，不会关心有多少消费者（consumer）没有作出响应，这样会使得比较慢的worker消息堆积过多，导致任务分配不均。Qos公平调度设置prefetch_count=1，即在同一时刻，不会发送超过1条消息给一个工作者（worker），直到它已经处理了上一条消息并且作出了响应。这样，RabbitMQ就会把消息分发给下一个空闲的工作者（worker）。
``` php
//$prefetch_size,$prefetch_count,$global
$channel->basic_qos(null, 1, null);
```
### 消息事务

将消息设为持久化并不能完全保证不会丢失。持久化只是告诉了RabbitMq要把消息存到硬盘，但从RabbitMq收到消息到保存之间还是有一个很小的间隔时间。因为RabbitMq并不是所有的消息都使用同步IO—它有可能只是保存到缓存中，并不一定会写到硬盘中。并不能保证真正的持久化，但已经足够应付我们的简单工作队列。如果你一定要保证持久化，我们需要改写代码来支持事务（transaction）。
``` php
$channel->tx_select();
$channel->basic_publish($msg, '', 'task_queue');
$channel->tx_commit();
```
### confirm消息确认

AMQP消息协议提供了事务支持，不过事务机制会导致性能急剧下降，所以rabbitmq特别引入了confirm机制。

Confirm有三种编程方式：

1. 普通confirm模式。每发送一条消息后，调用wait_for_pending_acks()方法，等待服务器端confirm。实际上是一种串行confirm。

2. 批量confirm模式。每次发送一批消息后，调用wait_for_pending_acks()方法，等待服务器端confirm。

3. 异步confirm模式。提供一个回调方法，服务器端confirm了一条(或多条)消息后客户端会回调这个方法。

代码示例：
``` php
//一旦消息被设为confirm模式，就不能设置事务模式，反之亦然
$channel->confirm_select();
//阻塞等待消息确认
$channel->wait_for_pending_acks();
//异步回调消息确认
$channel->set_ack_handle();
$channel->set_nack_handler();
```
## PHP使用 Rabbitmq
使用 php-amqplib/php-amqplib 进行二次封装可以满足基本需求
``` php
<?php
/*
 * rabbitMQ封装
 */
namespace App\Libraries\MQ;

// define('AMQP_PASSIVE', true);
// define('AMQP_DEBUG', false);
use App\Exceptions\AMQPException;
use PhpAmqpLib\Connection\AMQPConnection;
use PhpAmqpLib\Message\AMQPMessage;

class AMQP{

    private $channel;
    private $msg;
    private $conn;
    private $system;
    private $consumer_tag;
    private $isRePublish = false;
    private $mid;
    private $callback;
    private $config_path = 'queue.RMQ_CONFIG';

    public function __construct($system = ''){
        if(!empty($system)){
            $this->connection($system);
        }
    }

    public function __destruct(){
        $this->closeConnection();
    }

    /**
     * 实例化后rabbitMQ连接
     * @param string $system 系统名称
     * @return bool
     */
    public function connection($system){;
        if(empty($this->conn)){
            return $this->resetConnection($system);
        }
        if($system != $this->system){
            return $this->resetConnection($system);
        }
        return true;
    }

    /**
     * 强制重置rabbitMQ连接
     * @param string $system 系统名称
     * @return bool
     */
    public function resetConnection($system){
        $rmqconfig = config($this->config_path);
        if(isset($rmqconfig[$system])){
            $this->closeConnection();
            list($host, $user, $password, $port, $vhost) = $rmqconfig[$system];
            $this->conn         = new AMQPConnection($host, $port, $user, $password, $vhost );
            if(!$this->conn->isConnected()){
                throw new AMQPException($system.'链接失败');
            }
            $this->consumer_tag = getNow().':'.getmypid();
            $this->channel      = $this->conn->channel();
            $this->system       = $system;
            //			$this->ackHandler();
            return true;
        }else{
            throw new AMQPException('systme 没有配置');
        }
    }

    /*
     * 设置消息体大小限制
     * @param string|int $bytes 字节数
     */
    private function setBodySizeLimit($bytes=0){
        $this->channel->setBodySizeLimit($bytes);
    }

    /*
     * 添加交换器
     * @param string $ename 交换器名称
     * @param string $type 交换器的消息传递方式 可选:'fanout','direct','topic','headers'
     * 'fanout':不处理(忽略)路由键，将消息广播给绑定到该交换机的所有队列
     * 'diect':处理路由键，对路由键进行全文匹配。对于路由键为"aaa_rain"的消息只会分发给路由键绑定为"aaa_rain"的队列,不会分发给路由键绑定为"aaa_music"的队列
     * 'topic':处理路由键，按模式匹配路由键。模式符号 "#" 表示一个或多个单词，"*" 仅匹配一个单词。如 "aaa.#" 可匹配 "aaa.rain.music"，但 "aaa.*" 只匹配 "aaa.rain"和"aaa.music"。只能用"."进行连接，键长度不超过255字节
     * @param boolean $durable 是否持久化
     * @param boolean $auto_delete 当所有绑定队列都不再使用时，是否自动删除该交换机
     */
    public function addExchange($ename, $type = 'fanout', $durable = true, $auto_delete = false){
        $this->channel->exchange_declare($ename, $type, false, $durable, $auto_delete);
    }

    /*
     * 添加队列
     * @param string $qname 队列名称
     * @param boolean $durable 是否持久化
     * @param boolean $exclusive 仅创建者可以使用的私有队列，断开后自动删除
     * @param boolean $auto_delete 当所有消费客户端连接断开后，是否自动删除队列
     * return int 该队列的ready消息数量
     */
    public function addQueue($qname, $durable = true, $exclusive = false, $auto_delete = false){
        $this->channel->queue_declare($qname, false, $durable, $exclusive, $auto_delete);
    }

    /*
     * 绑定队列和交换器
     * @param string $qname 队列名称
     * @param string $ename 交换器名称
     * @param string $routing_key 路由键 注:在fanout的交换器中路由键会被忽略
     */
    public function bind($qname, $ename, $routing_key = ''){
        $this->channel->queue_bind($qname, $ename, $routing_key);
    }

    /*
     * 设置消费者预取消息数量
     * @param string|int $count 预取消息数量
     */
    public function setQos($count = 1){
        $this->channel->basic_qos(null, $count, null);
    }

    /**
     * 基础模型之消息发布
     * @param string $exchange 交换器名称
     * @param string|array $msg 发布内容
     * @param string $mqtype 发布消息的类型
     * @return bool
     */
    public function basicPublish($msg, $exchang,$routing_key=''){
        if(is_array($msg))$msg = json_encode($msg);
        $message  = new AMQPMessage( $msg,array('content_type' => 'text/plain', 'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT) );
        $this->channel->basic_publish($message, $exchang,$routing_key);
        $this->publishLog($exchange,$routing_key,1);
    }

    /**
     * confirm  pushlish
     * @param string $exchange 交换器名称
     * @param string|array $msg 发布内容
     * @param string $mqtype 发布消息的类型
     * @return bool
     */
    public function ackPublish($msg, $exchang,$routing_key=''){
        $this->ackHandler();
        $this->basicPublish($msg, $exchang,$routing_key);
        $this->waitAck();
        $this->publishLog($exchange,$routing_key,1);
    }

    /**
     * 批量发布
     * @param array $exchange 交换器名称
     * @param string|array $msg 发布内容
     * @param string $routing_key 路由
     */
    public function batchPublish($msg,$exchange,$routing_key=''){
        if(!is_array($msg)){
            throw new AMQPException("批量推送msg必须要为数组");
        }
        foreach ($msg as  $v) {
            if(is_array($v))$v = json_encode($v);
            $message = new AMQPMessage($v, array('content_type' => 'text/plain', 'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT));
            $this->channel->batch_basic_publish($message,$exchange,$routing_key);
        }
        $this->channel->publish_batch();
        $this->publishLog($exchange,$routing_key,count($msg));
    }

    /**
     * 基础模型之消息接受
     * @param string $exchange
     * @param string $queue
     * @param array $callback
     * @param string $mqtype
     * @return string
     */

    public function basicConsume($queue , $callback, $no_ack = false){
        $this->callback = $callback;
        $this->channel->basic_consume($queue, $this->consumer_tag, false, $no_ack, false, false, [$this,'process_message']);
/*        while(count($this->ch->callbacks)){
            $this->channel->wait();
        }*/
        while (count($this->channel->callbacks)) {
        	$read   = array($this->conn->getSocket()); // add here other sockets that you need to attend
        	$write  = null;
        	$except = null;
        	if (false == ($num_changed_streams = stream_select($read, $write, $except, 60))) {
                throw new AMQPException("队列接受异常或队列消息为空");
        	} elseif ($num_changed_streams > 0) {
        		$this->channel->wait();
        	}
        }
        
    }



    /**
     * Pub/Sub 之批量消息接受，默认接受200条数据
     * @param string $queue 队列名称
     * @param int $limit 返回条数
     * @return array 
     */
    public function batchGet($queue, $limit = 200){

        $messageCount = $this->channel->queue_declare($queue, false, true, false, false);
        $i        = 0;
        $max      = $limit < 200 ? $limit : 200;
        $data     = [];
        while($i < $messageCount[1] && $i < $max){
            $this->msg = $this->channel->basic_get($queue);
            $this->channel->basic_ack($this->msg->delivery_info['delivery_tag']);
            $data[] = $this->msg->body;
            $i++;
        }
        return $data;
    }

    /**
     * 重推消息
     * @param string|int $mid 重推消息id
     * @param string $exchange 交换器名称
     * @param string|array $msg 发布内容
     * @param string $routing_key 路由键 注:在fanout的交换器中路由键会被忽略
     * @return bool
     */

    public function rePublish($mid,$exchange, $msg, $routing_key=''){
        $this->isRePublish = true;
        $this->mid = $mid;
        $msg = is_array($msg) ? json_encode($msg) : $msg;
        $tosend = new AMQPMessage($msg, array('content_type' => 'text/plain', 'delivery_mode' => 2));
        $this->channel->basic_publish($tosend, $exchange, $routing_key);
        $this->waitAck();
        $this->isRePublish = false;//为了防止之后调用其他推送方法出现异常
    }

    /**
     * 销毁队列中的数据
     * @param $msg_obj
     * @return bool
     */
    public function basicAck($msg_obj){
        $this->channel->basic_ack($msg_obj->delivery_info['delivery_tag']);
    }


    /**
     * 推送回调处理
     */
    public function ackHandler(){
        $this->channel->set_ack_handler(
            function (AMQPMessage $message) {
                echo "Message acked with content " . $message->body . PHP_EOL;
            }
        );
        $this->channel->set_nack_handler(
            function (AMQPMessage $message) {
                echo "Message nacked with content " . $message->body . PHP_EOL;
            }
        );
        $this->channel->confirm_select();
    }

    public function waitAck(){
        $this->channel->wait_for_pending_acks();
    }


    /**
     * 默认回调函数
     * @param object $msg_obj
     * @return bool
     */
    public function process_message($msg_obj){
        $rerult = call_user_func($this->callback,$msg_obj);
        if(($rerult['ack'])){
            $this->basicAck($msg_obj);
        }
    }

    /**
     * 关闭消费者
     * @param $msg_obj
     * @return array
     */
    public function cancelConsumer($msg_obj){
        $msg_obj->delivery_info['channel']->basic_cancel($msg_obj->delivery_info['consumer_tag']);
    }

    public function closeConnection(){
        if(is_object($this->channel)){
            $this->channel->close();
        }
        if(is_object($this->conn)){
            $this->conn->close();
        }        
    }

    public function publishLog($exchange,$route,$count){
        echo getNow()."[交换机:{$exchange}][路由:{$route}][数量:$count]".PHP_EOL;
    }


}


```