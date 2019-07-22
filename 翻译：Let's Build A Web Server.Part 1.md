原文：《[Let’s Build A Web Server. Part 1.](https://ruslanspivak.com/lsbaws-part1/)》
***
&ensp;&ensp;&ensp;&ensp;有一天，一个女人出门散步，来到一个建筑工地，看到三个男人在工作。她问第一个男人，“你在做什么？”，他对这个问题很恼火，咆哮道，“你没看到我在砌砖吗？”；女人不满意他的回答，于是问第二个男人正在做什么。第二个人回答说：“我正在修建一堵砖墙。”，然后他便把注意力转向第一个男人，对他说，“嘿，你刚刚走过墙的尽头，你要拿掉最后那块砖。”；女人再次对回答不满意，于是问第三个人在做什么。那个男人一边仰望天空一边对她说：“我正在建造这个世界上最大的大教堂。”，当他站在那里仰望天空时，另外两个男人开始争论放错的砖块，他转向前两名男子说：“嘿伙计们，不要担心那块砖头。那是一面内墙，它将被覆盖掉，没有人会看到那块砖，继续前进另一层。”

这个故事的寓意是，当你了解整个系统并了解不同的部分如何组合在一起（砖块，墙壁，大教堂）时，你可以更快地识别和修复问题（放错的砖块）。

这与从头创建一个 Web 服务器有什么关联？

**我相信想要成为一个更优秀的开发者，你必须对每天在使用的底层软件系统有更好的了解，包括编程语言、编译器和解释器、数据库和操作系统、Web 服务器和框架等。 而且，为了更好、更深入地理解这些系统，你必须从头开始，一步一个脚印地重新构建它们。**

子曰：
>“我听到的会忘记”

![LSBAWS_confucius_hear](https://img-blog.csdnimg.cn/20190721124746797.png)
>“我看到的我能记住”

![LSBAWS_confucius_see](https://img-blog.csdnimg.cn/20190721124934194.png)
> “我做过的我就能理解”

![LSBAWS_confucius_do](https://img-blog.csdnimg.cn/20190721125045241.png)
我希望在这一点上你相信通过重新构建不同的软件系统来了解它们的工作原理是个好主意。

在这个由三部分组成的系列文章中，我将向你展示如何构建自己的基本 Web 服务器。让我们开始吧。

首先，什么是 Web 服务器？
![LSBAWS_HTTP_request_response](https://img-blog.csdnimg.cn/20190721125715752.png)
简而言之，它是一个位于物理服务器（oops，服务器上的服务器）上的网络服务器，它等待客户端发送请求。当它收到一个请求后，会生成响应并将其发送回客户端。客户端和服务器之间的通信使用 HTTP 协议进行。 客户端可以是您的浏览器或任何其他使用 HTTP 协议的软件。

Web 服务器的非常简单的实现是什么样的？ 以下是我的示例。示例是在 Python 中（在 Python3.7 +上测试过），但即使你不了解 Python（它是一种非常容易学习的语言，不妨试试！）你仍然应该能够从下面的代码和解释中理解概念：
```python
# Python3.7+
import socket

HOST, PORT = '', 8888

listen_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
listen_socket.bind((HOST, PORT))
listen_socket.listen(1)
print(f'Serving HTTP on port {PORT} ...')
while True:
    client_connection, client_address = listen_socket.accept()
    request_data = client_connection.recv(1024)
    print(request_data.decode('utf-8'))

    http_response = b"""\
HTTP/1.1 200 OK

Hello, World!
"""
    client_connection.sendall(http_response)
    client_connection.close()
```
将上面的代码保存为 `webserver1.py` 或直接从 [GitHub](https://github.com/rspivak/lsbaws/tree/master/part1) 下载并在命令行上运行，如下所示：
```
$ python webserver1.py
Serving HTTP on port 8888 …
```
现在，在 Web 浏览器的地址栏中键入以下 URL：`http://localhost:8888/hello`，按 Enter 键，然后看到魔法生效。你应该在浏览器中看到“Hello，World”，如下所示：
![browser_hello_world](https://img-blog.csdnimg.cn/20190721132606810.png)
Just do it，认真地。在你测试时我会等着你。

完成了吗？太好了。现在让我们讨论它到底是如何运作的。

首先让我们从你输入的网址开始。它被称为URL，这是它的基本结构：
![LSBAWS_URL_Web_address](https://img-blog.csdnimg.cn/20190721132907507.png)
这就是你告诉浏览器如何去查找和连接Web服务器的地址、以及要为你提取的服务器上的页面（路径）的方式。 在你的浏览器发送 HTTP 请求之前，它首先需要与 Web 服务器建立 TCP 连接。 然后它通过 TCP 连接向服务器发送 HTTP 请求，并等待服务器发回 HTTP 响应。 当您的浏览器收到响应时就会显示它，在这个示例中它显示“Hello，World！”

让我们更详细地探讨客户端和服务器在发送 HTTP 请求和响应之前如何建立 TCP 连接。为实现连接，它们都使用所谓的“套接字”。你不要直接使用浏览器，而是在命令行上使用 telnet 手动模拟浏览器。

在运行 Web 服务器的同一台计算机上，在新的命令行上启动 telnet 会话，指定连接到 localhost 的主机和 8888 端口，然后按 Enter 键：
```
$ telnet localhost 8888
正在连接 localhost …
Connected to localhost.
```
此时，你已与本地主机上运行的 Web 服务器建立 TCP 连接，并准备发送和接收 HTTP 消息。 在下图中，你可以看到服务器接受新的 TCP 连接必须经历的标准过程：
![LSBAWS_socket](https://img-blog.csdnimg.cn/20190721141111563.png)
在同一个 telnet 会话中键入 `GET /hello HTTP/1.1` 然后 Enter 键：
```
$ telnet localhost 8888
Trying 127.0.0.1 …
Connected to localhost.
GET /hello HTTP/1.1

HTTP/1.1 200 OK
Hello, World!
```
>译者注：在 Win10 的命令行中似乎无法完成，因为在你输入第一个字母的时候连接就断开了，但是依然能收到正确的响应。
>>打开本地回显提示：连接成功后终端是黑的，此时也可以输入命令，但不回显，所以可以先键入 `Ctrl + ]` 打开本地回显功能，再次回车，就能进入到回显模式输入命令。

你刚刚已经手动模拟了你的浏览器！你发送了 HTTP 请求并得到了 HTTP 响应。 这是HTTP请求的基本结构：
![LSBAWS_HTTP_request_anatomy](https://img-blog.csdnimg.cn/2019072114343332.png)
HTTP 请求包含指明 HTTP 方法的行（**GET**，因为我们要求我们的服务器返回一些东西），路径 `/hello` 指示我们想要的服务器上的“页”和协议版本。

为简单起见，我们此例中的 Web 服务器此时完全忽略了上述请求行。你也可以输入任何废话而不是“GET /hello HTTP / 1.1”，你仍然可以获得“Hello，World！”响应。

一旦你键入请求行并按下 Enter 键，客户端将请求发送到服务器，服务器将读取请求行，把它打印出来并返回相应的 HTTP 响应。

以下是服务器发送回客户端的 HTTP 响应（在本例中为 telnet）：
![LSBAWS_HTTP_response_anatomy](https://img-blog.csdnimg.cn/20190721144504166.png)
让我们剖析它看看。响应包括状态行 HTTP / 1.1 200 OK，后跟所需的空行，然后是 HTTP 响应主体。

响应状态行 HTTP / 1.1 200 OK 包含 HTTP 版本，HTTP 状态代码和 HTTP 状态代码原因短语OK。当浏览器获得响应时，它会显示响应的主体，这就是你在浏览器中看到“Hello，World！”的原因。

这就是 Web 服务器工作原理的基本模型。总结一下：Web 服务器创建一个监听套接字并通过循环接受新的连接。客户端启动 TCP 连接，并在成功建立 TCP 连接后，客户端向服务器发送 HTTP 请求，服务器响应 HTTP 响应，并显示给客户端。 要建立TCP连接，客户端和服务器都要使用套接字。

现在你有了一个非常基本的可运行的Web服务器，你可以使用浏览器或其他 HTTP 客户端进行测试。正如你所见并希望尝试的那样，你也可以通过使用 telnet 并手动输入 HTTP 请求来成为人工 HTTP 客户端。

这里有一个问题：“如何在新创建的 Web 服务器下运行 Django 应用程序，Flask 应用程序和Pyramid 应用程序，而无需对服务器进行单一更改以适应所有这些不同的 Web 框架？”

我将在本系列的 [Part 2](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%202.md) 中向您展示如何去做。敬请关注。

用于准备本文的资源（链接是代理链接）：
1. [Unix Network Programming, Volume 1: The Sockets Networking API (3rd Edition)](https://www.amazon.com/gp/product/0131411551/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0131411551&linkCode=as2&tag=russblo0b-20&linkId=2F4NYRBND566JJQL)
2. [Advanced Programming in the UNIX Environment, 3rd Edition](https://www.amazon.com/gp/product/0321637739/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0321637739&linkCode=as2&tag=russblo0b-20&linkId=3ZYAKB537G6TM22J)
3. [The Linux Programming Interface: A Linux and UNIX System Programming Handbook](https://www.amazon.com/gp/product/1593272200/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=1593272200&linkCode=as2&tag=russblo0b-20&linkId=CHFOMNYXN35I2MON)
4. [Lead with a Story](https://www.amazon.com/gp/product/0814420303/ref=as_li_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=0814420303&linkCode=as2&tag=russblo0b-20&linkId=HY2LNXTSGPPFZ2EV)

>**UPDATE: Sat, July 13, 2019**
> * Updated the server code to run under Python 3.7+
> * Added resources used in preparation for the article

**此系列的所有文章（已翻译）：**
* [Let’s Build A Web Server. Part 1.](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%201.md)
* [Let’s Build A Web Server. Part 2.](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%202.md)
* [Let’s Build A Web Server. Part 3.]()
***
>看得出来 Ruslan 对中国谚语是感兴趣的，我们可以去他博客的自我介绍（[About-Ruslan's Blog](https://ruslanspivak.com/pages/about/)）认识他：
>> “I hear and I forget. I see and I remember. I do and I understand.” - Confucius

不过，可能你发现了，这句话翻译成我们的谚语，应该是出自荀子《儒效篇》，荀子非常重视实践的作用，他认为：“不闻不若闻之，闻之不若见之，见之不若知之，知之不若行之；学至于行之而止矣。”
