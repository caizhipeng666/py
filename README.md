目录|内容
---|---
[对象](./对象.md)|变量保存的是引用<br>为现有的变量赋值, 不会修改之前绑定的变量<br>只会为现有的变量重新绑定
[列表](./列表.md)|
[集合](./集合.md)|
[元组](./元组.md)|
[字典](./字典.md)|
[函数](./函数.md)|
[协程](./协程.md)|yield通常出现在表达式的右边<br>可以产出值也可以不产出<br>把yield视作控制流程的方式
[期物](#./期物.md)|concurrent.futures.Future  & asyncio.Future<br>表示可能已经完成或者尚未完成的延迟计算
[类对象](./类对象.md)|当看到一只鸟走路像鸭子<br>游泳也像鸭子<br>称这只鸟为鸭子
[字符串](./字符串.md)|
[装饰器](./装饰器.md)|
[上下文](./上下文.md)|上下文管理器对象存在的目的是管理 with 语句<br>就像迭代器的存在是为了管理 for 语句一样
[异步io](./异步io.md)|Asyncio<br>真正执行I/O操作的函数是最内层的子生成器
[元编程](./元编程.md)|某类计算机程序的编写<br>这类计算机程序编写或者操纵其他程序（或者自身）作为它们的数据<br>或者在运行时完成部分本应在编译时完成的工作

---

### 概念

内容|desc
---|---
迭代器|从集合中取出元素<br>按需一次性获取一个数据项
生成器|"凭空"生成元素<br>所有生成器都是迭代器
with|设置一个临时的上下文,<br>交给上下文管理器对象控制,<br>并且负责清理上下文<br>(保证一段代码运行完毕后执行某项操作, 即使代码由于异常 & return & sys.exit()而中止)
else|else子句不仅能在if中使用,<br>还能在for(循环完毕, 没有break)、while(while False)、try(没有Exception)中使用
白鹅类型|只要cls是抽象基类(metaclass=abc.ABCMeta)<br>就可以使用isinstance(obj, cls)

### Important
问题|解答
---|---
[生成器和迭代器](#生成器和迭代器)|1.接口<br>2.实现方式<br>3.概念

##### 生成器和迭代器
* 接口

```
迭代器协议: 实现__next__和__iter__

> 生成器实现了这两个function, 
  ∴所有生成器都是迭代器
```
* 实现方式

```
> 生成器: 1. 含有yield关键字的函数
         2. 生成器表达式
得到的生成器对象属于语言内部的GeneratorType类型
而GeneratorType的实例实现了迭代器接口,
∴所有生成器都是迭代器
```
> 但是, 存在不是生成器的迭代器   
   Ex: 用C语言编写的拓展(enumerate)

```python
import types
from collections.abc import Iterator

e = enumerate("ABC")
isinstance(e, types.GeneratorType)

>>> False

isinstance(e, Iterator)

>>> True
```

* 概念

```
迭代器: 用于 遍历集合, 从中产出元素
       当调用next(item)时, 迭代器不能修改从数据源中读取的值, 只能原封不动的产出值

生成器: 无需遍历就能生成值
```
