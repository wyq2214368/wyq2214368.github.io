---
title: PHP多进程
date: 2019-02-15
categories: php
tag: fork
---


### PHP多进程 
* pcntl_fork()函数成功执行时会在父进程返回子进程的进程id(pid)，因为系统的初始进程init进程的pid为1，后来产生进程的pid都会大于此进程，所以我们可以通过判断pcntl_fork()的返回值大于1来确实当前进程是父进程；
而在子进程中，此函数的返回值会是固定值0，我们也可以通过判断pcntl_fork()的返回值为0来确定子进程；
* pcntl_fork()函数在执行失败时，会在父进程返回-1,当然也不会有子进程产生。

#### 简单的子进程生成例子
``` php
$pid = pcntl_fork();
 
if($pid == -1) {
    //错误处理：创建子进程失败时返回-1.
    die('fork error');
} else if ($pid) {
    //父进程会得到子进程号，所以这里是父进程执行的逻辑
    echo "parent \n";
    //等待子进程中断，防止子进程成为僵尸进程。
    pcntl_wait($status);
} else {
    //子进程得到的$pid为0, 所以这里是子进程执行的逻辑。
    echo "child \n";
 
    exit;
}
```
#### 多进程生成例子

``` php
define('FORK_NUMS', 3);
 
$pids = array();
 
for($i = 0; $i < FORK_NUMS; ++$i) {
    $pids[$i] = pcntl_fork();
    if($pids[$i] == -1) {
        die('fork error');
    } else if ($pids[$i]) {
        pcntl_waitpid($pids[$i], $status); //等待指定进程退出 
        echo "pernet \n";
    } else {
        sleep(3);
        echo "child id:" . getmypid() . " \n";
        exit;  //在子进程中,需通过exit来退出，不然会产生递归多进程,父进程中不需要exit,不然会中断多进程。
    }
}
```

#### 多进程写入文件

``` php
define('FORK_NUMS', 3);
$pids = array();
 
$fp = fopen('./test.log', 'wb');
$num = 1;
 
for($i = 0; $i < FORK_NUMS; ++$i) {
    $pids[$i] = pcntl_fork();
    if($pids[$i] == -1) {
        die('fork error');
    } else if ($pids[$i]) {
 
    } else {
        for($i = 0; $i < 5; ++$i) {
            flock($fp, LOCK_EX);  //并发文件锁
            fwrite($fp, getmypid() . ' : ' . date('Y-m-d H:i:s') . " : {$num} \r\n");
            flock($fp, LOCK_UN);
            echo getmypid(), ": success \r\n";
            ++$num;
        }
        exit;
    }
}
 
foreach($pids as $k => $v) {
    if($v) {
        pcntl_waitpid($v, $status); //等待子进程完成
    }
}
fclose($fp);
```
#### 多进程生产封装

##### 用方法生产进程

``` php
function createProcess($callback){
    $pid = pcntl_fork();
    if($pid < 0){
        die('fork error');
    }else if($pid > 0){
        return $pid ;
    }else{
        $pid = posix_getpid();
        $callback();
        exit("child {$pid} process end");
    }
}

function handle($args){
    print_r($args);
    echo 'handle'.PHP_EOL;
    sleep(3);
}

for($i=0;$i<2;$i++){
    $childList[] = createProcess('handle');
}
//阻塞等子进程结束
foreach ($childList as $v) {
    if($v){
        pcntl_waitpid($v, $status);
    }
}
print_r($childList);

```

##### 用类生产进程

``` php
class Process  
{
    private $callback; //进程处理方法
    private $args;//进程处理参数
    private $pid;//子进程pid
    private $ppid;//主进程id

    function __construct($callback,$args='')
    {
        $this->ppid     = posix_getpid();
        $this->callback = $callback;
        $this->args     = $args;
    }
    /**
     * 生成子进程
    */
    public function run(){
        $pid = pcntl_fork();
        if($pid < 0){
            throw new Exception("fork  error");
        }else if($pid > 0){
            $this->pid = $pid;
            return $pid;
        }else{
            posix_kill($this->ppid,0);//检查主进程是否存在 存在则返回true
            $callback = $this->callback;
            $callback($this->args);
            exit();
            
        }
    }

    /**
     * 同步阻塞等子进程结束
    */
    public function join(){
        return pcntl_waitpid($this->pid, $status);
    }
}

function handle($args){
    print_r($args);
    echo 'handle'.PHP_EOL;
    sleep(3);
}

$p = [];
for($i=0;$i<2;$i++){
    $cp  = new Process('handle',[1,2,3]);
    $cp->run();
    $p[] = $cp;
}

foreach ($p as $v) {
    $v->join();
}

echo '主进程执行完毕';

```

### 进程间通信（IPC）
通常linux中的进程通信方式有：消息队列、信号量、共享内存、信号、管道、socket。[转载](https://www.jianshu.com/p/08bcf724196b)

#### 消息队列
消息队列是存放在内存中的一个队列。如下代码将创建3个生产者子进程，2个消费者子进程。这5个进程将通过消息队列通信。
由于消息队列取数据是原子性的,所以不需要锁或者信号量

``` php
$parentPid = posix_getpid();
echo "parent progress pid:{$parentPid}\n";$childList = array();
// 创建消息队列,以及定义消息类型(类似于数据库中的库)
$id = ftok(__FILE__,'m');
$msgQueue = msg_get_queue($id);
const MSG_TYPE = 1;
// 生产者
function producer(){
    global $msgQueue;
    $pid = posix_getpid();
    $repeatNum = 5;
    for ( $i = 1; $i <= $repeatNum; $i++) {
        $str = "({$pid})progress create! {$i}";
        msg_send($msgQueue,MSG_TYPE,$str);
        $rand = rand(1,3);
        sleep($rand);
    }
}
// 消费者
function consumer(){
    global $msgQueue;
    $pid = posix_getpid();
    $repeatNum = 6;
    for ( $i = 1; $i <= $repeatNum; $i++) {
        $rel = msg_receive($msgQueue,MSG_TYPE,$msgType,1024,$message);
        echo "{$message} | consumer({$pid}) destroy \n";
        $rand = rand(1,3);
        sleep($rand);
    }
}
function createProgress($callback){
    $pid = pcntl_fork();
    if ( $pid == -1) {
        // 创建失败
        exit("fork progress error!\n");
    } else if ($pid == 0) {
        // 子进程执行程序
        $pid = posix_getpid();
        $callback();
        exit("({$pid})child progress end!\n");
    }else{
        // 父进程执行程序
        return $pid;
    }
}
// 3个写进程
for ($i = 0; $i < 3; $i ++ ) {
    $pid = createProgress('producer');
    $childList[$pid] = 1;
    echo "create producer child progress: {$pid} \n";
}
// 2个写进程
for ($i = 0; $i < 2; $i ++ ) {
    $pid = createProgress('consumer');
    $childList[$pid] = 1;
    echo "create consumer child progress: {$pid} \n";
}
// 等待所有子进程结束
while(!empty($childList)){
    $childPid = pcntl_wait($status);
    if ($childPid > 0){
        unset($childList[$childPid]);
    }
}
echo "({$parentPid})main progress end!\n";
```

#### 信号

信号是一种系统调用。通常我们用的kill命令就是发送某个信号给某个进程的。具体有哪些信号可以在liunx/mac中运行kill -l查看。
php 中发送信号用 posix_kill($pid,SIGINT);
下面这个例子中，父进程等待5秒钟，向子进程发送sigint信号。子进程捕获信号，掉信号处理函数处理。

``` php
$parentPid = posix_getpid();
echo "parent progress pid:{$parentPid}\n";

// 定义一个信号处理函数
function sighandler($signo) {
    $pid = posix_getpid();
    echo "{$pid} progress,oh no ,I'm killed!\n";
    exit(1);
}

$pid = pcntl_fork();
if ( $pid == -1) {
    // 创建失败
    exit("fork progress error!\n");
} else if ($pid == 0) {
    // 子进程执行程序
    // 注册信号处理函数
    declare(ticks=10);
    pcntl_signal(SIGINT, "sighandler");
    $pid = posix_getpid();
    while(true){
        echo "{$pid} child progress is running!\n";
        sleep(1);
    }
    exit("({$pid})child progress end!\n");
}else{
    // 父进程执行程序
    $childList[$pid] = 1;
    // 5秒后,父进程向子进程发送sigint信号.
    sleep(5);
    posix_kill($pid,SIGINT);
    sleep(5);
}
echo "({$parentPid})main progress end!\n";
```

####  信号量与共享内存

* 信号量：是系统提供的一种原子操作，一个信号量，同时只有你个进程能操作。一个进程获得了某个信号量，就必须被该进程释放掉。
* 共享内存：是系统在内存中开辟的一块公共的内存区域，任何一个进程都可以访问，在同一时刻，可以有多个进程访问该区域，为了保证数据的一致性，需要对该内存区域加锁或信号量。

``` php
$parentPid = posix_getpid();
echo "parent progress pid:{$parentPid}\n";
$childList = array();

// 创建共享内存,创建信号量,定义共享key
$shm_id = ftok(__FILE__,'m');
$sem_id = ftok(__FILE__,'s');
$shareMemory = shm_attach($shm_id);
$signal = sem_get($sem_id);
const SHARE_KEY = 1;
// 生产者
function producer(){
    global $shareMemory;
    global $signal;
    $pid = posix_getpid();
    $repeatNum = 5;
    for ( $i = 1; $i <= $repeatNum; $i++) {
        // 获得信号量
        sem_acquire($signal);
        
        if (shm_has_var($shareMemory,SHARE_KEY)){
            // 有值,加一
            $count = shm_get_var($shareMemory,SHARE_KEY);
            $count ++;
            shm_put_var($shareMemory,SHARE_KEY,$count);
            echo "({$pid}) count: {$count}\n";
        }else{
            // 无值,初始化
            shm_put_var($shareMemory,SHARE_KEY,0);
            echo "({$pid}) count: 0\n";
        }
        // 用完释放
        sem_release($signal);
        
        $rand = rand(1,3);
        sleep($rand);
    }
}
function createProgress($callback){
    $pid = pcntl_fork();
    if ( $pid == -1) {
        // 创建失败
        exit("fork progress error!\n");
    } else if ($pid == 0) {
        // 子进程执行程序
        $pid = posix_getpid();
        $callback();
        exit("({$pid})child progress end!\n");
    }else{
        // 父进程执行程序
        return $pid;
    }
}
// 3个写进程
for ($i = 0; $i < 3; $i ++ ) {
    $pid = createProgress('producer');
    $childList[$pid] = 1;
    echo "create producer child progress: {$pid} \n";
}
// 等待所有子进程结束
while(!empty($childList)){
    $childPid = pcntl_wait($status);
    if ($childPid > 0){
        unset($childList[$childPid]);
    }
}
// 释放共享内存与信号量
shm_remove($shareMemory);
sem_remove($signal);
echo "({$parentPid})main progress end!\n";
```

#### 管道

管道是比较常用的多进程通信手段，管道分为无名管道与有名管道，无名管道只能用于具有亲缘关系的进程间通信，而有名管道可以用于同一主机上任意进程。这里只介绍有名管道。下面的例子，子进程写入数据，父进程读取数据。

``` php
// 定义管道路径,与创建管道
$pipe_path = '/data/test.pipe';
if(!file_exists($pipe_path)){
    if(!posix_mkfifo($pipe_path,0664)){
        exit("create pipe error!");
    }
}
$pid = pcntl_fork();
if($pid == 0){
    // 子进程,向管道写数据
    $file = fopen($pipe_path,'w');
    while (true){
        fwrite($file,'hello world');
        $rand = rand(1,3);
        sleep($rand);
    }
    exit('child end!');
}else{
    // 父进程,从管道读数据
    $file = fopen($pipe_path,'r');
    while (true){
        $rel = fread($file,20);
        echo "{$rel}\n";
        $rand = rand(1,2);
        sleep($rand);
    }
}
```
