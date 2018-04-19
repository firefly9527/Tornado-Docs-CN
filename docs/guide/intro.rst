介绍
------------

`Tornado <http://www.tornadoweb.org>`_ 是一个Python web框架和异步网络库，最初由 `FriendFeed
<https://en.wikipedia.org/wiki/FriendFeed>`_ 开发。 通过使用非阻塞网络I/O，Tornado
可以支持上万级的连接，它适合构建
`long polling <http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_，
`WebSockets <http://en.wikipedia.org/wiki/WebSocket>`_，和其他需要与每个用户保持长连接的应用。

Tornado 大体上可以被分为4个主要的部分:

* web框架 (包括用于创建web应用的 `.RequestHandler` 子类，以及很多其他支持的类)。
* HTTP的客户端和服务端实现 (`.HTTPServer` 和
  `.AsyncHTTPClient`)。
* 异步网路库 (包括 `.IOLoop`
  和 `.IOStream` 类) 为HTTP组件提供构建模块，也可以用来实现其他协议。
* 协程库 (`tornado.gen`) 允许以更直接的方式编写异步代码，而不是链式回调的方式。

Tornado web 框架和HTTP 服务器一起为 `WSGI <http://www.python.org/dev/peps/pep-3333/>`_
提供了一个全栈式的选择。在WSGI容器中使用Tornado web框架 (`.WSGIAdapter`) 或者使用Tornado HTTP server作为一个其他WSGI框架
的容器 (`.WSGIContainer`)，这样的组合方式都是有局限性的；为了充分利用Tornado的特性,你需要一起使用Tornado的web框架和HTTP server。
