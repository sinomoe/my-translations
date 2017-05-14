# Node.js架构—单线程、事件循环

> 原文：[RAMBABU POSA - Node JS Architecture – Single Threaded Event Loop](http://www.journaldev.com/7462/node-js-architecture-single-threaded-event-loop)

> 译：sino

## Node.js 架构

在举开始一些例子之前，对 Node.js 架构有一个整体的认识是很重要的。本文中，我们将探讨 “Node 是怎么工作的？他遵循什么处理模型？Node 在单线程下是怎么处理并发的？” 等。

## 单线程事件循环（Single Threaded Event Loop）模型

正如我们已经讨论过的，Node 引用使用了 “单线程事件循环（Single Threaded Event Loop）模型” 架构处理多个并发客户机请求。

有很多像 JSP, Spring MVC, ASP.NET, HTML, Ajax, jQuery 等这样的 WEB 应用技术。然而这些技术都使用 “多线程请求相应” 的架构来处理多并发客户机。

由于 “多线程并发响应” 在 WEB 应用中的广泛使用，我们已经很熟悉这种架构了。但是为什么 Node 选择了另一种架构来开发 WEB 应用呢？这也是多线程与单线程事件循环（Single Threaded Event Loop）架构间的主要差异。

任何 WEB 开发者都能简单的上手开发 Node 应用。然而，要是不理解 Node 内部机制的话，我们不能很好的设计开发 Node 应用。所以在开始开发 Node 应用之前，我们应该先学习一下 Node 内部机制。

## Node 平台

Node 使用了 “单线程事件循环（Single Threaded Event Loop）” 架构来处理多并发客户机请求。那么他是怎么不通过多线程来处理多并发的呢？什么是事件循环呢？我们将在下面逐一讨论。

在讨论 “单线程事件循环（Single Threaded Event Loop）” 架构之前，我们介绍一下著名的 “多线程请求响应（Multi-Threaded Request-Response）” 架构。

## 传统 WEB 应用处理模型

任何不用 Node 开发的 WEB 应用，基本上都遵循着 “多线程请求响应（Multi-Threaded Request-Response）” 模型。简单来说，可以把它叫作 请求/响应 模型。

客户端请求到服务器，然后服务器处理客户机请求，准备响应并发送回客户机。

这个模型使用了 HTTP 协议。HTTP 是无状态协议，这个 请求/响应 模型也是无状态的模型。所以我们也把它叫作无状态请求响应模型。

然而，这个模型使用了多线程处理客户机并发请求。在讨论这个模型的内部机制之前，首先我们看下面的图。

### 请求/响应 模型处理步骤：

* 客户机发出请求到服务器。
* 服务器内部维护一个有限线程池为客户机请求提供服务。
* 服务器循环等待客户机的请求。
* 服务器接收到以下请求。
  * 服务器收到一个客户机请求。
  * 从线程池中挑出一个线程。
  * 为客户机请求分配这个线程。
  * 这个线程将会接管并处理客户机请求，执行 IO 阻塞操作（如果需要的话）并准备响应。
  * 这个线程发送准备好的响应给服务器。
  * 服务器反过来将这个响应发送给相应的客户机。

服务器循环等待每隔客户机的请求并执行上面的子步骤。这意味着这个模型为每个客户机请求创建一个线程。

如果有更多客户机请求需要执行阻塞 IO 操作的话，那么所有的线程都将忙于准备它们的响应。然后剩下的客户机请求会等待更长的时间。

![PIC1](http://cdn.journaldev.com/wp-content/uploads/2015/04/Request-Response-Model.png)

### 图表描述

* 有 n 个客户机对服务器发出请求。假设他们是并发的。
* 假设客户机是 Client-[1-n]
* 服务器内部维持着一个有限线程池，假设在线程池中有 m 个线程。
* 服务器一个接一个的收到这些请求。
  * 服务器选中 Client-1 Request-1，从线程池中选中 线程 T-1，并把这个请求分配给 T-1
    * 线程 T-1 读取 Client-1 Request-1 并处理
    * Client-1 Request-1 不进行阻塞 IO 操作
    * 线程 T-1 执行必要的步骤并准备 Response-1 并发送回服务器。
    * 服务器反过来将 Response-1 发给 Client-1
  * 服务器选中 Client-2 Request-2，从线程池中选中 线程 T-2，并把这个请求分配给 T-2
    * 线程 T-2 读取 Client-1 Request-2 并处理
    * Client-1 Request-2 不进行阻塞 IO 操作
    * 线程 T-2 执行必要的步骤并准备 Response-2 并发送回服务器。
    * 服务器反过来将 Response-2 发给 Client-2
  * 服务器选中 Client-n Request-n，从线程池中选中 线程 T-n，并把这个请求分配给 T-n
    * 线程 T-n 读取 Client-n Request-n 并处理
    * Client-n Request-n 不进行阻塞 IO 操作
    * 线程 T-n 执行必要的步骤并准备 Response-n 并发送回服务器。
    * 服务器反过来将 Response-n 发给 Client-n

  如果 n 比 m 大，首先服务器分配客户机请求到可用线程，在所有的线程都分配完了之后，剩下的客户机请求应该在队列中等待，直到一些线程完成了他们的请求响应工作且能在下次请求中被挑选出。

  如果这些线程都忙于 IO 阻塞任务时，请求等待的时间会更长，剩余的客户端也会等待更久。

* 一旦线程池有线程可用，服务器将选出这些线程并将它们分配给剩下的客户机请求。
* 每个线程都会使用很多资源，比如内存。所以在这些线程从忙碌转为等待之前，他们应该释放占用的资源。

### 请求/响应 无状态模型的缺点

* 难以处理大量的并发。
* 当并发数提升时，会使用更多的线程，最终吃光内存。
* 有时，客户机请求要等待可用线程来处理请求。
* 处理 IO 阻塞任务时浪费了 CPU 时间。

## Node 架构 - 单线程事件循环（Single Threaded Event Loop）

Node 平台不遵循 请求/响应 多线程无状态模型。Node 的处理模型主要基于 JavaScript 使用回调机制的事件模型。

也许你很清楚 JavaScript 事件和回调机制的工作原理。如果你不懂，请想看一下这些文章或者教程以对本文下本部分有一个粗略的了解。

正是因为 Node 使用了这一架构，他们非常容易地处理大量的并发请求。在讨论这个模型内部机制之前，首先浏览下面的图。

我尝试用下面的图解释 Node 内部的细节。

Node 处理模型的核心是 “事件循环”。如果我们理解它，我们也能轻松理解 Node 的内部机制。

### 单线程事件循环（Single Threaded Event Loop）模型处理步骤：

* 客户机发送请求到服务器。
* Node 服务器内部维持着一个有限线程池为客户机请求提供服务。
* Node 服务器接收到这些请求并将它们放到一个队列中，也就是 “事件队列”。
* Node 服务器内部有一个组件叫作 “事件循环”。取这个名字的原因是，它不断地循环接收请求并处理他们。
* 事件循环只使用一个线程。这就是 Node 处理模型的核心。
* 事件循环价差所有放进 “事件队列” 的客户机请求。如果没有，则循环等待请求到来。
* 如果有，从书剑队列中取出一个请求。
  * 开始处理请求
  * 如果不需要阻塞 IO 操作的话，处理完以后后准备响应并发送会客户机
  * 如果需要阻塞 IO 操作，比如与数据库交互。
    * 检测是否有可用内部线程。
    * 选出线程并将请求分配给该线程。
    * 该线程负责处理该请求，执行 IO 阻塞操作，并将准备的响应发送给事件循环。
    * 相反的，事件循环将响应发给相应客户机。

![pic1](http://cdn.journaldev.com/wp-content/uploads/2015/04/NodeJS-Single-Thread-Event-Model.png)

### 图示描述

* 有 n 个客户机发送请求给服务器。假设他们是并发的。
* 假设客户机是 Client-[1-n]
* 服务器内部维持着一个有限线程池，假设在线程池中有 m 个线程。
* Node 服务器收到了 Client-1, Client-2… and Client-n 的请求并将他们放到事件队列中。
* Node 事件循环将这些请求挨个取出。
  * 事件循环取出 Client-1 Request-1
    * 检查是否 Client-1 Request-1 需要进行阻塞 IO 操作或 CPU 密集型操作.
    * 如果只是简单的计算且不阻塞 IO，将不会有单独线程去处理它。
    * 事件循环处理 Client-1 Request-1 中的所有步骤并准备 Response-1
    * 事件循环将 Response-1 发送给 Client-1
  * 事件循环取出 Client-2 Request-2
    * 检查是否 Client-2 Request-2 需要进行阻塞 IO 操作或 CPU 密集型操作.
    * 如果只是简单的计算且不阻塞 IO，将不会有单独线程去处理它。
    * 事件循环处理 Client-2 Request-2 中的所有步骤并准备 Response-2
    * 事件循环将 Response-2 发送给 Client-2
  * 事件循环取出 Client-n Request-n
    * 检查是否 Client-2 Request-2 需要进行阻塞 IO 操作或 CPU 密集型操作.
    * 由于这个请求计算复杂或者阻塞 IO，事件循环不能处理这个请求。
    * 事件循环选取 线程 T-1 并分配它给 Client-n Request-n
    * 线程 T-1 读取并处理s Request-n, 执行必要的阻塞 IO 或计算任务, 最终准备 Response-n
    * 线程 T-1 将 Response-n 发给事件循环
    * 事件循环将 Response-n 发送给 Client-n

这里客户机请求是对一个或多个 JavaScript 函数的调用。JavaScript 函数也许会调用其他函数或使用它的原生回调函数。

每个客户机的请求会像下面这样：

![pic2](http://cdn.journaldev.com/wp-content/uploads/2015/04/javascript-callback-mechanism.png)

例如：

```js
function1(function2,callback1);
function2(function3,callback2);
function3(input-params);
```

### 注意

* 如果你不懂这些函数是怎么执行的。你应该是不熟悉 JavaScript 函数和回调机制。
* 我们应该了解 JavaScript 函数和回调机制。在开始开发 Node 应用之前，请浏览在线教程。

## Node 架构 - 单线程事件循环（Single Threaded Event Loop）的优点

1. 轻松处理大量并发请求。
1. 因为有事件循环，即使并发数越来越大，也没有必要创建多余的线程。
1. Node 应用使用较少的线程，所有它会占用更少的内存。