# 阅读 Node.js 核心源码的快速指南

> 原文：[Rich Trott - A Quick Guide To Reading Node.js Core Source](https://medium.com/@Trott/a-quick-guide-to-reading-node-js-core-source-c968d83e4194)

> 译：sino

这是我自己学习 Node.js 核心源码的方法，这个方法对我来说挺有用的，对你来说可能不是很一样。不保证对你也一样有用。

Node.js 的源代码可以从[这里找到](https://github.com/nodejs/node)。

本文假设你是一名 JavaScript 开发者，如果你是 C++ 程序员，你的路径可能有所不同。

黑喂狗！

1. 确保你至少懂一点 Node.js。我指的是在阅读它的核心之前，你至少写过一点简单的应用。相比你已经谢过了，那么就开始吧。
1. 看 API 文档。挑一个你感兴趣的通读他，或者浏览完全部。在我写这篇文章的时候，你能在[这里找到](http://nodejs.cn/api/)最新的文档。或者你也能阅读[源码树](https://github.com/nodejs/node/tree/master/doc/api)里的 markdown 文件，或者你也能从你下载的本地源码的 'src/api' 文件夹下阅读文档。
1. 当你阅读文档发现了类型错误、内容欠缺或者其它问题时，你的机会就来了。你可以熟悉 Node.js 的贡献流程。如果你只是想要理解源码而不是参与到项目中，你也可以忽略这些。但是如果你有兴趣参与的话，请阅读[如何提交更改](https://github.com/nodejs/node/blob/master/CONTRIBUTING.md)。
1. 还记得步骤二中让你读的 API 吗？现在是时候去阅读你感兴趣的 API 相关源码了。假如说你对 'http' 模块感兴趣，那就看 'lib/http.js'。当你遇到了不懂的东西时，比如说什么是 '*process.binding*'？在文档中寻找，或者在 freenode 的 #node-dev 频道 提问吧。
1. 不论你在看什么（比如 'http'），阅读 'test/parallel' 和 'test/sequential' 中的所有的测试代码。
1. 仔细看看 'deps' 文件夹的的某些依赖。哪些可能取决于你对什么感兴趣，以及你是否喜欢阅读 C++。那些能够深入理解工作机制的可能是 'npm'（用 JavaScript 实现的包管理器），'uv'（'libuv' 的简写，是 Node.js 用于与底层操作系统进行交互的 C 库），还有 'V8'（Node.js 所使用的 JavaScript 引擎，是用 C++ 编写的）。

照这样，仍还有很多事情没有被包络。但是，希望你能有一个好的开始。或许对你来说这是一个糟糕的方法，你会尝试其他的东西，写自己的帖子让这篇文章显得过时。当然，你一定会有比本文更好的结论。