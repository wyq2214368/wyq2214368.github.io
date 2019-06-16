---
title: 使用Swoole提升Laravel的性能
date: 2019-05-14 
categories: PHP
tag: laravel 
---
### 优化 Laravel 网站打开速度
1. 关闭 debug
打开.env 文件，把 debug 设置为 false. `barryvdh/laravel-debugbar`等开发环境使用的包一定要放在require-dev,线上就不要载入了，就算载入也要关闭。
2. 缓存路由和配置
php artisan route:cache。如果路由中有闭包是会报错的,所以路由中就不要添加处理逻辑
php artisan config:cache
3. composer 优化
composer dump-autoload --optimize
4. Laravel 优化命令
php artisan optimize
5. 使用 Laravel 缓存
``` php
$lists = Cache::remember('key', 20, function () {
    return $this->destination->getList();
});
```
6. 使用 PHP7 并开启 OPcache
开启opcache后需要重启 php-fpm哦

7. nginx 开启 gzip 压缩
Nginx 开启 gzip 可以有效减少服务器带宽的消耗，缺点是会增大 CPU 的占用率，但是很多时候 CPU 往往是空闲最多的。
在nginx的配置中添加如下:
``` bash
gzip on;
gzip_min_length 1k;
gzip_buffers 16 64k;
gzip_http_version 1.1;
gzip_comp_level 9;
gzip_types text/plain application/x-javascript application/javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png font/ttf font/otf image/svg+xml;
gzip_vary on;   
```

`GZIP_MIN_LENGTH `  设置允许压缩的页面最小字节数，页面字节数从 header 头中的 Content-Length 中进行获取。默认值是 0，不管页面多大都压缩。建议设置成大于 1k 的字节数，小于 1k 可能会越压越大。 即: gzip_min_length 1024 

响应头中 Content-Encoding 字段是 gzip，表示该网页是经过 gzip 压缩的。

### 使用swoole来laravel优化项目
我们这里使用[laravels](https://github.com/hhxsv5/laravel-s)来优化laravel5.5
1. composer require "hhxsv5/laravel-s:~3.5.0" -vvv
2. 配置ningx的代理到swoole端口。Nginx的配置,使用的是proxy_pass+upstream。
3. 由于swoole是常驻内存，所以单例的写法和HTTP请求参数获取需要注意，下面会说。
4. 如果某些包在加入swoole后出现了异常，可以用一个中间件来重置单例对象的状态。

``` bash
upstream laravels {
    # 通过 IP:Port 连接
    server 127.0.0.1:5200 weight=5 max_fails=3 fail_timeout=30s;
    # 通过 UnixSocket Stream 连接，小诀窍：将socket文件放在/dev/shm目录下，可获得更好的性能
    #server unix:/xxxpath/laravel-s-test/storage/laravels.sock weight=5 max_fails=3 fail_timeout=30s;
    #server 192.168.1.1:5200 weight=3 max_fails=3 fail_timeout=30s;
    #server 192.168.1.2:5200 backup;
    keepalive 16;
}
server {

        listen       80;
        server_name  laravels.valsun.cn;

        root        /data/web/niftyAdmin/public/;
        error_log   /data/web/niftyAdmin/storage/logs/error.log   error;
        access_log  /data/web/niftyAdmin/storage/logs/access.log  main;
        index index.php;

        location / {
             try_files $uri @laravels;
        }

	    location @laravels {
	        # proxy_connect_timeout 60s;
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

### 压测
PHP 7.0.11 + Laravel5.5
服务器配置，就一个小型测试项目在跑，装有一个mysql服务。

``` bash
top - 11:30:03 up 396 days, 10:08,  2 users,  load average: 0.14, 5.14, 7.21
Tasks: 315 total,   1 running, 297 sleeping,  16 stopped,   1 zombie
Cpu0  : 16.3%us,  4.3%sy,  0.0%ni, 75.4%id,  2.0%wa,  0.0%hi,  1.3%si,  0.7%st
Cpu1  :  1.7%us,  1.7%sy,  0.0%ni, 96.4%id,  0.0%wa,  0.0%hi,  0.0%si,  0.3%st
Cpu2  :  1.0%us,  2.0%sy,  0.0%ni, 96.7%id,  0.3%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu3  :  1.3%us,  1.3%sy,  0.0%ni, 97.4%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu4  :  1.3%us,  1.3%sy,  0.0%ni, 97.4%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu5  :  9.3%us,  2.7%sy,  0.0%ni, 88.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu6  :  1.0%us,  3.0%sy,  0.0%ni, 96.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Cpu7  :  0.7%us,  2.3%sy,  0.0%ni, 97.0%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  16333892k total, 15396272k used,   937620k free,   191568k buffers
Swap:        0k total,        0k used,        0k free, 10030768k cached

[root@tracknumber_share niftyAdmin]# free -m
             total       used       free     shared    buffers     cached
Mem:         15951      15036        914          0        187       9797
-/+ buffers/cache:       5051      10899
Swap:            0          0          0

```
测试都是在本地使用简单的ab测试，目的只是想横向对比 `php-fpm + nginx`  和 `swoole + nginx` 的性能

`./ab -n 5000 -c 500  http://nt.valsun.cn/api/getData`

逻辑很简单,启用laravel5.5的框架通过api路由直接输出,laravel已经使用上面的优化措施
``` php
public function getData()
{
    echo '3123123';
}
```

#### nginx + php-fpm 测试结果

压测时服务器的指标
``` bash
top - 11:41:58 up 396 days, 10:20,  2 users,  load average: 17.20, 12.25, 11.45
```

ab压测结果

``` bash
Server Software:        BWS
Server Hostname:        nt.valsun.cn
Server Port:            80

Document Path:          /api/getData
Document Length:        1436 bytes

Concurrency Level:      1000
Time taken for tests:   10.291 seconds
Complete requests:      5000
Failed requests:        954
   (Connect: 0, Receive: 0, Length: 954, Exceptions: 0)
Non-2xx responses:      5000
Total transferred:      7339154 bytes
HTML transferred:       5967972 bytes
Requests per second:    485.84 [#/sec] (mean)
Time per request:       2058.296 [ms] (mean)
Time per request:       2.058 [ms] (mean, across all concurrent requests)
Transfer rate:          696.42 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2  42.5      1    3004
Processing:   112 1783 1146.6   1527    3988
Waiting:        8 1579 1098.9   1368    3978
Total:        113 1785 1146.8   1528    3989

Percentage of the requests served within a certain time (ms)
  50%   1528
  66%   1751
  75%   2064
  80%   2434
  90%   3946
  95%   3969
  98%   3977
  99%   3978
 100%   3989 (longest request)
```

#### nginx + swoole 压测

压测时服务器的指标
``` bash
top - 11:43:03 up 399 days, 10:21,  4 users,  load average: 5.31, 2.59, 2.12
```
``` bash
./ab  -k -t 30 -c 1000    http://laravels.valsun.cn/api/getData
Benchmarking laravels.valsun.cn (be patient)
Completed 5000 requests
Completed 10000 requests
Completed 15000 requests
Completed 20000 requests
Completed 25000 requests
Completed 30000 requests
Completed 35000 requests
Completed 40000 requests
Completed 45000 requests
Completed 50000 requests
Finished 50000 requests


Server Software:        BWS
Server Hostname:        laravels.valsun.cn
Server Port:            80

Document Path:          /api/getData
Document Length:        38 bytes

Concurrency Level:      1000
Time taken for tests:   14.921 seconds
Complete requests:      50000
Failed requests:        1772
   (Connect: 0, Receive: 0, Length: 1772, Exceptions: 0)
Non-2xx responses:      1772
Keep-Alive requests:    48228
Total transferred:      13281128 bytes
HTML transferred:       2158712 bytes
Requests per second:    3351.01 [#/sec] (mean)
Time per request:       298.417 [ms] (mean)
Time per request:       0.298 [ms] (mean, across all concurrent requests)
Transfer rate:          869.24 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.3      0      11
Processing:     4  284 163.2    248    1243
Waiting:        4  274 142.4    247    1134
Total:          4  284 163.3    248    1243

Percentage of the requests served within a certain time (ms)
  50%    248
  66%    288
  75%    323
  80%    348
  90%    449
  95%    612
  98%    875
  99%    976
 100%   1243 (longest request)
```

这个采用2种ab测试方法主要是 php-fpm 用-t测试并发太高 request fail的比率就太高了，就是用柔和的方式测试。

从测试结果也看出来了swoole提升的性能很大，而且服务器的负载完全没有起来。

但是用swoole常驻内存需要注意下一些事项。

### swoole常驻内存注意事项

[laravels的注意事项](https://github.com/hhxsv5/laravel-s/blob/master/README-CN.md#%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)

1. 应通过Illuminate\Http\Request对象来获取请求信息 $_ENV是可读取的，$_SERVER是部分可读的，不能使用$_GET、$_POST、$_FILES、$_COOKIE、$_REQUEST、$_SESSION、$GLOBALS
2. 推荐通过返回Illuminate\Http\Response对象来响应请求，兼容echo、vardump()、print_r()，不能使用函数像 dd()、exit()、die()、header()、setcookie()、http_response_code()
3. 各种单例的连接将被常驻内存，建议开启持久连接
4. 你声明的全局、静态变量必须手动清理或重置
5. 无限追加元素到静态或全局变量中，将导致内存爆满
6. 是用swoole优化之后 尽量不适用static 声明静态变量,也没有那个必要,这里有简单的方法去偷懒,就是把laravels.php配置文件中的 max_request降低一点这样它重启的频率就会大一点，以免单进程的内存暴了
7. 单例问题 ,laravel框架到处都用了单例。
* 传统FPM下，单例模式的对象的生命周期仅在每次请求中，请求开始=>实例化单例=>请求结束后=>单例对象资源回收。
* Swoole Server下，所有单例对象会常驻于内存，这个时候单例对象的生命周期与FPM不同，请求开始=>实例化单例=>请求结束=>单例对象依旧保留，需要开发者自己维护单例的状态
* 详见laravels文档,作者提供了详细的解决方案主要就是 Session,Passport,JWT的使用
* 如果要使用laravels，强烈建议先把这个laravels结构熟读于心，并且了解初始larvel的功能,毕竟常驻内存和平常的php-fpm编程还是有很大的不同
8. 引入laravels 不仅仅是提高了并发，而且可以方便的使用swoole的功能，比如websocket,协程，多进程等
9. 由于larvels是常驻内存，所以每次修改代码之后要看见效果就需要重启./bin/laravels reload，这个作者也提供了解决方案哦 [自动reload](https://github.com/hhxsv5/laravel-s/blob/master/README-CN.md#%E4%BF%AE%E6%94%B9%E4%BB%A3%E7%A0%81%E5%90%8E%E8%87%AA%E5%8A%A8reload)
10. laravels 初步在小型项目使用 ，用户认证包括(session 和jwt) ,Eloquent ORM,路由，Artisan 命令行 ，Blade模板渲染都没有问题。
