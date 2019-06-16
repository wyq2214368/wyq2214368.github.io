---
date: 2018-05-11 
title: concurrent.futures 并发操作
categories: Python
tag: future 
---

[concurrent.futures]()模块的基础是Exectuor，Executor是一个抽象类，它不能被直接使用。但是它提供的两个子类ThreadPoolExecutor和ProcessPoolExecutor却是非常有用，顾名思义两者分别被用来创建线程池和进程池的代码。我们可以将相应的tasks直接放入线程池/进程池，不需要维护Queue来操心死锁的问题，线程池/进程池会自动帮我们调度。

### 进程池

``` php
from concurrent.futures import ProcessPoolExecutor
import os,time,random
def task(n):
    print('%s is running' %os.getpid())
    time.sleep(2)
    return n**2


if __name__ == '__main__':
    p=ProcessPoolExecutor()  #不填则默认为cpu的个数
    l=[]
    start=time.time()
    for i in range(10):
        obj=p.submit(task,i)   #submit()方法返回的是一个future实例，要得到结果需要用obj.result()
        l.append(obj)

    p.shutdown()  #类似用from multiprocessing import Pool实现进程池中的close及join一起的作用
    print('='*30)
    # print([obj for obj in l])
    print([obj.result() for obj in l])
    print(time.time()-start)

    #上面方法也可写成下面的方法
    # start = time.time()
    # with ProcessPoolExecutor() as p:   #类似打开文件,可省去.shutdown()
    #     future_tasks = [p.submit(task, i) for i in range(10)]
    # print('=' * 30)
    # print([obj.result() for obj in future_tasks])
    # print(time.time() - start)
```


### 线程池


``` php
from concurrent.futures import ProcessPoolExecutor,ThreadPoolExecutor
import threading
import os,time,random
def task(n):
    print('%s:%s is running' %(threading.currentThread().getName(),os.getpid()))
    time.sleep(2)
    return n**2

if __name__ == '__main__':
    p=ThreadPoolExecutor()   #不填则默认为cpu的个数*5
    l=[]
    start=time.time()
    for i in range(10):
        obj=p.submit(task,i) #异步
        l.append(obj)
    p.shutdown()
    print('='*30)
    print([obj.result() for obj in l])
    print(time.time()-start)

#上面方法也可写成下面的方法
    # start = time.time()
    # with ThreadPoolExecutor() as p:   #类似打开文件,可省去.shutdown()
    #     future_tasks = [p.submit(task, i) for i in range(10)]
    # print('=' * 30)
    # print([obj.result() for obj in future_tasks])
    # print(time.time() - star
asyncio.run_coroutine_threadsafe(do_some_work(4), new_loop)
```

