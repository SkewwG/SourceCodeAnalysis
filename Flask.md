## 1. WSGI��ȫ�� Web Server Gateway Interface

WSGI��Ϊ Python ���Զ���� Web �������� Web Ӧ�ó������֮���һ�ּ򵥶�ͨ�õĽӿ�

WSGI���ܣ��൱��һ���ӿڣ�ǰ��Խӷ�����������Խ�app�ľ��幦��

![](https://github.com/SkewwG/SourceCodeAnalysis/blob/master/img/wsgi.jpg?raw=true)


**Server �� Application ֮����ôͨ��?**��

WSGI�涨�� `app(environ, start_response)` �Ľӿڣ�server ����� application��������������������`environ` �����������������Ϣ��`start_response` �� application ������֮����Ҫ���õĺ�����������״̬�롢��Ӧͷ�����д�����Ϣ��

## 2. werkzeug��jinja

### 2.1 werkzeug

Werkzeug��һ��WSGI���߰�����������Ϊweb��ܵĵײ�⡣������ĵ��߼�ģ�飬����·�ɡ������Ӧ��ķ�װ��WSGI ��صĺ����ȣ�

### 2.2 jinja

����ģ�����Ⱦ����Ҫ������Ⱦ���ظ��û��� html �ļ����ݡ�

### 2.3 ������wsgi, Werkzeug, flask֮��Ĺ�ϵ
Flask��һ������Python������������jinja2ģ���Werkzeug WSGI�����һ��΢�Ϳ�ܣ�Werkzeugֻ�ǹ��߰��������ڽ���http���󲢶��������Ԥ����Ȼ�󴥷�Flask���;jinja2ģ������ʵ�ֶ�ģ��Ĵ�����ģ������ݽ�����Ⱦ������Ⱦ����ַ������ظ��û��������

### 2.4 Flask��ʲô
Flask��Զ����������ݿ�㣬Ҳ�����б������������������������Flask����ֻ��Werkzeug��Jinja2��֮���������ǰ��ʵ��һ�����ʵ�WSGIӦ�ã����ߴ���ģ�塣

---
# Flask Դ�����

## 1. ��������
���������һ������wsgiЭ���Ӧ�ó���demo
```
def application(environ, start_response):               #һ������wsgiЭ���Ӧ�ó���д��Ӧ�ý���2������  
    start_response('200 OK', [('Content-Type', 'text/html')])               #environΪhttp�������Ϣ��������ͷ�� start_response������Ӧ��Ϣ  
    return [b'<h1>Hello, web!</h1>']               #return��������Ӧ����  
```
*   environ��һ����������HTTP������Ϣ��`dict`����

*   start_response��һ������HTTP��Ӧ�ĺ�����

��ô�ٷ������������£�
```
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello World!"

if __name__ == "__main__":
    app.run()
```

���ȳ��򴴽�һ��Flask����

��ô����Flask�࣬����Flask��init�������ע�ͣ������������£�
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

* view_functions�����ڱ�����ͼ���������Ǻ�����,ֵ�Ǻ���������

* error_handlers�� ����ֵ������������еĴ�������ͼ�������ֵ�� key �Ǵ���������

* before_request_funcs�� �б�������������󱻷���֮ǰӦ��ִ�еĺ���

* before_first_request_funcs�� �б���������ڽ��յ���һ�������ʱ��Ӧ��ִ�еĺ�����

* after_request_funcs�� �б�����������������֮�󱻵��õĺ���

* self.url_map Map()��һ��ʵ��

**Flask���init�������ˣ�����ʼִ��app.run()����http�����server���͹�����ʱ�򣬻���һ�����������̣�**

�����������£���

app.run()-->app��Flask��ʵ������ô���ǵ��õ�Flask��run����.Դ�����£���
```
def run(self, host=None, port=None, debug=None, **options):
    from werkzeug.serving import run_simple
    try:
        run_simple(host, port, self, **options)
    finally:
        self._got_first_request = False
```
��Ҫ������run_simple���������룺��
```
def run_simple(hostname, port, application, use_reloader=False,
               use_debugger=False, use_evalex=True,
               extra_files=None, reloader_interval=1,
               reloader_type='auto', threaded=False,
               processes=1, request_handler=None, static_files=None,
               passthrough_errors=False, ssl_context=None):
```
ע��������ôһ�У���

:param application: the WSGI application to execute

������ `werkzeug.serving:WSGIRequestHandler` �� `run_wsgi` ���ҵ�execute��
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
���Կ��� `application_iter = app(environ, start_response)` ���ǵ��� `app` ʵ������ô������Ҫ������ `__call__` ������������Flask����� `__call__` ������Դ�����£���
```
class Flask(_PackageBoundObject):
    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)
```
���Կ���return����wsgi_app������

���Ե�http������������յ��õ���wsgi_app������

**�������£���**

```
app.run()
    run_simple(host, port, self, **options)
        __call__(self, environ, start_response)
            wsgi_app(self, environ, start_response)
```
�����������wsgi_app����

**wsgi_app��flask����**
```
def wsgi_app(self, environ, start_response):
    # �������������ģ�������ѹջ
    ctx = self.request_context(environ)
    ctx.push()
    error = None

    try:
        try:
            # ��ȷ��������·������ͨ��·���ҵ���Ӧ�Ĵ�����
            response = self.full_dispatch_request()
        except Exception as e:
            # ������Ĭ���� InternalServerError �����������ͻ��˻ῴ�������� 500 �쳣
            error = e
            response = self.handle_exception(e)
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        # ���ܴ����Ƿ����쳣������Ҫ��ջ�е����� pop ����
        ctx.auto_pop(error)
```

wsgi_app�ĺ��ķ���`full_dsipatch_request` �Ĵ������£�
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
full_dispatch_request�����������ĵ������� dispatch_request.

---

dispatch_request��**�ٷ�ע��**��Does the request dispatching.  Matches the URL and returns the return value of the view or error handler��

��⣺Ҫ���ľ����ҵ����ǵĴ������������ص��õĽ�������Ҳ����**·�ɵĹ���**

---

Ȼ��finalize_request������rv���л�ת���� Response����

preprocess_request��finalize_request�����˺ܶ�hooks��

* before_first_request  ��һ��������֮ǰִ�еĺ���

* before_request  ÿ��������֮ǰִ�еĺ���

* after_request  ÿ��������������֮��ִ�еĺ���

* teardown_request  ���������Ƿ��쳣��Ҫִ�еĺ���

**���ۣ�**

**��http�����server���͹�����ʱ�򣬻����__call__���ܣ�����ʵ���ǵ�����wsgi_app���ܲ�����environ��start_response**

---

**��һ�ڽ��������������̣�֪���˳�������ô���еġ���ô����������ν�·�ɺ���ͼ����������ϵ���أ�**

**��ô�Ϳ�ʼ�ڶ��ڷ���·�ɡ�**

---


## 2. ·�ɣ�
### ѧϰ�ýڵ�Ŀ�������rule�� endpoint�� view_func����ô���ཨ����ϵ�ģ�

���������ע����url��Ӧ�Ĵ���������

�ٷ�ע�ͽ������������ǵȼ۵�
```
@app.route('/')
def hello():
    return "hello, world!"
�ȼ���
def hello():
    return "hello, world!"

app.add_url_rule('/', 'hello', hello)
```


��ô����add_url_rule����������ôһ�д��룺��

```
self.view_functions[endpoint] = view_func
```

**���Կ���view_functions��һ���ֵ䣬����endpoint��ֵ�Ǻ����������������൱��endpoint��view_func����������**

**�ʣ�����endpoint�� view_func������ϵ�����ˣ���ôrule��endpoint����ô�����أ�**

**��ͨ��MapAdpter��match��������path_info��endpoint������**

**����������˸���һ����վ��·������view_func�Ĺ���**

** ������ϸ��������**

### 2.1 add_url_rule����
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

add_url_rule�ٷ�ע�ͣ����� URL ���� ����ԭ����: "route" װ������ȫһ���� ����ṩ�� view_func, ������endpoint��ע��

������rule������url����

������endpoint��Flask ����ٶ�view function �����Ƶ���endpoint

������view_func�� Ҫ���õĺ�������

������options�� Ҫת�����������ѡ��: '~werkzeug.routing.Rule' ���� ��Werkzeug�ĸ����Ǵ�����ѡ� �����ǽ��˹�������Ϊ ("GET"��"POST" ) �ȵķ����б�

���û�д��� view_func, ����Ҫ��endpoint ���ӵ�view_func, ���£�
```
app.view_functions['index'] = index
```

���Կ���view_functions ��һ���ֵ䣬ע����������ͼ�������ֵ䡣 ���Ǻ�����, ֵ�Ǻ���������

ע��������£���

```
self.view_functions[endpoint] = view_func
```

**ǿ����add_url_rule���ݵĵڶ�������endpoint �Ǻ����������ݵĵ���������view_func�Ǻ�������**

add_url_rule�������Ĺ��ܾ��Ǹ��� `self.url_map` �� `self.view_functions` ������������

`url_map` �� `werkzeug.routeing:Map` ��Ķ���            self.url_map = Map()

`rule` �� `werkzeug.routing:Rule` ��Ķ���               url_rule_class = Rule

---

**��ô���������� Class Map**

---

### 2.2 Class Map
**���Ƕ�Map�����bind��matchʹ�õ�һ��demo����**
```
m = Map([
    Rule('/', endpoint='index'),
    Rule('/downloads/', endpoint='downloads/index'),
    Rule('/downloads/<int:id>', endpoint='downloads/show')
])

urls = m.bind("example.com", "/")
ret1 = urls.match("/", "GET")
print(ret1)                             # ('index', {})
ret2 = urls.match("/downloads/42")      # Ĭ��get
print(ret2)                             # ('downloads/show', {'id': 42})
ret3 = urls.match("/downloads/")        
print(ret3)                             # ('downloads/index', {})
ret4 = urls.match("/downloads")         # ����
print(ret4)                             # RequestRedirect: 301 Moved Permanently: None
ret5 = urls.match("/missing")           # ����
print(ret5)                             # 404 Not Found: The requested URL was not found on the server.
```

demo����������޷���⣬��ô�Ұѹٷ��Ľ��ͽ���һ�飺��

�ٷ�ע�ͣ�
```
the map class stores all the URL rules and some configuration parameters.  Some of the configuration values are only stored on the `Map` instance since those affect all rules, others are just defaults and can be overridden for each rule.  Note that you have to specify all arguments besides the `rules` as keyword arguments!
```

��⣺map ��洢���� URL �����һЩ���ò����� ĳЩ����ֵ���洢�� "ӳ��" ʵ����, ��Ϊ��Щ����Ӱ�����й���, ����һ������ֻ��ȱʡֵ, ���ҿ���Ϊÿ��������д�� ��ע��, ���˽� "����" ��Ϊ�ؼ��ֲ�����, ������ָ�����в���!

���� rules�� Ruleʵ����ɵ�����

**���� def bind**

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
�ٷ�ע�ͣ�
```
Return a new :class:`MapAdapter` with the details specified to the call.  Note that `script_name` will default to ``'/'`` if not further specified or `None`.  The `server_name` at least is a requirement because the HTTP RFC requires absolute URLs for redirects and so all redirect exceptions raised by Werkzeug will contain the full canonical URL.
```

��⣺����һ���µ�: ��: "MapAdapter"��server_name���뱻����,������վ��������script_name��Ĭ��Ϊ "'/", ���û�н�һ��ָ���� "��"����� path_info û�д��ݸ�: "match", ��������Ĭ��·����Ϣ��bind��

### 2.3. Class MapAdapter

**Mapʵ��������bind֮�󣬷�����MapAdapter�࣬���ҵ�����MapAdapter��match����**

```
class MapAdapter(object):
    def match(self, path_info=None, method=None, return_rule=False, query_args=None):
```


* ���ݵ�ֵ��

�ٷ�ע�ͣ�The usage is simple: you just pass the match method the current path info as well as the method (which defaults to GET).  The following things can then happen:

��⣺����ƥ�䷽����path_info �� method��Ĭ��GET��
```
you receive a `NotFound` exception that indicates that no URL is matching.  A `NotFound` exception is also a WSGI application you can call to get a default page not found page (happens to be the same object as werkzeug.exceptions.NotFound
����յ�һ�� "NotFound" �쳣, ָʾû��ƥ�䵽 URL�� "NotFound" �쳣Ҳ��һ�� WSGI Ӧ�ó���, �����Ե���������ȡĬ��ҳ "NotFound" ҳ

you receive a `MethodNotAllowed` exception that indicates that there is a match for this URL but not for the current request method. This is useful for RESTful applications.
����յ�һ�� "MethodNotAllowed" �쳣, ָʾ�� URL ����ƥ����, ����֧�ֵ�ǰ���󷽷�������� RESTful Ӧ�ó���ǳ����á�

you receive a `RequestRedirect` exception with a `new_url` attribute.  This exception is used to notify you about a request Werkzeug requests from your WSGI application.  This is for example the case if you request ``/foo`` although the correct URL is ``/foo/`` You can use the `RequestRedirect` instance as response-like object similar to all other subclasses of `HTTPException`.
�����յ�һ�� "new_url" ���Ե� "RequestRedirect" �쳣�� ������� "/foo", ����ȷ�� URL �� "/foo/" ����ʹ�� "RequestRedirect" ʵ����Ϊ��Ӧ���ƵĶ���������������������� "HTTPException"��
```

* ����ֵ:

**�������ƥ������Ի��һ��Ԫ��(endpoint, arguments)�� ��� `return_rule`����True, ���������£���õ�Ԫ����(rule, arguments)**

**���û�д���path info, ��ʹ��Ĭ��·����Ϣ (���δ��ʽ����, ��Ĭ��Ϊ�� URL)��**

path_info:����ƥ��·����Ϣ�� ��д�󶨵�·����Ϣ��

method:����ƥ��HTTP ������ ��дָ���ķ�����

return_rule:����ƥ��Ĺ���, ������ֻ��endpoint (Ĭ��Ϊ "False")��

query_args:�����Զ��ض���Ϊ�ַ������ֵ�Ŀ�ѡ��ѯ������ ��ǰ�޷�ʹ�ò�ѯ�������� URL ƥ�䡣

**�ص��Ƿ��� def match**
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
**match����Ҫ������Map ������ Rule �б�match ��ʱ������ε������е� rule.match ���������ƥ����ҵ���**

---

**ͨ��Map��ķ����������˽���endpoint�����view func������ϵ������ȥ����Rule�࣬�˽�url��endpoint ����ϵ��**

---

### 2.4. Class Rule
**��Ҫ���ܾ��� url �� endpoint ��ת����ͨ�� url �ҵ������ url �� endpoint**

```
def __init__(self, string, defaults=None, subdomain=None, methods=None,
                 build_only=False, endpoint=None, strict_slashes=None,
                 redirect_to=None, alias=False, host=None):

Rule('/', endpoint='index'),
Rule('/downloads/', endpoint='downloads/index'),
Rule('/downloads/<int:id>', endpoint='downloads/show')
```

* Class Rule�ٷ�ע�ͣ�

A Rule represents one URL pattern.  There are some options for `Rule` that change the way it behaves and are passed to the `Rule` constructor. Note that besides the rule-string all arguments *must* be keyword arguments in order to not break the application on Werkzeug upgrades.

��⣺Rule��ʾһ�� URL ģʽ�� ��һЩ "����" ѡ����Ըı�������Ϊ��ʽ�����ݸ� "����" ���캯������ע��, ���˹����ַ���֮��, ���в��� * �������ǹؼ��ֲ���, �Ա���Werkzeug����ʱ���ж�Ӧ�ó���

---

* ����String�ٷ�ע�ͣ�

Rule strings basically are just normal URL paths with placeholders in the format ``<converter(arguments):name>`` where the converter and the arguments are optional.  If no converter is defined the `default` converter is used which means `string` in the normal configuration.URL rules that end with a slash are branch URLs, others are leaves. If you have `strict_slashes` enabled (which is the default), all branch URLs that are matched without a trailing slash will trigger a redirect to the same URL with the missing slash appended.

��⣺�����ַ���������ֻ��ͨ����** URL ·��**, ��ռλ���ĸ�ʽΪ "<converter(arguments):name>", ����ת�����Ͳ����ǿ�ѡ�ġ� ���δ����ת����, ��ʹ�� "Ĭ��" ת����, �����������б�ʾ "string".</converter(arguments):name>��б�߽�β�� URL �����Ƿ�֧ url, ��һЩ����Ҷ�����������** "strict_slashes" (Ĭ��ֵ)**, ����û��β��б�ߵ������ƥ������з�֧ url ���������ض�������׷�ӵ�ȱʧб����ͬ�� URL��

---

* ����endpoint�ٷ�ע�ͣ�

The endpoint for this rule. This can be anything. A reference to a function, a string, a number etc.  The preferred way is using a string because the endpoint is used for URL generation.

��⣺�����endpoint ���������κΡ��Ժ������ַ��������ֵȵ����á� ��ѡ�ķ�����**ʹ���ַ���**, ��Ϊendpoint �������� URL��

---

* ����defaults�Ĺٷ�ע�ͣ�

An optional dict with defaults for other rules with the same endpoint. This is a bit tricky but useful if you want to have unique URLs

��⣺������ͬendpoint�����������Ĭ�Ͽ�ѡ�ֵ���һ���е㼬��, �����õ�, ���������Ψһ����ַ

---

```
url_map = Map([
                Rule('/all/', defaults={'page': 1}, endpoint='all_entries'),
                Rule('/all/page/<int:page>', endpoint='all_entries')
            ])
```

����û����ڷ��� "http://example.com/all/page/1", �������ض��� "http://example.com/all/"�� ��� "Map" ʵ���ϵ� "redirect_defaults" ������, ��ֻ��Ӱ�� URL ���ɡ�

* ����subdomain

�˹����subdomain�����ַ��������δָ������, ���ƥ��ӳ��� "default_subdomain"�� ���mapû�а�subdomain�� �˹��ܽ������á�

�����ϣ���ڲ�ͬ��������ӵ���û������ļ�, ���ҽ���������ת��������Ӧ�ó���, ���ǳ����á�

```
url_map = Map([
                Rule('/', subdomain='<username>', endpoint='user/homepage'),
                Rule('/stats', subdomain='<username>', endpoint='user/stats')
            ])
```

* ����methods�Ĺٷ�ע�ͣ�

A sequence of http methods this rule applies to.  If not specified, all methods are allowed. For example this can be useful if you want different endpoints for `POST` and `GET`.  If methods are defined and the path matches but the method matched against is not in this list or in the list of another rule for that path the error raised is of the type `MethodNotAllowed` rather than `NotFound`.  If `GET` is present in the list of methods and `HEAD` is not, `HEAD` is added automatically

��⣺�˹������õ� http �������С� ���δָ��, ������ʹ�����з���������, **���ϣ�� "POST" �� "GET" ��endpoints��ͬ**, ����ܺ����á� **���������methods ����·��ƥ��, ��ƥ���methods ���ڴ��б��л��·������һ�������б���, �������Ĵ��������� "MethodNotAllowed", ������ "NotFound"��** ��� "GET" �Ǵ��ڵķ����б�� ' HEAD ' ����, ' HEAD ' ���Զ����

---

**�ڶ��������Ѿ�֪����·�ɺ���ͼ����ô������ϵ��������ͼ����ִ����ҪһЩ���������磺����� url�������������ȡ���Щ�ⲿ�����ͽ��������ġ���ô�����ھͿ�ʼ���������ġ�**

---

## 3. �����ģ�application context �� request context��

**�ʣ�ʲô�������ģ�**

**��**��ÿһ�γ����кܶ��ⲿ������ֻ����Add���ּ򵥵ĺ�������û���ⲿ�����ġ�һ�����һ�γ��������ⲿ��������γ���Ͳ����������ܶ������С���Ϊ��ʹ�������У���Ҫ�����е��ⲿ����һ��һ��дһЩֵ��ȥ����Щֵ�ļ��Ͼͽ������ġ�

��Դ��֪�� https://www.zhihu.com/question/26387327/answer/32611575

**�ʣ���������������Ρ��ں�ʱ���������أ�**

**��**�ڷ���������Ӧ�õ�ʱ��Flask �� `wsgi_app` �����´��룬���д�����Ǵ��������������Ĳ�ѹջ��
```
def wsgi_app(self, environ, start_response): 
    ctx = self.request_context(environ) 
    ctx.push()
```

**flask �������������ģ�����������`application context` �� ����������`request context`��**

`application context`���������� `current_app` �� `g`;

`request context` ���������� `request` �� `session`��

�����ĵ���������� `globals.py` �ļ���
```
# context locals
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
current_app = LocalProxy(_find_app)
request = LocalProxy(partial(_lookup_req_object, 'request'))
session = LocalProxy(partial(_lookup_req_object, 'session'))
g = LocalProxy(partial(_lookup_app_object, 'g'))
```


* current_app ����ǰ�������˵ĳ���ʵ��������helloworld�����е�app

* g ��������������ʱ����ʱ�洢��ÿ����������

* request ��������󣬰����˿ͻ���HTTP���������

* session ���û��Ự�����ֵ��ʽ�������洢֤����ݵ���Ϣ

���������Ѿ��˽��������ĵĶ��壬��ͼ������Ҫ��ʱ�򣬿���ʹ�� **from flask import request, current_app** ��ȡ��

**������ʹ�õ�demo����**

**����������**

```
from flask import request
@app.route('/useragent/')
def userAgent():
    user_agent = request.headers.get('User-Agent')
    return '<p>Your browser is %s</p>' % user_agent
```

**����������**
```
from flask import current_app
# print('current_app.name:',current_app.name)
app_ctx = app.app_context()
app_ctx.push()
current_app.name
app_ctx.pop()
```

---

**ͨ������������֪������ͼ��ε����ⲿ������ʵ�ֹ��ܡ����������Ѿ�֪���˳���������У�·�ɺ���ͼ��ƥ�䣬��ͼ��ε��������ġ���ô�����������������Ӧ������ôʵ�ֵġ�**

---

## 4. ����
��flask��������һ�����󣬽�http������������ݶ������һ���ֵ���

���� flask ���������ǳ��򵥣�ֻ��Ҫ `from flask import request`��
```
from flask import request

with app.request_context(environ):
    assert request.method == 'POST'
```

class Request��Դ�����£���
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

**`@property` װ�η��ܹ�����ķ���������ԣ���Щ���Ժ�Flask���߼����**

��Դ���֪��requests�̳��� werkzeug.wrappers:Request

����werkzeug.wrappers:Request ����Դ��
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
����ʹ���˻����̡�

��������BaseRequest��

```
class BaseRequest(object):
    def __init__(self, environ, populate_request=True, shallow=False):
        self.environ = environ
        if populate_request and not shallow:
            self.environ['werkzeug.request'] = self
        self.shallow = shallow
```
���Ǽ򵥵Ľ�environ��������������

���ſ�AcceptMixin��Щ�ࣺ
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
������ `@cached_property` װ��

��ô������ `@cached_property`Դ��
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
���װ����ʵ���� `__set__` �� `__get__` ������������װ�ε����ԣ��ͻ���� `__get__` ����������������� `obj.__dict__` ��Ѱ���Ƿ��Ѿ����ڶ�Ӧ��ֵ��������ڣ���ֱ�ӷ��أ���������ڣ����õײ�ĺ��� `self.func`�����ѵõ���ֵ�����������ٷ��ء���Ҳ������ʵ�ֻ����ԭ����Ϊ����Ѻ�����ֵ��Ϊ���Ա��浽�����С�

---

**���Ľ������˽������������ʵ�ֵģ�����ȥ�˽���Ӧ**

---

## 5. response ��Ӧ
HTTP ��Ӧ��Ϊ�������֣� ״̬����Ӧͷ��body

��ͼ�����ķ���ֵ��Ϊ��Ӧ

**��Ӧ�����ַ�����**

* ��ͼ����ֱ�ӷ���һ��Ԫ�� (response, status, headers)

��һ���Ƿ��ص����ݣ��ڶ�����״̬�룬����������Ӧͷ

**demo����**
```
@app.route('/')
def hello_world():
    return 'Hello, World!', 201, {'X-Foo': 'bar'}
```

* ��ͼ��������һ��make_resonse()������������Ӧ����

make_response()��������һ����������������ͼ�����ķ���ֵһ������Ȼ�󷵻�һ�� Response ��
��

**demo����**
```
from flask import make_response
@app.route('/response/')
def response():
    resp = make_response('<h1>Bad Request</h1>',400)
    return resp
```

---

**ǰ����Ѿ���flask����������ȫ��ѧϰ��һ�飬�����ٷ�����session����ô�����ġ�**

---

## 6. Session
**�ʣ�ʲô��session**

**��**������HTTPЭ������״̬��Э�飬���Է������Ҫ��¼�û���״̬ʱ������Ҫ��ĳ�ֻ�����ʶ������û���������ƾ���Session.

**session��cookies������**
1��session �ڷ������ˣ�cookie �ڿͻ��ˣ��������
2��session Ĭ�ϱ������ڷ�������һ���ļ�������ڴ棩
3��session ���������� session id���� session id �Ǵ��� cookie �еģ�Ҳ����˵���������������� cookie ��ͬʱ session Ҳ��ʧЧ�����ǿ���ͨ��������ʽʵ�֣������� url �д��� session_id��

**�ʣ���ô�����flask��ʹ��session�أ�**

**��**��ֻҪ`from flask import session` �Ϳ���ʹ�� session 

�����������ʱ��flask����������cookies����session��Ȼ������ͼ��Ϳ���ʹ�����ɵ�session����session���и��ģ���󷵻ظ�response������sessionд��cookies�

���Ĵ�����save_session�����
```
def save_session(self, app, session, response):
        domain = self.get_cookie_domain(app)
        path = self.get_cookie_path(app)

        # ��� session ����˿��ֵ䣬flask ��ֱ��ɾ����Ӧ�� cookie
        if not session:
            if session.modified:
                response.delete_cookie(app.session_cookie_name,
                                       domain=domain, path=path)
            return

        # �Ƿ���Ҫ���� cookie����� session �����˱仯����һ��Ҫ���� cookie�������û����� `SESSION_REFRESH_EACH_REQUEST` ���������Ƿ�Ҫ���� cookie
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
�� `app` �� `session` �����л�ȡ������Ҫ����Ϣ��Ȼ����� `response.set_cookie` ���� `cookie`���������ڿͻ��˵�cookies�ﱣ����session���Ժ���ʵ�ʱ��Я����cookies���session���ܹ�֤���Լ�����ݡ�

��ƪFlaskԴ�������漰������Ҫ���ݺͺ���Դ�������������ͼ�
![](https://raw.githubusercontent.com/SkewwG/SourceCodeAnalysis/master/img/Flask%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E6%B5%81%E7%A8%8B%E5%9B%BE.png)


## �ο�����

https://segmentfault.com/a/1190000010954763

http://mingxinglai.com/cn/2016/08/flask-source-code/

https://juejin.im/post/5a32513ff265da430f321f3d

http://cizixs.com/2017/01/10/flask-insight-introduction