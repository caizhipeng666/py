# Asyncio

内容|description
---|---
[协程基础](#协程框架基础)|**任何一个协程框架都首先必须是一个异步框架**
[asyncio](#消息循环)|消息循环
[Future](#Future)|回调管理器
[Coroutine](#Coroutine)|异步流程(生成器)<br>(*生成器跟Future的结合*)
[Task](#Task)|对于coroutine中的Future自动调用它的add\_callback<br>把coroutine放到Task的壳里, 使得coroutine在事件循环里独立地执行
[避免阻塞](#避免阻塞)|硬件中断、轮询
[组合协程](#组合协程)|wait & gather

## 协程框架基础
构成|作用
---|---
事件循环|实际启动之后执行的代码
事件队列|向事件循环发送要执行的任务
polling|使用multiplexing(多路复用)技术(如select或epoll)<br>用来**监控socket等IO活动**
timer队列|保存定时器，一般是个最小堆

执行顺序|过程
---|---
asyncio的事件队列:<br>**取出callable**|callable执行:<br>1. 调用asyncio的其他接口配置polling 或者 timer队列<br>2. 发送更多callable到事件队列里
polling 或者 timer队列:<br>**将socket活动和计时器关联到不同的回调**|这个回调不是立即执行的, 而是延迟回调<br>即不直接调用而是放进事件队列里，等着事件循环去调用

```
事件循环（即EventLoop）开始
    不断从事件队列里取callable
    一个一个call过去，call完换下一个
if 事件队列里没了
    去timer队列里取一个最近的timer作为超时时间
调用polling
    直到任意socket活动，或者超时为止

然后根据不同的条件，将socket的回调放到事件队列里，从头开始执行
```

---
###消息循环
1.从asyncio中直接取一个eventloop的引用

```python
loop = asyncio.get_event_loop()
```
2.把需要执行的协程扔到EventLoop中执行

```python
loop.run_until_complete(coroutine方法())
loop.close()
```
3.定义coroutine

```python
"""@asyncio.coroutine:将一个generator标记为coroutine类型"""

@asyncio.coroutine
def hello():
    print("Hello world!")
    r = yield from asyncio.sleep(1)
    print("Hello again!")

```

> yield from语法可以让我们方便地调用另一个generator<br>**asyncio中yield from是把控制权还给事件循环**

---
> asyncio.sleep()也是一个coroutine,所以线程不会等待asyncio.sleep(),<br>而是直接中断并执行下一个消息循环<br>当asyncio.sleep()返回时, 线程就可以从yield from拿到返回值(此处是None)<br>然后接着执行下一行语句。

---
> 从 Py3.4起, asyncio包只直接支持TCP和UDP<br>如果想使用HTTP或其他协议, 那么要借助第三方包, Ex: aiohttp

```python
import threading
import asyncio


@asyncio.coroutine
def hello():
    print('Hello world! (%s)' % threading.currentThread())
    yield from asyncio.sleep(3)
    print('Hello again! (%s)' % threading.currentThread())


loop = asyncio.get_event_loop()
wait_tasks = asyncio.wait([hello(), hello()])  # wait也是一个协程, 等传给它的所有协程运行完毕后退出
loop.run_until_complete(wait_tasks)
loop.close()
```

> 可以发现第一个线程到asyncio.sleep的时候跳出去执行第二个coroutine了

```python
Hello world! (<_MainThread(MainThread, started 4533507520)>)
Hello world! (<_MainThread(MainThread, started 4533507520)>)
Hello again! (<_MainThread(MainThread, started 4533507520)>)
Hello again! (<_MainThread(MainThread, started 4533507520)>)
```
---

### Future

q|a
---|---
Future是什么?|基本上是一个回调管理器<br>任何程序都可以通过add\_callback的接口为这个Future设置一个回调函数
如何创建Future?|通常情况下, 都不应该自己创建期物(Future)<br>只能由并发框架concurrent.futures 或者 asyncio 进行实例化
为什么Future只能由并发框架实例化?|**期物表示终将发生的事情**<br>而确定某件事情一定会发生的唯一方式是执行的时间已经排定<br>只有排定某件事情交给框架, 才会创建期物<br>concurrent.futures: Executor.submit()<br>asyncio: asyncio.create_task()
什么时候调用callback?|Future完成时会调用callback进行回调
Future的初始状态?|Future刚创建的时候是未完成状态
如何修改Future未完成状态?|通过set\_result接口将Future设置为完成状态<br>(也是这个set\_result接口调用了已经注册的那些callback)<br>如果add\_callback的时候Future已经完成,<br>就让add\_callback自己去直接调用了这个回调, 这样效果就统一了
为什么Future不直接调用callback?|Future直接调用callback容易反复触发, 导致栈溢出,<br>所以通过延迟调用, 将callable放到事件队列里, 让事件循环帮忙调用
与concurrent.futures.Future的区别?|唯独.result()区别较大<br>asyncio没有参数: 不会阻塞去等候结果, 而是排除asyncio.InvalidStateError异常

---

### Coroutine

q|a
---|---
Coroutine是什么?|一个生成器<br>(感觉更像一个Pipe)
Coroutine做什么?|1. 每次通过yield返回一个Future<br>2. 为这个Future设置一个callback<br>3. 这个callback触发的时候, 我们将Future里保存的结果发送给这个 coroutine, <br> 4. coroutine往下再进一步, 得到下一个 Future<br>5. Loop 1 ~ 4: 让这个coroutine在我们的异步模型当中以一种类似于同步的方式执行下去
Coroutine级联?|除了使用Task作为 Future来串联<br>Ex: `asyncio.Task(coroutine)` **`asyncio.create_task(coroutine)`**<br>**coroutine之间还可以级联在一个coroutine内**<br>将这个coroutine的**yield值和返回值, 以及外部发送的值**`全部都代理到内层coroutine里就行了`<br>(Py3: yield from)<br>(Py3.5: async语法中的await, await一个coroutine即为级联)

```python
"""coroutine级联"""
async def do_something():
    await asyncio.sleep(2)
    r = await some_other_thing()
    return r + 1
```
---

### Task

Future|Task
---|---
asyncio.Task也是一个Future的子类|asyncio中的Future
Future通过set_result接口将Future设置为完成状态|Task的结果是自动用coroutine返回值设置的<br>且Task的cancel不仅会设置 "已取消" 的结果, <br>还会立即在coroutine中抛出异常, 尝试使coroutine提前结束
concurrent.Executor.submit()|asyncio中有多个函数会自动把参数指定的协程(coroutine)<br>包装在asyncio.Task对象中<br>Ex:<br>`asyncio.create_task`<br>`eventloop.run_until_complete`

---
---

## 避免阻塞
q|a
---|---
如何避免阻塞?|1. 单独线程中运行阻塞操作<br>2. 把所有阻塞操作转换成非阻塞的异步调用
异步调用的实现?|不等待响应, 使用回调<br>注册一个函数, 在某事件发生时调用
为什么只有**异步应用程序底层的事件循环**<br>能够依靠基础设置的中断、线程、轮询和后台进程等?|确保多个并发请求能够取得进展并最终完成<br>当底层的事件循环获得响应后, 回过头来调用我们指定的回调
如何解决应用瓶颈?(文件)|loop.run\_in\_executor, 在线程池中运行

### 组合协程
> 一系列的协程可以通过await链式的调用, 但是有的时候我们需要在一个协程里等待多个协程

func|description
---|---
wait|**结果无序**<br>wait()使用一个set保存它创建的Task实例<br>因为set是无序的<br>所以任务不是顺序执行的
gather|并行运行, 等待最耗时的那个完成之后才返回结果<br>1.gather任务无法取消<br>2.返回值是一个结果列表<br>3.可以按照传入参数的顺序，顺序输出
