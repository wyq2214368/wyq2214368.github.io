---
title: Swoole 协程
date: 2019-04-29 
categories: php
tag: swoole
---
## Swoole 协程

协程可以理解为纯用户态的线程，其通过协作而不是抢占来进行切换。相对于进程或者线程，协程所有的操作都可以在用户态完成，创建和切换的消耗更低。协程主要用于优化IO操作频繁的任务,当然这个IO需要使用异步IO，能够yeild的异步IO。

### yield 实现协程多任务调度
这里有两篇分享很好讲诉了使用yeild来实现生成器，从而实现协程多任务调度，[PHP 多任务协程处理](https://learnku.com/articles/14482/php-multitask-co-process)，[PHP 协程实现](https://newt0n.github.io/2017/02/10/PHP-%E5%8D%8F%E7%A8%8B%E5%8E%9F%E7%90%86/ )，借花献佛哈哈。主要分以下两步。
这个和Python的asyncio协程实现很像。asyncio.event_loop:程序开启一个无限循环，把一些函数注册到事件循环上，当满足事件发生的时候，调用相应的协程函数。asyncio.task:一个协程对象就是一个原生可以挂起的函数，任务则是对协程进一步封装，其中包含了任务的各种状态。

#### Task 
Task 是普通生成器的装饰器。我们将生成器赋值给它的成员变量以供后续使用，然后实现一个简单的 run() 和 finished() 方法。run() 方法用于执行任务，finished() 方法用于让调度程序知道何时终止运行。
``` php
class Task
{
    protected $generator;

    protected $run = false;

    public function __construct(Generator $generator)
    {
        $this->generator = $generator;
    }

    public function run() 
    {
        if ($this->run) { //判断是否是第一次run,第一次用next那直接会跑到第二个yield
            $this->generator->next();
        } else {
            $this->generator->current();
        }

        $this->run = true;
    }

    public function finished()
    {
        return !$this->generator->valid();
    }
}
```
#### Scheduler 
Scheduler 用于维护一个待执行的任务队列。run() 会弹出队列中的所有任务并执行它，直到运行完整个队列任务。如果某个任务没有执行完毕，当这个任务本次运行完成后，我们将再次入列。
``` php
class Scheduler
{
    protected $queue;

    public function __construct()
    {
        $this->queue = new SplQueue(); //FIFO 队列
    }

    public function enqueue(Task $task)
    {
        $this->queue->enqueue($task);
    }

    public function run()
    {
        while (!$this->queue->isEmpty()) {
            $task = $this->queue->dequeue();
            $task->run();

            if (!$task->finished()) {
                $this->queue->enqueue($task);
            }
        }
    }
}
```
#### 使用
``` php
$scheduler = new Scheduler();

$task1 = new Task(call_user_func(function() {
    for ($i = 0; $i < 3; $i++) {
        print "task1: " . $i . "\n";
        yield sleep(1); //挂起IO操作
    }
}));

$task2 = new Task(call_user_func(function() {
    for ($i = 0; $i < 6; $i++) {
        print "task2: " . $i . "\n";
        yield sleep(1); //挂起IO操作
    }
}));

$scheduler->enqueue($task1);
$scheduler->enqueue($task2);
$startTime = microtime(true);
$scheduler->run();
print "用时: ".(microtime(true) - $startTime);
```
#### 执行结果
交替执行,task1执行到yeild交出控制权，轮到task2执行到yeild再交出控制权，再一次轮到task1，直到task1执行完，队列里只剩下task2自我陶醉了。
虽然执行结果是这样的，但是效果并不是我们想要的,执行了9秒那和我们同步执行有什么区别，因为sleep()是同步阻塞的，接下来我们把sleep换一下。
``` php
task1: 0
task1: 1
task2: 0
task2: 1
task1: 2
task2: 2
task2: 3
task2: 4
task2: 5
用时: 9.0115599632263
```
#### 异步sleep
需要用到swoole,co::sleep()是swoole自带的异步sleep,go()是 [swoole协程](https://wiki.swoole.com/wiki/page/p-coroutine.html) 的创建命令
``` php
function async_sleep($s){
	return  go(function ()use($s)  {
			    co::sleep($s); // 模拟请求接口、读写文件等I/O
			}); 
}

$scheduler = new Scheduler();

$task1 = new Task(call_user_func(function() {
    for ($i = 0; $i < 3; $i++) {
        print "task1: " . $i . "\n";
        yield async_sleep(1);
    }
}));

$task2 = new Task(call_user_func(function() {
    for ($i = 0; $i < 6; $i++) {
        print "task2: " . $i . "\n";
        yield async_sleep(1);
    }
}));

$scheduler->enqueue($task1);
$scheduler->enqueue($task2);
$startTime = microtime(true);
$scheduler->run();
print "用时: ".(microtime(true) - $startTime);
```
执行结果，这应该就我们想要的IO操作异步并发,一共9个IO实际时间=1个IO,如果这个异步IO是异步mysql，异步http等就大大提升了我们脚本的并发能力
``` php
task1: 0
task2: 0
task1: 1
task2: 1
task1: 2
task2: 2
task2: 3
task2: 4
task2: 5
用时: 1.0025930404663
```
### Swoole 协程
从4.0版本开始Swoole提供了完整的协程(Coroutine)+通道(Channel)特性。应用层可使用完全同步的编程方式，底层自动实现异步IO。这句话是[swoole](https://wiki.swoole.com/wiki/page/p-coroutine.html)说的。
``` php
for ($i = 0; $i < 10; ++$i) {
    // swoole 创建协程
    go(function () use ($i) {
        co::sleep(1.0); // 模拟异步请求接口、读写文件等I/O
        var_dump($i);
    });
}
swoole_event_wait(); //异步阻塞等所有协程完成任务
print "协程用时: ".(microtime(true) - $time);
```
运行时间是1秒这里就不多说了。协程之所以快是因异步IO可以yield，但是我们平常使用的mysql请求，http请求等都是同步的,就算使用协程调度也提升不了并发，这不swooleg提供了我们想要的东东。


#### Swoole 协程MySQL客户端

swoole的[Coroutine\\MySQL](https://wiki.swoole.com/wiki/page/p-coroutine_mysql.html)具体操作可以看这里,代码中举了异步和同步的mysql请求和并发试一下， dump需要引入symfony,方便打印对象的结构。

``` php
//异步mysql
function asyncMysql(){

    go(function () {
        $db = new \Swoole\Coroutine\Mysql();
        $server = array(
            'host' => '127.0.0.1',
            'user' => 'root',
            'password' => '123456',
            'database' => 'test',
            'port' => '3306',
        );

        $db->connect($server); //异步

        $result = $db->query('select * from users limit 1');
        // dump( $result);
    });

}
//同步msql
function synMysql(){
    $servername = "127.0.0.1";
    $username = "root";
    $password = "123456";
    $dbname = "test";
    
    $conn = mysqli_connect($servername, $username, $password, $dbname);

    if (!$conn) {
        die("连接失败: " . mysqli_connect_error());
    }
     
    $sql = "select * from users limit 1";
    $result = mysqli_query($conn, $sql);
     
    if (mysqli_num_rows($result) > 0) {
        while($row = mysqli_fetch_assoc($result)) {
            // dump($row);
        }
    } else {
        echo "0 结果";
    }
     
    mysqli_close($conn);
}

$startTime = microtime(true);

for($i=0;$i<100;$i++){
    asyncMysql();
}
swoole_event_wait();
$endTime = microtime(true);

dump($endTime-$startTime);

异步所花时间
0.029722929000854
0.017247200012207
0.029895067214966
0.024247884750366
同步所花时间
0.086297988891602
0.083254814147949
0.0831139087677
0.083254814147949

```
看运行时间不太对哈,这个怎么差了这么一点。我想的是这样的哈，Coroutine\MySQL 上面的例子异步IO操作应该是 connect 和 query，其他的例如创建客户端那就是同步操作了，这个消耗是同步阻塞的,而且占了比例不小,所以才出现这样的情况。
那想一下我们是不是可以这样写，把mysql异步客服端直接拿出来让协程共享。

``` php
function asyncMysql(){
    go(function(){
        $db = new \Swoole\Coroutine\Mysql();
        $server = array(
            'host' => '127.0.0.1',
            'user' => 'root',
            'password' => '4QqRbtNCc3LnHko4LQ9H',
            'database' => 'tracknumer_share',
            'port' => '3306',
        );   
        $db->connect($server); 
        $startTime = microtime(true);
        for($i=0;$i<10;$i++){
            go(function ()use($db) {
                $result = $db->query('select * from users limit 1');
            });
        }
        swoole_event_wait();
        $endTime = microtime(true);
        dump($endTime-$startTime);
    });
}
[2019-04-30 11:23:36 @4769.0]	ERROR	check_bind (ERROR 10002): mysql client has already been bound to another coroutine#2, reading or writing of the same socket in multiple coroutines at the same time is not allowed.
Stack trace:
#0  Swoole\Coroutine\MySQL->query() called at [/data/web/dev/swoole-demo/src/Coroutine/mysql.php:44]

``` 
哦天哪发生了什么，报错了，它说这个mysql客户端已经有其他协程占用了。是我太天真的了。官网说swoole这样做是为了防止多个协程同一时刻使用同一个客户端导致数据错乱。
那我们就简单实现一个mysql的连接池,复用协程客户端，实现长连接。

#### Swoole 协程MySQL连接池
``` php
<?php 
require __DIR__ . '/../bootstrap.php';
class MysqlPool
{
    protected $available = true;
    public $pool;
    protected $config; //mysql服务的配置文件
    protected $max_connection = 100;//连接池大小 
    protected $current_connection = 0;//当前链接池数

    public function __construct($config)
    {
        $this->config = $config;
        $this->pool   = new SplQueue;
    }

    public function put($mysql)
    {
        $this->pool->push($mysql);
    }

    /**
     * @return bool|mixed|\Swoole\Coroutine\Mysql
     */
    public function get()
    {
        //有空闲连接且连接池处于可用状态
        if ($this->available && $this->pool->length > 0) {
            return $this->pool->pop();
        }

        //无空闲连接，创建新连接
        $mysql = $this->newMysqlClient();
        if ($mysql == false) {
            return false;
        } else {
            return $mysql;
        }
    }

    protected function newMysqlClient()
    {

        if($this->current_connection >= $this->max_connection){
            throw new Exception("链接池已经满了"); 
        }
        $this->current_connection++;
        $mysql = new Swoole\Coroutine\Mysql();
        $mysql->connect($this->config); 
        return $mysql;
    }

    public function destruct()
    {
        // 连接池销毁, 置不可用状态, 防止新的客户端进入常驻连接池, 导致服务器无法平滑退出
        $this->available = false;
        while (!$this->pool->isEmpty()) {
            go(function(){
                $mysql = $this->pool->pop();
                $mysql->close();
            });
        }
    }

    public function __destruct(){
        $this->destruct();
    }
}

$config = array(
            'host' => '127.0.0.1',
            'user' => 'root',
            'password' => '123456',
            'database' => 'test',
            'port' => '3306',
        );

$pool = new MysqlPool($config);
``` 
好了，一个简单的连接池已经搞好了,我先用一下
``` php

go(function()use($config){
    $pool = new MysqlPool($config);
    for($i=0;$i<2;$i++){
        go(function ()use($pool) {
            $mysql = $pool->get();
            $result = $mysql->query('select * from users limit 1');
            dump($result);
            $pool->put($mysql);
        });
    }
    dump($pool);

});

``` 
好了结果出来了，新增一个defer()，在协程推出之前释放连接池的资源。
``` php

go(function()use($pool){ 
    $pool = new MysqlPool($config);
    defer(function () use ($pool) { //用于资源的释放, 会在协程关闭之前(即协程函数执行完毕时)进行调用, 就算抛出了异常, 已注册的defer也会被执行.
        echo "Closing connection pool\n";
        $pool->destruct();
    });
    for($i=0;$i<2;$i++){
        go(function ()use($pool) {
            $mysql = $pool->get();
            $result = $mysql->query('select * from users limit 1');
            dump($result);
            $pool->put($mysql);
        });
    }
     dump($pool);
});
``` 
这个有一个比较完善的 [协程客户端链接池包](https://github.com/open-smf/connection-pool)

#### Swoole 协程 Channel 实现并发数据收集
这里使用子协程+通道来并发收集数据，理想的情况是使用连接池,但是会遇到问题。
``` php
//每个子进程创建一个mysql连接
go(function()use($pool,$config){
    $chan = new chan(10);
    for($i=0;$i<2;$i++){
        go(function()use($pool,$chan,$config){
            $mysql = new \Swoole\Coroutine\Mysql();
            $mysql->connect($config); 
            $result = $mysql->query('select * from users limit 1');
            $chan->push($result);
            $mysql->close();
        });
    }

    for($i=0;$i<2;$i++){
        dump($chan->pop());//这个pop()如果遇到空会yield,直到子协程的push()数据之后才会重新唤醒
    }

});
//使用连接池
$pool = new MysqlPool($config);
go(function()use($pool,$config){
    $chan = new chan(10);
    for($i=0;$i<2;$i++){
        go(function()use($pool,$chan,$config){
            $mysql = $pool->get();
            $result = $mysql->query('select * from users limit 1');
            $chan->push($result);
            $pool->put($mysql);//这里如果不put回去，脚本就不会阻塞，不知道为啥，希望有大佬解惑！不put回去就要mysql->close()，等于每次都新建连接
        });
    }
    for($i=0;$i<2;$i++){
        dump($chan->pop());//这个pop()如果遇到空会yield,直到子协程的push()数据之后才会重新唤醒
    }

});
``` 
过了一圈swoole协程感觉还是没有Python的asyncio包好用，有些地方总是搞不明白。