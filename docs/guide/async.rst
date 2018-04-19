异步和非阻塞I/O
---------------------------------

实时web功能需要为每个用户提供一个长链接。在传统的同步web服务器中，
这意味着为每个用户提供一个线程，但这个成本是很昂贵的。

为了尽量减少并发链接带来的开销，Tornado使用但线程事件循环的方式。
这意味着所有的应用代码都需要是异步非阻塞的，因为在任意时间只有一
个操作在执行。

异步和非阻塞的关系是非常近且经常交换使用的，但他们并不是相同的意思

阻塞
~~~~~~~~

一个函数在等待某些执行的返回值的时候会被 **阻塞** 。一个函数被阻塞有很多原因：
网络I/O，硬盘I/O，互斥锁等。事实上， *每个* 函数在运行和使用CPU的时候或多或少
会被阻塞(举个极端的例子来说明为什么对待CPU阻塞要和对待一般阻塞一样的认真,
例如hash加密 `bcrypt <http://bcrypt.sourceforge.net/>`_, 需要消耗几百毫秒的CPU
时间, 而这已经超过一般的网络或者磁盘I/O消耗的时间了)。

一个函数可以在某些方面阻塞另外一些方面阻塞。例如，`tornado.httpclient` 的默认配置是会在DNS解析上阻塞，但是
其他网络请求不会(为了减轻这种默认配置的影响可以使用`.ThreadedResolver` 或者
通过正确配置``libcurl`` 然后用 ``tornado.curl_httpclient`` 来做)。在Tornado中，我们一般讨论的阻塞指的是
网络环境下的I/O阻塞，尽管其它的阻塞都被尽量减少了。

异步
~~~~~~~~

**异步** 函数在会在完成之前返回，在应用中触发下一个动作之前通常会在后台执行一些工作(和正常的 **同步**  函数在返回前就执行完所有的事情不同)。这里列 举了几种风格的异步接口:

* 回调参数
* 返回一个占位符 (`.Future`, ``Promise``, ``Deferred``)
* 将等待执行的动作添加到异步队列(例如 celery)
* 注册回调函数 (例如 POSIX signals)

不论使用哪种类型的接口, *按照定义*  异步函数与它们的调用者都有着不同的交互方式;也没有什么对调用者透明的方式使得同步函数异步(类似 
`gevent<http://www.gevent.org>`_ 使用轻量级线程的系统性能虽然堪比异步系统,但它们并 没有真正的让事情异步).


例子
~~~~~~~~

一个简单的同步函数:

.. testcode::

    from tornado.httpclient import HTTPClient

    def synchronous_fetch(url):
        http_client = HTTPClient()
        response = http_client.fetch(url)
        return response.body

.. testoutput::
   :hide:

把上面的例子用回调参数重写的异步函数:

.. testcode::

    from tornado.httpclient import AsyncHTTPClient

    def asynchronous_fetch(url, callback):
        http_client = AsyncHTTPClient()
        def handle_response(response):
            callback(response.body)
        http_client.fetch(url, callback=handle_response)

.. testoutput::
   :hide:

使用 `.Future` 代替回调:

.. testcode::

    from tornado.concurrent import Future

    def async_fetch_future(url):
        http_client = AsyncHTTPClient()
        my_future = Future()
        fetch_future = http_client.fetch(url)
        fetch_future.add_done_callback(
            lambda f: my_future.set_result(f.result()))
        return my_future

.. testoutput::
   :hide:

`.Future` 版本明显更加复杂，但是 ``Futures`` 却是Tornado中推荐的写法 因为它有两个主要的优势。
首先是错误处理更加一致，`因为`.Future.result` 方法可以简单的抛出异常(相较于常见的回调函数接口特别指定错误处理)，
而且 ``Futures`` 很适合和协程一起使用。协程会在后面深入讨论，这里是上 面例子的协程版本，和最初的同步版本很像:

.. testcode::

    from tornado import gen

    @gen.coroutine
    def fetch_coroutine(url):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch(url)
        raise gen.Return(response.body)

.. testoutput::
   :hide:

``raise gen.Return(response.body)`` 的写法适用在Python2中, 因为在其中生成器不允许返回值。
为了克服这个问题，Tornado的协程抛出一种特殊的叫 `.Return` 的异常。协程捕获这个异常并把它作为返回值。
在Python 3.3和更高版本,使用 ``return response.body`` 可以达到同样的目的。
