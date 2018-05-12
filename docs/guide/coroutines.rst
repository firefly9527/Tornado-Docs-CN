协程
==========

.. testsetup::

   from tornado import gen

Tornado中推荐通过 **协程** 的方式写异步代码.  协程使用了Python ``yield`` 关键字来挂起和恢复执行程序而不是链式回调(像在 `gevent
<http://www.gevent.org>`_ 中出现的轻量级线程有时也被称为协程, 但Tornado中所有协程都采用显式的上下文切换并作为异步函数被调用).

使用协程方式来写异步代码几乎跟写同步代码一样简单，并不需额外的线程开销。 还通过减少上下文切换来 `使并发编程更简单
<https://glyph.twistedmatrix.com/2014/02/unyielding.html>`_ 。

例子::

    from tornado import gen

    @gen.coroutine
    def fetch_coroutine(url):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch(url)
        # In Python versions prior to 3.3, returning a value from
        # a generator is not allowed and you must use
        #   raise gen.Return(response.body)
        # instead.
        return response.body

.. _native_coroutines:

Python 3.5: ``async`` and ``await``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Python 3.5 引入了 ``async`` 和 ``await`` 关键字(使用这些关键字的函数也被称为”原生协程”)。
从Tornado 4.3开始, 你可以用它们代替基于 yield 关键字的协程(请阅读以下相应的限制)。 
只需要简单的使用 ``async def foo()`` 在函数定义的时候代替 ``@gen.coroutine`` 装饰器, 并且
用 await 代替yield. 本文档的其他部分会继续使用 yield 的风格来和旧版本的Python兼容,
但是如果 async 和 await 可用的话，它们运行起来会更快::

    async def fetch_coroutine(url):
        http_client = AsyncHTTPClient()
        response = await http_client.fetch(url)
        return response.body

 ``await`` 关键字比 ``yield`` 关键字功能要少一些。
 例如,在一个基于 ``yield`` 的协程中， 你可以得到 ``Futures`` 列表， 
 但是在原生协程中，你必须把列表用 `tornado.gen.multi` 包起来，
 你也可以使用 `tornado.gen.convert_yielded` 来把任何使用 ``yield`` 工作
 的代码转换成使用 ``await`` 的形式::

    async def f():
        executor = concurrent.futures.ThreadPoolExecutor()
        await tornado.gen.convert_yielded(executor.submit(g))

虽然原生协程没有明显依赖于特定框架(例如它们没有使用装饰器,
例如 tornado.gen.coroutine 或 asyncio.coroutine),
不是所有的协程都和其他的兼容。
有一个 协程执行者(coroutine runner) 在第一个协程被调用的时候进行选择,
然后被所有用 await 直接调用的协程共享。
Tornado 的协程执行者(coroutine runner)在设计上是多用途的,
可以接受任何来自其他框架的awaitable对象;
其他的协程运行时可能有很多限制(例如, asyncio 协程执行者不接受来自其他框架的协程).
基于这些原因,我们推荐组合了多个框架的应用都使用Tornado的协程执行者来进行协程调度.
为了能使用Tornado来调度执行asyncio的协程, 可以使用 `tornado.platform.asyncio.to_asyncio_future` 适配器.


它是如何工作的
~~~~~~~~~~~~

一个函数包含了关键字 ``yield`` , 那么这个函数是一个生成器。所有的生成器都是异步的；
当调用生成器的时候会返回一个生成器对象而不是该生成器执行的结果。使用 ``@gen.coroutine`` 装饰器
装饰的生成器, 如此产生的协程的调用者能通过 ``yield`` 表达式和协程进行通信，并且调用者可以获得一个 `.Future` 与协程进行交互。

下面是一个协程装饰器内部循环的简单版本::

    # Simplified inner loop of tornado.gen.Runner
    def run(self):
        # send(x) makes the current yield return x.
        # It returns when the next yield is reached
        future = self.gen.send(self.next)
        def callback(f):
            self.next = f.result()
            self.run()
        future.add_done_callback(callback)


装饰器从生成器接收一个 `.Future` 对象，等待(非阻塞)这个 `.Future` 对象执行完成，
然后解开('unwrap')这个 `.Future` 对象，并把结果作为 `yield` 的结果传回给生成器器。
大多数异步代码从来不会接触到 `.Future` 对象, 除非立即将由异步函数返回的`.Future`传递给 `yield` 表达式。

如何调用协程
~~~~~~~~~~~~~~~~~~~~~~~

协程一般不会抛异常: 任何异常都会被 `.Future` 捕获直到 `.Future` 传递给了
`yield` 。这意味着正确地调用协程很重要，否则可能会无法得到错误信息::

    @gen.coroutine
    def divide(x, y):
        return x / y

    def bad_call():
        # 这里应该抛出一个ZeroDivisionError 异常, 但却没有
        # 因为这里的协程调用方式是错的
        divide(1, 0)

几乎所有情况下，任何函数想要调用协程必须自身也是协程，并且在调用的
时候使用 ``yield`` 关键字.
当你复写父类的方法，请看下文档，看是是否支持协程(文档里应该会有说这个
方法"可能是协程或者返回一个 `.Future` ")::

    @gen.coroutine
    def good_call():
        # yield will unwrap the Future returned by divide() and raise
        # the exception.
        yield divide(1, 0)

有时你可能想要只执行一个协程并不需要等待结果。在这种情况下，建议使用 `.IOLoop.spawn_callback`,
它使用 `.IOLoop`  负责调用。 如果 它失败了, `.IOLoop` 会在日志中把调用栈记录下来::

    # IOLoop 将会捕获异常,并且在日志中打印栈记录.
    # 注意这不像是一个正常的调用, 因为我们是通过
    # IOLoop 调用的这个函数.
    IOLoop.current().spawn_callback(divide, 1, 0)

使用 `.IOLoop.spawn_callback` 执行 ``@gen.coroutine`` 装饰的协程是 *推荐* 的方式，
但对于使用 ``async def`` 的原生协程，这是 *必须的* (否则，协程不会启动)


最后, 在程序顶层, 如果 `.IOLoop` 尚未运行, 你可以启动 `.IOLoop` ,
执行协程, 然后使用 `.IOLoop.run_sync` 方法停止 `.IOLoop`
这通常被 用来启动面向批处理程序的 ``main`` 函数::

    # run_sync() doesn't take arguments, so we must wrap the
    # call in a lambda.
    IOLoop.current().run_sync(lambda: divide(1, 0))

协程模式
~~~~~~~~~~~~~~~~~~

结合callback
^^^^^^^^^^^^^^^^^^^^^^^^^^

为了使用回调而不是 Future 与异步代码进行交互,  用 `.Task` 把调被调用的函数包起来。
这将会把一个函数添加为回调参数并返回一个可以yield的 `.Future` :

.. testcode::

    @gen.coroutine
    def call_task():
        # Note that there are no parens on some_function.
        # This will be translated by Task into
        #   some_function(other_args, callback=callback)
        yield gen.Task(some_function, other_args)

.. testoutput::
   :hide:

调用阻塞函数
^^^^^^^^^^^^^^^^^^^^^^^^^^

使用 `~concurrent.futures.ThreadPoolExecutor` 将一个阻塞函数变为可被协程调用的。
它将返回和协程兼容的 ``Futures`` ::

    thread_pool = ThreadPoolExecutor(4)

    @gen.coroutine
    def call_blocking():
        yield thread_pool.submit(blocking_func, args)

并行
^^^^^^^^^^^

协程装饰器能识别列表或者字典对象中的每个 ``Futures`` ，并且等待所有 ``Futures`` 并行执行完:

.. testcode::

    @gen.coroutine
    def parallel_fetch(url1, url2):
        resp1, resp2 = yield [http_client.fetch(url1),
                              http_client.fetch(url2)]

    @gen.coroutine
    def parallel_fetch_many(urls):
        responses = yield [http_client.fetch(url) for url in urls]
        # responses is a list of HTTPResponses in the same order

    @gen.coroutine
    def parallel_fetch_dict(urls):
        responses = yield {url: http_client.fetch(url)
                            for url in urls}
        # responses is a dict {url: HTTPResponse}

.. testoutput::
   :hide:

交叉存取
^^^^^^^^^^^^

有时候保存一个 `.Future` 比立即yield它更有用, 所以你可以在等待之前 执行其他操作:

.. testcode::

    @gen.coroutine
    def get(self):
        fetch_future = self.fetch_next_chunk()
        while True:
            chunk = yield fetch_future
            if chunk is None: break
            self.write(chunk)
            fetch_future = self.fetch_next_chunk()
            yield self.flush()

.. testoutput::
   :hide:

这种方式最好和 ``@gen.coroutine`` 一起使用。如果 ``fetch_next_chunk()`` 使用 ``async def`` 声明的,
那么必须像这样调用 ``fetch_future = tornado.gen.convert_yielded(self.fetch_next_chunk())`` 才能启动
后台执行进程。

循环
^^^^^^^
在协程中，处理循环是比较棘手的，因为在Python中没法在每一个 ``for`` 或者 ``while`` 循环
中 ``yield`` 并捕获yield的结果。事实上，你需要将循环条件从访问结果中分离出来，
下面是一个使用 `Motor <https://motor.readthedocs.io/en/stable/>`_ 的例子::

    import motor
    db = motor.MotorClient().test

    @gen.coroutine
    def loop_example(collection):
        cursor = db.collection.find()
        while (yield cursor.fetch_next):
            doc = cursor.next_object()

在后台执行
^^^^^^^^^^^^^^^^^^^^^^^^^
通常在协程中不使用 `.PeriodicCallback` 。事实上，一个协程使用 ``while True:`` 循环
并使用 `tornado.gen.sleep`::

    @gen.coroutine
    def minute_loop():
        while True:
            yield do_something()
            yield gen.sleep(60)

    # Coroutines that loop forever are generally started with
    # spawn_callback().
    IOLoop.current().spawn_callback(minute_loop)

有时候很多协程。例如，一个循环执行任何都超过60多毫秒 这个N是执行函数 ``do_something()`` 花费的时间。
为了 准确的每60秒运行,使用上面的交叉模式::

    @gen.coroutine
    def minute_loop2():
        while True:
            nxt = gen.sleep(60)   # Start the clock.
            yield do_something()  # Run while the clock is ticking.
            yield nxt             # Wait for the timer to run out.
