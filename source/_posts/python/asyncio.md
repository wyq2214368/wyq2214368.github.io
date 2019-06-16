---
date: 2018-04-11 
title: 协程 asyncio
categories: Python
tag: asyncio 
---

### 关于asyncio的一些关键字的说明：
* event_loop 事件循环：程序开启一个无限循环，把一些函数注册到事件循环上，当满足事件发生的时候，调用相应的协程函数
* coroutine 协程：协程对象，指一个使用async关键字定义的函数，它的调用不会立即执行函数，而是会返回一个协程对象。协程对象需要注册到事件循环，由事件循环调用。
* task 任务：一个协程对象就是一个原生可以挂起的函数，任务则是对协程进一步封装，其中包含了任务的各种状态
* future: 代表将来执行或没有执行的任务的结果。它和task上没有本质上的区别

### 定义一个协程

``` php
import asyncio
import time

now = lambda: time.time()

async def do_some_work(x):
    print("waiting:",x)
    await asyncio.sleep(x)
    return "Done after {}s".format(x)

async def main():
    coroutine1 = do_some_work(1)
    coroutine2 = do_some_work(2)
    coroutine3 = do_some_work(4)
    tasks = [
        asyncio.ensure_future(coroutine1),#创建任务
        asyncio.ensure_future(coroutine2),
        asyncio.ensure_future(coroutine3)
    ]
    return await asyncio.gather(*tasks)

start = now()

loop = asyncio.new_event_loop()
results = loop.run_until_complete(main())
for result in results:
    print("Task ret:",result)

print("Time:", now()-start)
```
执行结果
``` php
waiting: 1
waiting: 2
waiting: 4
Task ret: Done after 1s
Task ret: Done after 2s
Task ret: Done after 4s
Time: 4.022229909896851
```

在上面带中我们通过async关键字定义一个协程（coroutine）,当然协程不能直接运行，需要将协程加入到事件循环loop中asyncio.get_event_loop：创建一个事件循环，然后使用run_until_complete将协程注册到事件循环，并启动事件循环。task对象是Future类的子类，保存了协程运行后的状态，用于未来获取协程的结果。
使用async可以定义协程对象，使用await可以针对耗时的操作进行挂起，就像生成器里的yield一样，函数让出控制权。协程遇到await，事件循环将会挂起该协程，执行别的协程，直到其他的协程也挂起或者执行完毕，再进行下一个协程的执行耗时的操作一般是一些IO操作，例如网络请求，文件读取等。我们使用asyncio.sleep函数来模拟IO操作。协程的目的也是让这些IO操作异步化。

### 线程协程

很多时候，我们的事件循环用于注册协程，而有的协程需要动态的添加到事件循环中。一个简单的方式就是使用多线程。当前线程创建一个事件循环，然后在新建一个线程，在新线程中启动事件循环。当前线程不会被block。

``` php
import asyncio
import time
from threading import Thread

now = lambda :time.time()


def start_loop(loop):
    asyncio.set_event_loop(loop)
    loop.run_forever()

async def do_some_work(x):
    print('Waiting {}'.format(x))
    await asyncio.sleep(x)
    print('Done after {}s'.format(x))

def more_work(x):
    print('More work {}'.format(x))
    time.sleep(x)
    print('Finished more work {}'.format(x))

start = now()
new_loop = asyncio.new_event_loop()
t = Thread(target=start_loop, args=(new_loop,))
t.start()
print('TIME: {}'.format(time.time() - start))

asyncio.run_coroutine_threadsafe(do_some_work(6), new_loop)
asyncio.run_coroutine_threadsafe(do_some_work(4), new_loop)
```

上述的例子，主线程中创建一个new_loop，然后在另外的子线程中开启一个无限事件循环。 主线程通过run_coroutine_threadsafe新注册协程对象。这样就能在子线程中进行事件循环的并发操作，同时主线程又不会被block。一共执行的时间大概在6s左右。