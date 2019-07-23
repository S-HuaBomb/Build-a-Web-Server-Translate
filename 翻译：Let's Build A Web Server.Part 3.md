:books: :books: :books:

原文：《[Let's Build A Web Server. Part 3.](https://ruslanspivak.com/lsbaws-part3/)》
***
> ***“We learn most when we have to invent” —Piaget***

在 [Part 2](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%202.md) 中，您创建了一个可以处理基本 HTTP GET 请求的简约 WSGI 服务器。我问你一个问题，“你怎么能让你的服务器一次处理多个请求？”，在本文中你会找到答案。因此，抓紧扶好，老司机带你飞。真的是老司机带你飞的感觉哦。准备好 Linux，Mac OS X（或任何 *nix 系统）和 Python。文章中的所有源代码都可以在 [GitHub](https://github.com/rspivak/lsbaws/tree/master/part3) 上找到。

首先让我们回想起一个非常基本的 Web 服务器是什么样的，以及服务器需要做些什么来服务客户端的请求。您在 [Part 1](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%201.md) 和 [Part 2](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%202.md) 中创建的服务器是一个迭代服务器，一次处理一个客户端请求。在完成处理当前客户端请求之前，它不能接受新的客户端连接。有些客户可能对它不满意，因为他们必须排队等候，对于繁忙的服务器，这个等待队列可能会很长。
![lsbaws_part3_it1](https://img-blog.csdnimg.cn/20190722184158155.png)



>译者注：原作者还没有更新 Part 3 的代码，所以翻译的进度暂且到此，等待原作者的更新吧。。

>**UPDATE: Sat, July 13, 2019**
> * Updated the server code to run under Python 3.7+
> * Added resources used in preparation for the article

**此系列的所有文章（已翻译）：**
* [Let’s Build A Web Server. Part 1.](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%201.md)
* [Let’s Build A Web Server. Part 2.](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%202.md)
* [Let’s Build A Web Server. Part 3.](https://github.com/S-HuaBomb/Build-a-Web-Server-Translate/blob/master/%E7%BF%BB%E8%AF%91%EF%BC%9ALet's%20Build%20A%20Web%20Server.Part%203.md)【未完待续...】
