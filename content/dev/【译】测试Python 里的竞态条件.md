---
title: "【译】测试Python里的竞态条件"
date: "2017-09-27"
draft: false
tags: ["python"]
keywords: ["编程", "python", "竞态条件"]
categories: "技术"
---



[原文地址](http://www.oreills.co.uk/2015/03/01/testing-race-conditions-in-python.html)

不论何时，当有多于一个进程或者线程访问相同数据的时候，竞态条件都是一个危险的事情。本文探讨了发现竞态条件之后怎么去进行测试。  

## Incrmt  

（假设）你目前在一个只做一件事情并做的很好的热门初创企业Incrmt里工作。  

你部署了一个全球的计数器和一个加号的标志。人们可以点击那个加号，然后计数器增加1。它非常简单且让人上瘾。这是未来一个了不起的事情。  

投资者们都在登船旅行，但是你有了一个麻烦。  

## 竞态条件  

在内测的时候，Abraham和Belinda太兴奋了，分别点击了加号按钮100次。你的服务器日志显示接收了200次请求，但是计数器却只显示173。有一些并没有被加上。  

为了把”Incrmnt原来是Excrmnt”抛到脑后，你检查了以下代码。  

```python
# incrmnt.py
import db
def incremnt():
    count = db.get_count()
    new_count = count + 1
    db.set_count(new_count)
    return new_count
```

你的Web服务器使用了大量的进程来增加吞吐量，所以这个方法可能 同时再2个不同的线程中执行。如果在执行的时间线上你不够走运，下面的情况就会发生。  

```python
# 线程1和线程2同一时间在两个不同的进程中执行  
# 为了方便展示把他们并排放在一起  
# 垂直方向用来展示在某个时间点是那条语句在运行  
# 线程 1                                                    # 线程 2  
def increment():
                                                                 def increment():
    # get_count  return 0
    count = db.get_count()
                                                                      # get_count return 0 again
                                                                      count = db.get_count()
    new_count = count + 1
    # set_count called with 1
    db.set_count(new_count)
                                                                      new_count = count + 1
                                                                      # set_count called with 1 again
                                                                      db.set_count(new_count)
```

所以，虽然`count`应该是被增加了2次，实际上只增加了1。  

你知道你能修复这个代码让它是线程安全的，但是在你做这些之前，你想要写个测试来证明竞态存在。  

## 重现竞态  

尽可能接近得重现上面的情况是最理想的，关键的竞态是：  

> `get_count`调用都必须在`set_count`调用之前，这样`count`在两个线程中才拥有同一个值。  

`set_count`在什么时候调用无所谓，只要他们都在另一个`get_count`调用之后就行。  

简单起见，来试着重现这个情况，让线程2在线程1的`get_count`调用之后执行:  

```python
# 线程 1                                                  # 线程 2
def increment():
    # get_count returns 0
    count = db.get_count()
                                                                def increment():
                                                                    # get_count returns 0 again
                                                                    count = db.get_count()
                                                                    # set_count called with 1
                                                                    new_count = count + 1
                                                                    db.set_count(new_count)
    # set_count called with 1 again
    new_count = count + 1
    db.set_count(new_count)
```

[before_after](https://pypi.python.org/pypi/before_after/)库提供了一些功能可以帮助我们来重现这个情况。它能在方法的前面或者后面插入一些代码。  

`before_after`依赖[mock](https://pypi.python.org/pypi/mock)库来获取方法。如果你对他们不熟悉的话建议阅读[这些文档](http://www.voidspace.org.uk/python/mock/)。其中重要的部分是[Where To Patch](http://www.voidspace.org.uk/python/mock/patch.html#where-to-patch)。  

我们想要在线程1调用了`get_count`之后执行线程2，然后重新唤醒线程1继续执行。  

可以编写以下测试:  

```python
# test_incrmnt.py  
import unittest
import before_after
import db
import incrmnt

class TestIncrmnt(unitest.TestCase):
    def setUp(self):
        db.reset_db()

    def test_increment_race(self):
        # 在调用get_count之后，调用increment
        with before_after.after(‘increment.db.get_count’, incrmnt.increment):
            # 调用increment产生竞态  
            incrmnt.increment()
        count = db.get_count()
        self.assertEqual(count, 2)
```

在第一次`get_count`调用之后，我们使用了`before_after`的`after`上下文管理器来调用`increment`。  

`before_after`默认只调用一次`after`方法一次。这对大多数情况都是有用的，否则我们就需要清理栈了（`increment`调用`get_count`会循环再次调用`increment`,这又会继续调用`get_count`…）。  

这个测试失败了，因为`count`等于1而不是2.现在我们重现了竞态条件，接下来对它进行修复。  

## 减少竞态  

我们使用一个简单的锁来解决这个问题，这样可以用`before_after`来做一个更好的示例，另外`before_after`对于多线程应用的测试并不好用。显然这不是个理想的解决方案，最好是在数据存储层使用原子操作来进行数据更新。  

在`incrmnt.py`中添加一个新方法:  

```python
# incrmnt.py
def locking_increment():
    with db.get_lock():
        return increment()
```

这保证了同一时间只有一个线程对counter进行读写操作。当一个线程试图获取已经被另一个线程占用的锁时，将会抛出`CouldNotLock`异常。  

现在可以添加以下测试:   

```python
def test_locking_increment_race(self):

    def erroring_locking_increment():
        # 再试图获取被另一线程占用的锁时触发CouldNotLock异常
        # 这里捕获这个异常，否则测试将会失败
        with self.assertRaises(db.CouldNotLock):
            incrmnt.locking_increment()
    
    with before_after.after(‘incrmnt.db.get_count’, erroring_locking_increment):
        incrmnt.locking_increment()

    count = db.get_count()
    self.assertEqual(count, 1)
```

现在在某个时间点只有一个线程能使计数器增长了。  

## 减轻竞态  

这里还有一个问题，当两次请求碰撞的时候，有一个会被丢弃。为了解决这个问题，我们可以进行重试(使用像[funcy retry](http://funcy.readthedocs.org/en/stable/flow.html#retry)这样的库实现起来非常简洁):  

```python
# incrmnt.py  
def retrying_locking_increment():
    @retry(tries=5, errors=db.CouldNotLock)
    def _increment():
        return locking_increment()
    return _increment()
```

当我们需要比这个方法提供的结果更严苛的时候，可以把数据的增长交给数据库的原子更新或传输来做，不在应用层面进行处理。    

## 结论  

Incrmnt现在从竞态中解脱了，人们可以一整天都开心地进行点击而不用担心计数的问题。  

这是一个简单的例子，但是`before_after`可以在更复杂的竞态中使用来保证我们的方法正确地处理了这种情况。可以对使用线程的环境进行测试和重现能让我们在正确处理竞态条件时更加地自信。