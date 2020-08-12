---
title: 面向前端的 Node.js（一） - 前端需要了解的 Node.js 基础与实践
date: 2020-08-12
categories: ["后端", "前端"]
tags: ["Node.js"]
draft: true
---

## 系列前言

在学习极客时间上的 Node.js 课程之后我决定自己整理一下学到的知识。

在 Node.js 推出并越来越流行的今天，全栈工程师这种名词也开始流行起来，自然会有很多同学为了拓展自己的知识面和更好地找工作去学 Node.js。其实大多数这样做的人并不会期望着自己学完了之后变成一个“后端工程师”的，毕竟很多后端的知识和实践经验并不是前端开发者们一朝一夕就能拥有的。那么前端同学怎样入门 Node.js 最好呢？那就要从 Node.js 的运用来说起。因为大型项目的后台大多数是由 Java 等语言编写的，这有历史遗留问题，也有技术选型问题，这里不赘述了，总体来说用传统后端的项目还是远超 Node.js 的。而 Node.js 对于前端来说最常用的场景其实就是接驳前后端的一个中间层，也就是 Backend for Frontend。通过它我们能获得什么呢？一是可以自由拼装后端数据，二是可以做 SSR 来优化首屏加载速度和 SEO。

另外，这篇文章放在“前端”的栏目里主要是因为它是面向前端的，当然它对后端人员也有帮助，能够使后端工程师更好地理解这种构架，这样能使后端工程师与前端工程师的协作更加顺利。

## 为何要多做一层？

首先我们要清楚一点，前后端通信一般是使用 HTTP 协议，而后端服务间的数据传输往往会使用 RPC。而 RPC 往往会使用到二进制协议而不是 HTTP 中常用的文本协议。关于 RPC vs HTTP 和二进制协议 vs 文本协议，本文不会赘述，可以自行查询资料。总体来说 RPC + 二进制协议的效率会更高，因为我们的 BFF 层是可以和众多后端服务一起部署的，完全可以把它视作众多后端服务之一而融入进后端服务的 DevOps 中，可以算作微服务中的一个服务。至于这个构架怎样做最好，这大概率是构架师和后端的事情，前端同学基本不会参与这方面的决策。而前端同学最需要知道的就是，如果给了我们一个这样的任务，我们该怎么做。

另外关于这个 BFF 层的必要性，我个人从构架的角度认为，如果要做它，必定要有它的优越性才行，包括但不限于性能、DevOps、开发成本。例如后端如果已经提供了 HTTP 接口，那么除非是做 SSR 优化，BFF 层并没有存在的必要。至于前端常讲的自由组装数据，我觉得在非极限的情况下都是没有必要的。如果确定有某种业务场景无法满足性能需求，那么应当前后端与构架师一同商讨改进方案，不管是 BFF、传输协议还是接口数据格式等等，而不是加一个 Node.js 服务用服务器的运算能力去处理这么简单的事情。

## Node.js 基础

关于 Node.js 的基础，其实对于前端来说并不需要讲太多，我们在浏览器端应用开发中所用到的 JavaScript 语法、特性等等都是一样的，甚至编译器都是用的和 Chrome 一样的 V8，能在 Chrome 中 debug。那么对于前端来说，学习 Node.js 主要掌握差异化的部分和 API 即可。

首先我们要知道 Node.js 是什么。官网原话：

> Node.js® is a JavaScript runtime built on Chrome's V8 JavaScript engine.

> As an asynchronous event-driven JavaScript runtime, Node.js is designed to build scalable network applications.

那么这里有两个关键点，一是“基于 V8 的”，二是“异步的、事件驱动的”。

那么我们可以先来看下 Node.js 的构架，借用网上的一张图：

![](/images/bff/nodejs-architecture.jpg)

它包括了 Node.js 应用层、V8 引擎、Node.js Binding 层、以及异步 I/O 相关模块。

而作为对比， JavaScript 在浏览器环境下的构架大概是（以 Chrome 为例）：

![](/images/bff/browser-js-runtime.png)

对比二者，我们可以发现 V8 是二者都有的部分（事实上不同浏览器使用的 JavaScript 引擎不大一样，如果是 Chrome 就和 Node.js 一样）。Event Loop 也是二者都有，但是它们的实现方式有所区别：Node.js 靠的是 libuv，而在浏览器端它是由 [HTML 规范](https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model) 所定义的，具体交由各浏览器的开发者自行实现。另外浏览器端独有的是 Web API，这点我们应该都很熟悉。而 Node.js 独有的则是 Node.js Binding，它主要负责的是 Node.js 与操作系统的交互，这是后端语言必备的能力。举例来说，有可以读取系统信息的 API，可以用于性能监测；有读写文件的 API，这个是非常常用的。

那么这样分析我们大致可以了解到，在 Node.js 中 JavaScript 的执行机制其实和在 Chrome 浏览器中比较类似，然后在 Node.js 中我们不需要也不能使用 Web API，比如 window 全局对象、DOM API 等。

## RPC 原理与实现

## 二进制协议的实现 - Buffer
