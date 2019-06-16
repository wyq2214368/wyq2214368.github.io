---
title: PHP 守护进程
date: 2019-02-18
categories: php
tag: daemon
---


## PHP 守护进程 
守护进程是一种运行在后台的特殊进程，因为它不属于任何一个终端，所以不会收到任何终端发来的任何信号。它与前台进程显著的区别是：
* 它没有控制终端，不能直接和用户交互，在后台运行；
* 它不受用户登录和注销的影响，只受开机或关机的影响，可以长期运行；
* 通常我们编写的程序，都需要在 后台不终止的长期运行 ，此时就可以使用守护进程。当然，我们可以在代码中调用系统函数，或者直接在启动命令后追加&操作符，如下：
``` php
$ nohup php server.php start &
```
通常&与 nohup 结合使用，忽略 SIGHUP 信号来实现一个守护进程。该方式对业务代码侵入最小，方便且成本低，常用于临时执行任务脚本的场景。

### 守护进程要点

1. 进程守护化 使用 pcntl_fork()创建子进程，终止父进程,使得程序在 shell 终端里造成一个已经运行完毕的假象,一般会fork2次。

``` php
protected static function daemonize()
{

    $pid = pcntl_fork();
    if (-1 === $pid) {
         exit("process fork fail\n");
    } elseif ($pid > 0) { //父进程直接退出
        exit(0);
    }
    // 将当前进程提升为会话leader
    if (-1 === posix_setsid()) {
        exit("process setsid fail\n");
    }
    //改变工作目录
    chdir('/');
 	//重设文件创建的掩码
    umask(0);
    // 再次fork以避免SVR4这种系统终端再一次获取到进程控制
    $pid = pcntl_fork();
    if (-1 === $pid) {
        exit("process fork fail\n");
    } elseif ($pid > 0) {
        exit(0);
    }
}

```

2. 在子进程中创建新的会话
会话是一个或多个进程组的集合，一个会话有对应的控制终端。
setsid 函数用于创建一个新的会话，并担任该会话组的组长。调用 setsid 的三个作用：让进程摆脱原会话的控制、让进程摆脱原进程组的控制和让进程摆脱原控制终端的控制。
在调用 fork 函数时，子进程全盘拷贝父进程的会话期 (session，是一个或多个进程组的集合)、进程组、控制终端等，虽然父进程退出了，但原先的会话期、进程组、控制终端等并没有改变，因此，那还不是真正意义上使两者独立开来。setsid 函数能够使进程完全独立出来，从而脱离所有其他进程的控制。

3. 改变工作目录
使用 fork 创建的子进程也继承了父进程的当前工作目录。由于在进程运行过程中，当前目录所在的文件系统不能卸载，因此，把当前工作目录换成其他的路径，如 “/” 或 “/tmp” 等。改变工作目录的常见函数是 chdir。

4. 重设文件创建掩码
文件创建掩码是指屏蔽掉文件创建时的对应位。由于使用 fork 函数新建的子进程继承了父进程的文件创建掩码，这就给该子进程使用文件带来了诸多的麻烦。因此，把文件创建掩码设置为 0，可以大大增强该守护进程的灵活性。设置文件创建掩码的函数是 umask，通常的使用方法为 umask (0)。

5. 重定向标准输入输出
用 fork 新建的子进程会从父进程那里继承一些已经打开了的文件。这些被打开的文件可能永远不会被守护进程读或写，但它们一样消耗系统资源，可能导致所在的文件系统无法卸载。
``` php
protected static function resetStdFd()
{
    global $STDIN, $STDERR, $STDOUT;
    //重定向标准输出和错误输出
    @fclose(STDIN);
    @fclose(STDOUT);
    @fclose(STDERR);
    $STDIN  = fopen('/dev/null', 'r');
    $STDOUT = fopen(static::$stdoutFile, 'a');
    $STDERR = fopen(static::$stdoutFile, 'a');
}
```
如果你关闭了标准输出，标准错误输出文件描述符，那么你打开的前三个文件描述符将成为新的标准输入、输出、错误的描述符。
使用$STDIN, $STDOUT纯粹是障眼法而已, 必须指定为全局变量，否则文件描述符将在函数执行完毕之后被释放。

6. 信号处理
在 Linux 系统中，可使用kill -l命令查看这 62 个信号值,使用信号来实现进程间通信并控制进程的行为，注册信号处理器如下：

``` php
function installSignal()
{
    pcntl_signal(SIGINT,  'signalHandler', false);
    pcntl_signal(SIGTERM, 'signalHandler', false);

    pcntl_signal(SIGUSR1, 'signalHandler', false);
    pcntl_signal(SIGQUIT, 'signalHandler', false);

    // 忽略信号
    pcntl_signal(SIGUSR2, SIG_IGN, false);
    pcntl_signal(SIGHUP,  SIG_IGN, false);
}

function signalHandler($signal)
{
    switch($signal) {
        case SIGINT:
        case SIGTERM:
            static::stop();
            break;
        case SIGQUIT:
        case SIGUSR1:
            static::reload();
            break;
        default: break;
    }
}
```
其中，SIGINT 和 SIGTERM 信号会触发stop操作，即终止所有进程；SIGQUIT 和 SIGUSR1 信号会触发reload操作，即重新加载所有 Worker 进程；此处忽略了 SIGUSR2 和 SIGHUP 信号，但是并未忽略 SIGKILL 信号，即所有进程都可以被强制kill掉。

### Show My Code

``` php
class Daemon
{

    protected $daemonize = false;//是否进程守护化 

    protected $stdoutFile = '/dev/daemon.log'; //重定向标准输出文件 设置之后可以查看守护进程的错误

    protected $pidFile; //子进程pid文件

    const PIDPATH = '/var/run/'; //默认的进程pid保存路径

    public function __construct()
    {
        $this->setPidFile();
    }

    /**
     * 创建守护进程核心函数
     * @return string|void
     */
    private function daemonize()
    {
        if(!$this->daemonize){
            return;
        }
        $this->checkPcntl();
        //创建子进程
        $pid = pcntl_fork();
        if ($pid == -1) {
            exit("process fork fail\n");
        } elseif ($pid) {
            //终止父进程
            exit(0);
        }

        //在子进程中创建新的会话
        if (posix_setsid() === -1) {
            die('Could not detach');
        }

        // 再次fork以避免SVR4这种系统终端再一次获取到进程控制
        $pid = pcntl_fork();
        if ($pid == -1) {
            exit("process fork fail\n");
        } elseif (0 !== $pid) {
            exit(0);
        }
        
        chdir('/');//改变工作目录
        umask(0);//重设文件创建的掩码
        $this->saveMasterPid();//保存pid
        $this->setProcessTitle();//设置进程名字
        $this->resetStdFd();//关闭文件描述符
        return;
    }

    /**
     * 守护进程的任务，子类重写job，执行自定义方法
     */
    public function job()
    {
        //TODO 你的守护经常需要执行的任务
        while (true) {
            // echo 'job process'.PHP_EOL;
            file_put_contents('/var/job.log', 'do job' . PHP_EOL, FILE_APPEND);
            sleep(2);
        }
    }

    /**
     * 设置 进程pid保存文件 之类可以重新自定义pidFile路径
     * @return void
     */    
    public function setPidFile($file=''){
        if(empty($file)){
            $this->pidFile = static::PIDPATH.get_called_class().'_server.pid'; //get_called_class 继承之后获取的是之类的名字,或者使用后期静态绑定
        }else{
            $this->pidFile = $file;
        }
        
    }

    /**
     * 保存pid以实现stop
     */
    protected  function saveMasterPid()
    {
        $fp = fopen($this->pidFile, 'w') or die("Can't create pid file");
        //把当前进程的id写入到文件中
        fwrite($fp, posix_getpid());
        fclose($fp);
    }

    /**
     * 获取守护进程的id
     * @return int
     */
    private function getPid()
    {
        //判断存放守护进程id的文件是否存在
        if (!file_exists($this->pidFile)) {
            return 0;
        }
        $pid = intval(file_get_contents($this->pidFile));
        if (posix_kill($pid, SIG_DFL)) {//判断该进程是否正常运行中
            return $pid;
        } else {
            unlink($this->pidFile);
            return 0;
        }
    }

    /**
     * 关闭标准输出和错误输出.
     */
    protected  function resetStdFd()
    {
        global $STDERR, $STDOUT;
        //重定向标准输出和错误输出
        @fclose(STDOUT);
        @fclose(STDERR);
        $STDOUT = fopen($this->stdoutFile, 'a');
        $STDERR = fopen($this->stdoutFile, 'a');
    }

    /**
     * 设置进程名.
     *
     * @param string $title 进程名.
     */
    protected static function setProcessTitle($title='')
    {
        if(empty($title)){
            $title = get_called_class().'Server';
        }
        if (extension_loaded('proctitle') && function_exists('setproctitle')) {
            @setproctitle($title);
        } elseif (version_compare(phpversion(), "5.5", "ge") && function_exists('cli_set_process_title')) {
            @cli_set_process_title($title);
        }
    }

    /**
     * 判断pcntl拓展
     */
    private function checkPcntl()
    {
        !function_exists('pcntl_signal') && die('Error:Need PHP Pcntl extension!');
    }

    private function message($message)
    {
        printf("%s  %d  %s" . PHP_EOL, date("Y-m-d H:i:s"), $this->getPid(), $message);
    }

    /**
     * 开启守护进程
     */
    private function start()
    {
        if ($this->getPid() > 0) {
            $this->message('Running');
        } else {
            $this->daemonize();
            $this->message('Start');
            $this->job();
        }
    }

    /**
     * 停止守护进程
     */
    private function stop()
    {
        $pid = $this->getPid();
        if ($pid > 0) {
            //通过向进程id发送终止信号来停止进程
            posix_kill($pid, SIGTERM);
            unlink($this->pidFile);
            echo $pid.' Stoped' . PHP_EOL;
        } else {
            echo "Not Running" . PHP_EOL;
        }
    }

    private function status()
    {
        if ($this->getPid() > 0) {
            $this->message('Is Running');
        } else {
            echo 'Not Running' . PHP_EOL;
        }
    }

    public function run()
    {
        global $argv;
        $command = isset($argv[1]) ? $argv[1] : '';
        $command2 = isset($argv[2]) ? $argv[2] : '';
        switch ($command) {
            case 'start':
                if ($command2 === '-d') {
                    $this->daemonize = true;
                }
                $this->start();
                break;
            case 'stop':
                $this->stop();
                break;
            case 'status':
                $this->status();
                break;
            default:
                echo "Argv request start|stop|status " . PHP_EOL;
                break;
        }
    }

}
``` 
* php deamon.php  start  正常运行
* php deamon.php  start  -d  进程守护化运行
* php deamon.php  status  查看进程运行状态
* php deamon.php  stop 停止运行

### 使用
将Daemon作为基类,子类继承Deamon自定义job
``` php
class Work extends Daemon 
{
    public function job(){
        //TODO 你的守护经常需要执行的任务
        while (true) {
            // echo 'job process'.PHP_EOL;
            file_put_contents('/var/job.log', 'do work job' . PHP_EOL, FILE_APPEND);
            sleep(2);
        }
    }
}


$work = new Work();
$work->run();
```