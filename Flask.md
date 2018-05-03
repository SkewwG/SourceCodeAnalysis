## 1. WSGI，全称 Web Server Gateway Interface

WSGI：为 Python 语言定义的 Web 服务器和 Web 应用程序或框架之间的一种简单而通用的接口

WSGI功能：相当于一个接口，前面对接服务器，后面对接app的具体功能

![](https://github.com/SkewwG/SourceCodeAnalysis/blob/master/img/wsgi.jpg?raw=true)


**Server 和 Application 之间怎么通信?**。

WSGI规定了 `app(environ, start_response)` 的接口，server 会调用 application，并传给它两个参数：`environ` 包含了请求的所有信息，`start_response` 是 application 处理完之后需要调用的函数，参数是状态码、响应头部还有错误信息。

## 2. werkzeug和jinja

### 2.1 werkzeug

Werkzeug是一个WSGI工具包，它可以作为web框架的底层库。负责核心的逻辑模块，比如路由、请求和应答的封装、WSGI 相关的函数等；

### 2.2 jinja

负责模板的渲染，主要用来渲染返回给用户的 html 文件内容。

### 2.3 如何理解wsgi, Werkzeug, flask之间的关系
Flask是一个基于Python开发并且依赖jinja2模板和Werkzeug WSGI服务的一个微型框架，Werkzeug只是工具包，其用于接收http请求并对请求进行预处理，然后触发Flask框架;jinja2模板用来实现对模板的处理，将模板和数据进行渲染，将渲染后的字符串返回给用户浏览器。

### 2.4 Flask是什么
Flask永远不会包含数据库层，也不会有表单库或是这个方面的其它东西。Flask本身只是Werkzeug和Jinja2的之间的桥梁，前者实现一个合适的WSGI应用，后者处理模板。

---
# Flask 源码分析

## 1. 启动流程
下面代码是一个符合wsgi协议的应用程序demo
```
def application(environ, start_response):               #一个符合wsgi协议的应用程序写法应该接受2个参数  
    start_response('200 OK', [('Content-Type', 'text/html')])               #environ为http的相关信息，如请求头等 start_response则是响应信息  
    return [b'<h1>Hello, web!</h1>']               #return出来是响应内容  
```
*   environ：一个包含所有HTTP请求信息的`dict`对象；

*   start_response：一个发送HTTP响应的函数。

那么官方给的例子如下：
```
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run()
```

首先程序创建一个Flask对象

那么跟到Flask类，过滤Flask的init方法里的注释，核心内容如下：
```
class Flask:
    def __init__(self, package_name):

        self.package_name = package_name
        self.root_path = _get_package_path(self.package_name)

        self.view_functions = {}
        self.error_handlers = {}
        self.before_request_funcs = []
        self.after_request_funcs = []
        self.url_map = Map()
```

* view_functions：用于保存视图函数，键是函数名,值是函数对象本身。

* error_handlers： 这个字典用来保存所有的错误处理视图函数，字典的 key 是错误类型码

* before_request_funcs： 列表：用来存放在请求被分派之前应当执行的函数

* before_first_request_funcs： 列表：用来存放在接收到第一个请求的时候应当执行的函数。

* after_request_funcs： 列表：用来存放在请求完成之后被调用的函数

* self.url_map Map()的一个实例

**Flask类的init介绍完了，程序开始执行app.run()，当http请求从server发送过来的时候，会是一个怎样的流程？**

分析流程如下：↓

app.run()-->app是Flask的实例，那么就是调用的Flask的run方法.源码如下：↓
```
def run(self, host=None, port=None, debug=None, **options):
    from werkzeug.serving import run_simple
    try:
        run_simple(host, port, self, **options)
    finally:
        self._got_first_request = False
```
主要方法是run_simple，继续跟入：↓
```
def run_simple(hostname, port, application, use_reloader=False,
               use_debugger=False, use_evalex=True,
               extra_files=None, reloader_interval=1,
               reloader_type='auto', threaded=False,
               processes=1, request_handler=None, static_files=None,
               passthrough_errors=False, ssl_context=None):
```
注释里有这么一行：↓

:param application: the WSGI application to execute

接着在 `werkzeug.serving:WSGIRequestHandler` 的 `run_wsgi` 中找到execute：
```
def execute(app):
    application_iter = app(environ, start_response)
    try:
        for data in application_iter:
            write(data)
        if not headers_sent:
            write(b'')
    finally:
        if hasattr(application_iter, 'close'):
            application_iter.close()
            application_iter = None
```
可以看到 `application_iter = app(environ, start_response)` 就是调用 `app` 实例，那么它就需要定义了 `__call__` 方法，继续找Flask类里的 `__call__` 方法，源码如下：↓
```
class Flask(_PackageBoundObject):
    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)
```
可以看到return的是wsgi_app方法。

所以当http请求过来后，最终调用的是wsgi_app方法。

**流程如下：↓**

```
app.run()
    run_simple(host, port, self, **options)
        __call__(self, environ, start_response)
            wsgi_app(self, environ, start_response)
```
下面继续分析wsgi_app：↓

**wsgi_app是flask核心**
```
def wsgi_app(self, environ, start_response):
    # 创建请求上下文，并把它压栈
    ctx = self.request_context(environ)
    ctx.push()
    error = None

    try:
        try:
            # 正确的请求处理路径，会通过路由找到对应的处理函数
            response = self.full_dispatch_request()
        except Exception as e:
            # 错误处理，默认是 InternalServerError 错误处理函数，客户端会看到服务器 500 异常
            error = e
            response = self.handle_exception(e)
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        # 不管处理是否发生异常，都需要把栈中的请求 pop 出来
        ctx.auto_pop(error)
```

wsgi_app的核心方法`full_dsipatch_request` 的代码如下：
```
def full_dispatch_request(self):
    self.try_trigger_before_first_request_functions()
    try:
        request_started.send(self)
        rv = self.preprocess_request()
        if rv is None:
            rv = self.dispatch_request()
    except Exception as e:
        rv = self.handle_user_exception(e)
    return self.finalize_request(rv)
```
full_dispatch_request方法里的最核心的内容是 dispatch_request.

---

dispatch_request的**官方注释**：Does the request dispatching.  Matches the URL and returns the return value of the view or error handler。

理解：要做的就是找到我们的处理函数，并返回调用的结果，这个也就是**路由的过程**

---

然后finalize_request方法将rv序列化转换成 Response对象。

preprocess_request和finalize_request包含了很多hooks，

* before_first_request  第一次请求处理之前执行的函数

* before_request  每个请求处理之前执行的函数

* after_request  每个请求正常处理之后执行的函数

* teardown_request  不管请求是否异常都要执行的函数

**结论：**

**当http请求从server发送过来的时候，会调用__call__功能，最终实际是调用了wsgi_app功能并传入environ和start_response**

---

**第一节介绍了启动的流程，知道了程序是怎么运行的。那么程序又是如何将路由和视图函数建立联系的呢？**

**那么就开始第二节分析路由。**

---


## 2. 路由：
### 学习该节的目的是理解rule， endpoint， view_func是怎么互相建立联系的？

下面代码是注册与url对应的处理函数：↓

官方注释解释以下例子是等价的
```
@app.route('/')
def hello():
    return "hello, world!"
等价于
def hello():
    return "hello, world!"

app.add_url_rule('/', 'hello', hello)
```


那么分析add_url_rule方法，有这么一行代码：↓

```
self.view_functions[endpoint] = view_func
```

**可以看到view_functions是一个字典，键是endpoint，值是函数对象本身，这样就相当于endpoint和view_func建立了连接**

**问：现在endpoint， view_func可以联系起来了，那么rule和endpoint该怎么连接呢？**

**答：通过MapAdpter的match方法，将path_info和endpoint相连接**

**这样就完成了根据一个网站的路径调用view_func的过程**

** 现在详细分析：↓**

### 2.1 add_url_rule方法
```
def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
    """Connects a URL rule.  Works exactly like the :meth:`route`
    decorator.  If a view_func is provided it will be registered with the
    endpoint.
    """

    methods = options.pop('methods', None)

    rule = self.url_rule_class(rule, methods=methods, **options)
    self.url_map.add(rule)
    if view_func is not None:
        old_func = self.view_functions.get(endpoint)
        if old_func is not None and old_func != view_func:
            raise AssertionError('View function mapping is overwriting an '
                                 'existing endpoint function: %s' % endpoint)
        self.view_functions[endpoint] = view_func

```

add_url_rule官方注释：连接 URL 规则。 工作原理与: "route" 装饰器完全一样。 如果提供了 view_func, 它将在endpoint上注册

参数：rule：传递url规则

参数：endpoint：Flask 本身假定view function 的名称当作endpoint

参数：view_func： 要调用的函数本身

参数：options： 要转发到基础类的选项: '~werkzeug.routing.Rule' 对象。 对Werkzeug的更改是处理方法选项。 方法是将此规则限制为 ("GET"、"POST" ) 等的方法列表。

如果没有传递 view_func, 则需要将endpoint 连接到view_func, 如下：
```
app.view_functions['index'] = index
```

可以看到view_functions 是一个字典，注册了所有视图函数的字典。 键是函数名, 值是函数对象本身。

注册代码如下：↓

```
self.view_functions[endpoint] = view_func
```

**强调：add_url_rule传递的第二个参数endpoint 是函数名，传递的第三个参数view_func是函数本身**

add_url_rule方法核心功能就是更新 `self.url_map` 和 `self.view_functions` 这两个变量。

`url_map` 是 `werkzeug.routeing:Map` 类的对象            self.url_map = Map()

`rule` 是 `werkzeug.routing:Rule` 类的对象               url_rule_class = Rule

---

**那么接下来分析 Class Map**

---

### 2.2 Class Map
**这是对Map对象的bind和match使用的一个demo：↓**
```
m = Map([
    Rule('/', endpoint='index'),
    Rule('/downloads/', endpoint='downloads/index'),
    Rule('/downloads/<int:id>', endpoint='downloads/show')
])

urls = m.bind("example.com", "/")
ret1 = urls.match("/", "GET")
print(ret1)                             # ('index', {})
ret2 = urls.match("/downloads/42")      # 默认get
print(ret2)                             # ('downloads/show', {'id': 42})
ret3 = urls.match("/downloads/")        
print(ret3)                             # ('downloads/index', {})
ret4 = urls.match("/downloads")         # 报错
print(ret4)                             # RequestRedirect: 301 Moved Permanently: None
ret5 = urls.match("/missing")           # 报错
print(ret5)                             # 404 Not Found: The requested URL was not found on the server.
```

demo的例子如果无法理解，那么我把官方的解释解释一遍：↓

官方注释：
```
the map class stores all the URL rules and some configuration parameters.  Some of the configuration values are only stored on the `Map` instance since those affect all rules, others are just defaults and can be overridden for each rule.  Note that you have to specify all arguments besides the `rules` as keyword arguments!
```

理解：map 类存储所有 URL 规则和一些配置参数。 某些配置值仅存储在 "映射" 实例上, 因为这些参数影响所有规则, 而另一部分则只是缺省值, 并且可以为每个规则重写。 请注意, 除了将 "规则" 作为关键字参数外, 还必须指定所有参数!

参数 rules： Rule实例组成的序列

**方法 def bind**

```
        server_name = server_name.lower()
        if self.host_matching:
            if subdomain is not None:
                raise RuntimeError('host matching enabled and a '
                                   'subdomain was provided')
        elif subdomain is None:
            subdomain = self.default_subdomain
        if script_name is None:
            script_name = '/'
        try:
            server_name = _encode_idna(server_name)
        except UnicodeError:
            raise BadHost()
        return MapAdapter(self, server_name, script_name, subdomain,
                          url_scheme, path_info, default_method, query_args)
```
官方注释：
```
Return a new :class:`MapAdapter` with the details specified to the call.  Note that `script_name` will default to ``'/'`` if not further specified or `None`.  The `server_name` at least is a requirement because the HTTP RFC requires absolute URLs for redirects and so all redirect exceptions raised by Werkzeug will contain the full canonical URL.
```

理解：返回一个新的: 类: "MapAdapter"；server_name必须被传递,传递网站的域名；script_name将默认为 "'/", 如果没有进一步指定或 "无"；如果 path_info 没有传递给: "match", 它将传递默认路径信息给bind。

### 2.3. Class MapAdapter

**Map实例调用了bind之后，返回了MapAdapter类，并且调用了MapAdapter的match方法**

```
class MapAdapter(object):
    def match(self, path_info=None, method=None, return_rule=False, query_args=None):
```


* 传递的值：

官方注释：The usage is simple: you just pass the match method the current path info as well as the method (which defaults to GET).  The following things can then happen:

理解：传递匹配方法的path_info 和 method（默认GET）
```
you receive a `NotFound` exception that indicates that no URL is matching.  A `NotFound` exception is also a WSGI application you can call to get a default page not found page (happens to be the same object as werkzeug.exceptions.NotFound
你会收到一个 "NotFound" 异常, 指示没有匹配到 URL。 "NotFound" 异常也是一个 WSGI 应用程序, 您可以调用它来获取默认页 "NotFound" 页

you receive a `MethodNotAllowed` exception that indicates that there is a match for this URL but not for the current request method. This is useful for RESTful applications.
你会收到一个 "MethodNotAllowed" 异常, 指示此 URL 存在匹配项, 但不支持当前请求方法。这对于 RESTful 应用程序非常有用。

you receive a `RequestRedirect` exception with a `new_url` attribute.  This exception is used to notify you about a request Werkzeug requests from your WSGI application.  This is for example the case if you request ``/foo`` although the correct URL is ``/foo/`` You can use the `RequestRedirect` instance as response-like object similar to all other subclasses of `HTTPException`.
您会收到一个 "new_url" 属性的 "RequestRedirect" 异常。 如果请求 "/foo", 但正确的 URL 是 "/foo/" 可以使用 "RequestRedirect" 实例作为响应类似的对象类似于所有其他子类的 "HTTPException"。
```

* 返回值:

**如果存在匹配项，可以获得一个元组(endpoint, arguments)。 如果 `return_rule`设置True, 在这个情况下，获得的元组是(rule, arguments)**

**如果没有传递path info, 则使用默认路径信息 (如果未显式定义, 则默认为根 URL)。**

path_info:用于匹配路径信息。 重写绑定的路径信息。

method:用于匹配HTTP 方法。 重写指定的方法。

return_rule:返回匹配的规则, 而不是只是endpoint (默认为 "False")。

query_args:用于自动重定向为字符串或字典的可选查询参数。 当前无法使用查询参数进行 URL 匹配。

**重点是方法 def match**
```
def match(self, path):
    self.map.update()
    if path_info is None:
        path_info = self.path_info
    else:
        path_info = to_unicode(path_info, self.map.charset)
    if query_args is None:
        query_args = self.query_args
    method = (method or self.default_method).upper()
    ...
```
**match的主要功能是Map 保存了 Rule 列表，match 的时候会依次调用其中的 rule.match 方法，如果匹配就找到了**

---

**通过Map类的分析，我们了解了endpoint如何与view func建立联系，接下去分析Rule类，了解url和endpoint 的联系。**

---

### 2.4. Class Rule
**主要功能就是 url 到 endpoint 的转换：通过 url 找到处理该 url 的 endpoint**

```
def __init__(self, string, defaults=None, subdomain=None, methods=None,
                 build_only=False, endpoint=None, strict_slashes=None,
                 redirect_to=None, alias=False, host=None):

Rule('/', endpoint='index'),
Rule('/downloads/', endpoint='downloads/index'),
Rule('/downloads/<int:id>', endpoint='downloads/show')
```

* Class Rule官方注释：

A Rule represents one URL pattern.  There are some options for `Rule` that change the way it behaves and are passed to the `Rule` constructor. Note that besides the rule-string all arguments *must* be keyword arguments in order to not break the application on Werkzeug upgrades.

理解：Rule表示一个 URL 模式。 有一些 "规则" 选项可以改变它的行为方式并传递给 "规则" 构造函数。请注意, 除了规则字符串之外, 所有参数 * 都必须是关键字参数, 以便在Werkzeug升级时不中断应用程序。

---

* 参数String官方注释：

Rule strings basically are just normal URL paths with placeholders in the format ``<converter(arguments):name>`` where the converter and the arguments are optional.  If no converter is defined the `default` converter is used which means `string` in the normal configuration.URL rules that end with a slash are branch URLs, others are leaves. If you have `strict_slashes` enabled (which is the default), all branch URLs that are matched without a trailing slash will trigger a redirect to the same URL with the missing slash appended.

理解：规则字符串基本上只是通常的** URL 路径**, 其占位符的格式为 "<converter(arguments):name>", 其中转换器和参数是可选的。 如果未定义转换器, 则使用 "默认" 转换器, 在正常配置中表示 "string".</converter(arguments):name>以斜线结尾的 URL 规则是分支 url, 另一些则是叶。如果启用了** "strict_slashes" (默认值)**, 则在没有尾随斜线的情况下匹配的所有分支 url 都将触发重定向到与所追加的缺失斜线相同的 URL。

---

* 参数endpoint官方注释：

The endpoint for this rule. This can be anything. A reference to a function, a string, a number etc.  The preferred way is using a string because the endpoint is used for URL generation.

理解：规则的endpoint 。可以是任何。对函数、字符串、数字等的引用。 首选的方法是**使用字符串**, 因为endpoint 用于生成 URL。

---

* 参数defaults的官方注释：

An optional dict with defaults for other rules with the same endpoint. This is a bit tricky but useful if you want to have unique URLs

理解：具有相同endpoint的其他规则的默认可选字典是一个有点棘手, 但有用的, 如果你想有唯一的网址

---

```
url_map = Map([
                Rule('/all/', defaults={'page': 1}, endpoint='all_entries'),
                Rule('/all/page/<int:page>', endpoint='all_entries')
            ])
```

如果用户现在访问 "http://example.com/all/page/1", 他将被重定向到 "http://example.com/all/"。 如果 "Map" 实例上的 "redirect_defaults" 被禁用, 这只会影响 URL 生成。

* 参数subdomain

此规则的subdomain规则字符串。如果未指定规则, 则仅匹配映射的 "default_subdomain"。 如果map没有绑定subdomain， 此功能将被禁用。

如果您希望在不同的子域上拥有用户配置文件, 并且将所有子域转发到您的应用程序, 则会非常有用。

```
url_map = Map([
                Rule('/', subdomain='<username>', endpoint='user/homepage'),
                Rule('/stats', subdomain='<username>', endpoint='user/stats')
            ])
```

* 参数methods的官方注释：

A sequence of http methods this rule applies to.  If not specified, all methods are allowed. For example this can be useful if you want different endpoints for `POST` and `GET`.  If methods are defined and the path matches but the method matched against is not in this list or in the list of another rule for that path the error raised is of the type `MethodNotAllowed` rather than `NotFound`.  If `GET` is present in the list of methods and `HEAD` is not, `HEAD` is added automatically

理解：此规则适用的 http 方法序列。 如果未指定, 则允许使用所有方法。例如, **如果希望 "POST" 和 "GET" 的endpoints不同**, 这可能很有用。 **如果定义了methods 并且路径匹配, 但匹配的methods 不在此列表中或该路径的另一个规则列表中, 则引发的错误是类型 "MethodNotAllowed", 而不是 "NotFound"。** 如果 "GET" 是存在的方法列表和 ' HEAD ' 不是, ' HEAD ' 是自动添加

---

**第二节我们已经知道了路由和视图是怎么建立联系，但是视图函数执行需要一些参数，例如：请求的 url，参数，方法等。这些外部变量就叫做上下文。那么第三节就开始介绍上下文。**

---

## 3. 上下文（application context 和 request context）

**问：什么是上下文？**

**答**：每一段程序都有很多外部变量。只有像Add这种简单的函数才是没有外部变量的。一旦你的一段程序有了外部变量，这段程序就不完整，不能独立运行。你为了使他们运行，就要给所有的外部变量一个一个写一些值进去。这些值的集合就叫上下文。

来源：知乎 https://www.zhihu.com/question/26387327/answer/32611575

**问：请求上下文是如何、在何时被创建的呢？**

**答：**在服务器调用应用的时候，Flask 的 `wsgi_app` 有以下代码，这行代码就是创建了请求上下文并压栈。
```
def wsgi_app(self, environ, start_response): 
    ctx = self.request_context(environ) 
    ctx.push()
```

**flask 中有两种上下文：程序上下文`application context` 和 请求上下文`request context`。**

`application context`有两个变量 `current_app` 和 `g`;

`request context` 有两个变量 `request` 和 `session`。

上下文的相关内容在 `globals.py` 文件中
```
# context locals
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
g = LocalProxy(partial(_lookup_app_object, 'g'))
```


* current_app ：当前被激活了的程序实例，比如helloworld例子中的app

* g ：用作处理请求时的临时存储，每次请求都重设

* request ：请求对象，包含了客户端HTTP请求的内容

* session ：用户会话，是字典格式，用来存储证明身份的信息

上面我们已经了解了上下文的定义，视图函数需要的时候，可以使用 **from flask import request, current_app** 获取。

**下面是使用的demo：↓**

**请求上下文**

```
from flask import request
@app.route('/useragent/')
def userAgent():
    user_agent = request.headers.get('User-Agent')
    return '<p>Your browser is %s</p>' % user_agent
```

**程序上下文**
```
from flask import current_app
# print('current_app.name:',current_app.name)
app_ctx = app.app_context()
app_ctx.push()
current_app.name
app_ctx.pop()
```

---

**通过第三节我们知道了视图如何调用外部变量来实现功能。现在我们已经知道了程序如何运行，路由和视图的匹配，视图如何调用上下文。那么接下来分析请求和响应又是怎么实现的。**

---

## 4. 请求
在flask里请求是一个对象，将http请求的所有内容都存放在一个字典里

访问 flask 的请求对象非常简单，只需要 `from flask import request`：
```
from flask import request

with app.request_context(environ):
    assert request.method == 'POST'
```

class Request的源码如下：↓
```
from werkzeug.wrappers import Request as RequestBase
class Request(RequestBase):
    @property
    def max_content_length(self):
        """Read-only view of the ``MAX_CONTENT_LENGTH`` config key."""
        ctx = _request_ctx_stack.top
        if ctx is not None:
            return ctx.app.config['MAX_CONTENT_LENGTH']

    @property
    def endpoint(self):
        """The endpoint that matched the request.  This in combination with
        :attr:`view_args` can be used to reconstruct the same or a
        modified URL.  If an exception happened when matching, this will
        be ``None``.
        """
        if self.url_rule is not None:
            return self.url_rule.endpoint
```

**`@property` 装饰符能够把类的方法变成属性，这些属性和Flask的逻辑相关**

由源码可知，requests继承自 werkzeug.wrappers:Request

跟入werkzeug.wrappers:Request 分析源码
```
class Request(BaseRequest, AcceptMixin, ETagRequestMixin,
              UserAgentMixin, AuthorizationMixin,
              CommonRequestDescriptorsMixin):

    """Full featured request object implementing the following mixins:

    - :class:`AcceptMixin` for accept header parsing
    - :class:`ETagRequestMixin` for etag and cache control handling
    - :class:`UserAgentMixin` for user agent introspection
    - :class:`AuthorizationMixin` for http auth handling
    - :class:`CommonRequestDescriptorsMixin` for common headers
    """
```
这里使用了混入编程。

继续跟入BaseRequest类

```
class BaseRequest(object):
    def __init__(self, environ, populate_request=True, shallow=False):
        self.environ = environ
        if populate_request and not shallow:
            self.environ['werkzeug.request'] = self
        self.shallow = shallow
```
就是简单的将environ变量保存下来。

接着看AcceptMixin这些类：
```
class AcceptMixin(object):

    @cached_property
    def accept_mimetypes(self):
        """List of mimetypes this client supports as
        :class:`~werkzeug.datastructures.MIMEAccept` object.
        """
        return parse_accept_header(self.environ.get('HTTP_ACCEPT'), MIMEAccept)

    @cached_property
    def accept_charsets(self):
        """List of charsets this client supports as
        :class:`~werkzeug.datastructures.CharsetAccept` object.
        """
        return parse_accept_header(self.environ.get('HTTP_ACCEPT_CHARSET'),
                                   CharsetAccept)
```
方法被 `@cached_property` 装饰

那么分析下 `@cached_property`源码
```
class cached_property(property):

    def __init__(self, func, name=None, doc=None):
        self.__name__ = name or func.__name__
        self.__module__ = func.__module__
        self.__doc__ = doc or func.__doc__
        self.func = func

    def __set__(self, obj, value):
        obj.__dict__[self.__name__] = value

    def __get__(self, obj, type=None):
        if obj is None:
            return self
        value = obj.__dict__.get(self.__name__, _missing)
        if value is _missing:
            value = self.func(obj)
            obj.__dict__[self.__name__] = value
        return value
```
这个装饰器实现了 `__set__` 和 `__get__` 方法，访问它装饰的属性，就会调用 `__get__` 方法，这个方法先在 `obj.__dict__` 中寻找是否已经存在对应的值。如果存在，就直接返回；如果不存在，调用底层的函数 `self.func`，并把得到的值保存起来，再返回。这也是它能实现缓存的原因：因为它会把函数的值作为属性保存到对象中。

---

**第四节我们了解了请求是如何实现的，接下去了解响应**

---

## 5. response 响应
HTTP 响应分为三个部分： 状态、响应头、body

视图函数的返回值即为响应

**响应有两种方法：**

* 视图函数直接返回一个元组 (response, status, headers)

第一个是返回的数据，第二个是状态码，第三个是响应头

**demo：↓**
```
@app.route('/')
def hello_world():
    return 'Hello, World!', 201, {'X-Foo': 'bar'}
```

* 视图函数返回一个make_resonse()函数产生的响应对象

make_response()函数接受一个或多个参数（和视图函数的返回值一样），然后返回一个 Response 对
象。

**demo：↓**
```
from flask import make_response
@app.route('/response/')
def response():
    resp = make_response('<h1>Bad Request</h1>',400)
    return resp
```

---

**前五节已经把flask的运作流程全部学习了一遍，下面再分析下session是怎么工作的。**

---

## 6. Session
**问：什么是session**

**答**：由于HTTP协议是无状态的协议，所以服务端需要记录用户的状态时，就需要用某种机制来识具体的用户，这个机制就是Session.

**session和cookies的区别：**
1，session 在服务器端，cookie 在客户端（浏览器）
2，session 默认被存在在服务器的一个文件里（不是内存）
3，session 的运行依赖 session id，而 session id 是存在 cookie 中的，也就是说，如果浏览器禁用了 cookie ，同时 session 也会失效（但是可以通过其它方式实现，比如在 url 中传递 session_id）

**问：那么如何在flask中使用session呢？**

**答**：只要`from flask import session` 就可以使用 session 

当请求过来的时候，flask会把请求里的cookies生成session，然后在视图里就可以使用生成的session，对session进行更改，最后返回给response，并把session写入cookies里。

核心代码在save_session方法里：
```
def save_session(self, app, session, response):
        domain = self.get_cookie_domain(app)
        path = self.get_cookie_path(app)

        # 如果 session 变成了空字典，flask 会直接删除对应的 cookie
        if not session:
            if session.modified:
                response.delete_cookie(app.session_cookie_name,
                                       domain=domain, path=path)
            return

        # 是否需要设置 cookie。如果 session 发生了变化，就一定要更新 cookie，否则用户可以 `SESSION_REFRESH_EACH_REQUEST` 变量控制是否要设置 cookie
        if not self.should_set_cookie(app, session):
            return

        httponly = self.get_cookie_httponly(app)
        secure = self.get_cookie_secure(app)
        expires = self.get_expiration_time(app, session)
        val = self.get_signing_serializer(app).dumps(dict(session))
        response.set_cookie(app.session_cookie_name, val,
                            expires=expires,
                            httponly=httponly,
                            domain=domain, path=path, secure=secure)
```
从 `app` 和 `session` 变量中获取所有需要的信息，然后调用 `response.set_cookie` 生成 `cookie`。这样就在客户端的cookies里保存了session，以后访问的时候携带的cookies里的session就能够证明自己的身份。

本篇Flask源码解读所涉及到的重要内容和核心源码在下面的流程图里。
![](https://raw.githubusercontent.com/SkewwG/SourceCodeAnalysis/master/img/Flask%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E6%B5%81%E7%A8%8B%E5%9B%BE.png)


## 参考资料

https://segmentfault.com/a/1190000010954763

http://mingxinglai.com/cn/2016/08/flask-source-code/

https://juejin.im/post/5a32513ff265da430f321f3d

http://cizixs.com/2017/01/10/flask-insight-introduction