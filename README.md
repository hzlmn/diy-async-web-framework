<div align="center">
  <img src="media/logo.png"/>
</div>

### Translations
> Feel free to create an issue if you have translated this guide or want to request a translation.
- :ukraine:[Ukrainian](https://codeguida.com/post/2020)

  
### Introduction

   Asynchronous programming became much popular in the last few years in the Python community. Libraries like `aiohttp` show incredible growth in usage. They handle a large amount of concurrent connections while still maintain good code readability and simplicity. Not a long time ago, Django [committed](https://docs.djangoproject.com/en/dev/releases/3.0/#asgi-support) on adding async support in a next major version. So future of asynchronous python is pretty bright as you may realise. However, for a large number of developers, who came from a standard blocking model, the working mechanism of these tools may seem confusing. In this short guide, I tried to go behind the scene and clarify the process, by re-building a  little  `aiohttp`  clone from scratch. We will start just with a basic sample from official documentation and progressively add all necessary functionality that we all like. So let's start.
   
  I assume that you already have a basic understanding of [asyncio](https://docs.python.org/3/library/asyncio.html) to follow this guide, but if you need a refresher here are few articles that may help
  
   - [Intro to asyncio](https://www.blog.pythonlibrary.org/2016/07/26/python-3-an-intro-to-asyncio)
   - [Understanding asynchronous programming in Python](https://dbader.org/blog/understanding-asynchronous-programming-in-python)

For impatients, final source code available at [`hzlmn/sketch`](https://github.com/hzlmn/sketch)

## Related projects
- [500 Lines or Less](https://github.com/aosabook/500lines)
   

## Table of contents :book:

* [Asyncio low-level APIs, Transports & Protocols](#asyncio-low-level-apis-transports--protocols)

* [Making server protocol](#making-server-protocol)

* [Request/Response objects](#requestresponse-objects)

* [Application & UrlDispatcher](#application--urldispatcher)
    
 * [Going further](#going-further)
      * [Route params](#route-params)
    
      * [Middlewares](#middlewares)
      
      * [App lifecycle hooks](#app-lifecycle-hooks)
  
      * [Better exceptions](#better-exceptions)
      
      * [Graceful shutdown](#graceful-shutdown)

  * [Sample application](#sample-application)
      
  * [Conclusion](#conclusion)
      
## Asyncio low-level APIs, Transports & Protocols
Asyncio has come a long journey to become what it looks like now. Back in those days, it was created as a lower-level tool called a "tulip", and writing higher-level applications was not as enjoyable as it is today.

Right now for most usecases `asyncio` is pretty high-level API, but it also provides set of low-level helpers for library authors to manage event loops, and implement networking/ipc protocols.

Out of the box it only supports `TCP`, `UDP`, `SSL` and subprocesses. Libraries implement their own higher level (HTTP, FTP, etc.) based on base transports and available APIs.

All communications done over chaining `Transport` and `Protocols`. In simple words `Transport` describes how we can exchange data and `Protocol` is responsible for choosing which data specifically.

`Asyncio` has a pretty great official docs so you can read more about it [here](https://docs.python.org/3.8/library/asyncio-protocol.html#asyncio-transport)

To get a first grasp let's write simple `TCP` server that will echo messages.


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

As you can see from the example above, the code is pretty simple, but as you may realize it is not scalable for writing a high-level application yet. 

As `HTTP` works over `TCP` transport we already can send `HTTP` requests to our server, however, we receive them in raw format and working with it will be annoying as you may guess. So next step we need to add better `HTTP` handling mechanism.


## Making server protocol

Let's add request parsing so we can extract some useful info like headers, body, path and work with them instead of raw text. Parsing is a complex topic and it is certainly out of the scope of this guide, thats why we will use [httptools](https://github.com/MagicStack/httptools) by MagicStack for this as it rapidly fast, standard compatible and pretty flexible. 

`aiohttp`, on the other hand, has own hand-written Python based parser as well as binding to Node's [`http-parser`](https://github.com/nodejs/http-parser/tree/77310eeb839c4251c07184a5db8885a572a08352).

Lets write our parsing class, that will be used as mixin for our main `Server` class.

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
Now when we have working `HttpParserMixin`, lets modify a bit our `Server` and apply mixin.

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

<!-- Now when we have actual server lets try to run in -->
So far, we have our server that can understand incoming `HTTP` requests and obtain some important information from it. Now let's try to add simple runner to it.

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

## Request/Response objects
At this moment we have working server that can parse `HTTP` calls, but for our apps we need better abstractions to work with. 

Let's create base `Request` class that will group together all incoming `HTTP` request information. We will use `yarl` library for dealing with urls, make sure you installed it with pip.

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

As a next step, we also need a structure, that will helps us to describe outgoing `HTTP` response in programmer-friendly manner and convert it to raw `HTTP`, which can be processed by `asyncio.Transport`.

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

As you can see code is pretty straight forward we incapsulate all our data and provide proper getters. Also we have few helpers for reading body `text` and `json`, that will be used later. We also need to update our `Server` to actually construct `Request` object from message.

It should be created, when a whole request processed, so we need to add it to `on_message_complete` event handler in our parser mixin.

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

Server also need a little modification to create `Response` object and
pass encoded value to `asyncio.Transport`.

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

Now running our `server.py` we will be able to see `Received request on /path` in response to curl call `http://localhost:8080/path`.

## Application & UrlDispatcher

At this stage we already have simple working server that can process HTTP requests and Request/Response objects for dealing with request cycles. However, our hand-crafted toolkit still miss few important concepts. First of all right now we have only one main request handler, in large applications we have lots of them for different routes so we certainly need mechanism for registring multiple route handlers.

So, let's try to build simplest possible `UrlDispatcher`, just object with internal dict, that store as a key method and path tuple and actual handler as a value. We also need a handler for situation where user try to reach unrecognized route.

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

Sure thing, we miss lots of stuff like parameterized routes but we will add them later on. For now let keep it simple as it is.

<!-- Now when we have our server up and running lets lets combine together `Application` container and it's runner. -->

Next things we need an `Application` container, that will actually combine together all app related information, because dealing with underlaying `Server` will be annoying for us.

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

We need to modify our `Server` a bit and add `response_writer` method, that will be responsible for passing data to transport. Also initializer should be changed to add `handler` and `app` properties that will be used to call corresponding handlers. 

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

Finally, when we have basic functionality ready, can register new routes and handlers, let's add simple helper for actually running our app instance (similar to `web.run_app` in `aiohttp`).

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

And now, time to make simple app with our fresh toolkit.

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
If you will run it and then make `GET` request to `/`, you will be able to see `Hello at /` and `404` response for all other routes. Hooray, we done it however there are still big room for improvements.

```shell
$ curl 127.0.0.1:8080/
Hello at /

$ curl 127.0.0.1:8080/invalid
Not found /invalid on this server

```

## Going further

So far, we have all basic functionality up & running, but we still need to change certain things in our "framework". First of all, as we discussed earlier our router are missing parametrized-routes, it is "must have" feature of all modern libraries. Next we need to add support for middlewares, it is also very common and powerfull concept. Great thing about `aiohttp` that i am pretty much in love with is application lifecycle hooks (eg. `on_startup`, `on_shutdown`, `on_cleanup`) so we certainly should try to implement it as well.

## Route params 
Currently, our `UrlDispatcher` is pretty lean and it works with registered url pathes as a string. First thing, that we need is actually add support for patterns like `/user/{username}` to our `resolve` method. Also we need `_format_pattern` helper that will be responsible for generating actual regular expression from parametrized string. Also as you may noted we have another helper `_method_not_allowed` and methods for simpler definition of `GET`, `POST`, etc. routes.


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

We also need to modify our application container, right now `resolve` method of `UrlDispatcher` returns `match_info` and `handler`. So inside `Application._handler` change the following lines.

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
## Middlewares

For those, who aren't familiar with a this concept, in simple words `middleware` is just a coroutine, that can modify incoming request object or change response of a handler. It will be fired before each request to the server. Implementation is pretty trivial for our needs. First of all we need to add list of registered middlewares inside our `Application` object and change a little bit `Application._handler` to run through them. Each middleware should work with result of previous one in chain.

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

Now lets try to add request logging middleware to our simple application.

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
If we try to run it, we should see `Received request to /` message in response to incoming request.


## App lifecycle hooks

Next step let's add support for running certain coroutines in response to events like starting server and stopping it. It is pretty neat feature of `aiohttp`. There are many signals like `on_startup`, `on_shutdown`, `on_response_prepared` to name a few, but for our need let's keep it simple as possible and just implement `startup` & `shutdown` helpers.

Inside `Application` we need to add list of actual handlers for each event with proper encapsulation and provide getters. Then actual `startup` and `shutdown` coroutines and add corresponding calls to `run_app` helper.

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

## Better exceptions

At this steps we have most of the core features added, however, we still have lack of exceptions handling. Great feature about `aiohttp` is that it allows you to work with web exceptions as a native python exceptions.
It's done with implementing both `Exception` and `Response` classes and it's really flexible mechanism we would like to have as well. 

So, first thing lets create our base `HTTPException` class and few helpers based on it that we might need like `HTTPNotFound` for unrecognized pathes `HTTPBadRequest` for user side issues and `HTTPFound` for redirecting.

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

Then we need modify a bit our `Application._handler` to actually catch web exceptions.

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

Also now we can drop `_not_found` & `_method_not_allowed` helpers from our `UrlDispatcher` and instead just raise proper exceptions.

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

Another thing that might be a good addition is a standard formatted response for internal server errors, because we don't want to break actual app in some inconsistent situations. Let's add just a simple html template as well as tiny helper for formatting exceptions.

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

As simple as it is, and now just catch all `Exception` inside our `Application._handler` and generate actual html response with our helper.

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

## Graceful shutdown
As a final touch, we need to add signal processing for the proper process of shutting down our application. So, let's change `run_app` to the following lines.

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

## Sample application
Now when we have our toolkit ready, let's try to complete our previous sample application lifecycle hooks and exceptions that we just added.

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

If we done all properly you will see log messages on each request, welcome message in response to registered user and `HTTPNotFound` for unregistered users and unrecognized path.

## Conclusion

Summing it up, in ~500 lines we hand-crafted pretty simple yet powerfull micro framework inspired by `aiohttp` & `sanic`. Of course, it is not production ready software as it still miss lot's of usefull & important features like more robust server, better HTTP support to fully correlate with specification, web sockets to name a few. However, i belive that through this process we developed better understanding how such tools built. As a famous physicist Richard Feynman said “What I cannot create, I do not understand”. So I hope you enjoyed this guide, see ya! :wave:


