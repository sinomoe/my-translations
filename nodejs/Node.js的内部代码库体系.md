# Node.js的内部代码库体系

> 原文：[Aren Li - Architecture of Node.js’ Internal Codebase](https://arenli.com/architecture-of-node-js-internal-codebase-57cd8376b71f)

> 译：sino

首先，关于 JavaScript...

Stack Overflow 的联合创始人 Jeff Atwood 曾经在他的博客 [Coding Horror](https://blog.codinghorror.com/) 这么写过：

> any application that can be written in JavaScript, will eventually be written in JavaScript

在过去数年间 JavaScript 已经取得了如此大的影响和成就，以至于它俨然已成为了最流行的编程语言之一。确实，在 2016 的 SO 开发者调查中，JavaScript 位列[最受欢迎技术](http://stackoverflow.com/research/developer-survey-2016#technology-most-popular-technologies)以及 [Stack Overflow 上的头号技术](http://stackoverflow.com/research/developer-survey-2016#technology-top-tech-on-stack-overflow) 排行之首，在其他的一些排名中也是名列前茅。

Node.js 是一种服务端 JavaScript 运行环境，它提供了服务端功能的基础，比如二进制操作、文件系统 I/O 操作、数据库访问、网络等。它的独有特性使得它在与已有的框架竞争中脱颖而出。按理说，这些特性使得像 PayPal, Tinder, Medium, LinkedIn 以及 Netflix 这样的科技公司选择它，甚至在 Node.js 达到 1.0 版本之前。

我最近在 Stack Overflow 上回答了一个有关 Node.js 内部代码库体系的[问题](http://stackoverflow.com/questions/36766696/which-is-correct-node-js-architecture/37512766#37512766)，这促使了我写下本文。

官方文档对 Node.js 到底是什么解释的不够好，如下：

> 基于 chrome 的 v8 引擎构建的 JavaScript 运行时。Node.js 基于事件驱动，无阻塞 I/O 模型。。。

为了理解这句话以及背后的真正含义，我们分解 Node.js 组件，通过阐明某些关键词，并解释不同的片段将如何相互影响，来理解 Node.js 的强大：

![Node.js Architecture (High-Level to Low-Level)](https://cdn-images-1.medium.com/max/1000/1*i9MvBlVGHqywu4EuWxdZEQ.png)

## 组件/依赖

**[V8](https://developers.google.com/v8/)：** 谷歌开源的高性能 JavaScript 引擎，使用 C++ 编写。这和你的 chrome 浏览器使用的是相同的引擎。V8 将你的 JavaScript 代码编译成机器码并执行。V8 有多快？Check out [this SO answer](http://stackoverflow.com/a/41932/4603550).

**[libuv](https://github.com/libuv/libuv)：** 提供异步特性的 C 语言库。它维持着事件循环，线程池，文件系统 I/O，DNS 功能，网络 I/O 以及其它关键功能。

**[其它 C/C++ 组件/依赖](https://nodejs.org/en/docs/meta/topics/dependencies/)：** 例如 c-ares, crypto (OpenSSL), http-parser 和 zlib.这些依赖为服务器建立重要的功能，例如网络、压缩、加密等提供了底层交互。

**应用/模块：** 这就是所有 JavaScript 代码所在的地方：你的应用代码，Node.js 核心模块，你从 npm 安装的模块，以及你自己写的模块。这是你多数时间工作的东西。

**Bindings：** 也许你已经注意到，Node.js 是用 JavaScript 和 C/C++ 写的。有很多 C/C++ 代码/库的原因很简单：他们运行很快。然而，你的 JavaScript 代码最终是怎么与 C/C++ 代码交流的呢？他们不是三种不同的编程语言吗？是的，它们是。通常，不同语言写的代码不能彼此交流。没有 Bindings 所以不能交流。Bindings（绑定），正如其名，是不同语言间的胶水。所以它们能进行交互。在 Node.js 中，bindings 将 Node.js 核心中采用 C/C++ 写的内部库（c-ares, zlib, OpenSSL, http-parser, etc.）暴露给了 JavaScript。写 bindings 的一个动机是代码重用：如果想要的功能已经实现了，为什么还要再写一遍呢，只是因为语言不同？为什么不桥接他们呢？另一个动机是性能：像 C/C++ 这样的系统编程语言一般比其他高级语言要快得多。因此，将 CPU 密集型操作指定为 C/C++ 编写的代码可能是明智之举。

**C/C++ Addons：** Bindings 只提供了访问 Node.js 核心库的胶水代码，比如 zlib, OpenSSL, c-ares, http-parser 等。如果想在应用中包含第三方或者自己的 C/C++ 库，你写的这些胶水代码就被称之为插件。bindings 和插件是 JavaScript 代码和 Node.js 的 C/C++ 核心代码沟通的桥梁。

## 术语

**I/O：** Input/Output 的缩写。主要指由系统 I/O 子系统处理的操作。I/O-bound 操作通常会与磁盘/光驱交互，比如数据库访问和文件系统。还有一些其他的相关概念，包括 CPU-bound，memory-bound 等。判断一个操作是否属于 I/O 绑定，CPU 绑定等的好方法是，当增加指定的资源时，检查是否能获得更好的性能。比如，如果提升了 CPU 性能后，一个操作显著的提升了运行速度，那么他就是 CPU-bound。

**无阻塞/异步：** 一般，当收到一个请求后，应用会处理这个请求并停止其它操作直到请求被处理完成。这直接导致了：但大量请求同时涌入，每个请求都要等待前一个请求被处理完成。也就是说，前一个请求将会阻塞下一个请求。更糟的是，如果前一个请求需要长时间响应（比如计算前 1000 个素数或者从数据库中读取 3GB 数据），它之后的请求都会被暂停/阻塞相当长的一段时间。为了解决这个问题，可以采用多线程和/或多进程的处理方案，每个都有各自的利弊。Node.js 与这些相比有点不同。不是通过为每个请求分配一个单独线程来进行处理，而是将所有的请求放到同一个主线程（单线程）上进行处理，这几乎就是它所做的一切：处理请求——请求里的所有（I/O）操作，比如访问文件系统，数据库读写等被发送给了 worker 线程，并由 libuv 在后台维持。换句话说，请求中的 I/O 操作是异步处理的，而不是在主线程中被处理。这样的话，主线程永远就不会被阻塞了。所以你的应用代码只能使用唯一一个主线程。在 libuv 线程池中的 worker 线程对你来说是被屏蔽的。你永远也不需要直接操作（或考虑）它们。Node.js 已经替你管理好了。这种架构使得 I/O 操作尤为高效。然而，他也不是没有缺点。操作不仅仅是只包含了 I/O-bound 类型，还有 CPU-bound 类型，memory-bound  类型等。Node.js 只提供了针对 I/O 的异步函数。虽然也有办法处理 CPU 密集性操作，但是就不在本文关注的范围之内了。

**事件驱动：** 通常，在大多数现代系统中，当主应用启动之后，进程被请求初始化。然而在不同的技术间，在这之后的步骤就会有所不同，甚至差异很大。典型的实现是用程序处理请求：为每个请求创建一个线程；操作一个接一个的完成；如果操作很慢，该线程上在这之后的所有操作都被暂停；但所偶的操作都完成，返回响应。然而，在 Node.js 中，所有的操作都被当作待触发的事件，不论是主应用还是请求。

**运行时 (系统)：** Node.js 运行时是一整个代码库（上面提及的组件），所有的低级的、高级的特性，共同为 Node.js 应用的运行提供服务。

## 综合考虑

现在我们对 Node.js 组件有了一个更深层的整体认识，我们将调查它的工作流程以了解它的架构以及各组件间的相互作用。

当一个 Node.js 应用开始运行，V8 引擎将会运行你所写的代码。你应用里的对象将维持一个观察者的列表（被注册到事件上的函数）。这些观察者能感知到它们对应事件的触发。

当事件被发出，它的回调函数将会被入队到一个事件队列中。只要事件队列不为空，事件循环会将事件队列中的时间不断地出队并将它们放置于调用栈。值得注意的是只有当前一个时间被处理了（调用栈被清空）事件循环才会将下一个事件推入调用栈。

在调用栈中如果出现了 I/O 操作，将会由 libuv 接手处理。默认，libuv 维持着一个含有四个 worker 线程的线程池，尽管还能添加更多。如果一个请求与文件系统、I/O 和 DNS 相关，那么它将被分配给线程池处理。否则，对于诸如网络的其他请求，将部署特定于平台的机制来处理这些请求（[libuv 设计概览](http://docs.libuv.org/en/v1.x/design.html)）。

对于使用了线程池的 I/O 操作（比如 文件 I/O, DNS 等），worker 进程将与 Node.js 的底层库交互并执行诸如数据库事物，文件系统访问等操作。当处理结束后，libuv 会将事件再次入队回事件循环，以便主线程的工作。在 libuv 处理异步 I/O 操作的期间，主线程不等待处理的结果，而是继续向下执行。当 libuv 返回的事件被事件循环推入调用栈中后，将有机会在主线程中再次被处理。这就是 Node.js 中一个事件的生命周期。

mbq 曾经拿 Node.js 和餐馆做了一个[形象的比喻](http://stackoverflow.com/a/3491931/4603550)。我借用并修改他的例子使  Node.js 生命周期更易于理解：

将 Node.js 应用想作是星巴克。一位老练麻利的服务员接受点单（当作是唯一的主线程）。有大量的顾客同时进入店内，他们排成一队（进入事件队列）等待点单。只要顾客接受到了服务，服务员就把订单交给经理（libuv），经理将订单分配给咖啡师（worker 线程或特定平台机制）。咖啡师使用不同的原料和机器（底层 C/C++ 组件）制作顾客所需的饮品。通常有四位咖啡师专门负责（线程池）制作拿铁（文件 I/O, DNS 等）然而当正值旺季时，更多的咖啡师也能被叫来工作（这应该在一天开始之前，而不是在一天开始之后，比如午饭后）。只要服务员将订单交给经理，他不需要等待咖啡做好了再去服务下一位顾客。相反，他会叫下一位顾客（事件循环让下一个时间出队并将其压入调用栈）。你能想象，现在在调用栈上的时间就像正在柜台被服务的顾客。*当咖啡做好后，咖啡会被送到顾客队列的最后。当咖啡移动到柜台后，服务员将会叫顾客的名字来取咖啡*。（斜体的部分的描述在现实生活中听起来有点奇怪，但是你从程序的角度来考虑的话，它又是说得通的）

我们完成了 Node.js 的内部代码库和它的事件循环的典型生命周期的高阶概览。即使如此，本文也有很多问题以及细节未能提及，比如 CPU-bound 操作，Node.js 设计模式等。更多的话题将在其他文章中被讨论。