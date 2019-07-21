原文：《[Let’s Build A Web Server. Part 2.](https://ruslanspivak.com/lsbaws-part2/)》
***
还记得，在 [Part 1](https://blog.csdn.net/Run_Bomb/article/details/96710139) 中，我问了一个问题：“如何在新创建的 Web 服务器下运行 Django 应用程序，Flask 应用程序和 Pyramid 应用程序，而无需对服务器进行单一更改以适应所有这些不同的Web框架？”， 继续阅读，找出答案。

过去，你选择的 Python Web 框架会限制你对可用 Web 服务器的选择，反之亦然。 如果框架和服务器设计为一起工作，那么没有任何问题：
![lsbaws_part2_before_wsgi](https://img-blog.csdnimg.cn/20190721153452334.png)

但是当你尝试将服务器和并非一起工作的框架组合在一起时，你可能会遇到以下问题（也许你曾经遇到过）：
![lsbaws_part2_after_wsgi](https://img-blog.csdnimg.cn/20190721153928628.png)

基本上你必须使用一起工作的服务器和框架，而不是你可能想要使用的那个。

那么，你如何确保可以运行具有多个 Web 框架的 Web 服务器，而无需对 Web 服务器或 Web 框架进行代码更改？而这个问题的答案就在于 Python Web 服务器网关接口（简称WSGI，发音为“wizgy”）。
![lsbaws_part2_wsgi_idea](https://img-blog.csdnimg.cn/20190721154420414.png)

WSGI 允许开发人员将 Web 框架的选择与 Web 服务器的选择分开。现在，你可以放心地混合和匹配 Web 服务器和 Web 框架，并选择适合你需求的配对。例如，你可以使用 Gunicorn 或 Nginx / uWSGI 或 Waitress 来运行 Django，Flask 或 Pyramid。 真正的混合和匹配，得益于服务器和框架中的 WSGI 支持：
![lsbaws_part2_wsgi_interop](https://img-blog.csdnimg.cn/20190721155538694.png)

因此，WSGI 是我在 [Part 1](https://blog.csdn.net/Run_Bomb/article/details/96710139) 末尾向你提出并在本文开头重复的问题的答案。 你的 Web 服务器必须实现 WSGI 接口的服务器部分，并且所有现代 Python Web 框架都已实现了 WSGI 接口的框架端，这允许你将这些框架与 Web 服务器一起使用，而无需修改服务器的代码去适应特定的 Web 框架。

现在你知道 Web 服务器和 Web 框架的 WSGI 支持允许你选择适合你的配对，它同时对服务器和框架开发人员也是有益的，因为他们可以专注于他们擅长的专业领域而不是彼此手忙脚乱。其他语言也有类似的接口：例如，Java 有 Servlet API，Ruby 有 Rack。

现在自我感觉良好，但我打赌你会说：“Show me the code!”。好的，看看这个非常简约的 WSGI 服务器实现：
```python
# 使用 Python 3.7+ 测试通过 (Mac OS X)
import io
import socket
import sys

class WSGIServer(object):

    address_family = socket.AF_INET
    socket_type = socket.SOCK_STREAM
    request_queue_size = 1

    def __init__(self, server_address):
        # 创建一个监听套接字
        self.listen_socket = listen_socket = socket.socket(
            self.address_family,
            self.socket_type
        )
        # 允许重用同一个地址
        listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        # 绑定网址
        listen_socket.bind(server_address)
        # 激活（开始监听）
        listen_socket.listen(self.request_queue_size)
        # 获取服务器主机和端口
        host, port = self.listen_socket.getsockname()[:2]
        self.server_name = socket.getfqdn(host)
        self.server_port = port
        # 用于存放 Web 框架或 Web 应用返回的请求头
        self.headers_set = []

    def set_app(self, application):
        self.application = application

    def serve_forever(self):
        listen_socket = self.listen_socket
        while True:
            # 新的客户端连接
            self.client_connection, client_address = listen_socket.accept()
            # 处理完一个请求后关闭客户端连接. 接着
            # 循环等待其他客户端的连接
            self.handle_one_request()

    def handle_one_request(self):
        request_data = self.client_connection.recv(1024)
        self.request_data = request_data = request_data.decode('utf-8')
        # 打印格式化后的请求数据
        print(''.join(
            f'< {line}\n' for line in request_data.splitlines()
        ))

        self.parse_request(request_data)

        # 使用请求数据构造环境字典
        env = self.get_environ()

        # 是时候调用我们的 application 方法并获得 HTTP 的响应主体的结果
        result = self.application(env, self.start_response)

        # 构建一个响应并返回到客户端
        self.finish_response(result)

    def parse_request(self, text):
        request_line = text.splitlines()[0]
        request_line = request_line.rstrip('\r\n')
        # 将请求行分解为组件
        (self.request_method,  # GET
         self.path,            # /hello
         self.request_version  # HTTP/1.1
         ) = request_line.split()

    def get_environ(self):
        env = {}
        # 以下代码段没有遵循 PEP8 约定，但它的格式与演示目的相同，以强调所需的变量及其值
        #
        # 需要用到的 WSGI 参数
        env['wsgi.version']      = (1, 0)
        env['wsgi.url_scheme']   = 'http'
        env['wsgi.input']        = io.StringIO(self.request_data)
        env['wsgi.errors']       = sys.stderr
        env['wsgi.multithread']  = False
        env['wsgi.multiprocess'] = False
        env['wsgi.run_once']     = False
        # 需要用到的 CGI 参数
        env['REQUEST_METHOD']    = self.request_method    # GET
        env['PATH_INFO']         = self.path              # /hello
        env['SERVER_NAME']       = self.server_name       # localhost
        env['SERVER_PORT']       = str(self.server_port)  # 8888
        return env

    def start_response(self, status, response_headers, exc_info=None):
        # 添加必要的服务器请求头
        server_headers = [
            ('Date', 'Mon, 15 Jul 2019 5:54:48 GMT'),
            ('Server', 'WSGIServer 0.2'),
        ]
        self.headers_set = [status, response_headers + server_headers]
        # 为了遵守 WSGI 规范，start_response 必须返回可调用的 'write'。 
        # 为了简单起见，我们现在忽略那个细节。返回 self.finish_response
        # return self.finish_response

    def finish_response(self, result):
        try:
            status, response_headers = self.headers_set
            response = f'HTTP/1.1 {status}\r\n'
            for header in response_headers:
                response += '{0}: {1}\r\n'.format(*header)
            response += '\r\n'
            for data in result:
                response += data.decode('utf-8')
            # 依照 'Ctrl -v' 打印格式化后的响应数据
            print(''.join(
                f'> {line}\n' for line in response.splitlines()
            ))
            response_bytes = response.encode()
            self.client_connection.sendall(response_bytes)
        finally:
            self.client_connection.close()


SERVER_ADDRESS = (HOST, PORT) = '', 8888


def make_server(server_address, application):
    server = WSGIServer(server_address)
    server.set_app(application)
    return server


if __name__ == '__main__':
    if len(sys.argv) < 2:
        sys.exit('Provide a WSGI application object as module:callable')
    app_path = sys.argv[1]
    module, application = app_path.split(':')
    module = __import__(module)
    application = getattr(module, application)
    httpd = make_server(SERVER_ADDRESS, application)
    print(f'WSGIServer: Serving HTTP on port {PORT} ...\n')
    httpd.serve_forever()
```
它肯定比 [Part 1](https://blog.csdn.net/Run_Bomb/article/details/96710139) 中的服务器代码更大，但它也足够小（不到150行），你可以理解而不会被细节困扰。上面的服务器也能做到更多——它可以运行用你心爱的 Web 框架编写的基本 Web 应用程序，无论是 Pyramid，Flask，Django 还是其他一些 Python WSGI 框架。

不相信我？试一试，亲身体会吧。 将上述代码保存为 `webserver2.py` 或直接从 [GitHub](https://github.com/rspivak/lsbaws/blob/master/part2/webserver2.py) 下载。 如果你试图在没有任何参数的情况下运行它，它会报错并退出。
```
$ python webserver2.py
Provide a WSGI application object as module:callable
```
它迫切地的想要为你的 Web 应用程序提供服务，这样才会变得有趣。要运行服务器，你唯一需要安装的就是 Python（确切地说是Python 3.7+）。 但要运行使用 Pyramid，Flask 和 Django 编写的应用程序，你需要先安装这些框架。让我们安装所有这三个。我首选的方法是使用 venv（默认情况下在Python 3.3及更高版本中可用）。只需按照以下步骤创建并激活虚拟环境，然后安装这三个 Web 框架。
>译者注：先创建项目目录，再在目录中创建虚拟环境
```
$ mkdir part2
$ cd part2
```
>译者注：请注意，在一些操作系统中，你可能需要在上面的命令中使用 python 而不是 python3 。
```
$ python -m venv lsbaws
```
>译者注：不管你用什么方法创建虚拟环境，创建完毕之后还需要激活才能够进入这个虚拟环境。 要激活你的全新虚拟环境，需使用以下命令：（以下是 Microsoft Windows 命令提示符窗口）
```
$ lsbaws\Scripts\activate
(lsbaws) $ _
```
>译者注：激活一个虚拟环境，终端会话的环境配置就会被修改，之后你键入 `python` 的时候，实际上是调用的虚拟环境中的 Python 解释器。 此外，终端提示符也被修改成包含被激活的虚拟环境的名称的格式。这种激活是临时的和私有的，因此在关闭终端窗口时它们将不会保留，也不会影响其他的会话。 那么，当你需要同时打开多个终端窗口来调试不同的应用时，每个终端窗口都可以激活不同的虚拟环境而不会相互影响。
>成功创建和激活了虚拟环境之后，你可以安装那三个框架了，命令如下：
```

(lsbaws) $ pip install -U pip
(lsbaws) $ pip install pyramid
(lsbaws) $ pip install flask
(lsbaws) $ pip install django
```
>译者注：我在 Windows10 命令行安装 pip 时遇到文件权限限制，修改整个 anaconda3 的文件权限无果，
便直接在命令行中加入` --user` ：`pip install --user -U pip` 成功安装 pip-19。

原文作者电脑系统是 MacOS，创建虚拟环境、激活虚拟环境以及安装的过程如下：
```
$ python3 -m venv lsbaws
$ ls lsbaws
bin   include   lib   pyvenv.cfg
$ source lsbaws/bin/activate
(lsbaws) $ pip install -U pip
(lsbaws) $ pip install pyramid
(lsbaws) $ pip install flask
(lsbaws) $ pip install django
```
此时，你需要创建一个 Web 应用程序。 让我们先从 Pyramid 开始吧。 将以下代码保存为 `pyramidapp.py`，放到你保存 `webserver2.py` 的同一目录下或直接从 [GitHub](https://github.com/rspivak/lsbaws/blob/master/part2/pyramidapp.py) 下载该文件：
>译者注：这些文件最好都放在上面创建的目录中，即刚刚创建的虚拟环境 `lsbaws` 的同级目录下，否则要在命令行运行这些文件时就要带上它们的绝对路径才行。
```python
from pyramid.config import Configurator
from pyramid.response import Response


def hello_world(request):
    return Response(
        'Hello world from Pyramid!\n',
        content_type='text/plain',
    )

config = Configurator()
config.add_route('hello', '/hello')
config.add_view(hello_world, route_name='hello')
app = config.make_wsgi_app()
```
现在，你已准备好使用自己的 Web 服务器为 Pyramid 应用程序提供服务：
```
(lsbaws) $ python webserver2.py pyramidapp:app
WSGIServer: Serving HTTP on port 8888 ...
```
>译者注：可以注意到，要在激活的虚拟环境中运行上面的命令

你刚刚告诉你的服务器从 python 模块 '*pyramidapp*' 加载可调用的 '*app*' 。你的服务器现在已准备好接收请求并将它们转发到你的 Pyramid 应用程序。应用程序现在只处理一个路径：`/hello` 路由。在浏览器中键入 `http://localhost:8888/hello` 地址，按 Enter 键，然后观察结果：
![lsbaws_part2_browser_pyramid](https://img-blog.csdnimg.cn/20190721185532139.png)

你还可以使用 *'curl'* 工具程序在另一个命令窗口上测试这个服务器：
```
$ curl -v http://localhost:8888/hello
...
```
查看服务器和 `curl -v` 打印到标准输出的内容。
>译者注：这是我的运行结果：
>>![mytest-curl-pyramid](https://img-blog.csdnimg.cn/20190721190059614.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1J1bl9Cb21i,size_16,color_FFFFFF,t_70)

现在轮到 Flask。我们按照相同的步骤。
```python
from flask import Flask
from flask import Response
flask_app = Flask('flaskapp')


@flask_app.route('/hello')
def hello_world():
    return Response(
        'Hello world from Flask!\n',
        mimetype='text/plain'
    )

app = flask_app.wsgi_app
```
将上面的代码保存为 `flaskapp.py` 或从 [GitHub](https://github.com/rspivak/lsbaws/blob/master/part2/flaskapp.py)下载，然后运行服务器如下：
```
(lsbaws) $ python webserver2.py flaskapp:app
WSGIServer: Serving HTTP on port 8888 ...
```
现在在浏览器中键入 `http://localhost:8888/hello`，然后按 Enter 键：
![lsbaws_part2_browser_flask](https://img-blog.csdnimg.cn/20190721190647621.png)

再次尝试 *'curl'* 并亲自看看服务器返回 Flask 应用程序生成的消息：
```
$ curl -v http://localhost:8888/hello
...
```
>译者注：这是我的运行结果：
>>![mytest-curl-flask](https://img-blog.csdnimg.cn/20190721191218216.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1J1bl9Cb21i,size_16,color_FFFFFF,t_70)

服务器还可以处理 Django 应用程序吗？ 试试看！但是，它有点复杂，我建议克隆整个 repo 并使用其中的 `djangoapp.py`，它是 [GitHub repository](https://github.com/rspivak/lsbaws/) 的一部分。这是源代码，它基本上将Django 'helloworld' 项目（使用 Django 的 `django-admin.py startproject` 命令预先创建）添加到当前的 Python 路径，然后导入项目的 WSGI 应用程序。
```python
import sys
sys.path.insert(0, './helloworld')
from helloworld import wsgi


app = wsgi.application
```
将上面的代码保存为 `djangoapp.py` 并使用 Web 服务器运行这个 Django 应用程序：
```
(lsbaws) $ python webserver2.py djangoapp:app
WSGIServer: Serving HTTP on port 8888 ...
```
输入以下网址，然后按 Enter 键：
![lsbaws_part2_browser_django](https://img-blog.csdnimg.cn/20190721192754833.png)

就像你之前已经做过的那几次，你也可以在命令行上进行测试，并确认这次是处理你的请求的 Django 应用程序：
```
$ curl -v http://localhost:8888/hello
...
```
>译者注：服务器已经成功在 Windows10 下的命令行终端中挂起，但在 chrome 浏览器中访问 `http://localhost:8888/hello` 时出现 `DisallowedHost at /
Invalid HTTP_HOST header: 'DESKTOP-E61P8KD:8888'. You may need to add 'desktop-e61p8kd' to ALLOWED_HOSTS.` 这样的错误。暂且搁置，待日后重提。

你试过吗？你已经确认了这个服务器能和这三个框架一起工作了吗？ 如果没有，那么请试一试。 阅读很重要，但这个系列是关于重构的，这意味着你需要亲力亲为。去尝试吧，我会等着你，别担心。不是说，你必须尝试它，只是那样的话会更好，自己重新输入所有内容，并确保它按预想的那样运行。

好的，你已经体验过 WSGI 的强大功能：它允许你混合搭配 Web 服务器和 Web 框架。 WSGI 在Python Web 服务器和 Python Web 框架之间提供了一个最小的接口。它非常简单，并且很容易在服务器端和框架端实现。以下代码段显示了接口的服务器端和框架端：
```python
def run_application(application):
    """Server code."""
    # 这是应用程序/框架存储 HTTP 状态和 HTTP 响应头的位置，供服务器传输到客户端
    headers_set = []
    # 带有 WSGI / CGI 变量的环境字典
    environ = {}

    def start_response(status, response_headers, exc_info=None):
        headers_set[:] = [status, response_headers]

    # 服务器调用'application'并返回响应主体
    result = application(environ, start_response)
    # 服务器构建HTTP响应并将其传输到客户端
    ...

def app(environ, start_response):
    """A barebones WSGI app."""
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return [b'Hello world!']

run_application(app)
```
以下是它的运行方式：
1. 框架提供了一个可调用的 *“application”*（WSGI 规范没有规定应该如何实现）
2. 服务器为从 HTTP 客户端接收的每个请求调用 *“application”* 。 它传递一个包含 WSGI / CGI 变量的字典 *'environ'* 和一个可调用 *'start_response'* 作为 *'application'* 的参数。
3. 框架 / 应用程序生成 HTTP 状态和 HTTP 响应标头，并将它们传递给可调用的 *“start_response”*，以便服务器存储它们。框架 / 应用程序还返回响应主体。
4. 服务器将状态、响应头和响应主体组合成 HTTP 响应并将其发送到客户端（此步骤不是规范的一部分，但它是流程中的下一个逻辑步骤，为了清楚起见我添加进来）

以下是这个接口的直观展示：
![lsbaws_part2_wsgi_interface](https://img-blog.csdnimg.cn/20190721205141288.png)

到目前为止，你已经看过 Pyramid、Flask 和 Django Web 应用程序，并且你已经看到了实现服务器端 WSGI 规范的服务器代码。你甚至已经看到了不使用任何框架的非正式 WSGI 应用程序代码片段。

问题在于，当你使用其中一个框架编写 Web 应用程序时，你在更高级别的层次工作而不是直接使用 WSGI，但我知道你对 WSGI 接口的框架方面也很好奇，因为你在读这篇文章。因此，让我们创建一个简约的 WSGI Web 应用程序 / Web 框架，而不使用 Pyramid、Flask 或 Django，并在你的服务器上运行它：
```python
def app(environ, start_response):
    """A barebones WSGI application.

    This is a starting point for your own Web framework :)
    """
    status = '200 OK'
    response_headers = [('Content-Type', 'text/plain')]
    start_response(status, response_headers)
    return [b'Hello world from a simple WSGI application!\n']
```
再次，将上述代码保存成 `wsgiapp.py` 文件或直接从 [GitHub](https://github.com/rspivak/lsbaws/blob/master/part2/wsgiapp.py) 下载并在 Web 服务器下运行应用程序：
```
(lsbaws) $ python webserver2.py wsgiapp:app
WSGIServer: Serving HTTP on port 8888 ...
```
输入以下网址，然后按 Enter 键。 你应该看到这样的结果：
![lsbaws_part2_browser_simple_wsgi_app](https://img-blog.csdnimg.cn/20190721210736385.png)

在学习如何创建 Web 服务器的同时，你刚刚编写了自己的简约 WSGI Web 框架！不得了啊。

现在，让我们回到服务器传输给客户端的内容。 以下是使用 HTTP 客户端调用 Pyramid 应用程序时服务器生成的 HTTP 响应：
![lsbaws_part2_http_response](https://img-blog.csdnimg.cn/20190721211208398.png)

响应中有一些你在 [Part 1](https://blog.csdn.net/Run_Bomb/article/details/96710139) 中看到的熟悉部分，但它也有一些新的东西。例如，它有四个你以前没有见过的 HTTP 标头：*Content-Type，Content-Length，Date* 和 *Server*。 这些是来自 Web 服务器的响应通常应该具有的标头。但是，没有一个是严格要求的。标头的目的是传输有关 HTTP 请求 / 响应的其他信息。

现在你已经了解了有关 WSGI 接口的更多信息，以下是同样的 HTTP 响应，其中包含有关生成它的部件的更多信息：
![lsbaws_part2_http_response_explanation](https://img-blog.csdnimg.cn/20190721212122891.png)

我还没有提到过 *'environ'* 字典，但基本上它是一个 Python 字典，必须包含 WSGI 规范规定的某些 WSGI 和 CGI 参数变量。解析请求后，服务器从 HTTP 请求中获取字典的值。这就是字典的内容：
![lsbaws_part2_environ](https://img-blog.csdnimg.cn/20190721212338260.png)

Web 框架使用来自该字典的信息来决定指定的路由、请求方法等使用哪个视图，从哪里读取请求主体以及在何处写入错误（如果有的话）。

到目前为止，你已经创建了自己的 WSGI Web 服务器，并且已经使用不同的 Web 框架编写了 Web 应用程序。而且，你还在此过程中创建了你的准系统 Web 应用程序 / Web 框架。这是一段艰难的旅程。让我们回顾一下你的 WSGI Web 服务器必须做什么来提供针对 WSGI 应用程序的请求：
* 首先，服务器启动并加载由 Web 框架 / 应用程序提供的可调用 *“application”*
* 然后，服务器读取一个请求
* 接着，服务器解析这个请求
* 再者，然后，服务器使用请求数据构建 *“environ”* 字典
* 然后，服务器调用可调用的 *“application”*、*“environ”* 字典和可调用的 *“start_response”* 作为参数并返回响应主体。
* 接着，服务器使用可调用 *'application'* 对象返回的数据以及 *'start_response'* 设置的状态和响应头来构造 HTTP 响应。
* 最后，服务器将这个 HTTP 响应返回给客户端
![lsbaws_part2_server_summary](https://img-blog.csdnimg.cn/2019072121363354.png)

这就是它的全部内容。你现在拥有一个可用的 WSGI 服务器，可以为使用 WSGI 兼容的 Web 框架（如 Django，Flask，Pyramid 或你自己的 WSGI 框架）编写的基本 Web 应用程序提供服务。最好的一点就是服务器可以与多个 Web 框架一起使用，而无需对服务器代码库进行任何更改。简直漂亮。

在你继续之前，这是另一个你可以考虑一下的问题：“你如何让你的服务器一次处理多个请求？”

请继续关注，我将在 [Part 3]() 中向你展示完成它的一种方法。Cheers！

>**UPDATE: Mon, July 15, 2019**
> * Updated the server code to run under Python 3.7+
> * Added resources used in preparation for the article

**此系列的所有文章（已翻译）：**
* [Let’s Build A Web Server. Part 1.](https://blog.csdn.net/Run_Bomb/article/details/96710139)
* [Let’s Build A Web Server. Part 2.](https://blog.csdn.net/Run_Bomb/article/details/96726186)
* [Let’s Build A Web Server. Part 3.]()
