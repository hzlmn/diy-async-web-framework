<div align="center">
  <img src="media/logo.png"/>
</div>

### 简介

近几年来，异步编程在Python社区中变得越来越受欢迎。诸如`aiohttp`之类的异步库，在使用量上呈现出惊人的增长态势，因为它们能够并发处理大量链接，并在此基础上保持代码的可读性与简洁程度。而就在不久前，Django也[承诺](https://docs.djangoproject.com/en/dev/releases/3.0/#asgi-support)将在下个大版本中增加对异步的支持。种种迹象都表明，Python的异步编程拥有非常不错的前景。然而，对于很大一部分习惯于使用标准阻塞模型的开发人员来说，这些异步工具的工作机制显得十分令人困惑。因此，我将在这份简短的指南中从零构建一个简化版的`aiohttp`，并通过这种方式深入幕后，理清Python异步编程的工作过程。我们将从官方文档中的一个基本示例出发，并逐步增加我们所感兴趣的必要功能。让我们立刻开始吧！

在这篇指南中，我将假设你已经对[asyncio](https://docs.python.org/3/library/asyncio.html)有了最基本的了解。如果你需要回顾一些相关知识的话，这几篇文章或许能帮到你：

   - [Intro to asyncio](https://www.blog.pythonlibrary.org/2016/07/26/python-3-an-intro-to-asyncio)
   - [Understanding asynchronous programming in Python](https://dbader.org/blog/understanding-asynchronous-programming-in-python)

当然，如果你已经等不及了，可以直接在这里找最终的源码：[`hzlmn/sketch`](https://github.com/hzlmn/sketch)

## 相关项目

- [500 Lines or Less](https://github.com/aosabook/500lines)

## 目录 :book:

* [Asyncio库的低层级API：Transports与Protocols](#asyncio库的低层级apitransports与protocols)
* [在服务器程序上实现协议](#在服务器程序上实现协议)
* [Request/Response对象](#requestresponse对象)
* [Application与UrlDispatcher](#application与urldispatcher)
* [更进一步](#更进一步)
   * [路由参数](#路由参数)
   * [中间件](#中间件)
   * [App的生命周期钩子](#app的生命周期钩子)
   * [完善异常处理](#完善异常处理)
   * [优雅地退出](#优雅地退出)
* [应用程序示例](#应用程序示例)
* [总结](#总结)

## Asyncio库的低层级API：Transports与Protocols

`Asyncio`库经过了漫长的演变才成为现在这个样子。曾经的asyncio是作为一个名为“tulip”的底层工具而被创造出来的。那个时候，开发高层级的应用程序可不像今天这么愉快。

在如今大多数情况下，`asyncio`都被作为一种高层级的API来使用，不过该库也提供了一些低层级的助手来供那些库的设计者管理事件循环，以及实现网络或进程间的通信协议。

`Asyncio`库仅为`TCP`, `UDP`, `SSL` 以及子进程提供了开箱即用的支持。而其他异步库则基于`asyncio`库所提供基础传输与编程接口实现了它们所需的更高层级的协议，如`HTTP`、`FTP`等等。

所有通信都是通过链接Transports和Protocols来完成的。简单地说，Transports描述了我们该如何传送数据，而Protocols负责决定传送哪些数据。

关于Transports与Protocols，`asyncio`库提供了一份非常棒的官方文档，你可以在[这里](https://docs.python.org/3.8/library/asyncio-protocol.html#asyncio-transport)访问它，并进行更深入的了解。

作为项目的第一步，让我们先来编写一个简单的`TCP`回显服务器。

`server.py`

```python
import asyncio

class Server(asyncio.Protocol):
    def connection_made(self, transport):
        self._transport = transport

    def data_received(self, data):
        message = data.decode()

        self._transport.write(data)

        self._transport.close()

loop = asyncio.get_event_loop()

coro = loop.create_server(Server, '127.0.0.1', 8080)
server = loop.run_until_complete(coro)

try:
    loop.run_forever()
except KeyboardInterrupt:
    pass

server.close()
loop.run_until_complete(server.wait_closed())
loop.close()
```

```shell
$ curl http://127.0.0.1:8080
GET / HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: curl/7.54.0
Accept: */*
```

从以上示例可以看出，构建异步服务器程序的代码非常简单。不过如果你想构建一个更高层级的应用程序，仅凭这些还不太够。

由于 `HTTP` 协议工作在 `TCP` 协议之上，我们现在已经可以向我们的服务器程序发送 `HTTP` 请求了。然而，接收并使用未经格式化处理的 `HTTP`报文显然是非常困难的。所以我们下一步的工作就是去增加一种更好的 `HTTP` 处理机制。


## 在服务器程序上实现协议

让我们为服务器程序增加一个解析 `HTTP`请求的功能，这样我们就可以提取并使用请求头、请求正文以及请求路径等信息。如何解析 `HTTP`请求是一个非常复杂的话题，这远远超出了本指南所研究的范围，因此我们将直接使用[httptools](https://github.com/MagicStack/httptools)来解析请求。[httptools](https://github.com/MagicStack/httptools)是一个效率高，兼容性好，并且相当灵活的`HTTP`解析器。

此外，`aiohttp`项目也实现了一个基于Python的`HTTP`解析器，并且这个解析器已经被集成到了Node的 [`http-parser`](https://github.com/nodejs/http-parser/tree/77310eeb839c4251c07184a5db8885a572a08352)中。

接下来，我们需要实现一个用来与服务器类组合的解析器类。

`http_parser.py`

```python
class HttpParserMixin:
    def on_body(self, data):
        self._body = data

    def on_url(self, url):
        self._url = url

    def on_message_complete(self):
        print(f"Received request to {self._url.decode(self._encoding)}")

    def on_header(self, header, value):
        header = header.decode(self._encoding)
        self._headers[header] = value.decode(self._encoding)
```

实现解析器类 `HttpParserMixin`后，将它与我们的 `Server` 类组合到一起。

`server.py`

```python
import asyncio

from httptools import HttpRequestParser

from .http_parser import HttpParserMixin

class Server(asyncio.Protocol, HttpParserMixin):
    def __init__(self, loop):
        self._loop = loop
        self._encoding = "utf-8"
        self._url = None
        self._headers = {}
        self._body = None
        self._transport = None
        self._request_parser = HttpRequestParser(self)

    def connection_made(self, transport):
        self._transport = transport

    def connection_lost(self, *args):
        self._transport = None

    def data_received(self, data):
        # Pass data to our parser
        self._request_parser.feed_data(data)
```

现在，我们终于拥有了一个能够解析传入的 `HTTP` 请求，并从中提取重要信息的服务器。让我们把它运行起来。

`server.py`

```python
if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    serv = Server(loop)
    server = loop.run_until_complete(loop.create_server(lambda: serv, port=8080))

    try:
        print("Started server on ::8080")
        loop.run_until_complete(server.serve_forever())
    except KeyboardInterrupt:
        server.close()
        loop.run_until_complete(server.wait_closed())
        loop.stop()

```

```sh
> python server.py
Started server on ::8080
```

```sh
> curl http://127.0.0.1:8080/hello
```

## Request/Response对象

目前，我们已经拥有了一个可以解析 `HTTP`请求的服务器程序。但为了构建应用程序，我们还需要在某些方面做进一步的抽象。

现在让我们来创建一个用于将所有 `HTTP` 请求信息组合到一起的 `Request` 类。请确保已经安装了 `yarl` 库，我们将使用它来处理url。

`request.py`

```python
import json

from yarl import URL

class Request:
    _encoding = "utf_8"

    def __init__(self, method, url, headers, version=None, body=None, app=None):
        self._version = version
        self._method = method.decode(self._encoding)
        self._url = URL(url.decode(self._encoding))
        self._headers = headers
        self._body = body

    @property
    def method(self):
        return self._method

    @property
    def url(self):
        return self._url

    @property
    def headers(self):
        return self._headers

    def text(self):
        if self._body is not None:
            return self._body.decode(self._encoding)

    def json(self):
        text = self.text()
        if text is not None:
            return json.loads(text)

    def __repr__(self):
        return f"<Request at 0x{id(self)}>"
```

下一步，我们还需要这样一个结构：它能帮助我们以程序员友好的方式描述 `HTTP` 响应，并将其转化为原始的 `HTTP` 报文。这种转化后的报文可以通过 `asyncio.Transport`处理。

`response.py`

```python
import http.server

web_responses = http.server.BaseHTTPRequestHandler.responses

class Response:
    _encoding = "utf-8"

    def __init__(
        self,
        body=None,
        status=200,
        content_type="text/plain",
        headers=None,
        version="1.1",
    ):
        self._version = version
        self._status = status
        self._body = body
        self._content_type = content_type
        if headers is None:
            headers = {}
        self._headers = headers

    @property
    def body(self):
        return self._body

    @property
    def status(self):
        return self._status

    @property
    def content_type(self):
        return self._content_type

    @property
    def headers(self):
        return self._headers
    
    def add_body(self, data):
        self._body = data

    def add_header(self, key, value):
        self._headers[key] = value
    
    def __str__(self):
        """We will use this in our handlers, it is actually generation of raw HTTP response,
        that will be passed to our TCP transport
        """
        status_msg, _ = web_responses.get(self._status)
        
        messages = [
            f"HTTP/{self._version} {self._status} {status_msg}",
            f"Content-Type: {self._content_type}",
            f"Content-Length: {len(self._body)}",
        ]

        if self.headers:
            for header, value in self.headers.items():
                messages.append(f"{header}: {value}")

        if self._body is not None:
            messages.append("\r\n" + self._body)

        return "\r\n".join(messages)

    def __repr__(self):
        return f"<Response at 0x{id(self)}>"
```

如上所示，代码非常简单。我们封装了所有的数据，并为其属性定义了对应的getter方法。我们还定义了一些之后要用到的助手方法，用于处理 `text` 以及 `json`格式的报文体。接下来的任务就是更新一下服务器程序 ，使之能够通过接收到的消息来创建 `Request` 对象。

 `Request` 对象应当在解析完整个请求后创建，因此我们把创建工作添加到解析器类的 `on_message_complete`事件的处理方法中。

`http_parser.py`

```python
class HttpParserMixin:
    ...

    def on_message_complete(self):
        self._request = self._request_class(
            version=self._request_parser.get_http_version(),
            method=self._request_parser.get_method(),
            url=self._url,
            headers=self._headers,
            body=self._body,
        )

    ...
```

`Server`类也需要改造一下，使之能够创建 `Response` 对象，并将编码后的消息传递给`asyncio.Transport`。

`server.py`

```python
from .response import Response
...

class Server(asyncio.Protocol, HttpParserMixin):
    ...

    def __init__(self, loop):
        ...
        self._request = None
        self._request_class = Request

    ...

    def data_received(self, data):
        self._request_parser.feed_data(data)

        resp = Response(body=f"Received request on {self._request.url}")
        self._transport.write(str(resp).encode(self._encoding))

        self._transport.close()
```

现在再去运行 `server.py`，我们就可以使用curl去请求`http://localhost:8080/path`，并在响应中看到 `Received request on /path`了 。

## Application与UrlDispatcher

现阶段，我们已经拥有了能够解析`HTTP`请求的服务器，以及能够处理请求周期的Request/Response对象。然而，我们这个手写的工具包中还缺少一些重要的概念。首先，我们现在只有一个主请求处理器，而在大型的应用程序中，我们需要很多请求处理器来处理不同的路由。因此我们还需要一种机制来为不同路由分别注册处理程序。

现在让我们用内置的字典来实现一个尽可能简单的 `UrlDispatcher`。该字典的键是一个由请求方法与请求路径组成的二元组，而值是一个处理程序。此外我们还需要一个单独的处理程序去处理那些无法识别路由的请求。

`router.py`

```python
from .response import Response

class UrlDispatcher:
    def __init__(self):
        self._routes = {}

    async def _not_found(self, request):
         return Response(f"Not found {request.url} on this server", status=404)

    def add_route(self, method, path, handler):
        self._routes[(method, path)] = handler

    def resolve(self, request):
        key = (request.method, request.url.path)
        if key not in self._routes:
            return self._not_found
        return self._routes[key]
```

当然，我们还缺少很多别的东西，比如参数化的路由等等。我们会在之后增加它们，现在还是让程序尽可能保持简单吧。

直接与底层的 `Server` 进行交互是非常麻烦的，所以，接下来我们需要一个`Applicatio `容器，用来组合所有与应用相关的信息。

```python
import asyncio

from .router import UrlDispatcher
from .server import Server
from .response import Response

class Application:
    def __init__(self, loop=None):
        if loop is None:
            loop = asyncio.get_event_loop()

        self._loop = loop
        self._router = UrlDispatcher()

    @property
    def loop(self):
        return self._loop

    @property
    def router(self):
        return self._router

    def _make_server(self):
        return Server(loop=self._loop, handler=self._handler, app=self)

    async def _handler(self, request, response_writer):
        """Process incoming request"""
        handler = self._router.resolve(request)
        resp = await handler(request)

        if not isinstance(resp, Response):
            raise RuntimeError(f"expect Response instance but got {type(resp)}")

        response_writer(resp)

```

我们需要对 `Server` 稍加修改，并增加一个 `response_writer` 方法来将数据传送给transport。同时，我们需要在 `Server` 的构造函数中增加 `handler` 属性和 `app`属性。这些属性将被用来调用相应的处理程序。

`server.py`

```python
class Server(asyncio.Protocol, HttpParserMixin):
    ...

    def __init__(self, loop, handler, app):
        self._loop = loop
        self._url = None
        self._headers = {}
        self._body = None
        self._transport = None
        self._request_parser = HttpRequestParser(self)
        self._request = None
        self._request_class = Request
        self._request_handler = handler
        self._request_handler_task = None

    def response_writer(self, response):
        self._transport.write(str(response).encode(self._encoding))
        self._transport.close()
    
    ...

```

`http_parser.py`

```python
class HttpParserMixin:
    def on_body(self, data):
        self._body = data

    def on_url(self, url):
        self._url = url

    def on_message_complete(self):
        self._request = self._request_class(
            version=self._request_parser.get_http_version(),
            method=self._request_parser.get_method(),
            url=self._url,
            headers=self._headers,
            body=self._body,
        )

        self._request_handler_task = self._loop.create_task(
            self._request_handler(self._request, self.response_writer)
        )

    def on_header(self, header, value):
        header = header.decode(self._encoding)
        self._headers[header] = value.decode(self._encoding)
```



终于，我们完成了基本功能的开发，并且可以注册新的路由和处理程序了。接下来，我们要写一个简单的助手方法来运行我们的应用实例（就像 `aiohttp`中的 `web.run_app`）。

`application.py`

```python
def run_app(app, host="127.0.0.1", port=8080, loop=None):
    if loop is None:
        loop = asyncio.get_event_loop()

    serv = app._make_server()
    server = loop.run_until_complete(
        loop.create_server(lambda: serv, host=host, port=port)
    )

    try:
        print(f"Started server on {host}:{port}")
        loop.run_until_complete(server.serve_forever())
    except KeyboardInterrupt:
        server.close()
        loop.run_until_complete(server.wait_closed())
        loop.stop()
```

现在，是时候用我们新开发的工具包来创建简单的应用程序了。

`app.py`

```python
import asyncio

from .response import Response
from .application import Application, run_app

app = Application()

async def handler(request):
    return Response(f"Hello at {request.url}")

app.router.add_route("GET", "/", handler)

if __name__ == "__main__":
    run_app(app)

```

如果你已经运行了程序，并向 `/`发送了一个 `GET` 请求，就可以看到 `Hello at /`响应。同时，如果你访问其他路由，则会收到一个 `404`响应。

```shell
$ curl 127.0.0.1:8080/
Hello at /

$ curl 127.0.0.1:8080/invalid
Not found /invalid on this server

```

不错，我们终于完成了！但不得不说，这个项目还有很多需要改进的地方。

## 更进一步

到目前为止，我们已经开发并运行了所有的基本功能，但我们的“框架”中的某些东西还有待改进。首先，正如之前提到过的，我们的路由程序缺少参数化路由的功能，这是所有现代的框架都必须具有的特性。然后我们需要添加对中间件的支持，这也是十分常见，并且非常强大的概念。此外，在`aiohttp`的炫酷特性中，应用的生命周期钩子深得我喜爱（如`on_startup`, `on_shutdown`, `on_cleanup`），所以我们也应当尝试着去实现它。



## 路由参数

目前我们的 `UrlDispatcher`非常精简，它把被注册的url路径当作字符串来处理。我们首先要做的是在`resolve`方法中添加对` /user/{username} `等模式的支持。同时，我们还需要一个`_format_pattern` 助手方法，该方法可以从参数化字符串生成实际的正则表达式。也许你已经注意到了，我们还定义了`_method_not_allowed` 助手方法，以及另外几个用来处理 `GET`, `POST`等简单路由的方法。

`router.py`

```python
import re

from functools import partialmethod

from .response import Response

class UrlDispatcher:
    _param_regex = r"{(?P<param>\w+)}"

    def __init__(self):
        self._routes = {}

    async def _not_found(self, request):
        return Response(f"Could not find {request.url.raw_path}")

    async def _method_not_allowed(self, request):
        return Response(f"{request.method} not allowed for {request.url.raw_path}")

    def resolve(self, request):
        for (method, pattern), handler in self._routes.items():
            match = re.match(pattern, request.url.raw_path)

            if match is None:
                return None, self._not_found

            if method != request.method:
                return None, self._method_not_allowed

            return match.groupdict(), handler

    def _format_pattern(self, path):
        if not re.search(self._param_regex, path):
            return path

        regex = r""
        last_pos = 0

        for match in re.finditer(self._param_regex, path):
            regex += path[last_pos: match.start()]
            param = match.group("param")
            regex += r"(?P<%s>\w+)" % param
            last_pos = match.end()

        return regex

    def add_route(self, method, path, handler):
        pattern = self._format_pattern(path)
        self._routes[(method, pattern)] = handler

    add_get = partialmethod(add_route, "GET")

    add_post = partialmethod(add_route, "POST")

    add_put = partialmethod(add_route, "PUT")

    add_head = partialmethod(add_route, "HEAD")

    add_options = partialmethod(add_route, "OPTIONS")
```

我们还需要改造一下`Applicatio `容器，使`UrlDispatcher `的`resolve `方法能够返回`match_info `以及对应的`handler `。修改 `Application._handler` 中的以下几行。

`application.py`

```python
class Application:
    ...
    async def _handler(self, request, response_writer):
        """Process incoming request"""
        match_info, handler = self._router.resolve(request)

        request.match_info = match_info
            
        ...

```

## 中间件

可能有些读者会对中间件这个概念感到陌生。简单来说，中间件是一个协程，且该协程会在请求到达服务器之前启动，并修改传入处理程序的 `Request`对象，或修改处理程序生成的 `Response`对象。我们的需求实现起来非常简单。首先，我们要在`Application` 对象中添加一个用于注册中间件的列表，并修改 `Application._handler` 来运行这些中间件。注意，每个中间件的运行都要基于前一个中间件的工作结果，而不是基于最初的处理程序的工作结果。

`application.py`

```python
from functools import partial
...

class Application:
    def __init__(self, loop=None, middlewares=None):
        ...
        if middlewares is None:
            self._middlewares = []

    ...

    async def _handler(self, request, response_writer):
        """Process incoming request"""
        match_info, handler = self._router.resolve(request)
        
        request.match_info = match_info

        if self._middlewares:
            for md in self._middlewares:
                handler = partial(md, handler=handler)

        resp = await handler(request)

        ...
```

然后，为我们的应用程序添加一个请求日志中间件。

`app.py`

```python
import asyncio

from .response import Response
from .application import Application, run_app

async def log_middleware(request, handler):
    print(f"Received request to {request.url.raw_path}")
    return await handler(request)

app = Application(middlewares=[log_middleware])

async def handler(request):
    return Response(f"Hello at {request.url}")

app.router.add_route("GET", "/", handler)

if __name__ == "__main__":
    run_app(app)

```

现在再运行这个程序，我们就可以看到每个请求所对应的 `Received request to /` 消息了。

## App的生命周期钩子

下一步我们需要添加一些功能，使得应用程序可以在服务启动、服务停止等事件发生时执行对应的协程。这也是 `aiohttp`所拥有的一项非常灵巧的特性。可以处理的信号非常多，例如 `on_startup`、 `on_shutdown`、`on_response_prepared` 等等。但是我们想让程序尽可能保持简洁，因此只要实现`startup` 和 `shutdown`即可。

我们要先在 `Application` 内部为每个事件设置一个列表，用来添加各自的处理程序，并将其封装为属性，提供对应的getter。然后我们要编写实际的 `startup` 和 `shutdown` 协程，并在 `run_app`增加相应的调用。

`application.py`

```python
class Application:
    def __init__(self, loop=None, middlewares=None):
        ...
        self._on_startup = []
        self._on_shutdown = []

    ... 

    @property
    def on_startup(self):
        return self._on_startup

    @property
    def on_shutdown(self):
        return self._on_shutdown

    async def startup(self):
        coros = [func(self) for func in self._on_startup]
        await asyncio.gather(*coros, loop=self._loop)

    async def shutdown(self):
        coros = [func(self) for func in self._on_shutdown]
        await asyncio.gather(*coros, loop=self._loop)

    ...

def run_app(app, host="127.0.0.1", port=8080, loop=None):
    if loop is None:
        loop = asyncio.get_event_loop()

    serv = app._make_server()

    loop.run_until_complete(app.startup())

    server = loop.run_until_complete(
        loop.create_server(lambda: serv, host=host, port=port)
    )

    try:
        print(f"Started server on {host}:{port}")
        loop.run_until_complete(server.serve_forever())
    except KeyboardInterrupt:
        loop.run_until_complete(app.shutdown())
        server.close()
        loop.run_until_complete(server.wait_closed())
        loop.stop()
```

## 完善异常处理

至此，我们已经开发好了大部分核心特性，但是我们还缺少异常处理机制。 `Aiohttp`允许开发人员以处理原生Python异常的方式去处理web异常， 这也是其强大的特性之一。它实现上结合了 `Exception` 类以及 `Response` 类，非常的灵活，因此我们也来实现类似的机制。

首先，我们要创建 `HTTPException` 基类，并基于该类来实现一些我们可能会需要的助手类：`HTTPNotFound` 用于路径无法识别的情况、`HTTPBadRequest` 用于用户侧的问题、`HTTPFound` 用于重定向。

```python
from .response import Response

class HTTPException(Response, Exception):
    status_code = None

    def __init__(self, reason=None, content_type=None):
        self._reason = reason
        self._content_type = content_type

        Response.__init__(
            self,
            body=self._reason,
            status=self.status_code,
            content_type=self._content_type or "text/plain",
        )

        Exception.__init__(self, self._reason)


class HTTPNotFound(HTTPException):
    status_code = 404


class HTTPBadRequest(HTTPException):
    status_code = 400


class HTTPFound(HTTPException):
    status_code = 302

    def __init__(self, location, reason=None, content_type=None):
        super().__init__(reason=reason, content_type=content_type)
        self.add_header("Location", location)
```

然后，我们需要修改一下 `Application._handler` 来实际捕获web异常。

`application.py`

```python
class Application:
    ...
    async def _handler(self, request, response_writer):
        """Process incoming request"""
        try:
            match_info, handler = self._router.resolve(request)

            request.match_info = match_info

            if self._middlewares:
                for md in self._middlewares:
                    handler = partial(md, handler=handler)

            resp = await handler(request)
        except HTTPException as exc:
            resp = exc

        ...
```

现在我们可以删除 `UrlDispatcher` 中的`_not_found` 和`_method_not_allowed` 助手方法了。取而代之的是抛出对应的异常。

`router.py`

```python
class UrlDispatcher:
    ...
    def resolve(self, request):
        for (method, pattern), handler in self._routes.items():
            match = re.match(pattern, request.url.raw_path)

            if match is None:
                raise HTTPNotFound(reason=f"Could not find {request.url.raw_path}")

            if method != request.method:
                raise HTTPBadRequest(reason=f"{request.method} not allowed for {request.url.raw_path}")

            return match.groupdict(), handler

        ...
```

在出现某些反常的情况时，我们并不想去的破坏应用程序的运行，因此我们最好为服务器内部错误添加一个标准格式的响应。让我们编写一个简单的html模板，以及用于格式化异常的助手方法。

`helpers.py`

```python
import traceback

from .response import Response

server_exception_templ = """
<div>
    <h1>500 Internal server error</h1>
    <span>Server got itself in trouble : <b>{exc}</b><span>
    <p>{traceback}</p>
</div>
"""


def format_exception(exc):
    resp = Response(status=500, content_type="text/html")
    trace = traceback.format_exc().replace("\n", "</br>")
    msg = server_exception_templ.format(exc=str(exc), traceback=trace)
    resp.add_body(msg)
    return resp
```

这非常简单，我们现在捕获了 `Application._handler` 中生成的所有 `Exception` ，并使用我们的助手方法生成实际的html响应。

`application.py`

```python
class Application:
    ...
    async def _handler(self, request, response_writer):
        """Process incoming request"""
        try:
            match_info, handler = self._router.resolve(request)

            request.match_info = match_info

            if self._middlewares:
                for md in self._middlewares:
                    handler = partial(md, handler=handler)

            resp = await handler(request)
        except HTTPException as exc:
            resp = exc
        except Exception as exc:
            resp = format_exception(exc)
        ...
```

## 优雅地退出

最后，我们要为用于正确关闭应用程序的过程设置信号处理机制。让我们把 `run_app`修改成下面这个样子：

`application.py`

```python
...

def run_app(app, host="127.0.0.1", port=8080, loop=None):
    if loop is None:
        loop = asyncio.get_event_loop()

    serv = app._make_server()

    loop.run_until_complete(app.startup())

    server = loop.run_until_complete(
        loop.create_server(lambda: serv, host=host, port=port)
    )

    loop.add_signal_handler(
        signal.SIGTERM, lambda: asyncio.ensure_future(app.shutdown())
    )

    ...
```

## 应用程序示例

我们的工具包已经准备就绪了。现在，让我们为之前的应用示例添加生命周期钩子和异常处理。

`app.py`

```python
from .application import Application, run_app

async def on_startup(app):
    # you may query here actual db, but for an example let's just use simple set.
    app.db = {"john_doe",}

async def log_middleware(request, handler):
    print(f"Received request to {request.url.raw_path}")
    return await handler(request)

async def handler(request):
    username = request.match_info["username"]
    if username not in request.app.db:
        raise HTTPNotFound(reason=f"No such user with as {username} :(")
      
    return Response(f"Welcome, {username}!")

app = Application(middlewares=[log_middleware])

app.on_startup.append(on_startup)

app.router.add_get("/{username}", handler)

if __name__ == "__main__":
    run_app(app)
```

如果我们正确完成了所有操作，现在就可以看到每个请求的日志消息了。同时，应用程序会响应欢迎信息给已注册用户的请求，响应 `HTTPNotFound` 给那些未注册用户或无法识别路由的请求。

## 总结

受 `aiohttp` 和 `sanic`的启发，我们用了500行代码手写了一个非常简单，而又功能强大的微型框架。诚然，它还不能用于生产环境，因为它还缺少很多实用且重要的特性，如更健壮的服务器，对http规范的完整支持，以及web套接字等等。但是，我相信在这个过程中，我们更好地理解了这些工具是如何被构建的。正如著名物理学家理查德·费曼所说：“如果我不能创造某个事物，那就说明我对它的理解还不够”。希望你能够喜欢这个指南，再见:wave:。

