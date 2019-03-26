---
title: twisted笔记（二）：reactor
date: 2019-03-23 16:22:44
categories: twisted 笔记
tags: twisted
---

reactor 意为：反应堆，reactor 对象代表事件循环，是 twisted 的核心。

<!--more-->

一个 reactor 循环什么都不做的例子：

{% codeblock lang:python %}
    from twisted.internet import reactor
    reactor.run()
{% endcodeblock %}

一个简单的 reactor 例子：

{% codeblock lang:python %}
    from twisted.internet import reactor

    def hello():
        print('hello world')

    reactor.callWhenRunning(hello)
    print('start reactor loop')
    reactor.run()
    print('使用ctrl+c手动关闭reactor后，才会显示该信息')
{% endcodeblock %}

结论：

- reactor运行在单线程中
- reactor对象是单例模式，引入即创建
- reactor循环只能通过`reactor.run()`启动
- reactor循环并不会消耗任何CPU资源
- reactor循环一旦启动会一直运行下去，可以手动ctrl+c关闭

事件循环
----------

reactor通过`select`函数监控文件描述符来实现对事件的监控（linux下）。当然并非只能通过select函数，只是因为select函数比较古老并被广泛应用，可以使用`twisted.internet.pollreactor`来调用poll代替select方法（代码略）。

原理：reactor监视一系列 文件描述符 并阻塞程序，直到至少有一个准备好的I/O操作。

![](/images/twisted-notes/p02_reactor-1.png)

回调与阻塞
-----------

回调过程中发生的一切：

![](/images/twisted-notes/p03_reactor-callback.png)

结论：

- 我们的代码与Twisted代码运行在同一个线程中
- 当我们的代码运行时，Twisted代码是处于暂停状态的。
- 同样，当Twisted代码处于运行状态时，我们的代码处于暂停状态。
- reactor事件循环会在我们的回调函数返回后恢复运行。

所以**我们应当在回调函数中尽量避免使用会阻塞I/O的函数**，例如：

- 文件读/写
- 数据库操作
- sleep()函数
- os.system系统函数

退出Twisted
----------

- `reactor.stop()`停止循环
- reactor一旦停止就无法再启动了（再次启动会raise ReactorNotRestartable）

例1，循环只能启动一次：

{% codeblock lang:python %}
    from twisted.internet import reactor

    def hello():
        print('hello world')
        reactor.stop()

    def hello_again():
        print('hello world again')

    reactor.callWhenRunning(hello)
    reactor.run()
    reactor.callWhenRunning(hello_again)
    reactor.run()
{% endcodeblock %}

输出：

{% codeblock lang:python %}
    hello world
    Traceback (most recent call last):
      File "test/tmp.py", line 13, in <module>
        reactor.run()
      File "D:\Anaconda3\lib\site-packages\twisted\internet\base.py", line 1260, in run
        self.startRunning(installSignalHandlers=installSignalHandlers)
      File "D:\Anaconda3\lib\site-packages\twisted\internet\base.py", line 1240, in startRunning
        ReactorBase.startRunning(self)
      File "D:\Anaconda3\lib\site-packages\twisted\internet\base.py", line 748, in startRunning
        raise error.ReactorNotRestartable()
    twisted.internet.error.ReactorNotRestartable
{% endcodeblock %}


回调中发生异常
--------------

如果在一个回调函数中发生异常，并不会影响其他回调的执行。
向外反馈一个`Unhandled Error`异常，并打印错误栈。


{% codeblock lang:python %}
    from twisted.internet import reactor

    def hello():
        print('hello world')

    def hello_exception():
        raise Exception('oh, shit!')
        
    reactor.callWhenRunning(hello)
    reactor.callWhenRunning(hello_exception)
    reactor.run()
{% endcodeblock %}

输出：

{% codeblock lang:python %}
    hello world
    Unhandled Error
    Traceback (most recent call last):
      File "D:\Anaconda3\lib\site-packages\twisted\internet\base.py", line 428, in fireEvent
        DeferredList(beforeResults).addCallback(self._continueFiring)
      File "D:\Anaconda3\lib\site-packages\twisted\internet\defer.py", line 322, in addCallback
        callbackKeywords=kw)
      File "D:\Anaconda3\lib\site-packages\twisted\internet\defer.py", line 311, in addCallbacks
        self._runCallbacks()
      File "D:\Anaconda3\lib\site-packages\twisted\internet\defer.py", line 654, in _runCallbacks
        current.result = callback(current.result, *args, **kw)
    --- <exception caught here> ---
      File "D:\Anaconda3\lib\site-packages\twisted\internet\base.py", line 441, in _continueFiring
        callable(*args, **kwargs)
      File "tmp.py", line 7, in hello_exception
        raise Exception('oh, shit!')
    builtins.Exception: oh, shit!
{% endcodeblock %}

调用回调
--------

调用回调函数常见方法，以我所知的举例：

- reactor.callWhenRunning(_callable, *args, **kw)
- reactor.callLater(_seconds, _f, *args, **kw)

{% codeblock lang:python %}
    from twisted.internet import reactor

    def foo(num1, num2):
        print(f'{num1}+{num2}={num1+num2}')

    reactor.callWhenRunning(foo, 1, 2)
    reactor.run()
{% endcodeblock %}

#### `callWhenRunning`：

当reactor循环开始时执行回调，若其中一个回调中关闭了reactor循环，并不影响其他回调执行。如果有多个回调中都执行了关闭循环动作，则后执行的回调会报错：

> twisted.internet.error.ReactorNotRunning: Can't stop reactor that isn't running

{% codeblock lang:python %}
    from twisted.internet import reactor

    def echo(*, close=False):
        print('hello world')
        if close:
            reactor.stop()

    reactor.callWhenRunning(echo, close=True)
    reactor.callWhenRunning(echo)
    reactor.run()
{% endcodeblock %}

输出：

{% codeblock lang:python %}
    hello world
    hello world
{% endcodeblock %}

如果连续两个`echo`回调都设置`close=True`则报错：

{% codeblock lang:python %}
    hello world
    hello world
    Unhandled Error
    Traceback (most recent call last):
      ...略...
      File "tmp.py", line 6, in echo
        reactor.stop()
      File "D:\Anaconda3\lib\site-packages\twisted\internet\base.py", line 630, in stop
        "Can't stop reactor that isn't running.")
    twisted.internet.error.ReactorNotRunning: Can't stop reactor that isn't running.
{% endcodeblock %}

#### `callLater`：

callLater并非指的是从当前时间n秒后开始调用指定回调，而是指上一个回调执行完毕n秒后才开始调用指定回调。如果上一个回调函数阻塞了很长时间，则callLater的调用也会跟着延迟。

如果callLater之前的回调函数执行了`reactor.stop()`动作，则callLater注册的回调函数并不会执行：

{% codeblock lang:python %}
    from twisted.internet import reactor

    def echo():
        print('hello world')
        reactor.stop()

    def echo_again():
        print('HELLO WORLD')

    reactor.callWhenRunning(echo)
    reactor.callLater(1, echo_again)   # echo_again()并不会被回调
    reactor.run()
{% endcodeblock %}


