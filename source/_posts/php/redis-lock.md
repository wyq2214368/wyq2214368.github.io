---
title: Redis 分布式锁
date: 2018-12-04
categories: php
tag: redis
---
实现一个分布式锁定，我们至少要考虑它能满足一下的这些需求:

* 互斥，就是要在任何的时刻，同一个锁只能够有一个客户端用户锁定.
* 不会死锁，就算持有锁的客户端在持有期间崩溃了，但是也不会影响后续的客户端加锁
* 谁加锁谁解锁，很好理解，加锁和解锁的必须是同一个客户端

### Redis 分布式锁

这里使用PRedis来访问Redis
``` php
<?php

use PRedis;

class RedisLock 
{

	protected is_block = 1; //获取阻塞获取锁

    protected $_redis = null;

    /**
     * 当前请求id
     * @var integer
     */
    protected static $request_id = '';

    /**
     * 锁过期时间
     * 单位：秒
     * @var integer
     */
    protected $expiredTime = 30;

    /**
     * 获取阻塞锁最长等待时间
     * 不宜过长，请考虑实际情况单次锁释放时间设置
     * 高并发情况，较长阻塞时间会造成大量进程堆积
     * 单位：秒
     * @var integer
     */
    protected $waitTime = 10;

    /**
     * 阻塞锁重试频率
     * 每次请求的间隔时间
     * 单位: 微秒
     * @var integer
     */
    protected $frequency = 200;


    public function __construct()
    {
        $this->_redis = PRedis::connection('lock');
    }

    public function lock($name, $policy=0, $config=[])
    {
        $req_id = $this->getRequestId();

        if ($policy === self::BLOCKED) {
            $result = 'false';
            $beginTime = time();
            while(true) {
                $result = $this->_redis->set($name, $req_id, 'PX', $this->expiredTime * 1000, 'NX');
                if ('OK' === (string) $result) {
                    return true;
                } elseif (time()-$beginTime<=$this->waitTime) {
                    usleep($this->frequency * 1000);
                } else {
                    return false;
                }
            }
            
            return 'OK' === (string) $result;
        } else {
            $result = $this->_redis->set($name, $req_id, 'PX', $this->expiredTime * 1000, 'NX');
            return 'OK' === (string) $result;
        }

        return false;
    }

    /**
     * 使用lua 保证原子操作,客户端 A 加锁成功后一段时间再来解锁，在执行删除 del 操作的时候锁过期了，而且这时候又有其他客户端 B 来加锁 (这时候加锁是肯定成功的，因为客户端 A 的锁过期了), 这时客户端 A 再执行删除 del 操作，会把客户端 B 的锁给清了.
     */
    public function unlock($name)
    {
        $lua = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";

        $req_id = $this->getRequestId();
        $result = $this->_redis->eval($lua, 1, $name, $req_id);

        return (int) $result === 1;
    }

    /**
     * @return $request_id 最好保证唯一值
     */
    protected function getRequestId()
    {
        $request_id = static::$request_id ?: (static::$request_id = uniqid());

        return $request_id . getmypid();
    }
}

```
$this->_redis->set($name, $req_id, 'PX', $this->expiredTime * 1000, 'NX');
* 第一个 name 是锁的名字，这个由具体业务逻辑控制，保证唯一即可，比如并发更新一个 sku的库存的时候 SKU0001就可以加上锁以免超卖
* 第二个是请求 ID,这样做的目的主要是为了保证加解锁的唯一性。这样我们就可以知道该锁是哪个客户端加的.
* 第三个参数是一个标识符，标识时间戳以毫秒为最小单位
* 具体的过期时间
* 这个参数是 NX, 表示当 key 不存在时我们才进行 set 操作，这样锁就不会形成覆盖。
* 分布式唯一ID使用的[snowflack](https://github.com/Sxdd/php_snowflake)这个是编译安装，还有用swoole_lock实现的扩展包