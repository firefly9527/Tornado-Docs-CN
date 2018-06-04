.. currentmodule:: tornado.web

.. testsetup::

   import tornado.web

Tornado web应用结构
======================================

通常一个Tornado web应用包括一个或者多个 `.RequestHandler` 子类,
一个可以将收到的请求路由到对应handler的 `.Application`  对象,
和 一个启动服务的 ``main()`` 函数.


一个最小的 "hello world" 例子就像下面这样:

.. testcode::

    import tornado.ioloop
    import tornado.web

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    def make_app():
        return tornado.web.Application([
            (r"/", MainHandler),
        ])

    if __name__ == "__main__":
        app = make_app()
        app.listen(8888)
        tornado.ioloop.IOLoop.current().start()

.. testoutput::
   :hide:

 ``Application`` 对象
~~~~~~~~~~~~~~~~~~~~~~~~~~
 `.Application`  对象是负责全局配置的, 包括映射请求转发给处理程序的路由表.

路由表是 `.URLSpec` 对象(或元组)的列表, 其中每个都包含(至少)一个正则 表达式和一个处理类。
关于解析顺序; 第一个匹配的规则会被使用。如果正则表达 式包含捕获组，
这些组会被作为 *路径参数* 传递给处理函数的HTTP方法。
如果一个字典作为 `.URLSpec` 的第三个参数被传递, 它会作为 *初始参数* 传递给 `.RequestHandler.initialize`。
最后 URLSpec 可能有一个名字，这将允许它被 `.RequestHandler.reverse_url` 使用。

例如, 在这个片段中根URL ``/`` 映射到了 ``MainHandler``，
像 ``/story/`` 后跟着一个数字这种形式的URL被映射到了 ``StoryHandler``。
这个数字被传递(作为字符串)给 ``StoryHandler.get``。::

    class MainHandler(RequestHandler):
        def get(self):
            self.write('<a href="%s">link to story 1</a>' %
                       self.reverse_url("story", "1"))

    class StoryHandler(RequestHandler):
        def initialize(self, db):
            self.db = db

        def get(self, story_id):
            self.write("this is story %s" % story_id)

    app = Application([
        url(r"/", MainHandler),
        url(r"/story/([0-9]+)", StoryHandler, dict(db=db), name="story")
        ])

 `.Application` 构造函数有很多关键字参数可以用于自定义应用程序的行为和使用某些特性(或者功能);
 完整列表请查看 `.Application.settings`。

``RequestHandler`` 子类
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Tornado web 应用程序的大部分工作是在 RequestHandler 子类下完成的。
处理子类的主入口点是一个命名为处理HTTP方法的函数: get(), post(), 等等。
每个处理程序可以定义一个或者多个这种方法来处理不同的HTTP动作。
如上所述, 这些方法将被匹配路由规则的捕获组对应的参数调用.

在处理程序中, 调用方法如 `.RequestHandler.render` 或者 `.RequestHandler.write` 产生一个响应.
``render()`` 通过名字加载一个 `.Template` 并使用给定的参数渲染它。
``write()``  被用于非模板基础的输出;
它接受字符串, 字节, 和字典(字典会被编码成JSON)。

在  `.RequestHandler` 中的很多方法的设计是为了在子类中复写和在整个应用中使用。
常用的方法是定义一个 ``BaseHandler`` 类,
复写一些方法例如 `~.RequestHandler.write_error` 和  `~.RequestHandler.get_current_user` 然
后子类继承使用你自己的 ``BaseHandler`` 而不是 `.RequestHandler` 在你所有具体的处理程序中.


处理输入请求
~~~~~~~~~~~~~~~~~~~~~~

处理器可以使用 ``self.request`` 访问当前请求对象。通过
`~tornado.httputil.HTTPServerRequest` 类的定义查看完整的属性列表。

通过form表单传递的请求数据会被解析并且能过通过 `~.RequestHandler.get_query_argument`
和  `~.RequestHandler.get_body_argument` 得到。

.. testcode::

    class MyFormHandler(tornado.web.RequestHandler):
        def get(self):
            self.write('<html><body><form action="/myform" method="POST">'
                       '<input type="text" name="message">'
                       '<input type="submit" value="Submit">'
                       '</form></body></html>')

        def post(self):
            self.set_header("Content-Type", "text/plain")
            self.write("You wrote " + self.get_body_argument("message"))

.. testoutput::
   :hide:


由form表单编码是不能确定一个标签的值是单一的还是一多个组成的列表  `.RequestHandler` 有明确的
方式来允许应用程序表明它期望接收的值类型。对于列表，使用 `~.RequestHandler.get_query_arguments`
和 `~.RequestHandler.get_body_arguments` 而非获取单个值的方法。

通过表单上传的文件， 可以从 ``self.request.files`` 获取相关信息，它遍历文件名字(HTML标签 ``<input type="file">``) 到一个列表
。 每个文件都是一个字典形式 ``{"filename":..., "content_type":..., "body":...}`` 。
 ``files`` 对象如果文件上传是通过一个表单那么它是当前唯一的
(比如 ``multipart/form-data`` Content-Type); 反之可以使用 ``self.request.body`` 获取文件对象。
默认上传的文件是完全缓存在内存中的，如果你需要处理的文件比较大那么可以看看 `.stream_request_body` 类装饰器。

文件上传的例子： `file_receiver.py <https://github.com/tornadoweb/tornado/tree/master/demos/file_upload/>`_
展示了两种方式上传文件。

由于HTML表单编码格式的怪异 (比如 在单数和复数参数的含糊不清),
Tornado 不会试图统一表单参数和其他输入类型的参数。特别是, 我们不解析JSON请求体。
应用程序希望使用JSON代替表单编码可以复写  `~.RequestHandler.prepare`  来解析它们的请求::

    def prepare(self):
        if self.request.headers["Content-Type"].startswith("application/json"):
            self.json_args = json.loads(self.request.body)
        else:
            self.json_args = None


复写RequestHandler的方法
~~~~~~~~~~~~~~~~~~~~~~~

除了 ``get()``/``post()``/等，在 `.RequestHandler` 中的某些其他方法 被设计成了在必要的时候让子类重写，
在每个请求中, 会发生下面这样的的顺序调用:

1. 在每次请求时生成一个新的 `.RequestHandler` 对象
2. `~.RequestHandler.initialize()` 被 `.Application` configuration中的初始化参数被调用。 ``initialize`` 通常应该只保存成员变量传递的参数;
它不可能产生任何输出或者调用方法, 例如  `~.RequestHandler.send_error`.
3. `~.RequestHandler.prepare()` 被调用。这在你所有处理子类共享的基类中是最有用的,
   ``prepare`` 无论是使用哪种HTTP方法, prepare 都会被调用。 ``prepare`` 可能会产生输出；
   如果它调用 `~.RequestHandler.finish` (或者 redirect, 等), 处理会在这里结束
4. 其中一种HTTP方法被调用: ``get()``, ``post()``, ``put()`` 等。
如果URL的正则表达式包含捕获组, 它们会被作为参数传递给这个方法.
5. 当请求结束, `~.RequestHandler.on_finish()` 方法被调用。
对于同步 处理程序会在 ``get()``(等)后立即返回; 对于异步处理程序,会在调用 `~.RequestHandler.finish()` 后返回.

所有的方法都被设计为可以复写并记录在 `.RequestHandler` 的文档中。
其中最常被复写的方法包括:

- `~.RequestHandler.write_error` -
  输出报告错误的HTML页面。
- `~.RequestHandler.on_connection_close` - 当客户端断开连接时；应用可以检测这种情况，
并中断后续处理。注意这不能保证当一个连接关闭时被及时发现。
- `~.RequestHandler.get_current_user` - 参考 :ref:`user-authentication`
- `~.RequestHandler.get_user_locale` - 返回 `.Locale` 对象给当前用户使用
- `~.RequestHandler.set_default_headers` - 可以被用来给response响应设置额外的头部字段(例如自定义的 ``Server``
  头部字段)

错误处理
~~~~~~~~~~~~~~

如果一个处理器抛出异常，Tornado会调用 `.RequestHandler.write_error` 方法生成一个错误报告页。
`tornado.web.HTTPError` 可以被用来生成一个指定的状态码；所有其他的异常都会返回一个500状态码。

默认的错误也包含一个debug模式下的调用堆和以及一行错误描述(例如 "500: Internal Server Error")。
为了创建自定义的错误页面，可以复写 `RequestHandler.write_error`
(可能在一个所有处理器的共有父类里)。 这个方法可能通过常用的方法产生输出，例如
 `~RequestHandler.write` 和 `~RequestHandler.render`。
如果错误是由异常引起的，``exc_info`` 将作为一个关键字参数传递(注意这个异常不能保证的是
`sys.exc_info`, 所以 ``write_error`` 必须使用比如 `traceback.format_exception` 代替
`traceback.format_exc`).

也可以在常规的处理方法中调用 `~.RequestHandler.set_status` 代替 ``write_error`` 生成一个response并返回。
特殊的例外是 `tornado.web.Finish` 在直接返回不方便的情况下能够在不调用  ``write_error``
前结束处理程序。

对于404错误, 使用  ``default_handler_class`` `Application setting
<.Application.settings>`.  这个处理程序会复写
`~.RequestHandler.prepare` 而不是一个更具体的方法比如
``get()`` 所以它可以在任何HTTP方法下工作。 它应该会产生如上所说的错误页面：
要么抛出一个 ``HTTPError(404)``
要么复写 ``write_error``, 或者调用 ``self.set_status(404)``
或者在  ``prepare()`` 直接生成响应。


重定向
~~~~~~~~~~~
在Tornado里有 `.RequestHandler.redirect` 和 `.RedirectHandler` 这两种主要的方式重定向请求。

你可以在使用 `.RequestHandler` 里的 ``self.redirect()`` 方法重定向用户到其他地方。
还有个可选参数 ``permanent`` ，你可以使用它来决定这个重定向是永久的。这个参数默认是
``False``, 这会生成一个 ``302 Found`` HTTP 响应码。 适合在比如
``POST`` 请求成功后的重定向。如果这个参数的值为 ``True`` , 那么HTTP 响应码将会是 ``301 Moved
Permanently`` ，更适合比如 SEO友好方法中把重定向到一个权威的URL。

`.RedirectHandler` 让你可以直接在 `.Application` 路由表中配置。
例如，配置静态重定向::

    app = tornado.web.Application([
        url(r"/app", tornado.web.RedirectHandler,
            dict(url="http://itunes.apple.com/my-app-id")),
        ])

`.RedirectHandler` 也支持正则表达式替换。下面的规则重定向所有 ``/pictures/`` 开头的请求用
 ``/photos/`` 替换::

    app = tornado.web.Application([
        url(r"/photos/(.*)", MyPhotoHandler),
        url(r"/pictures/(.*)", tornado.web.RedirectHandler,
            dict(url=r"/photos/{0}")),
        ])

不像  `.RequestHandler.redirect`, `.RedirectHandler` 默认使用永久重定向。这是
因为路由表在运行时不会改变，所以被认为是永久的。 当在处理程序中发现重定向的时候，可能是其他处理逻辑想要的结果，用 `.RedirectHandler` 发送临时重定向, 需要将 `.RedirectHandler` 初始化参数这样设置
``permanent=False`` 。

异步处理
~~~~~~~~~~~~~~~~~~~~~

Tornado的处理器默认是同步的: 当
``get()``/``post()`` 方法返回, 请求被视为结束并返回响应。当一个处理器在运行的时候其他所有
请求都被阻塞， 任何长时间运行的处理器都应当是异步的，这样就可以通过非阻塞的方式执行较慢的操作。
这个话题更详细的内容包含在 :doc:`async` ；这部分是关于在
 `.RequestHandler` 子类中的异步技术的细节

使用 `.coroutine` 装饰器是让处理器异步执行的最简单方式。 这允许你使用 ``yield`` 关键字执行非阻塞I/O，并且直到协程返回才发送响应
。查看 :doc:`coroutines` 了解更多细节。

In some cases, coroutines may be less convenient than a
callback-oriented style, in which case the `.tornado.web.asynchronous`
decorator can be used instead.  When this decorator is used the response
is not automatically sent; instead the request will be kept open until
some callback calls `.RequestHandler.finish`.  It is up to the application
to ensure that this method is called, or else the user's browser will
simply hang.

Here is an example that makes a call to the FriendFeed API using
Tornado's built-in `.AsyncHTTPClient`:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            http = tornado.httpclient.AsyncHTTPClient()
            http.fetch("http://friendfeed-api.com/v2/feed/bret",
                       callback=self.on_response)

        def on_response(self, response):
            if response.error: raise tornado.web.HTTPError(500)
            json = tornado.escape.json_decode(response.body)
            self.write("Fetched " + str(len(json["entries"])) + " entries "
                       "from the FriendFeed API")
            self.finish()

.. testoutput::
   :hide:

When ``get()`` returns, the request has not finished. When the HTTP
client eventually calls ``on_response()``, the request is still open,
and the response is finally flushed to the client with the call to
``self.finish()``.

For comparison, here is the same example using a coroutine:

.. testcode::

    class MainHandler(tornado.web.RequestHandler):
        @tornado.gen.coroutine
        def get(self):
            http = tornado.httpclient.AsyncHTTPClient()
            response = yield http.fetch("http://friendfeed-api.com/v2/feed/bret")
            json = tornado.escape.json_decode(response.body)
            self.write("Fetched " + str(len(json["entries"])) + " entries "
                       "from the FriendFeed API")

.. testoutput::
   :hide:

For a more advanced asynchronous example, take a look at the `chat
example application
<https://github.com/tornadoweb/tornado/tree/stable/demos/chat>`_, which
implements an AJAX chat room using `long polling
<http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_.  Users
of long polling may want to override ``on_connection_close()`` to
clean up after the client closes the connection (but see that method's
docstring for caveats).
