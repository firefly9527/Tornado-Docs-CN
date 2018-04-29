.. title:: Tornado Web Server

.. meta::
    :google-site-verification: g4bVhgwbVO1d9apCUsT-eKlApg31Cygbp8VGZY8Rf0g

|Tornado Web Server|
====================

.. |Tornado Web Server| image:: tornado.png
    :alt: Tornado Web Server

`Tornado <http://www.tornadoweb.org>`_ 是一个Python web框架和异步网络库，最初由 `FriendFeed <http://friendfeed.com>`_ 开发。 
通过使用非阻塞网络I/O，Tornado 可以支持上万级的连接，它适合构建 `长连接 <http://en.wikipedia.org/wiki/Push_technology#Long_polling>`_ , `WebSockets <http://en.wikipedia.org/wiki/WebSocket>`_ ，和其他需要与每个用户保持长连接的应用。

相关链接
-----------

* 当前版本: |version| (`PyPI下载 <https://pypi.python.org/pypi/tornado>`_, :doc: `版本记录 <releases>`)
* `源码 (github) <https://github.com/tornadoweb/tornado>`_
* 邮件列表: `discussion <http://groups.google.com/group/python-tornado>`_ 和 `announcements <http://groups.google.com/group/python-tornado-announce>`_
* `Stack Overflow <http://stackoverflow.com/questions/tagged/tornado>`_
* `Wiki <https://github.com/tornadoweb/tornado/wiki/Links>`_

Hello, world
------------

这是一个简单的Tornado web应用::

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

这个例子没有使用Tornado的任何异步特性；更具体的可以看 `这个简单的聊天室
<https://github.com/tornadoweb/tornado/tree/stable/demos/chat>`_ 。

安装
------------

::

    pip install tornado

Tornado在 `PyPI列表中 <http://pypi.python.org/pypi/tornado>`_ ，可以使用
 ``pip`` 安装。 注意源码发布中包含的示例应用可能不会出现在这种方式安装
 的代码中，所以你也可能希望通过下载一份源码的拷贝来进行安装 `git repository <https://github.com/tornadoweb/tornado>`_ 。

**安装提示**: Tornado 可以运行在 Python 2.7, and 3.4+
对于Python 2, 2.7.9及更新的版本。 *强烈* 推荐提高对SSL的支持。
另外Tornado的依赖包可能通过 ``pip`` 或 ``setup.py install``,
被自动安装，下面的这些可选包可能是有用的:

* `concurrent.futures <https://pypi.python.org/pypi/futures>`_ is the
  recommended thread pool for use with Tornado and enables the use of
  `~tornado.netutil.ThreadedResolver`.  It is needed only on Python 2;
  Python 3 includes this package in the standard library.
* `pycurl <http://pycurl.sourceforge.net>`_ is used by the optional
  ``tornado.curl_httpclient``.  Libcurl version 7.22 or higher is required.
* `Twisted <http://www.twistedmatrix.com>`_ may be used with the classes in
  `tornado.platform.twisted`.
* `pycares <https://pypi.python.org/pypi/pycares>`_ is an alternative
  non-blocking DNS resolver that can be used when threads are not
  appropriate.
* `monotonic <https://pypi.python.org/pypi/monotonic>`_ or `Monotime
  <https://pypi.python.org/pypi/Monotime>`_ add support for a
  monotonic clock, which improves reliability in environments where
  clock adjustments are frequent. No longer needed in Python 3.

**平台**: Tornado可以运行在任何版本的Unix上, 虽然为了最好的性能和可扩展性，只有Linux(使用 ``epoll``)
和BSD (使用 ``kqueue``) 是推荐的产品部署环境(尽管Mac OS X通过BSD发展来并且支持kqueue,但它的网络质量很差，
所以它只适合开发使用)。Tornado也可以运行在Windows上，虽然它的配置不是官方支持的,同时也仅仅推荐开发使用.
如果不修改Tornado IOLoop接口，就不可能添原生的Tornado Windows IOLoop实现，或者利用Windows IOCP 提供像
异步IO或者Twisted的绿程那样的支持。

文档
-------------

这份中文文档只提供 `Epub 格式
<https://readthedocs.org/projects/tornado/downloads/>`_.

.. toctree::
   :titlesonly:

   guide
   webframework
   http
   networking
   coroutine
   integration
   utilities
   faq
   releases

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`

讨论和支持
----------------------

你可以在 `Tornado开发者邮件列表<http://groups.google.com/group/python-tornado>`_ 讨论Tornado，
并在 `GitHub issue tracker <https://github.com/tornadoweb/tornado/issues>`_ 报告bug.
其他资源可以在 `Tornado wiki <https://github.com/tornadoweb/tornado/wiki/Links>`_ 找到.
先版本会在 `这个邮件列表<http://groups.google.com/group/python-tornado-announce>`_
宣布发布。

Tornado的开源许可是 `Apache License, Version 2.0 <http://www.apache.org/licenses/LICENSE-2.0.html>`_.

这份文档的所有内容许可是 `Creative Commons 3.0 <http://creativecommons.org/licenses/by/3.0/>`_.
