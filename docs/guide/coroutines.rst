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
While native coroutines are not visibly tied to a particular framework
(i.e. they do not use a decorator like `tornado.gen.coroutine` or
`asyncio.coroutine`), not all coroutines are compatible with each
other. There is a *coroutine runner* which is selected by the first
coroutine to be called, and then shared by all coroutines which are
called directly with ``await``. The Tornado coroutine runner is
designed to be versatile and accept awaitable objects from any
framework; other coroutine runners may be more limited (for example,
the ``asyncio`` coroutine runner does not accept coroutines from other
frameworks). For this reason, it is recommended to use the Tornado
coroutine runner for any application which combines multiple
frameworks. To call a coroutine using the Tornado runner from within a
coroutine that is already using the asyncio runner, use the
`tornado.platform.asyncio.to_asyncio_future` adapter.


How it works
~~~~~~~~~~~~

A function containing ``yield`` is a **generator**.  All generators
are asynchronous; when called they return a generator object instead
of running to completion.  The ``@gen.coroutine`` decorator
communicates with the generator via the ``yield`` expressions, and
with the coroutine's caller by returning a `.Future`.

Here is a simplified version of the coroutine decorator's inner loop::

    # Simplified inner loop of tornado.gen.Runner
    def run(self):
        # send(x) makes the current yield return x.
        # It returns when the next yield is reached
        future = self.gen.send(self.next)
        def callback(f):
            self.next = f.result()
            self.run()
        future.add_done_callback(callback)

The decorator receives a `.Future` from the generator, waits (without
blocking) for that `.Future` to complete, then "unwraps" the `.Future`
and sends the result back into the generator as the result of the
``yield`` expression.  Most asynchronous code never touches the `.Future`
class directly except to immediately pass the `.Future` returned by
an asynchronous function to a ``yield`` expression.

How to call a coroutine
~~~~~~~~~~~~~~~~~~~~~~~

Coroutines do not raise exceptions in the normal way: any exception
they raise will be trapped in the `.Future` until it is yielded. This
means it is important to call coroutines in the right way, or you may
have errors that go unnoticed::

    @gen.coroutine
    def divide(x, y):
        return x / y

    def bad_call():
        # This should raise a ZeroDivisionError, but it won't because
        # the coroutine is called incorrectly.
        divide(1, 0)

In nearly all cases, any function that calls a coroutine must be a
coroutine itself, and use the ``yield`` keyword in the call. When you
are overriding a method defined in a superclass, consult the
documentation to see if coroutines are allowed (the documentation
should say that the method "may be a coroutine" or "may return a
`.Future`")::

    @gen.coroutine
    def good_call():
        # yield will unwrap the Future returned by divide() and raise
        # the exception.
        yield divide(1, 0)

Sometimes you may want to "fire and forget" a coroutine without waiting
for its result. In this case it is recommended to use `.IOLoop.spawn_callback`,
which makes the `.IOLoop` responsible for the call. If it fails,
the `.IOLoop` will log a stack trace::

    # The IOLoop will catch the exception and print a stack trace in
    # the logs. Note that this doesn't look like a normal call, since
    # we pass the function object to be called by the IOLoop.
    IOLoop.current().spawn_callback(divide, 1, 0)

Using `.IOLoop.spawn_callback` in this way is *recommended* for
functions using ``@gen.coroutine``, but it is *required* for functions
using ``async def`` (otherwise the coroutine runner will not start).

Finally, at the top level of a program, *if the IOLoop is not yet
running,* you can start the `.IOLoop`, run the coroutine, and then
stop the `.IOLoop` with the `.IOLoop.run_sync` method. This is often
used to start the ``main`` function of a batch-oriented program::

    # run_sync() doesn't take arguments, so we must wrap the
    # call in a lambda.
    IOLoop.current().run_sync(lambda: divide(1, 0))

Coroutine patterns
~~~~~~~~~~~~~~~~~~

Interaction with callbacks
^^^^^^^^^^^^^^^^^^^^^^^^^^

To interact with asynchronous code that uses callbacks instead of
`.Future`, wrap the call in a `.Task`.  This will add the callback
argument for you and return a `.Future` which you can yield:

.. testcode::

    @gen.coroutine
    def call_task():
        # Note that there are no parens on some_function.
        # This will be translated by Task into
        #   some_function(other_args, callback=callback)
        yield gen.Task(some_function, other_args)

.. testoutput::
   :hide:

Calling blocking functions
^^^^^^^^^^^^^^^^^^^^^^^^^^

The simplest way to call a blocking function from a coroutine is to
use a `~concurrent.futures.ThreadPoolExecutor`, which returns
``Futures`` that are compatible with coroutines::

    thread_pool = ThreadPoolExecutor(4)

    @gen.coroutine
    def call_blocking():
        yield thread_pool.submit(blocking_func, args)

Parallelism
^^^^^^^^^^^

The coroutine decorator recognizes lists and dicts whose values are
``Futures``, and waits for all of those ``Futures`` in parallel:

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

Interleaving
^^^^^^^^^^^^

Sometimes it is useful to save a `.Future` instead of yielding it
immediately, so you can start another operation before waiting:

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

This pattern is most usable with ``@gen.coroutine``. If
``fetch_next_chunk()`` uses ``async def``, then it must be called as
``fetch_future =
tornado.gen.convert_yielded(self.fetch_next_chunk())`` to start the
background processing.

Looping
^^^^^^^

Looping is tricky with coroutines since there is no way in Python
to ``yield`` on every iteration of a ``for`` or ``while`` loop and
capture the result of the yield.  Instead, you'll need to separate
the loop condition from accessing the results, as in this example
from `Motor <https://motor.readthedocs.io/en/stable/>`_::

    import motor
    db = motor.MotorClient().test

    @gen.coroutine
    def loop_example(collection):
        cursor = db.collection.find()
        while (yield cursor.fetch_next):
            doc = cursor.next_object()

Running in the background
^^^^^^^^^^^^^^^^^^^^^^^^^

`.PeriodicCallback` is not normally used with coroutines. Instead, a
coroutine can contain a ``while True:`` loop and use
`tornado.gen.sleep`::

    @gen.coroutine
    def minute_loop():
        while True:
            yield do_something()
            yield gen.sleep(60)

    # Coroutines that loop forever are generally started with
    # spawn_callback().
    IOLoop.current().spawn_callback(minute_loop)

Sometimes a more complicated loop may be desirable. For example, the
previous loop runs every ``60+N`` seconds, where ``N`` is the running
time of ``do_something()``. To run exactly every 60 seconds, use the
interleaving pattern from above::

    @gen.coroutine
    def minute_loop2():
        while True:
            nxt = gen.sleep(60)   # Start the clock.
            yield do_something()  # Run while the clock is ticking.
            yield nxt             # Wait for the timer to run out.
