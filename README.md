## TinyWeb

Simple, lightweight, async HTTP webserver for tiny devices like **ESP8266** / **ESP32** running [micropython](https://github.com/micropython/micropython).

This repository merges the work of [belyalov](https://github.com/belyalov/tinyweb) (original author) and [metachris](https://github.com/metachris/tinyweb).

### Features

* Fully asynchronous when using with [uasyncio](https://github.com/micropython/micropython-lib/tree/v1.0/uasyncio) library for MicroPython.
* [Flask](http://flask.pocoo.org/) / [Flask-RESTful](https://flask-restful.readthedocs.io/en/latest/) like API.
* *Tiny* memory usage. So you can run it on devices like **ESP8266 / ESP32** with 64K/96K of onboard RAM. BTW, there is a huge room for optimizations - so your contributions are warmly welcomed.
* Support for static content serving from filesystem.
* Great unittest coverage
* No requirements on MicroPython 1.13+ (else `uasyncio`)

**Easy installation**

```wget https://raw.githubusercontent.com/metachris/tinyweb/master/tinyweb/server.py -O tinyweb.py```

Or the minified version (9.3kb instead of 27kb):

```wget https://raw.githubusercontent.com/metachris/tinyweb/master/tinyweb/server_min.py -O tinyweb.py```

* [logging](https://github.com/micropython/micropython-lib/tree/master/python-stdlib/logging)

**Changes from [upstream tinyweb](https://github.com/belyalov/tinyweb):**

* [uasyncio](https://github.com/micropython/micropython-lib/tree/v1.0/uasyncio) - micropython version of *async* python library.
* [uasyncio-core](https://github.com/micropython/micropython-lib/tree/v1.0/uasyncio.core)
* Removed logging depedency -- this can be downloaded without dependencies like this:
* `send_file` with auto mime-type detection (based on filename)
* `response.html(your_content_str)` and `response.json(your_response_obj)` helper
* Minified distribution files (9.3kb instead of 27kb. using [python-minifier](https://github.com/dflook/python-minifier)):

```
pyminify tinyweb/server.py --remove-literal-statements > tinyweb/server_min.py
```

# Getting started

Let's develop a hello world web app:

```python
import tinyweb
app = tinyweb.webserver()

# Index page
@app.route('/')
async def index(request, response):
    # Send HTML page with content-type text/html
    await response.html('<html><body><h1>Hello, world! (<a href="/table">table</a>)</h1></html>\n')

# Catch all requests with an invalid URL
@app.catchall()
async def catchall_handler(request, response):
    response.code = 404
    await response.html('<html><body><h1>My custom 404</h1></html>\n')

# HTTP redirection
@app.route('/redirect')
async def redirect(request, response):
    await response.redirect('/')

# Another one, more complicated page
@app.route('/table')
async def table(request, response):
    # Start HTTP response with content-type text/html
    await response.html('<html><body><h1>Simple table</h1><table border=1 width=400><tr><td>Name</td><td>Some Value</td></tr>')
    for i in range(10):
        await response.send('<tr><td>Name{}</td><td>Value{}</td></tr>'.format(i, i))
    await response.send('</table></html>')

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8081)
```

Save this as webserver.py

Let's try it! Flash your device with firmware, open REPL and type:

```python
# Connect to WiFi
>>> import network
>>> sta_if = network.WLAN(network.STA_IF)
>>> sta_if.active(True)
>>> sta_if.connect('<ssid>', '<password>')

# Run the code
>>> import webserver
>>> webserver.app.run(host='0.0.0.0', port=80)
```

That's it! :) Try it by open page `http://<your_ip>`

Like it? Check more [examples](https://github.com/metachris/tinyweb/tree/master/examples) then :)

### Limitations
* HTTP protocol support - due to memory constrains only **HTTP/1.0** is supported (with exception for REST API - it uses HTTP/1.1 with `Connection: close`). Support of HTTP/1.1 may be added when `esp8266` platform will be completely deprecated.

### Reference
#### class `webserver`
Main tinyweb app class.

* `__init__(self, request_timeout=3, max_concurrency=None)` - Create instance of webserver class.
    * `request_timeout` - Specifies timeout for client to send complete HTTP request (without HTTP body, if any), after that connection will be closed. Since `uasyncio` has very short queue (about 42 items) *Avoid* using values > 5 to prevent events queue overflow.
    * `max_concurrency` - How many connections can be processed concurrently. It is very important to limit it mostly because of memory constrain. Default value depends on platform, **3** for `esp8266`, **6** for `esp32` and **10** for others.
    * `backlog` - Parameter to socket.listen() function. Defines size of pending to be accepted connections queue. Must be greater than `max_concurrency`.
    * `debug` - Whether send exception info (text + backtrace) to client together with HTTP 500 or not.

* `add_route(self, url, f, **kwargs)` - Map `url` into function `f`. Additional keyword arguments are supported:
    * `methods` - List of allowed methods. Defaults to `['GET', 'POST']`
    * `save_headers` - Due to memory constrains you most likely want to minimze memory usage by saving only headers
    which you really need in. E.g. for POST requests it is make sense to save at least 'Content-Length' header.
    Defaults to empty list - `[]`.
    * `max_body_size` - Max HTTP body size (e.g. POST form data). Be careful with large forms due to memory constrains (especially with esp8266 which has 64K RAM). Defaults to `1024`.
    * `allowed_access_control_headers` - Whenever you're using xmlHttpRequest (send JSON from browser) these headers are required to do access control. Defaults to `*`
    * `allowed_access_control_origins` - The same idea as for header above. Defaults to `*`.

* `@route` - simple and useful decorator (inspired by *Flask*). Instead of using `add_route()` directly - just decorate your function with `@route`, like this:
    ```python
    @app.route('/index.html')
    async def index(req, resp):
        await resp.send_file('static/index.simple.html')
    ```
* `add_resource(self, cls, url, **kwargs)` - RestAPI: Map resource class `cls` to `url`. Class `cls` is arbitrary class with with implementation of HTTP methods:
    ```python
    class CustomersList():
        def get(self, data):
            """Return list of all customers"""
            return {'1': {'name': 'Jack'}, '2': {'name': 'Bob'}}

        def post(self, data):
            """Add customer"""
            db[str(next_id)] = data
        return {'message': 'created'}, 201
    ```
  `**kwargs` are optional and will be passed to handler directly.
    **Note**: only `GET`, `POST`, `PUT` and `DELETE` methods are supported. Check [restapi full example](https://github.com/belyalov/tinyweb/blob/master/examples/rest_api.py) as well.

* `@resource` - the same idea as for `route` but for resource:
    ```python
    # Regular version
    @app.resource('/user/<id>')
    def user(data, id):
        return {'id': id, 'name': 'foo'}

    # Generator based / different HTTP method
    @app.resource('/user/<id>', method='POST')
    async def user(data, id):
        yield '{'
        yield '"id": "{}",'.format(id)
        yield '"name": "test",'
        yield '}'
    ```

* `run(self, host="127.0.0.1", port=8081, loop_forever=True, backlog=10)` - run web server. Since *tinyweb* is fully async server by default it is blocking call assuming that you've added other tasks before.
    * `host` - host to listen on
    * `port` - port to listen on
    * `loop_forever` - run `async.loop_forever()`. Set to `False` if you don't want `run` to be blocking call. Be sure to call `async.loop_forever()` by yourself.
    * `backlog` - size of pending connections queue (basically argument to `listen()` function)

* `shutdown(self)` - gracefully shutdown web server. Meaning close all active connections / server socket and cancel all started coroutines. **NOTE** be sure to it in event loop or run event loop at least once, like:
    ```python
    async def all_shutdown():
        await asyncio.sleep_ms(100)

    try:
        web = tinyweb.webserver()
        web.run()
    except KeyboardInterrupt as e:
        print(' CTRL+C pressed - terminating...')
        web.shutdown()
        uasyncio.get_event_loop().run_until_complete(all_shutdown())
    ```


#### class `request`
This class contains everything about *HTTP request*. Use it to get HTTP headers / query string / etc.
***Warning*** - to improve memory / CPU usage strings in `request` class are *binary strings*. This means that you **must** use `b` prefix when accessing items, e.g.

    >>> print(req.method)
    b'GET'

So be sure to check twice your code which interacts with `request` class.

* `method` - HTTP request method.
* `path` - URL path.
* `query_string` - URL path.
* `headers` - `dict` of saved HTTP headers from request. **Only if enabled by `save_headers`.
    ```python
    if b'Content-Length' in self.headers:
        print(self.headers[b'Content-Length'])
    ```

* `read_parse_form_data()` - By default (again, to save CPU/memory) *tinyweb* doesn't read form data. You have to call it manually unless you're using RESTApi. Returns `dict` of key / value pairs.

#### class `response`
Use this class to generate HTTP response. Please be noticed that `response` class is using *regular strings*, not binary strings as `request` class does.

* `code` - HTTP response code. By default set to `200` which means OK, no error.
* `version` - HTTP version. Defaults to `1.0`. Please be note - that only HTTP1.0 is internally supported by `tinyweb`. So if you changing it to `1.1` - be sure to support protocol by yourself.
* `headers` - HTTP response headers dictionary (key / value pairs).
* `add_header(self, key, value)` - Convenient way to add HTTP response header
    * `key` - Header name
    * `value` - Header value

* `add_access_control_headers(self)` - Add HTTP headers required for RESTAPI (JSON query)

* `redirect(self, location)` - Generate HTTP redirection (HTTP 302 Found) to `location`. This *function is coroutine*.

* `start_html(self)`- Start response with HTML content type. This *function is coroutine*. This function is basically sends response line and headers. Refer to [hello world example](https://github.com/belyalov/tinyweb/blob/master/examples/hello_world.py).

* `send(self, payload)` - Sends your string/bytes `payload` to client. Be sure to start your response with `start_html()` or manually. This *function is coroutine*.

* `send_file(self, filename)`: Send local file as HTTP response. File type will be detected automatically unless you explicitly change it. If file doesn't exists - HTTP Error `404` will be generated.
Additional keyword arguments
    * `content_type` - MIME filetype. By default - `None` which means autodetect.
    * `content_encoding` - Specifies used compression type, e.g. `gzip`. By default - `None` which means don't add this header.
    * `max_age` - Cache control. How long browser can keep this file on disk. Value is in `seconds`. By default - 30 days. To disable caching, set it to `0`.

* `error(self, code)` - Generate HTTP error response with error `code`. This *function is coroutine*.
