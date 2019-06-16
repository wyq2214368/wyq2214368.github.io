---
title: Websokcet 实现mini聊天室
date: 2019-05-21
categories: php
tag: websokcet
---

前端经常需要与服务器进行持续的通讯以保持双方信息的同步，long long ago 我们会使用长轮询的方式来实现比如:
``` javascript
setInterval(function() {
    $.get("/api/getData", function(data, status) {
        console.log(data);
    });
}, 10000);
```
每隔10秒向服务器同步一次数据，这种方式缺点很明显:
1. 每次请求都需要新建HTTP连接
2. 只能是客户端向服务器的单向请求，服务器无法主动请求。
3. 无法即时更新，就算将时间间隔调整为1秒，同时有1万个客户端打开，服务器也受不了。

那有没有一种像打电话一样，接通了之后双方有事就可以直接通知的呢，websocket就很好的满足了这个需求。

## websokcet

> WebSocket 是基于 TCP 的独立的协议。它与 HTTP 唯一的关系是它的握手是由 HTTP 服务器解释为一个 Upgrade 请求

> WebSocket 的最大特点就是，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息，是真正的双向平等对话，属于服务器推送技术的一种。其他特点包括：

* 建立在 TCP 协议之上，服务器端的实现比较容易。
* 与 HTTP 协议有着良好的兼容性。默认端口也是 80 和 443 ，并且握手阶段采用 HTTP 协议，因此握手时不容易屏蔽，能通过各种 HTTP 代理服务器。
* 数据格式比较轻量，性能开销小，通信高效。
* 可以发送文本，也可以发送二进制数据。
* 没有同源限制，客户端可以与任意服务器通信。
* 协议标识符是ws（如果加密，则为wss），服务器网址就是 URL。
* 推荐这篇文章  [WebSocket 详解](https://segmentfault.com/a/1190000012948613#articleHeader1)

### websokcet 客户端

* Chrome/Firefox/高版本IE/Safari等浏览器内置了JS语言的WebSocket客户端
* 微信小程序开发框架内置的WebSocket客户端
* 异步的PHP程序中可以使用Swoole\Http\Client作为WebSocket客户端
* apache/php-fpm或其他同步阻塞的PHP程序中可以使用swoole/framework提供的[同步WebSocket客户端](https://github.com/swoole/framework/blob/master/libs/Swoole/Client/WebSocket.php)
* 非WebSocket客户端不能与WebSocket服务器通信


#### 简易的js websokcet 客户端

``` bash
<!DOCTYPE html>
<html>
<head>
    <title></title>
    <meta charset="UTF-8">
    <script type="text/javascript">
        if(window.WebSocket){
            var webSocket = new WebSocket("ws://10.201.10.242:5200");
            webSocket.onopen = function (event) {
                //webSocket.send("Hello,WebSocket!");
            };
            webSocket.onmessage = function (event) {
                var content = document.getElementById('content');
                if(event.data instanceof Blob) {
                    var img = document.createElement("img");
                    img.src = window.URL.createObjectURL(event.data);
                    content.appendChild(img);
                }else {
                    content.innerHTML = content.innerHTML.concat('<p style="margin-left:20px;height:20px;line-height:20px;">'+event.data+'</p>');
                }
            };

            var sendMessage = function(){
                var data = document.getElementById('message').value;
                webSocket.send(data);
                document.getElementById('message').value = '';
            }
        }else{
            console.log("您的浏览器不支持WebSocket");
        }
    </script>
</head>
<body>
<div style="width:600px;margin:0 auto;border:1px solid #ccc;">
    <div id="content" style="overflow-y:auto;height:300px;"></div>
    <hr/>
    <div style="height:40px">
        <input type="text" id="message" style="margin-left:10px;height:25px;width:450px;">
        <button onclick="sendMessage()" style="height:28px;width:75px;">发送</button>
    </div>
</div>
</body>
</html>
```


### websokcet 服务端
- 使用 swoole的`swoole_websocket_server`  来搭建websokcet的服务端
- 使用 swoole的内存表`swoole_table` 来存储数据，线上项目还是建议使用redis

``` php
<?php

class WebSocketServer
{
    private $config;
    private $table;
    private $server;

    /**
     * @param array $config 配置文件 ['host'=>'' , 'port'=>'']
     */
    public function __construct($config)
    {
        // 内存表 实现进程间共享数据，也可以使用redis替代
        $this->createTable();
        // 实例化配置
        $this->config = $config；
    }

    public function run()
    {
        $this->server = new Swoole\WebSocket\Server(
            $this->config['host'],
            $this->config['port']
        );

        $this->server->on('open', [$this, 'open']);
        $this->server->on('message', [$this, 'message']);
        $this->server->on('close', [$this, 'close']);

        $this->server->start();
    }

    public function open(Swoole\WebSocket\Server $server, Swoole\Http\Request $request)
    {
        $user = [
            'fd' => $request->fd,
            'name' => 'S'. $request->fd, // 这里就随便写了,做登陆设计会有身份
        ];
        // 放入内存表
        $this->table->set($request->fd, $user);

        $server->push($request->fd, json_encode(
                array_merge(['user' => $user], ['all' => $this->allUser()], ['type' => 'openSuccess'])
            )
        );
    }

    private function allUser()
    {
        $users = [];
        foreach ($this->table as $row) {
            $users[] = $row;
        }
        return $users;
    }

    public function message(Swoole\WebSocket\Server $server, Swoole\WebSocket\Frame $frame)
    {
        $this->pushMessage($server, $frame->data, 'message', $frame->fd);
    }

    /**
     * 推送消息
     *
     * @param \swoole_websocket_server $server
     * @param string $message
     * @param string $type
     * @param int $fd
     */
    private function pushMessage(Swoole\WebSocket\Server $server, string $message, string $type, int $fd)
    {
        $message = htmlspecialchars($message);
        $datetime = date('Y-m-d H:i:s', time());
        $user = $this->table->get($fd);

        foreach ($this->table as $item) {
            // 自己不用发送 也可以发 看前端怎么设计
            if ($item['fd'] == $fd) {
                continue;
            }

            $server->push($item['fd'], json_encode([
                'type' => $type,
                'message' => $message,
                'datetime' => $datetime,
                'user' => $user
            ]));
        }
    }

    /**
     * 客户端关闭的时候
     *
     * @param \swoole_websocket_server $server
     * @param int $fd
     */
    public function close(Swoole\WebSocket\Server $server, int $fd)
    {
        $user = $this->table->get($fd);
        $this->pushMessage($server, "{$user['name']}离开聊天室", 'close', $fd);
        $this->table->del($fd);
    }

    /**
     * 创建内存表
     */
    private function createTable()
    {
        $this->table = new Swoole\Table(1024);
        $this->table->column('fd', Swoole\Table::TYPE_INT);
        $this->table->column('name', Swoole\Table::TYPE_STRING, 255);
        $this->table->create();
    }
}
```

## mini聊天室 实战

> * 照着[webim](https://github.com/shisiying/webim)搭建的一个聊天项目[chat](https://github.com/yujiarong/chat),涉及swoole,websocket,laravels等知识点 
> * [聊天室入口](http://chat.dwyjr.cn/chat)  
> * [后台入口](http://chat.dwyjr.cn/chat/room/index)

### 主要知识点
* 使用Laravel 快速搭建后台管理系统，这里使用的是之前集成的一个项目 [niftyAdmin](https://github.com/yujiarong/niftyAdmin)，Laravel5.5。
* 集成[Laravels](https://github.com/hhxsv5/laravel-s)插件来使用Swoole的功能。
* 通过Nginx反向代理Swoole来加速HTTP服务，提高并发。
* 通过Swoole将Laravel常驻内存需要解决的一些注意事项。
* 在Laravel中使用多表登陆，前后台用户分开登陆管理，直接使用的Laravels的`guards`来处理。
* 通过Swoole来搭建高性能的Websocket的服务。
* 使用Swoole的异步Task功能来Push Websokcet 的Message。
* 使用Swoole的Tabel的直接管理一些不重要的数据。


### 主要功能
* 可以在聊天室群聊，也可以私聊。
* 通过laravels使用swoole搭建websocket 服务。
* 使用Task 异步发送websocket message 提高性能。
* 使用swoole_table 存储数据，如果是生产环境建议还是改成redis。

### 主要改动

* 新增一个聊天室后台，设置了onRequest回调，WebSocket\Server同时作为http服务器。
* 后台可以管理聊天室，主要是新增和查看聊天房间 [链接](http://chat.dwyjr.cn/chat/room/index)。
* 支持多表登陆,聊天用户管理,后台用户管理分开。
* 新增自动登录注册，也就是说页面会记住当前登录用户，不需要每次刷新抖登录。
* 新增小爱同学智能聊天，`@小爱`  聊天 ，他就会回复你哦，这个机器人很笨。

### 搭建流程

1. git clone https://github.com/yujiarong/chat
2. composer install
3. php artisan key:generate  composer里面应该集成了脚本
4. php artisan migrate 数据表迁移
5. php artisan laravels publish  发布laravels的配置文件
6. 修改配置文件
```bash
 `.env` 里面的 `JS_DOMIND` 图片域名  ，`WS_SERVER` JSd的websokcet连接地址
 `.env` 里面的 `LARAVELS_LISTEN_IP`和`LARAVELS_LISTEN_PORT`用于swoole的启动监听地址
```

7. nginx  使用以下的配置,后台域名HTTP访问直接代理到laravels,websocket直接使用ip+port访问。

```bash
upstream laravels {
    # 通过 IP:Port 连接
    server 127.0.0.1:9090  weight=5 max_fails=3 fail_timeout=30s;
    # 通过 UnixSocket Stream 连接，小诀窍：将socket文件放在/dev/shm目录下，可获得更好的性能
    #server unix:/xxxpath/laravel-s-test/storage/laravels.sock weight=5 max_fails=3 fail_timeout=30s;
    #server 192.168.1.1:5200 weight=3 max_fails=3 fail_timeout=30s;
    #server 192.168.1.2:5200 backup;
    keepalive 16;
}
server {

        listen       80;
        server_name  chat.dwyjr.cn;

        root        /data/web/chat/public/;
        error_log   /data/web/chat/storage/logs/error.log   error;
        access_log  /data/web/chat/storage/logs/access.log  main;
        index index.php;

        location / {
             try_files $uri @laravels;
        }


     location @laravels {
        # proxy_connect_timeout 60s;//看情况设置
        # proxy_send_timeout 60s;
        # proxy_read_timeout 120s;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Real-PORT $remote_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header Scheme $scheme;
        proxy_set_header Server-Protocol $server_protocol;
        proxy_set_header Server-Name $server_name;
        proxy_set_header Server-Addr $server_addr;
        proxy_set_header Server-Port $server_port;
        proxy_pass http://laravels;
    }

}

```


### 开启 

使用 ./bin/laravels start 就可以开始了,后台挂起加 -d,这里通过[Laravels](https://github.com/hhxsv5/laravel-s/blob/master/Settings-CN.md)来使用swoole
```bash
 _                               _  _____ 
| |                             | |/ ____|
| |     __ _ _ __ __ ___   _____| | (___  
| |    / _` | '__/ _` \ \ / / _ \ |\___ \ 
| |___| (_| | | | (_| |\ V /  __/ |____) |
|______\__,_|_|  \__,_| \_/ \___|_|_____/ 
                                           
Speed up your Laravel/Lumen
>>> Components
+---------------------------+---------+
| Component                 | Version |
+---------------------------+---------+
| PHP                       | 7.1.10  |
| Swoole                    | 4.2.1   |
| LaravelS                  | 3.5.2   |
| Laravel Framework [local] | 5.5.45  |
+---------------------------+---------+
>>> Protocols
+----------------+--------+-------------------------------+--------------+
| Protocol       | Status | Handler                       | Listen At    |
+----------------+--------+-------------------------------+--------------+
| Main HTTP      | On     | Laravel Framework             | 0.0.0.0:9090 |
| Main WebSocket | On     | App\Services\WebSocketService | 0.0.0.0:9090 |
+----------------+--------+-------------------------------+--------------+

```

### 遇到的坑
> 基于swoole，将HTTP服务和Websocket服务整合在一起，使用laravels插件，swoole是常驻内存的,所以单例对象的使用是非常要注意的，Laravel内有很多功能使用单例模式。

1. Laravel的Controller里的构造方法如果初始化了一些参数，初始化后每次请求都是一样的，除非worker重启，这里如果遇到问题则每次请求后需要手动重置Controller。
2. 某些服务提供者在加入了swoole之后因为会出现问题在，需要每次请求后重置，可以加在`config/laravels.php`的`register_providers`数组中。
3. 如果使用到了Session一定要把`config/laravels.php`的`cleaners`中`SessionCleaner`和`AuthCleaner`开启，原因和上面一样。
4. 若按照上面这样设置了之后还是有问题，则自己手动使用中间件清理或者重新绑定服务，比如以下方式。

```bash
<?php

namespace App\Http\Middleware;

use Closure;
use Route;
use Log;

class SwooleCleaner
{
    /**
     * 使用swoole时 清理一些常驻内存有问题的实例 
     * 因控制器是单例，会常驻于内存，控制器中使用了静态变量,或者在构造函数__construct() 初始化了一些东西，就需要重置这个控制器
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    
    public $resetProvider = [
       'datatables.request' => \Yajra\DataTables\Utilities\Request::class, //如果不清理，这个插件搜索的时候就会出问题。
    ];

    public function handle($request, Closure $next)
    {
        if (PHP_SAPI === 'cli') { 
            $this->resetProvider();
            $response  = $next($request);
            if(isset(Route::current()->controller )){
                unset( Route::current()->controller );
            }
            Log::info( "Swoole 请求之后删除controller");
            return $response ;
        }else{
            return $next($request);
        }
    }

    /*
    * 重新绑定一些服务提供者,有些服务提供者有 boot()有初始化可能需要更多的操作
    */
    public function resetProvider(){
        foreach ($this->resetProvider as $key => $provider) {
            if(app($key)){
                Log::info("Swoole 重置 {$key}");
                app()->singleton($key, function ()use($provider) {
                    return new $provider;
                });                
            }
        }
    }
}

```
