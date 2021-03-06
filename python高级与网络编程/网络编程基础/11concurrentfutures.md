# 11 concurrent.future并发模块

## 11.1 模块详解

​	这个模块是python3.2版本引入的，是一个新的高级异步执行库，它只在“任务”级别进行操作。也就是说，我们通过它就不需要过分关注同步和线程/进程的管理了。

| 类 | 描述 |
| :--- | :--- |
| Future | 封装可调用的异步执行，一般有Executor类的submit方法调用创建实例 |
| Executor | 调用异步执行的一个抽象类，不能直接实例化。 |
| ProcessPoolExecutor | Executor的子类，用来创建异步进程池 |
| ThreadPoolExecutor | Executor的子类，用来创建异步线程池 |

​	注：`Future`可以理解为一个在未来完成的操作，这是异步编程的基础。通常情况下，我们执行io操作，访问url时（如下）在等待结果返回之前会产生阻塞，cpu不能做其他事情，而Future的引入帮助我们在等待的这段时间可以完成其他的操作。它有着丰富的方法，常用的两个方法如下表。

| Future类的实例方法 | 描述 |
| :--- | :--- |
| done\(\) | 当异步调用成功取消或者成功执行完返回真 |
| result\(timeout=None\) | 异步执行的返回结果 |

除了Future，其它三个属于同一种类，即异步执行类。实际上方法也差不多。主要常用的方法有：

_**submit\(fn,\*args,\*\*kwargs\)**_

创建关于函数fn的子进程或线程,即生成一个Future类实例，以便实现异步执行。类似于threading.Thread或者multiprocessing.Process。

_**map\(self,fn,\*iterables,timeout=None,chunksize=1\)**_

类似submit，但这个方法返回一个map\(func, \*iterables\)迭代器，迭代器中的回调执行返回的结果有序的。

_**shutdown\(wait=True\)**_

释放进程池或线程池所有资源：wait=True，等待完成后关闭；wait=False，直接关闭。

我们通过几个实例来理解一下这几个方法。

**submit**

```python
''''''
from concurrent.futures import ThreadPoolExecutor
import time


def return_future(msg):
    time.sleep(1)
    return msg


# 创建一个线程池
pool = ThreadPoolExecutor(max_workers=2)

# 往线程池加入2个task
f1 = pool.submit(return_future, 'hello')
f2 = pool.submit(return_future, 'world')
time.sleep(0.9)
print(f1.done()) # 判断线程f1是否已执行
time.sleep(1)
print(f2.done())

print(f1.result()) # 输出结果
print(f2.result())
```

执行结果：

```
False
True
hello
world
```

**map **

```python

from concurrent.futures import ThreadPoolExecutor as Pool
import requests

URLS = ['http://www.baidu.com', 'http://qq.com', 'http://sina.com']


def task(url, timeout=10):
    return requests.get(url, timeout=timeout)


pool = Pool(max_workers=3)
results = pool.map(task, URLS)

for ret in results:
    print('%s, %s' % (ret.url, len(ret.content)))
```

执行结果：

```
http://www.baidu.com/, 2381
http://www.qq.com/, 251688
http://sina.com/, 21733
```

concurrent.futures模块本身也提供了两个重要的基础函数：

_**concurrent.futures.wait\(fs,timeout=None,return\_when=ALL\_COMPLETED\)**_

它类似于Executor类的shutdown方法，但比它更强大。wait方法会返回一个tuple\(元组\)，tuple中包含两个set\(集合\)，一个是completed\(已完成的\)另外一个是uncompleted\(未完成的\)。使用wait方法的一个优势就是获得更大的自由度，它接收三个参数FIRST\_COMPLETED, FIRST\_EXCEPTION和ALL\_COMPLETE，默认设置为ALL\_COMPLETED。其它具体参数含义如下图。

![](/assets/concurrent.png)

_**concurrent.futures.as\_completed\(fs, timeout=None\)**_

它返回一个迭代器，里面是刚完成的线程/进程。但与map相比，它返回的结果是无序的。

## 11.2 ThreadPoolExecutor实现静态web服务器

​	和第九章修改多线程时，一样，我们只需要修改第九章第二节的HTTPServer类中的serve\_forver方法中的处理请求方式就可以了，将其用本章模块ThreadPoolExecutor代替就可以。完整源码可查看net11\_concurrent\_futures/net11\_web\_PWB9.py。

```python
from concurrent.futures import ThreadPoolExecutor


    def serve_forever(self):
        """永久运行监听接收连接"""
        pool = ThreadPoolExecutor() # 创建线程池
        while True:
            client_socket, client_address = self.tcp_socket.accept()
            print(client_address, '向服务器发起了请求')
            pool.submit(self.handlerequest, client_socket) # 添加新线程处理请求
```



